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

You'll also want to ensure that you accept all the required [End User Licenses Agreements](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-install-tanzu-cli.html#accept-the-end-user-license-agreements-0).

## AWS CLI

This walkthrough will use the AWS CLI to both query and configure resources in AWS such as IAM roles.

To install the AWS CLI, use the [following guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

## EKSCTL

The EKSCTL command line makes managing the lifecycle of EKS clusters very simple.  This guide will use it to create clusters.

To install the eksctl cli, use the [following guide](https://eksctl.io/introduction/#installation).

## Tanzu CLI

The Tanzu CLI will enable us to install and interact with the Tanzu Application Platform.

To install the Tanzu, use the [following guide](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-install-tanzu-cli.html#cli-and-plugin)

# Export Environment Variables

Some variables are used throughout the guide, to simplify the process and minimize the opportunity for errors, let's export these variables.

```
export ACCOUNT_ID=012345678901
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
eksctl create cluster --name tap-on-aws --managed --region $AWS_REGION --instance-types t3.large --version 1.22 --with-oidc -N 5
```

Note: This step is optional if you already have an existing EKS Cluster.  Note, however that it has to be at least version 1.22 and have the OIDC authentication enabled.  To enable the OIDC provider, use the [following guide](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

# Create the Container Repositories

ECR requires the container repositories to be precreated.   While it is [recommended](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-install.html#relocate-images-to-a-registry-0) to replicate the images for Tanzu Application Platform, for this guide will use images hosted on the Tanzu Network for simplicity.

As part of the install process, we only need a repository for the Tanzu Build Service images.  To create a repository, run the following command:

```
aws ecr create-repository --repository-name tap-build-service --region $AWS_REGION
```

# Create IAM Roles

By default, the EKS cluster will be provisioned with an [EC2 instance profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) that provides read-only access for the entire EKS cluster to the ECR registery within your AWS account.

However, some of the services within the Tanzu Application Platform require write access to the container repositories. In order to provide that access, we will create IAM roles and add the ARN to the K8S service accounts that those services use.  This will ensure that only the required services have access to write container images to ECR, rather than a blanket policy that applies to the entire cluster.

We'll create two IAM Roles.

1.  Tanzu Build Service: Gives write access to the repository to allow the service to automatically upload new images.  This is limited in scope to the service account for kpack and the dependency updater.
2.  Workload:  Gives write access to the entire ECR registry.  This prevents you from having to update the policy for each new workload created.  For those looking for a more strict model, we'll include examples for the sample workload deployed later in the guide.

In order to create the roles, we need to establish two policys:

1. Trust Policy: Limits the scope to the OIDC endpoint for the Kubernetes Cluster and the Kubernetes service account we'll attach the role to
1. Permission Policy:  Limits the scope of what actions can be taken on what resources by the role.  

**Note** Both of these policies are best effort at a least privledge model, but be sure to review to ensure if they adhere to your organizations policies.

In order to simplify this guide, we'll use a script to create these policy documents and create the roles.  This script will output the files, then create the IAM roles using the policy documents.

To run the script, do the following:

```
curl -O https://raw.githubusercontent.com/ryan-a-baker/tap-on-aws/master/create-iam-roles.sh
chmod u+x create-iam-roles.sh
./create-iam-roles.sh
```

# Tanzu Application Platform Pre-Requesites

Before we install the Tanzu Application Platform, we to install the Cluster Essentials and set up the credentials for the Tanzu Network to be able to pull images.

Follow the [guide](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.2/cluster-essentials/GUID-deploy.html) to install cluster essentials to the EKS Cluster.

Next, let's install the Tanzu Application Platform package.  First, let's set the variables to use:

```
export INSTALL_REGISTRY_USERNAME=<tanzu-net-username>
export INSTALL_REGISTRY_PASSWORD=<tanzu-net-password>
export TAP_VERSION=1.2.0
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
```

Then we create the tap-install namepspace and add a secret to be able to authenticate to Tanzu Network to install the Tanzu Application platform.

```
kubectl create namespace tap-install
tanzu secret registry add tap-registry \
  --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
  --server ${INSTALL_REGISTRY_HOSTNAME} \
  --export-to-all-namespaces --yes --namespace tap-install
```

Finally, install the package

```
tanzu package repository add tanzu-tap-repository \
  --url ${INSTALL_REGISTRY_HOSTNAME}/tanzu-application-platform/tap-packages:$TAP_VERSION \
  --namespace tap-install
```

# Build Configuration for the Tanzu Application Platform

Before we install the Tanzu Application Platform, we need to generate the configuration to use to install it.  

Let's build the configuration file and populate it with variables.

```
cat << EOF > aws-values.yaml
profile: full
ceip_policy_disclosed: true

buildservice:
  kp_default_repository: ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/tap-build-service
  # Specify the ARN created earlier for the build service
  kp_default_repository_aws_iam_role_arn: "arn:aws:iam::${ACCOUNT_ID}:role/tap-build-service"

ootb_templates:
  # Allow the config writer service to use cloud based iaas authentication
  iaas_auth: true

supply_chain: testing

ootb_supply_chain_testing:
  registry:
    server: ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
    # The prefix of the ECR repository
    repository: tanzu-application-platform
  gitops:
    ssh_secret: ""

learningcenter:
  ingressDomain: tap-on-aws.com

contour:
  envoy:
    service:
      type: LoadBalancer

cnrs:
  domain_name: tap-on-aws.com

tap_gui:
  service_type: LoadBalancer
  app_config:
    app:
      baseUrl: http://tap-gui.tap-on-aws.com:7000
    catalog:
      locations:
        - type: url
          target: https://GIT-CATALOG-URL/catalog-info.yaml
    backend:
      baseUrl: http://tap-gui.tap-on-aws.com:7000
      cors:
        origin: http://tap-gui.tap-on-aws.com:7000

metadata_store:
  ns_for_export_app_cert: "default"

scanning:
  metadataStore:
    url: "" # Disable embedded integration since it's deprecated
EOF
```

This will output a `aws-values.yaml` file.  You can review it and notice there are no credentials in the file at all.  That's because we're leveraging the IAM roles to authenticate to ECR.

# Install TAP

Once the values file is generated, we can use it to install the Tanzu Application Platform.  The following command will do it:

```
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION -n tap-install --values-file aws-values.yaml
```

This will take 20-30 minutes.  Once completed, you will have the Tanzu Application Platform installed!

# Creating your first workload

Now that we have the platform installed, let's create the first workload!

Before we create it, we need to configure our "developer namespace".  This is outlined in the [documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-set-up-namespaces.html).  Since we are using IAM roles for our container registry, we do not need to set up registry secrets.  Instead, we'll specify the ARN of the role to use to authenticate to ECR.

To save some time, here's a developer namespace configuration we can apply that will automatically generate the ARN of the role:

```
cat <<EOF | kubectl -n default apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${ACCOUNT_ID}:role/tap-workload
imagePullSecrets:
  - name: tap-registry
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-permit-deliverable
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: deliverable
subjects:
  - kind: ServiceAccount
    name: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-permit-workload
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: workload
subjects:
  - kind: ServiceAccount
    name: default
EOF
```

Since we used the testing supply chain, we need to define the Tekton pipeline to use to test our sample application.  Use the following [documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-getting-started-add-test-and-security.html#tekton-pipeline-config-example-3) to do so.

Finally - we have to create repositories for our application, because ECR requires repositories to be created before pushing an image.  We'll create repository for both the image, as well as the bundle that's used to deploy the workload.

```
aws ecr create-repository --repository-name tanzu-application-platform/tanzu-java-web-app-default --region $AWS_REGION
aws ecr create-repository --repository-name tanzu-application-platform/tanzu-java-web-app-default-bundle --region $AWS_REGION
```

Now that we have our developer workspace setup, you can deploy your workload:

```
tanzu apps workload create tanzu-java-web-app \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
  --git-branch main \
  --type web \
  --label apps.tanzu.vmware.com/has-tests=true \
  --yes
  ```

