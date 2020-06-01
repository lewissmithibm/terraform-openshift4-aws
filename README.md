# Automated OpenShift v4 installation on AWS

This project automates the Red Hat OpenShift Container Platform 4.x installation on Amazon AWS platform. It focuses on the OpenShift User-provided infrastructure installation (UPI) where implementers provide pre-existing infrastructure including VMs, networking, load balancers, DNS configuration etc.

* [Terraform Automation](#terraform-automation)
* [Infrastructure Architecture](#infrastructure-architecture)
* [Installation Procedure](#installation-procedure)
* [Airgapped installation](#airgapped-installation)
* [Removal procedure](#removal-procedure)
* [Advanced topics](#advanced-topics)

## Terraform Automation

This project uses mainly Terraform as infrastructure management and installation automation driver. All the user provisioned resource are created via the terraform scripts in this project.

### Prerequisites

The best way to run this terraform install is to run it from an EC2 instance. You could also run it from your local machine, to do this use the manual setup steps. If you choose to use an EC2 instance, you can use the automated setup detailed below.

There are also a couple of steps detailed after that you must run regardless of your install type.

#### Manual Install
1. To use Terraform automation, download the Terraform binaries [here](https://www.terraform.io/). The code here supports Terraform 0.12 - 0.12.13; there are warning messages to run this on 0.12.14 and later.

   On MacOS, you can acquire it using [homebrew](brew.sh) using this command:

   ```bash
   brew install terraform
   ```

2. Install git

   ```bash
   sudo yum install git-all
   git --version
   ```

3. Install OpenShift command line `oc` cli:

   ```bash
   wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux-4.x.xx.tar.gz
   tar -xvf openshift-client-linux-4.x.xx.tar.gz
   chmod u+x oc kubectl
   sudo mv oc /usr/local/bin
   sudo mv kubectl /usr/local/bin
   oc version
   ```

4. Install wget command:

    - MacOS:
      ```
      brew install wget
      ```
    - Linux: (choose the command depending on your distribution)
      ```
      apt-get install wget
      yum install wget
      zypper install wget
      ```

5. Get the Terraform code

   ```bash
   git clone https://github.com/ibm-cloud-architecture/terraform-openshift4-aws.git
   ```
   
6. Install the AWS CLI using ```https://aws.amazon.com/cli/```
   
#### Automated EC2 Setup

1. Create an EC2 instance using the AMI with the AWS CLI installed, usually second on the list.

2. Choose the `t2.micro` instance type, which can be used here as we don't need anything more than this provides, if you are still within the first year of the account it will also be covered by the free tier.
    
3. On the Configure Instance page, paste the following into the User Data field at the bottom of the form. This will install all of the required components to run the install including cloning this repository into the home folder.
    
   ```
   #!/bin/bash
   sudo yum update
   sudo yum install git-all -y
   wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.18/openshift-client-linux-4.3.18.tar.gz
   tar -xvf openshift-client-linux-4.3.18.tar.gz
   chmod u+x oc kubectl
   sudo mv oc /usr/local/bin
   sudo mv kubectl /usr/local/bin
   sudo rm openshift-client-linux-4.3.18.tar.gz
   git clone https://github.com/ibm-cloud-architecture/terraform-openshift4-aws.git
   sudo mv terraform-openshift4-aws home/ec2-user/
   cd home/ec2-user
   chown -R ec2-user terraform-openshift4-aws
   cd /usr/local/src
   wget https://releases.hashicorp.com/terraform/0.12.25/terraform_0.12.25_linux_386.zip
   unzip terraform_0.12.25_linux_386.zip
   mv terraform /usr/local/bin/
   cd /home/ec2-user/terraform-openshift4-aws/
   terraform init
   ```
   
   **Note:** You can change the OpenShift Client version to match your requirements.

4. Add the tags to match your project

5. Create a new security group with just port 22 for SSH enabled

6. Launch instance

    When the status checks are complete you will be able to ssh to your instance and see this repo in your home folder ready to go.
    
#### Required for both installs
1. Prepare the DNS

   OpenShift requires a valid DNS domain, you can get one from AWS Route53 or using existing domain and registrar. The DNS must be registered as a Public Hosted Zone in Route53. (Even if you plan to use an airgapped environment)
    
2. Prepare AWS Account Access

   Please reference the [Required AWS Infrastructure components](https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html#installation-aws-user-infra-requirements_installing-aws-user-infra) to setup your AWS account before installing OpenShift 4.

   We suggest creating an AWS IAM user dedicated for OpenShift installation with permissions documented above.
   On the bastion host, configure your AWS user credential as environment variables:

    ```bash
    export AWS_ACCESS_KEY_ID=RKXXXXXXXXXXXXXXX
    export AWS_SECRET_ACCESS_KEY=LXXXXXXXXXXXXXXXXXX/ng
    export AWS_DEFAULT_REGION=us-east-2
    ```

## Infrastructure Architecture

For detail on OpenShift UPI, please reference the following:

* [https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html](https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html)
* [https://github.com/openshift/installer/blob/master/docs/user/aws/install_upi.md](https://github.com/openshift/installer/blob/master/docs/user/aws/install_upi.md)

The terraform code in this repository supports 3 installation modes:

- External facing cluster in a private network: ![External Open](img/openshift_aws_external.png)

- Internal cluster with internet access: ![Internal](img/openshift_aws_internal.png)

- Airgapped cluster with no access: ![Airgapped](img/openshift_aws_airgapped.png)

There are other installation modes that are possible with this terraform set, but we have not tested all the possible combinations, see [Advanced usage](#advanced-topics)

## Installation Procedure

This project installs the OpenShift 4 in several stages where each stage automates the provisioning of different components from infrastructure to OpenShift installation. The design is to provide the flexibility of different topology and infrastructure requirement.

1. The deployment assumes that you run the terraform deployment from a Linux based environment. This can be performed on an AWS-linux EC2 instance. The deployment machine has the following requirements:

    - git cli
    - terraform 0.12 or later
    - wget command

2. Deploy the OpenShift 4 cluster using the following modules in the folders:

 	- route53: generate a private hosted zone using route 53
  - vpc: Create the VPC, subnets, security groups and load balancers for the OpenShift cluster
	- install: Build the installation files, ignition configs and modify YAML files
	- iam: define AWS authorities for the masters and workers
	- bootstrap: main module to provision the bootstrap node and generates OpenShift installation files and resources
	- master: create master nodes manually (UPI)

	You can also provision all the components in a single terraform main module, to do that, you need to use a terraform.tfvars, that is copied from the terraform.tfvars.example file. The variables related to that are:

	Create a `terraform.tfvars` file with following content:

```
cluster_id = "ocp4-9n2nn"
clustername = "ocp4"
base_domain = "example.com"
openshift_pull_secret = "./openshift_pull_secret.json"
openshift_installer_url = "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest"

aws_access_key_id = "AAAA"
aws_secret_access_key = "AbcDefGhiJkl"
aws_ami = "ami-06f85a7940faa3217"
aws_extra_tags = {
  "kubernetes.io/cluster/ocp4-9n2nn" = "owned",
  "owner" = "admin"
  }
aws_azs = [
  "us-east-1a",
  "us-east-1b",
  "us-east-1c"
  ]
aws_region = "us-east-1"
aws_publish_strategy = "External"
```

|name | required  | description and value        |
|----------------|------------|--------------|
| `cluster_id` | yes | This id will be prefixed to all the AWS infrastructure resources provisioned with the script - typically using the clustername as its prefix.  |
| `clustername`     | yes  | The name of the OpenShift cluster you will install     |
| `base_domain` | yes | The domain that has been created in Route53 public hosted zone |
| `openshift_pull_secret` | no | The value refers to a file name that contain downloaded pull secret from https://cloud.redhat.com/openshift/install; the default name is `openshift_pull_secret.json` |
| `openshift_installer_url` | no | The URL to the download site for Red Hat OpenShift installation and client codes.  |
| `aws_region`   | yes  | AWS region that the VPC will be created in.  By default, uses `us-east-2`.  Note that for an HA installation, the AWS selected region should have at least 3 availability zones. |
| `aws_extra_tags`  | no  | AWS tag to identify a resource for example owner:myname     |
| `aws_ami` | yes | Red Hat CoreOS ami for your region (see [here](https://docs.openshift.com/container-platform/4.2/installing/installing_aws_user_infra/installing-aws-user-infra.html#installation-aws-user-infra-rhcos-ami_installing-aws-user-infra)). Other platforms images information can be found [here](https://github.com/openshift/installer/blob/master/data/data/rhcos.json) |
| `aws_secret_access_key` | yes | adding aws_secret_access_key to the cluster |
| `aws_access_key_id` | yes | adding aws_access_key_id to the cluster |
| `aws_azs` | yes | list of availability zones to deploy VMs |
| `aws_publish_strategy` | no | Whether to publish the API endpoint externally - Default: "External" |
| `airgapped` | no | A map with enabled (true/false) and repository name - This must be used with `aws_publish_strategy` of `Internal` |


See [Terraform documentation](https://www.terraform.io/intro/getting-started/variables.html) for the format of this file.

### Deploying the cluster

Initialize the Terraform:

```bash
terraform init
```

Run the terraform provisioning:

```bash
terraform plan
terraform apply
```

### Removing bootstrap node
 
Once the cluster is installed, the bootstrap node is no longer used at all. One of the indication that the bootstrap has been completed is that the API load balancer target group shows that the bootstrap address is `unhealthy`. 

```
terraform destroy -target=module.bootstrap.aws_instance.bootstrap
```



## Airgapped Installation

For performing a completely airgapped cluster, there are two capabilities that would not be available from the cluster's automation capabilities, the IAM and Route53 management access. The airgapped solution can address this by pre-creating the roles and secret that are needed for OpenShift to complete its functions, but the DNS update on Route53 must be performed manually after the installation.

**Note** This installation uses OpenShift Version 4.3.18.

Setting up the mirror repository using AWS ECR:

1. Create the repository

    ```
    aws ecr create-repository --repository-name ocp4318
    ```

2. Prepare your credential to access the ECR repository (ie the credential only valid for 12 hrs)

    ```
    aws ecr get-login
    ```

    Extract the password token (`-p` argument) and create a Base64 string:

    ```
    echo -n "AWS:<token>" | base64 -w0
    ```

    Add that into your existing OpenShift pull pull secret:

    ```
    {"<ecr_uri>":{"auth":"<base64string>","email":"abc@example.com"}}
    ```

3. Mirror quay.io and other OpenShift source into your repository

    ```
    export OCP_RELEASE="4.3.18-x86_64"
    export LOCAL_REGISTRY='<ecr_uri>'
    export LOCAL_REPOSITORY='ocp4318'
    export PRODUCT_REPO='openshift-release-dev'
    export LOCAL_SECRET_JSON='/home/ec2-user/openshift_pull_secret.json'
    export RELEASE_NAME="ocp-release"

    oc adm -a ${LOCAL_SECRET_JSON} release mirror --max-per-registry=1 \
       --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
       --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
       --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}
    ```

    Once the mirror registry is created - use the terraform.tfvars similar to below:
    
    To get the AMI for your region see page 137 of: https://access.redhat.com/documentation/en-us/openshift_container_platform/4.3/pdf/installing_on_aws/OpenShift_Container_Platform-4.3-Installing_on_AWS-en-US.pdf 
    
    ```
    cluster_id = "ocp4-9n2nn"
    clustername = "ocp4"
    base_domain = "example.com"
    openshift_pull_secret = "./openshift_pull_secret.json"
    openshift_installer_url = "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.18/"
    
    aws_access_key_id = "AAAA"
    aws_secret_access_key = "AbcDefGhiJkl"
    aws_ami = "<ami_for_your_region>"
    aws_extra_tags = {
      "kubernetes.io/cluster/ocp4-9n2nn" = "owned",
      "owner" = "admin"
      "projec" = "project name"
      }
    aws_azs = [
      "us-east-1a",
      "us-east-1b",
      "us-east-1c"
      ]
    aws_region = "us-east-1"
    aws_publish_strategy = "Internal"
    airgapped = {
      enabled = true
      repository = "<ecr_uri>/ocp4318"
    }
    ```

4. Deploying the cluster

    Initialize the Terraform:
    
    ```bash
    terraform init
    ```

    Run the terraform provisioning:
    
    ```bash
    terraform plan
    terraform apply
    ```

    Once the terraform has been run, get the `private_key_pem` from the `terraform.tfstate` file. This is what we will use in our jump server to connect to the bootstrap node.
    
    ```
    cat terraform.tfstate
    ```

    This will likely be at the bottom of the state file.
    
    To convert this to a usable key file run, from your local machine
    
    ```
    vi key.pem
    ```

    Paste in the contents of `private_key_pem`
    
    Run this command in vi to format the file
    
    ```
    :%s/\\n/\r/g
    ```
    
    Once saved, update the permissions on the `key.pem` file, so we can SSh with it.
    
    ```
    chmod 0600 key.pem
    ```

5. We need connect to your cluster in order to create a load balancer, in order to do this we need a jump server within the private VPC that has public ssh access

Within the AWS console, go to the VPC service

Create a new Internet Gateway, then attach it to your private VPC

Go to your VPC, click your route table and click edit routes

```
Destination
0.0.0.0/0

Target
The internet gate way we created in the last step
```

Save

Navigate back to EC2 instances console, launch a new instance

In the configure instance tab, make sure to choose your private VPC and create a *new* subnet. Set Auto-assign public Ip to enable.

Launch and then connect to your EC2 instance

Move your `key.pem` file from your local machine to the EC2 instance

Ssh to your bootstrap node

``
ssh -i key.pem core@<bootstrap_node_ip>
``

Copy your `kubeconfig` file to the bootstrap node, then export it

``
export KUBECONFIG=kubeconfig
``

You should now be able to run

``
oc get co
``

Update the `router-internal-default` service type from `ClusterIp` to `NodePort`

``
oc edit svc router-internal-default -n openshift-ingress
`` 

Run

``
oc get svc -n openshift-ingress
``

Take note of the ports assigned to port 80 and port 443.

5. Create a classic load balancer

Make sure you pick a classic load balancer, provide a name and choose the VPC that was generated by the terraform script

Edit the existing listener to use the instance port matching the one you recorded for port 80 in the last step.

Add a new record with the Load Balancer Protocol of `TCP` with port 443 and then instance port matching that of port 443 in the last step.

Select all the subnets, excluding your manually created subnet. As this has public access and will be later deleted.

On Assign Security Groups, select create new

On Health Check set

``
Ping Protocol
TCP

Ping Port
Use the recorded port for TCP for port 80 in the previous step

Ping Path
/
``

On Add EC2 Instance page, select all the workers

Then create

Navigate to Route53 and then click on the private hosted zone that mates your cluster, it will take the form below

``<cluster_name>.<domain_name>.``

Create a new record set

``
Name
*.apps

Type
A - IPV4 Address

Alias - yes
Select the load balancer we just created
``

Save

6. Update worker security group

Navigate to your EC2 Instances, click on a worker then the security group.

Edit inbound rules

Add rule

``
Type
Custom TCP

Protocol
TCP

Port Range
30000 - 32767

Source
0.0.0.0/0
``
*The source in a production environment should not be the world. This is set to open just for demonstrative purposes*

Save.

7. Final check

After 5 - 10 minutes, running:

``
oc get co
``

Should show all to be available.

**Note**
You should now remove your public subnet and jump server, this will return the cluster to an airgapped state.  

## Removal Procedure

For the removal of the cluster, there are several considerations for removing AWS resources that are created by the cluster directly, but not using Terraform. These resources are unknown to terraform and must be deleted manually from AWS console.
Some of these resources also hamper the ability to run `terraform destroy` as it becomes a dependent resource that prevent its parent resource to be deleted.

The cluster created resources are:

- Resources that prevents `terraform destroy` to be completed:
  - Worker EC2 instances
  - Application Load Balancer (classic load balancer) for the `*.apps.<cluster>.<domain>`
  - Security Group for the application load balancer
- Other resources that are not deleted:
  - S3 resource for image-registry
  - IAM users for the cluster
  - Public Route53 Record set associated with the application load balancer

## Advanced topics

Additional configurations and customization of the implementation can be performed by changing some of the default variables.
You can check the variable contents in the following terraform files:

- variable-aws.tf: AWS related customization, such as machine sizes and network changes
- config.tf: common installation variables for installation (not cloud platform specific)

**Note**: Not all possible combinations of options has been tested - use them at your own risk. 
