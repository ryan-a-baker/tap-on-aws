{{toc}}

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
eksctl create cluster --name tap-on-aws --managed --region $AWS_REGION --instance-types t3.large --version 1.22 --with-oidc -N 4
```

Note: This step is optional if you already have an existing EKS Cluster.  Note, however that it has to be at least version 1.22 and have the OIDC authentication enabled.  To enable the OIDC provider, use the [following guide](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

# Create the Container Repositories

ECR requires the container repositories to be precreated.   While it is [recommended](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-install.html#relocate-images-to-a-registry-0) to replicate the images for Tanzu Application Platform, for this guide will use images hosted on the Tanzu Network for simplicity.

As part of the install process, we only need a repository for the Tanzu Build Service images.  To create a repository, run the following command:

```
aws ecr create-repository tap-build-service
```

# Create IAM Roles

By default, the EKS cluster will be provisioned with an [EC2 instance profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) that provides read-only access for the entire EKS cluster to the ECR registery within your AWS account.

However, some of the services within the Tanzu Application Platform require write access to the container repositories. In order to provide that access, we will create IAM roles and add the ARN to the K8S service accounts that those services use.  This will ensure that only the required services have access to write container images to ECR, rather than a blanket policy that applies to the entire cluster.

We'll create two IAM Roles.

1.  Tanzu Build Service: Gives write access to the repository to allow the service to automatically upload new images.  This is limited in scope to the service account for kpack and the dependency updater.
2.  Workload:  Gives write access to the entire ECR registry.  This prevents you from having to update the policy for each new workload created.  For those looking for a more strict model, we'll include examples for the sample workload deployed later in the guide.

We've already exported the region and account id, but let's get the OIDC provider now that our cluster has been created:

```
aws eks describe-cluster --name tap-on-aws --region $AWS_REGION | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///'
```

Now, we can create the policy documents we'll use to create the role.  We need two policy documents:

1. Trust Policy: Limits the scope to the OIDC endpoint and the service account we'll attach the role to
1. Permission Policy:  Limits the scope of what actions can be taken on what resources by the role.  

**Note** Both of these policies are best effort at a least privledge model, but be sure to review to ensure if they adhere to your organizations policies.

Build Service Trust Policy:

```
cat << EOF > build-service-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$oidcProvider"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "$oidcProvider:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "$oidcProvider:sub": [
                        "system:serviceaccount:kpack:controller",
                        "system:serviceaccount:build-service:dependency-updater-controller-serviceaccount"
                    ]
                }
            }
        }
    ]
}
EOF
```

Workload Trust Policy:
```
cat << EOF > workload-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$oidcProvider"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "$oidcProvider:sub": "system:serviceaccount:tap-workload:default",
                    "$oidcProvider:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
EOF
```

Build Service Permission Policy:

```
cat << EOF > build-service-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:DescribeRegistry",
                "ecr:GetAuthorizationToken",
                "ecr:GetRegistryPolicy",
                "ecr:PutRegistryPolicy",
                "ecr:PutReplicationConfiguration",
                "ecr:DeleteRegistryPolicy"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "TAPEcrBuildServiceGlobal"
        },
        {
            "Action": [
                "ecr:DescribeImages",
                "ecr:ListImages",
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:BatchGetRepositoryScanningConfiguration",
                "ecr:DescribeImageReplicationStatus",
                "ecr:DescribeImageScanFindings",
                "ecr:DescribeRepositories",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetLifecyclePolicy",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetRegistryScanningConfiguration",
                "ecr:GetRepositoryPolicy",
                "ecr:ListTagsForResource",
                "ecr:TagResource",
                "ecr:UntagResource",
                "ecr:BatchDeleteImage",
                "ecr:BatchImportUpstreamImage",
                "ecr:CompleteLayerUpload",
                "ecr:CreatePullThroughCacheRule",
                "ecr:CreateRepository",
                "ecr:DeleteLifecyclePolicy",
                "ecr:DeletePullThroughCacheRule",
                "ecr:DeleteRepository",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:PutImageScanningConfiguration",
                "ecr:PutImageTagMutability",
                "ecr:PutLifecyclePolicy",
                "ecr:PutRegistryScanningConfiguration",
                "ecr:ReplicateImage",
                "ecr:StartImageScan",
                "ecr:StartLifecyclePolicyPreview",
                "ecr:UploadLayerPart",
                "ecr:DeleteRepositoryPolicy",
                "ecr:SetRepositoryPolicy"
            ],
            "Resource": [
                "arn:aws:ecr:$AWS_REGION:$ACCOUNT_ID:repository/tap-build-service"
            ],
            "Effect": "Allow",
            "Sid": "TAPEcrBuildServiceScoped"
        }
    ]
}
EOF
```

Workflow Permission Service:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:DescribeRegistry",
                "ecr:GetAuthorizationToken",
                "ecr:GetRegistryPolicy",
                "ecr:PutRegistryPolicy",
                "ecr:PutReplicationConfiguration",
                "ecr:DeleteRegistryPolicy"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "TAPEcrWorkloadGlobal"
        },
        {
            "Action": [
                "ecr:DescribeImages",
                "ecr:ListImages",
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:BatchGetRepositoryScanningConfiguration",
                "ecr:DescribeImageReplicationStatus",
                "ecr:DescribeImageScanFindings",
                "ecr:DescribeRepositories",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetLifecyclePolicy",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetRegistryScanningConfiguration",
                "ecr:GetRepositoryPolicy",
                "ecr:ListTagsForResource",
                "ecr:TagResource",
                "ecr:UntagResource",
                "ecr:BatchDeleteImage",
                "ecr:BatchImportUpstreamImage",
                "ecr:CompleteLayerUpload",
                "ecr:CreatePullThroughCacheRule",
                "ecr:CreateRepository",
                "ecr:DeleteLifecyclePolicy",
                "ecr:DeletePullThroughCacheRule",
                "ecr:DeleteRepository",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:PutImageScanningConfiguration",
                "ecr:PutImageTagMutability",
                "ecr:PutLifecyclePolicy",
                "ecr:PutRegistryScanningConfiguration",
                "ecr:ReplicateImage",
                "ecr:StartImageScan",
                "ecr:StartLifecyclePolicyPreview",
                "ecr:UploadLayerPart",
                "ecr:DeleteRepositoryPolicy",
                "ecr:SetRepositoryPolicy"
            ],
            "Resource": [
                "arn:aws:ecr:$AWS_REGION:$ACCOUNT_ID:repository/tanzu-application-platform/tanzu-java-web-app",
                "arn:aws:ecr:$AWS_REGION:$ACCOUNT_ID:repository/tanzu-application-platform/tanzu-java-web-app-bundle",
                "arn:aws:ecr:$AWS_REGION:$ACCOUNT_ID:repository/tanzu-application-platform/*"
            ],
            "Effect": "Allow",
            "Sid": "TAPEcrWorkloadScoped"
        }
    ]
}
EOF
```

