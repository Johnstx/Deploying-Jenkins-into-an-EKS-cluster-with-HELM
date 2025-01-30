

### Deploying Jenkins using HELM into an EKS cluster provisioned wioth Terraform.



####  The EKS cluster set-up with Terraform.

1. Create a working directory on your local device, name is ```eks``` 
```mkdir eks`` 

2. Create an S3 bucket with aws CLI
```
aws s3api create-bucket --bucket name --region state-region
```


3. Create a ```backend.tf``` file.

```
terraform {
  backend "s3" {
    bucket         = "choose-name"
    key            = "global/s3/terraform.tfstate"
    region         = "region"
    dynamodb_table = "terraform-locks" #optional
    encrypt        = true
  }
}
``` 

4. Create a file – ```network.tf``` and provision Elastic IP for Nat Gateway, VPC, Private and public subnets.

```
# reserve Elastic IP to be used in our NAT gateway
resource "aws_eip" "nat_gw_elastic_ip" {
  vpc = true

  tags = {
    Name            = "${var.cluster_name}-nat-eip"
    iac_environment = var.iac_environment_tag
  }
}

# create VPC using the official AWS module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"

  name = "${var.name_prefix}-vpc"
  cidr = var.main_network_block
  azs  = data.aws_availability_zones.available_azs.names

  private_subnets = [
    # this loop will create a one-line list as ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", ...]
    # with a length depending on how many Zones are available
    for zone_id in data.aws_availability_zones.available_azs.zone_ids :
    cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) - 1)
  ]

  public_subnets = [
    # this loop will create a one-line list as ["10.0.128.0/20", "10.0.144.0/20", "10.0.160.0/20", ...]
    # with a length depending on how many Zones are available
    # there is a zone Offset variable, to make sure no collisions are present with private subnet blocks
    for zone_id in data.aws_availability_zones.available_azs.zone_ids :
    cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) + var.zone_offset - 1)
  ]

  # enable single NAT Gateway to save some money
  # WARNING: this could create a single point of failure, since we are creating a NAT Gateway in one AZ only
  # feel free to change these options if you need to ensure full Availability without the need of running 'terraform apply'
  # reference: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/2.44.0#nat-gateway-scenarios
  enable_nat_gateway     = true
  single_nat_gateway     = true
  one_nat_gateway_per_az = false
  enable_dns_hostnames   = true
  reuse_nat_ips          = true
  external_nat_ip_ids    = [aws_eip.nat_gw_elastic_ip.id]

  # add VPC/Subnet tags required by EKS
  tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    iac_environment                             = var.iac_environment_tag
  }
  public_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = "1"
    iac_environment                             = var.iac_environment_tag
  }
  private_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
    iac_environment                             = var.iac_environment_tag
  }
}
```

**Note**: The tags added to the subnets is very important. The Kubernetes Cloud Controller Manager (cloud-controller-manager) and AWS Load Balancer Controller (aws-load-balancer-controller) needs to identify the cluster's. To do that, it querries the cluster's subnets by using the tags as a filter.

* For public and private subnets that use load balancer resources: each subnet must be tagged

```
Key: kubernetes.io/cluster/cluster-name
Value: shared```
```

* For private subnets that use internal load balancer resources: each subnet must be tagged

```
Key: kubernetes.io/role/internal-elb
Value: 1
```

* For public subnets that use internal load balancer resources: each subnet must be tagged

```
Key: kubernetes.io/role/elb
Value: 1
```


5. Create a file - ```variables.tf```

