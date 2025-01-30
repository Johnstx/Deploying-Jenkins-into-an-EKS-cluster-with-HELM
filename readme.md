
### Deploying Jenkins using HELM into an EKS cluster provisioned with Terraform.

This Lab further builds upon the previous projects we have done on kubernetes.  Here instead on manually creating a kubernetes cluster or even using eksctl to provision an EKS cluster, we will automate the creation swiftly with Terraform. We will  then see how using HELM to deploy an application compares with use of yml manifest files for the application deployment. 
This lab ellaborates  the preferred real life DevOps approach for software deployments.

KEY STEPS
1. Provisioning EKS cluster - Build Terraform config file for.
2. Depoy an App in the CLuster - Use HELM, best practice.
3. Jenkins App deployed with HELM.
4. CI/CD pipelines with Jenkins