# Installing PKS on Azure
This document leverages a Ubuntu jump box/bastion host to deploy `PKS on Azure`. This is for demo/education purpose only and not intended for production use. Please use Concourse and Platform Automation for deploying in a more formal environment. 

Prerequisites for the installation

* Access to the Azure account. 
*  The following packages installed on the bastion host
	* jq, wget, ruby-full, go-devel
	```console
	sudo apt install jq wget ruby-full go-devel
	```
*  Access to Pivotal Download site - https://network.pivotal.io

## Stage 1
### Preparing the jump box/bastion
 The following packages need to be installed on the bastion host - 
 * Azure CLI (optional)
	```console
	sudo apt install azcli
	```
* Terraform 
	* Download the **latest** version of Terraform (v0.12.13 in this example) and move to /usr/local/bin
	```console
	wget https://releases.hashicorp.com/terraform/0.12.13/terraform_0.12.13_linux_386.zip
	unzip terraform*.zip
	chmod +x terraform
	sudo mv terraform /usr/local/bin/terraform
	```
* OM Cli
	* Download the **latest** version of OpsMan CLI  - om cli (v4.2.1 in this example) and move to /usr/local/bin
	```console
	wget https://github.com/pivotal-cf/om/releases/download/4.2.1/om-linux-4.2.1
	chmod +x om-linux-4.2.1
	sudo mv om-linux-4.2.1 /usr/local/bin/om
	```
* Pivnet Cli
	* Download the **latest** version of CLI to interact with Pivotal Network  - pivnet cli (v0.0.72 in this example) and move to /usr/local/bin
	```console
	wget https://github.com/pivotal-cf/pivnet-cli/releases/download/v0.0.72/pivnet-linux-amd64-0.0.72
	chmod +x pivnet-linux-amd64-0.0.72
	sudo mv pivnet-linux-amd64-0.0.72 /usr/local/bin/pivnet
	```
* UAA Cli
	* Install the latest Cloudfoundry UAA Cli - uaac 
	```console
	sudo gem install cf-uaac
	```
* Texplate
	* Install the CLI wrapper around Golang text/template - texplate cli and move to  /usr/local/bin
	```console
	mkdir -p $HOME/go/src/github.com/pivotal-cf
	cd $HOME/go/src/github.com/pivotal-cf
	git clone https://github.com/pivotal-cf/texplate.git
	cd texplate/scripts/
	./build
	cd ../out/
	sudo mv texplate_linux_amd64 /usr/local/bin/texplate
	```

## Stage 2

### Setting up the plumbing and installing OpsMan

Download the `paving-pks` git repo. Contact the author if you do not have access to the repo. 
```console
git clone https://github.com/pivotal/paving-pks.git
cd azure/examples/open-network
```
**IMPORTANT** Going forward, all jump box activity will be done within this directory. 

Within the directory, create a variables file called `terraform.tfvars` with values specific to your environment - 

```
subscription_id       = "SUBSCRIPTION_ID"
tenant_id             = "TENANT_ID"
client_id             = "CLIENT_ID"
client_secret         = "CLINET_SECRET"

env_name              = "env"
location              = "East US"
ops_manager_image_uri = "https://opsmanagereastus.blob.core.windows.net/images/ops-manager-2.7.1-build.189.vhd"
dns_suffix            = "domain.com"
dns_subdomain         = "subdomain"
```
#### Variables

-   subscription_id:  **(required)**  Azure account subscription id
-   tenant_id:  **(required)**  Azure account tenant id
-   client_id:  **(required)**  Azure automation account client id
-   client_secret:  **(required)**  Azure automation account client secret
-   env_name:  **(required)**  An arbitrary unique name for namespacing resources
-   location:  **(required)**  Azure location to stand up environment in
-   ops_manager_image_uri:  **(optional)**  URL for an OpsMan image hosted on Azure (if not provided you get no Ops Manager)
-   dns_suffix:  **(required)**  Domain to add environment subdomain to
-  dns_subdomain: **(required)**  Subdomain to the environment