```
# create some variables
variable "cluster_name" {
  type        = string
  description = "EKS cluster name."
}
variable "iac_environment_tag" {
  type        = string
  description = "AWS tag to indicate environment name of each infrastructure object."
}
variable "name_prefix" {
  type        = string
  description = "Prefix to be used on each infrastructure object Name created in AWS."
}
variable "main_network_block" {
  type        = string
  description = "Base CIDR block to be used in our VPC."
}
variable "subnet_prefix_extension" {
  type        = number
  description = "CIDR block bits extension to calculate CIDR blocks of each subnetwork."
}
variable "zone_offset" {
  type        = number
  description = "CIDR block bits extension offset to calculate Public subnets, avoiding collisions with Private subnets."
}

# create some variables
variable "admin_users" {
  type        = list(string)
  description = "List of Kubernetes admins."
}
variable "developer_users" {
  type        = list(string)
  description = "List of Kubernetes developers."
}

variable "asg_instance_types" {
  description = "List of EC2 instance machine types to be used in EKS."
}
variable "autoscaling_minimum_size_by_az" {
  type        = number
  description = "Minimum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_maximum_size_by_az" {
  type        = number
  description = "Maximum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_average_cpu" {
  type        = number
  description = "Average CPU threshold to autoscale EKS EC2 instances."
}
```

6. Create a file - ```data.tf ```- This will pull the available AZs for use.

```
# get all available AZs in our region
data "aws_availability_zones" "available_azs" {
  state = "available"
}

data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN
```


7. Create a file -``` eks.tf``` and provision EKS cluster

```
module "eks_cluster" {
  source                                 = "terraform-aws-modules/eks/aws"
  version                                = "~> 18.0"
  cluster_name                      = var.cluster_name
  cluster_version                    = "1.28"
  subnet_ids                          = module.vpc.private_subnets
  vpc_id                                  = module.vpc.vpc_id
  cluster_endpoint_public_access = true
  cluster_endpoint_private_access = true


#   # Self Managed Node Group(s)
  self_managed_node_group_defaults = {
      instance_type                          = var.asg_instance_types[0]
      update_launch_template_default_version = true
  }
  self_managed_node_groups = local.self_managed_node_groups
 
    # aws-auth configmap
  create_aws_auth_configmap     = true
  manage_aws_auth_configmap  = true
  aws_auth_users                         = concat(local.admin_user_map_users, local.developer_user_map_users)
  tags = {
    Environment = "prod"
    Terraform   = "true"
  }
}
#  worker_groups_launch_template = local.worker_groups_launch_template

#   # map developer & admin ARNs as kubernetes Users
#   map_users = concat(local.admin_user_map_users, local.developer_user_map_users)
# }


# module "eks_cluster" {
#   source                          = "terraform-aws-modules/eks/aws"
#   version                         = "~> 18.0"
#   cluster_name                    = var.cluster_name
#   cluster_version                 = "1.28"
#   vpc_id                          = module.vpc.vpc_id
#   subnet_ids                      = module.vpc.private_subnets
#   cluster_endpoint_private_access = true
#   cluster_endpoint_public_access  = true

#   # Self Managed Node Group(s)
#   self_managed_node_group_defaults = {
#     instance_type                          = var.asg_instance_types[0]
#     update_launch_template_default_version = true
#   }
#   self_managed_node_groups = local.self_managed_node_groups

#
```

8. Create a file – ```data.tf``` – lists available AZs in the region

```
# get all available AZs in our region
data "aws_availability_zones" "available_azs" {
  state = "available"
}

data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN

# get EKS cluster info to configure Kubernetes and Helm providers
data "aws_eks_cluster" "cluster" {
  name = module.eks_cluster.cluster_id
}
data "aws_eks_cluster_auth" "cluster" {
  name = module.eks_cluster.cluster_id
}
```

9. create a file - ```ebs-csi.tf```.

```
data "aws_iam_policy" "ebs_csi_policy" {
  arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}

module "irsa-ebs-csi" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version = "5.32.0"

  create_role                   = true
  role_name                     = "AmazonEKSTFEBSCSIRole-${var.cluster_name}"
  provider_url                  = module.eks_cluster.oidc_provider
  role_policy_arns              = [data.aws_iam_policy.ebs_csi_policy.arn]
  oidc_fully_qualified_subjects = ["system:serviceaccount:kube-system:ebs-csi-controller-sa"]
}

resource "aws_eks_addon" "ebs-csi" {
  cluster_name             = var.cluster_name
  addon_name               = "aws-ebs-csi-driver"
  service_account_role_arn = module.irsa-ebs-csi.iam_role_arn
  tags = {
    "eks_addon" = "ebs-csi"
    "terraform" = "true"
  }
}
```
10. Create a file -``` locals.tf``` to create local variable. Assigning variable to variables is not allowed in terraform, to prevent code repetitions.  A terraform way to achieve this ```[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)``` approach is the use of locals.

