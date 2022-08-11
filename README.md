# Introduction

With the introduction of Tanzu Application Platform version 1.2, VMware and Amazon partnered to create the VMware Tanzu Application Platform on AWS Cloud Quick Start.  This Quick Start, allows you to get started building applications with the Tanzu Application Platform in the fastest way available by deploying the infrastructure required and the Tanzu Application Platform on the AWS Cloud.

As part of this effort, several of the services that make up the Tanzu Application Platform were enhanced to support using an (IAM role bound to a Kubernetes service account)[https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html] for authentication rather than the typical username and password stored in a kubernetes secret strategy.  This is important when using the Elastic Container Registry (ECR), because authenticating to ECR is a two step process.  You must first retrieve a token using your AWS credentials, and then you use that token to authenticate to the registry.  The bad news is, has a lifetime of 12 hours, so it would constantly need to be refreshed.  Leveraging an IAM role on the service account mitigates the need to retrieve the token at all, as it is handled by credential helpers in the services.  As I like to say, the only thing better than storing a secret securly is to not have a secret at all!

While these enhancements were made for the Quick Start, the good news is, those who wish to deploy the Tanzu Application Platform on top of EKS and use ECR as the container registry can take advantage of them as well. Let’s walk through how we can deploy the Tanzu Application Platform on AWS EKS using ECR.

# Prerequisites
# Create an EKS Cluster
# Create the container registries
# Enable OIDC authentication on EKS Cluster
# Create the IAM Roles
# Configure the Tanzu Application Platform Values
# Deploy a workload

## Prerequisites

### AWS CLI

### EKSCTL

## Create an EKS Cluster

This step is optional if you already have an existing EKS Cluster.  Note, however that it has to be at least version 1.22 and have the OIDC authentication enabled.

```eksctl create cluster --name tap-aws --region us-west-1 --managed --instance-types t3.large --version 1.22 --with-oidc -N 4```

## Create the Container Registries

ECR requires the container registries to be precreated.  Let’s precreate the registries required:

```aws ecr create-repository tap-build-service-images```