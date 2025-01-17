## NOTE: This is a work in progress. As of 2 Aug, resources are manually configured with AWS IAM role

# Before You Begin

- Install aws cli, eksctl, crossplane CLI, helm, flux, kubectl

# Create EKS Cluster

`eksctl create cluster -f crossplane-team1.yaml`

`aws eks update-kubeconfig --region us-east-1 --name crossplane-team1` # get kube-config

- Edit the configmap to give other humans access

https://github.com/defenseunicorns/eks-cluster-quickstart#add-users

# Install Crossplane

```
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --create-namespace --namespace crossplane-system crossplane-stable/crossplane
```

# Install AWS Crossplane Provider and Controller

- Apply AWS Controller Config for Provider (needed to credential workers)

`kubectl apply -f controller-config.yaml`

- Apply Provider to use k8s Node IAM Role (create k8s service account for AWS)
#similar to the AWS terraform registry

`kubectl apply -f provider.yaml`

#wait for provider to reconcile
`kubectl get providers`

- Apply Provider Config configure for injected identity
`kubectl apply -f provider-config.yaml`

`kubectl get providers.pkg.crossplane.io crossplane-provider-aws -o jsonpath="{.status.currentRevision}"`


# Credential EKS Cluster 

- Associate IAM OIDC Provider
```
eksctl utils associate-iam-oidc-provider \
--cluster "crossplane-team1" \
--region "us-east-1" \
--approve
```

- Create / Bind IAM Role to Crossplane Service Account

`export SERVICE_ACCOUNT_NAME=$(kubectl get providers.pkg.crossplane.io crossplane-provider-aws -o jsonpath="{.status.currentRevision}")`

```
eksctl create iamserviceaccount \
--cluster "crossplane-team1" \
--region "us-east-1" \
--name="$SERVICE_ACCOUNT_NAME" \
--namespace="crossplane-system" \
--role-name="provider-aws" \
--role-only \
--attach-policy-arn="arn:aws:iam::aws:policy/AdministratorAccess" \
--approve
```

# Configure AWS Infra Compositions

- Install `Getting Started with AWS` Composition Package

`kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-aws:v1.8.1`

- Install Custom Unicorn Infra Composition Package
#similar to a terraform modules / vars

Build Source: https://github.com/defenseunicorns/crossplane-config-aws-enclave
NOTE: Recommend review of this build source to understand the specfic AWS VPC resources that will be provisioned by the applying the enclave.yaml custom resource in later steps.

`kubectl crossplane install configuration ghcr.io/defenseunicorns/crossplane-config-aws-enclave:0.0.5`

- Ensure Packages are Installed / Healthy Before Proceeding

`watch kubectl get pkg`


# Apply Resource Claims against our Compositions using the k8s Node IAM role which is credentialed to provision infra
#similar to terragrunt files which leverage vars 

- RDS
`kubectl apply -f db.yaml`

`kubectl get rdsinstance -w`

- VPC, Subnets (pub & priv), IGW, NGW, RTB's & DB Subnet group
`kubectl apply -f enclave.yaml`

`kubectl get enclaves`

- Ensure All Resources All Ready / Synced

`watch kubectl get managed` 

## View a Specific Resource's Configuration (allows you to view what the composition and resource claim collectively provisioned)

kubectl get <`resource name` from kubectl get managed> -oyaml


# Clean up resources

`kubectl delete enclaves my-enclave`

`watch kubectl get managed`

`kubectl delete -f db.yaml`

`kubectl get rdsinstance -w`

# Uninstall Crossplane

https://crossplane.io/docs/v1.8/reference/uninstall.html#uninstalling