#### NOTE
For the demo/POC purpose, we will not be deploying/using custom certificates. We will be leveraging self signed certificates. As a result, the following tweaks need to be made to the a file in the git repo downloaded above. 

* Create a new file called `pks-config-new.yml` in the correct directory. 
* Copy and paste the contents from the file in the current Github repo [\[here\]](https://github.com/papivot/kickstart-pks-on-azure/blob/master/pks-config-new.yml)
* Save the file content.
* Update the original `pks-config.yml` file with the new one.
```console
cp -pav ../../../ci/assets/azure/pks-config.yml ../../../ci/assets/azure/pks-config-orig.yml
cp pks-config-new.yml ../../../ci/assets/azure/pks-config.yml
``` 
---

Once the file has been updates, deploy the environment using terraform. 

```console
terraform init
terraform plan -out=plan
terraform apply plan
```

This should deploy the OpsMan VM and all the required Azure artifacts(networking/security groups, DNS etc) necessary to PKS environment. 

The `terraform.tfstate` and `output.tf` file contains the required outputs from the terraform run.

* [IMPORTANT] Modify the DNS configurations, so that the newly created network and DNS domain address is resolvable on the network.  

* The OpsMan UI can be accessed in the browser at the following URL - https://pcf.subdomain.domainname.com.
*  Access the OpsMan UI, accept the certificate warnings and proceed. 
* Select `Internal Authentication` when asked to select the authentication method. 
* Enter the values for `Username`, `Password`, `Password confirmation`, `Decryption Password`, `Decryption Password confirmation`
* If using Proxy, enter the proxy server specific configurations. 
* Accept the EULA and press `Setup Authentication`

## Stage 3
### BOSH Director configuration and installation

Leveraging the `terraform.tfstate` file, the BOSH director is configured. While inside the directory where the Stage2 was executed, run the following commands - 

```console
jq -e -r '.outputs|map_values(.value)' terraform.tfstate > tf.output
texplate execute ../../../ci/assets/azure/director-config.yml -f tf.output -o yaml > director-config.yml
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k configure-director --config director-config.yml
```

this should produce an output similar to this - 

```shell
started configuring director options for bosh tile
finished configuring director options for bosh tile
started configuring availability zone options for bosh tile
...
finished configuring network assignment options for bosh tile
started configuring resource options for bosh tile
finished configuring resource options for bosh tile
```

Upon successful configuration, the following changes need to be applies to create and configure the BOSH Director VM. This can be done by -

```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k apply-changes
```

This takes a while (approx. 10+ minutes) and the end result is a fully configured BOSH Director. 

## Stage 4

### Installation of the PKS Control plane.

During this stage, the required product binaries(tiles) and associated stemcell are uploaded to OpsManager. Once done, the tile is configured and the changes applied.

To upload the tile, sign in to https://network.pivotal.io. Once signed in, search for Pivotal Container Service. Select the appropriate version/release - for example - 1.5.1.

Click on the **i** next to the product text. Copy the `pivnet` cli command text and paste it in the bastion shell. For example - 

```console
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=505925
```
The PKS product file would be named similar to `pivotal-container-service-[version_#]-build.[build_#].pivotal`

Similarly, copy the Linux specific PKS CLI and Kubectl CLI pivnet cli command text and past it in the bastion shell. For example -

```console
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=496006 # For Linux specific PKS CLI
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=484672 # For Linux specific Kubectl CLI
```

Find the link to the relevant stemcell in the Product Description tab and download the **IaaS specific** stemcell.  For e.g.

```console
pivnet download-product-files --product-slug='stemcells-ubuntu-xenial' --release-version='315.81' --product-file-id=460228
```
The stemcell file would be named similar to `bosh-stemcell-315.81-azure-hyperv-ubuntu-xenial-go_agent.tgz`

Once these 4 files have been downloaded, the PKS Product file and the stemcell needs to be uploaded to the OpsManager,

This is done by the following commands - 

```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k upload-product  -p [name_of_the_product_file]
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k upload-stemcell -s [name_of_stemcell_file]
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k stage-product   -p pivotal-container-service -v [version_# e.g. 1.5.1-build.8]
```

Similar to the BOSH Director, the PKS tile needs to be configured. The can be done similar to the BOSH Director by running the following commands - 

```console
texplate execute ../../../ci/assets/azure/pks-config.yml -f tf.output > pks-config.yml
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k configure-product --config pks-config.yml
```

This should partially configure the tile with the output similar to this - 

```shell
configuring pivotal-container-service...
setting up network
...
applying errand configuration for the following errands:
	smoke-tests
could not execute "configure-product": configuration not complete.
The properties you provided have been set,
but some required properties or configuration details are still missing.
Visit the Ops Manager for details: 
```
This is expected, as we did not provide the api end point certificates (optional) during Stage 1. This can be manually fixed by performing the following steps - 

* Within the bastion host, capture the value of `pks_api_endpoint` from the `tf.output` file
```console
grep pks_api_endpoint tf.output
```
* Login to the OpsManager UI.
* Click on the `Enterprise PKS` tile.  
* Click on `Settings`
* Click on `PKS API` [should be orange in color]
* In the `API Hostname (FQDN) *` field, provide the value captured for `pks_api_endpoint` earlier. 
*  Click `[Generate RSA Certificate]`  and enter the values of `pks_api_endpoint` 
* Click `Generate`
* Click `Save`

Validate that all the remaining setting are completed and `green`. Once validated, you can apply the configuration by either 

* Going back to the main screen of the OpsManager and the clicking on `Review Pending Changes` followed by `Apply Changes`

or

* executing the following command within the Bastion host

```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k apply-changes
```
This takes a while (approx. 10+ minutes) and the end result is a functioning PKS API Control plane.

## Stage 5

### Configure PKS Control plane

Once the PKS API VM has been installed and configured, we need to create a user(s) that can access the API with the correct privileges to deploy/manage K8s clusters. 

* Get the UAA Management admin credentials from the PKS tile 
	* This can be done thru the UI or by running the following command - 
```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k credentials -p pivotal-container-service -c .properties.pks_uaa_management_admin_client -t json	
# Copy the value of the secret.
```
* Adjust the VM security group. Make sure that the security group `[pks_api_lb_security_group]` is added to the `pivotal-container-service` EC2 instance in the AWS console. 
* Connect to the PKS UAA
```console
uaac target https://[pks_api_endpoint]:8443 --skip-ssl-validation
```

should return something like this - 
```shell
Target: https://[pks_api_endpoint]:8443
Context: admin, from client admin
```
```console
uaac token client get admin -s [pks_uaa_management_admin_client_credential copied above]
```
should return something like this - 
```shell
Successfully fetched token via client credentials grant.
Target: https://[pks_api_endpoint]:8443
Context: admin, from client admin
```

Create a new user `cody` [example] with the admin privileges.

```console
uaac user add cody --emails cody@example.com -p password
#user account successfully added
uaac member add pks.clusters.admin cody
#success
```

The newly created user `cody` can now leverage the PKS CLI to connect to the API endpoint and manage/create Kubernetes clusters. 

```console
pks login -a [pks_api_endpoint] -u cody -k
```

```shell
Password: ********
API Endpoint: [pks_api_endpoint]
User: cody
Login successful.
```

```console
pks plans
```

```shell
Name   ID                                    Description
small  [PLAN ID]                             Example: This plan will configure a lightweight kubernetes cluster. Not recommended for production workloads.
```

## Destroy

Make sure that all EC2 instances created by the PKS environment, **excluding the OpsMan VM**, has been destroyed.

```console
terraform destroy
```
This will destroy all the plumbing and OpsMan VM, that were created in **Stage 2**.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA3NzIyMTc2MiwtMTk1NTc5NTI4MF19
-->