# Introduction

With the introduction of Tanzu Application Platform version 1.2, VMware and Amazon partnered to create the [VMware Tanzu Application Platform on AWS Cloud Quick Start](https://aws.amazon.com/quickstart/architecture/vmware-tanzu-application-platform/).  This Quick Start, allows you to get started building applications with the Tanzu Application Platform in the fastest way available by deploying the infrastructure required and the Tanzu Application Platform on the AWS Cloud.

As part of this effort, several of the services that comprise the Tanzu Application Platform were enhanced to support using an [IAM role bound to a Kubernetes service account](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for authentication rather than the typical username and password stored in a kubernetes secret strategy.  This is important when using the Elastic Container Registry (ECR) because authenticating to ECR is a two step process:  

1. Retrieve a token using your AWS credentials
1. Use the token to authenticate to the registry  

To increase security posture, the token has a lifetime of 12 hours.  This makes storing it as a secret for a service impracticle, as it would have to be refereshed every 12 hours.

Leveraging an IAM role on a service account mitigates the need to retrieve the token at all, as it is handled by credential helpers in the services.  As I like to say, the only thing better than storing a secret securly is to not have a secret at all!

While the Quick Start drove these enhancements, they work for standalone Tanzu Application Platform as well.  The purpose of this guide is to walk you through how to go from zero to a running instance of Tanzu Application Platform on EKS leveraging ECR, with an emphasis on how to create the resource with AWS (EKS Cluster, ECR registries, IAM roles, etc).

# Scope of this Guide

The purpose of this guide is to illustrate how to configure both the Tanzu Application Platform and Amazon Web Services using a very simple deloyment and workload.  It is not intended to be a full explaination of all the options for both platforms, nor a full explanation of how to deploy the platform.  It will reference official documenation as much as possible.

For more complex and production ready deployments, we highly suggest referencing the [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html) and [Amazon Web Services](https://docs.aws.amazon.com/) documentation.

Prerequisites
Create an EKS Cluster
Create the container registries
Create the IAM Roles
Configure the Tanzu Application Platform Values
Deploy a workload

# Prerequisites

## AWS Account

We will be creating all of our resources within Amazon Web Services, so therefore we will need an Amazon account.  This guide was designed to use minimal resources to keep the cost low.

To create an Amazon account, use the [following guide](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).

Note that you will need your account ID during the guide.

## Tanzu Network Account

The images used for the Tanzu Application Platform are hosted on the [Tanzu Network](https://network.pivotal.io/).  You will need an account to log in.

## AWS CLI

This walkthrough will use the AWS CLI to both query and configure resources in AWS such as IAM roles.

To install the AWS CLI, use the [following guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

## EKSCTL

The EKSCTL command line makes managing the lifecycle of EKS clusters very simple.  This guide will use it to create clusters.

To install the eksctl cli, use the [following guide](https://eksctl.io/introduction/#installation).

# Export Environment Variables

Some variables are used throughout the guide, to simplify the process and minimize the opportunity for errors, let's export these variables.

```
export ACCOUNT_ID=12346789
export AWS_REGION=us-west-2
```

Where:

| Variable | Description |
| -------- | ----------- |
| ACCOUNT_ID | Your AWS account ID |
| AWS_REGION | The AWS region you are going to deploy to |


# Create an EKS Cluster

The eksctl cli makes creating an EKS Cluster a breeze.  Let's create an EKS in the specified region using the following command.  Creating the control plane and node group can take  take anywhere from 30-60 minutes. 

```
eksctl create cluster --name tap-aws --managed --region $AWS_REGION --instance-types t3.large --version 1.22 --with-oidc -N 4
```

Note: This step is optional if you already have an existing EKS Cluster.  Note, however that it has to be at least version 1.22 and have the OIDC authentication enabled.  To enable the OIDC provider, use the [following guide](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

# Create the Container Registries

ECR requires the container repositories to be precreated.   While it is [recommended](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-install.html#relocate-images-to-a-registry-0) to replicate the images for Tanzu Application Platform, for this guide will use images hosted on the Tanzu Network for simplicity.

As part of the install process, we only need a repository for the Tanzu Build Service images.  To create a repository, run the following command:

```
aws ecr create-repository tap-build-service-images
```