```
# render Admin & Developer users list with the structure required by EKS module
locals {
  admin_user_map_users = [
    for admin_user in var.admin_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${admin_user}"
      username = admin_user
      groups   = ["system:masters"]
    }
  ]
  developer_user_map_users = [
    for developer_user in var.developer_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${developer_user}"
      username = developer_user
      groups   = ["${var.name_prefix}-developers"]
    }
  ]

  self_managed_node_groups = {
    worker_group1 = {
      name = "${var.cluster_name}-wg"

      min_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      desired_size  = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      max_size      = var.autoscaling_maximum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      instance_type = var.asg_instance_types[0].instance_type

      bootstrap_extra_args = "--kubelet-extra-args '--node-labels=node.kubernetes.io/lifecycle=spot'"

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            delete_on_termination = true
            encrypted             = false
            volume_size           = 80
            volume_type           = "gp2"
          }
        }
      }

      use_mixed_instances_policy = true
      mixed_instances_policy = {
        instances_distribution = {
          spot_instance_pools = 4
        }

        override = var.asg_instance_types
      }
    }
  }
}

```

11. Create a - ``` terraform.tfvars ``` file. Used to define values for variables in ```variables.tf```

```
cluster_name                                     = "staxx-tooling-app"
iac_environment_tag                         = "development"
name_prefix                                       = "staxx-eks"
main_network_block                          = "10.0.0.0/16"
subnet_prefix_extension                    = 4
zone_offset                                        = 8

# Ensure that these users already exist in AWS IAM. Another approach is that you can introduce an iam.tf file to manage users separately, get the data source and interpolate their ARN.
admin_users                                        = ["staxx", "solomon"]
developer_users                                  = ["john", "david"]
asg_instance_types                             = [{ instance_type = "t3.medium" },  { instance_type = "t2.medium" } ]
autoscaling_minimum_size_by_az      = 1
autoscaling_maximum_size_by_az     = 10
autoscaling_average_cpu                   = 30
```
12. Create file – ```provider.tf```

```
provider "aws" {
  region = "us-east-2"
}

provider "random" {
}

# get EKS authentication for being able to manage k8s objects from terraform
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}
```

**See the terraform file [here]()**

13. Run terraform init

![alt text](<images/terraform init pass.jpg>)

*If your provider is not correctly specified, the ```init``` step may not run correctly*

14. Run terraform plan

If the code checks are okay, then

![alt text](<images/t-plan pass-2.jpg>)


15. Run terraform apply

![alt text](<images/cluster create with ebs.jpg>)

Check the cluster dashboard in the AWS console
![alt text](<images/cluster aws.jpg>)



**This setup above has successfully provision an kubernetes cluster  (EKS on AWS) which will be the infrastructure we will run our servers/applications on**
Next, we deploy the apps.

**Deploy applications with Helm**

