
## Deploying Applications into an EKS cluster with Helm

**Automation technologies** used  - 
* **Kubernetes** as in **EKS** (Ochestrating the deployments)
* **Terraform** (Automating EKS infrastructure setup)
* **Jenkins** (CI/CD)
* **Helm** (Application Deployment)


## Deploying EKS with Terraform

1. Create a working directory, **eks**.
```mkdir eks```

2. Using AWS CLI create an s3 bucket, **terraform eks**. 
    ```
    aws s3api create-bucket --bucket terraform-eks --region us-east-2
    ```

3.  Create a *backend.tf* file. Confiure the backend for remote state in s3

4. create a *network.tf* file for the network resources e.g Elastic IP, VPC and subnets.






After successful cluster deployment, the applications now have a infrastructure to run on, so we can go ahead an deploy them, using HELM.

what is HELM?


## Setting up HELM

1. Download HELM zip file and extract. (You can alternatively use wget to run a more direct install of the desired binary version).

``` 
$ wget https://github.com/helm/helm/archive/refs/tags/v3.6.3.tar.gz
```  
2. Unpack the tar.gz file

```
tar -zxvf v3.6.3.tar.gz 
```
3. cd into the directory and 'build' the source code using the 'make' utility 

```  $ cd helm-3.6.3 ``` 

build source code

 ``` $ make build ```




