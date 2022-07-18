## NOTE: This is a work in progress. As of 15 Jul, resources are manually configured with AWS IAM role

# Before You Begin

- Install aws cli, eksctl, crossplane CLI, helm, flux, kubectl

# Create EKS Cluster

`eksctl create cluster -f crossplane-team1.yaml`

`aws eks update-kubeconfig --region us-east-1 --name crossplane-team1` # get kube-config

- Edit the configmap to give other humans access

https://github.com/defenseunicorns/eks-cluster-quickstart#add-users

# Install Crossplane

`helm repo add crossplane-stable https://charts.crossplane.io/stable`
`helm repo update`
`helm install crossplane --create-namespace --namespace crossplane-system crossplane-stable/crossplane`

#Configure Crossplane

- Install `Getting Started` Composition Package

`kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-aws:v1.8.1`

- Install Infra Composition Package

Build Source: https://github.com/defenseunicorns/crossplane-config-aws-enclave

`kubectl crossplane install configuration ghcr.io/defenseunicorns/crossplane-config-aws-enclave:0.0.5`

`watch kubectl get pkg`

- Apply Controller Config, Provider & Provider Config to use k8s Node IAM Role

`kubectl apply -f controller-config.yaml`

# Credential EKS Cluster & Bind IAM Role to Crossplane Service Account

```
eksctl utils associate-iam-oidc-provider \
--cluster "crossplane-team1" \
--region "us-east-1" \
--approve
```

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

# Test Provisioning Infra with k8s Node IAM role

## RDS
`kubectl apply -f db.yaml`

`kubectl get rdsinstance -w`

## VPC, Subnets (pub & priv), IGW, NGW, RTB's & DB Subnet group
`kubectl apply -f enclave.yaml`

`watch kubectl get managed` or `kubectl get enclaves`

# Clean up resources

`kubectl delete enclaves my-enclave`

`watch kubectl get managed`

`kubectl delete -f db.yaml`

`kubectl get rdsinstance -w`

# Uninstall Crossplane

https://crossplane.io/docs/v1.8/reference/uninstall.html#uninstalling