NB: We have previously deployed apps in pods by using manifest files to to create ```deployments```, ```pods```, etc. This time we will use  it in a different way such that will not pass through kubectl. This method uses a tool called **[HELM](https://helm.sh/docs/intro/quickstart/)**. Helm is a great tool used in kubernetes to manage kubernetes applications. It is used to define, create and manage applications by using of helm charts, which are pre-configured packages of the kubernetes resources.

A Helm chart is a definition of the resources that are required to run an application in Kubernetes. Instead of having to think about all of the various deployments/services/volumes/configmaps/ etc that make up your application, you can use a command like
``` 
helm install stable/mysql
``` 
and Helm will make sure all the required resources are installed. In addition you will be able to tweak helm configuration by setting a single variable to a particular value and more or less resources will be deployed. For example, enabling slave for MySQL so that it can have read only replicas.
 
 HELM has some useful features, an exaple is TEMPLATES which makes it easy to generate kubernetes resources dynamically. It makes use of  {{ }} to define dynamic sections that can be populated with variables from a values file. There are other features of HELM that makes it a great tool. See the documentation link **[HELM](https://helm.sh/docs/intro/quickstart/)** for more.


 **Helm Setup**

 1. Download the tar.gz file
 ```
 wget https://github.com/helm/helm/archive/refs/tags/v3.6.3.tar.gz
```

2. Unpack the file.
```
tar -zxvf v3.6.3.tar.gz 
```

3. cd into the unpacked directory
```
cd helm-3.6.3
```

4. Build the source code using the ```make``` utility

```
make build
```

If you do not have ```make``` installed or for any other reason, you cannot install the tool, simply use the official documentation [here](https://helm.sh/docs/intro/install/) for other options.

5. Move the the helm binary to the ```bin``` directory.

```
sudo mv bin/helm /usr/local/bin/
```
6. Confirm helm is installed
``` helm version ```

![alt text](<images/helm install.jpg>)


**Deploy Jenkins with Helm**

Lets Illustrate the utility of HELM in app deployments with **Jenkins** deployments.
With helm, one can deploy applications that are already packaged from a public helm repository directly with very minimal configuration.

Pre-packaged applications are available as Helm Charts in **[Artifact Hub](https://artifacthub.io/packages/search)**

* In the Artifact hub, search for jenkins
* Add the repo to the helm

![alt text](<images/jenkins artifact hub.jpg>)

```
helm repo add jenkins https://charts.jenkins.io
```
* Update helm repo

```
helm repo update 
```
*(Optional) Create a namespace for each workspace to group your apps into, and then specify the -n tag when install the helm chart of the concerned appliction*
E.g helm install [RELEASE_NAME] jenkins/jenkins -n dev
```dev``` being a namepace created in the cluster

![alt text](<images/create ns dev.jpg>)

* Install the chart
```
helm install [RELEASE_NAME] jenkins/jenkins --kubeconfig [kubeconfig file]
```
![alt text](<images/helm install success.jpg>)

*Confirm install is successful with an output like above.*

* Check the Helm deployment

```
helm ls --kubeconfig [kubeconfig file]
```
*To avoid calling the --kubeconfig [kubeconfig file] everytime we query the helm chart, we will apply some setups that will let allow merging of all existing kubeconfig files together, then select the one we prefer to work with easily so that any query being run afterwards will be return information on  the selected cluster. We will be able to merge the kubeconfig files together using a kubectl plugin called [Konfig].*

  * Install a package manager for kubectl called [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/).

  * Install krew
      ```
      kubectl krew install konfig
      ```
![alt text](<images/kubectl krew version.jpg>)

  * Import the kubeconfig into the default kubeconfig file. Ensure to accept the prompt to overide.
      ```
      sudo kubectl konfig import --save  [kubeconfig file]
      ``` 
  * Show all the contexts - Meaning all the clusters configured in the kubeconfig. If there are more than one Kubernetes clusters configured, they will be listed.
    ```
    kubectl config get-contexts
    ``` 
  * Set the current context to use for all the kubectl and helm commands
    ```
    kubectl config use-context [name of EKS cluster]
    ```
![alt text](<images/k get contexts - one context.jpg>)
*Theres only one context available in my cluster*

* Now, check the pods
```
kubectl get pods -n dev
```
![alt text](<images/ebs helm release get pods.jpg>)

![alt text](<images/kubectl describe pods dev.jpg>)


* Use the commands on the jenkins install page to access the jenkins UI.
```
kubectl exec --namespace default -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
```
![alt text](<images/jenkins password.jpg>)

* Use port forwarding to access Jenkins from the UI.
```
kubectl --namespace default port-forward svc/jenkins 8080:8080
```

Jenkins UI
![alt text](<images/jenkins UI.jpg>)











