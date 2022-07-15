# Before You Begin

- Install aws cli, eksctl, crossplane CLI, helm, flux, kubectl

# Create EKS Cluster

`eksctl create cluster -f crossplane-team1.yaml`

`aws eks update-kubeconfig --region us-east-1 --name crossplane-team1`

- Edit the configmap to give other humans access

https://github.com/defenseunicorns/eks-cluster-quickstart#add-users

# Install Crossplane

`helm repo add crossplane-stable https://charts.crossplane.io/stable`
`helm repo update`
`helm install crossplane --create-namespace --namespace crossplane-system crossplane-stable/crossplane`

#Configure Crossplane

- Install `Getting Started` Configuration Package

`kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-aws:v1.8.1`

`watch kubectl get pkg`

- Apply Controller Config, Provider & Provider Config to use k8s Node IAM Role

`kubectl apply -f controller-config.yaml`

- Credential EKS Cluster & Bind IAM Role to Crossplane Service Account

```
eksctl utils associate-iam-oidc-provider \
--cluster "crossplane-team1" \
--region "us-east-1" \
--approve
```

`SERVICE_ACCOUNT_NAME=$(kubectl get providers.pkg.crossplane.io crossplane-provider-aws -o jsonpath="{.status.currentRevision}")`

```
eksctl create iamserviceaccount \
--cluster "crossplane-team1" \
--region "us-east-1" \
--name="crossplane-provider-aws-a2e16ca2fc1a" \
--namespace="crossplane-system" \
--role-name="provider-aws" \
--role-only \
--attach-policy-arn="arn:aws:iam::aws:policy/AdministratorAccess" \
--approve
```

<!-- aws iam create-role \
    --role-name "provider-aws" \
    --assume-role-policy-document file://trust.json \
    --description "IAM role for provider-aws"

aws iam attach-role-policy --role-name "provider-aws" --policy-arn=arn:aws:iam::aws:policy/AdministratorAccess -->
