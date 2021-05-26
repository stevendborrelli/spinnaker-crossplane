# spinnaker-crossplane

This Repository Demonstration of Spinnaker and Universal Crossplane

## Getting Started

### Requirements

Please Download the following utilities:

* The [AWS Cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* (Optional) [Kind](https://kind.sigs.k8s.io/docs/user/quick-start)
* The [up] CLI
* The [Crossplane Kubectl Plugin](https://crossplane.io/docs/v1.2/getting-started/install-configure.html#install-crossplane-cli)
* [Helm](https://helm.sh/docs/intro/install/)

## Creating a deployment Cluster

We will be deploying 2 EKS clusters. While you can combine Spinnaker and Crossplane
on a single cluster, most organizations will have their CI and Infrastructure
deployments managed on separate Cluster. 

1. An EKS cluster running Spinnaker
2. An optional EKS cluster running Universal Crossplane


## Creating the Spinnaker Cluster using Crossplane

Since Crossplane is effective at managing infrstructure
We can use a custom composition to create the AWS Components needed to run 
Spinnaker on Kubernetes:

* VPC
* S3 Bucket
* EKS Cluster

Crossplane can run on any Kubernetes Cluster, in this example we use [Kind](https://kind.sigs.k8s.io/docs/user/quick-start) to create and manage the environment.

### Creating the Deployment Cluster

Any Kubernetes cluster can run Crossplane and provision resources. For
this demonstration, we will be using Kind that can run on your laptop
or workstation:

```shell
kind create cluster --name spinnaker-demo
```

## Installing Universal Crossplane (UXP) to your Deployment Cluster

Next we will install [Universal Crosslpane](TODO://LINK),
an open-source compliant distribution of Crossplane supported by Upbound
and is available via the [AWS marketplace](https://aws.amazon.com/marketplace/pp/prodview-uhc2iwi5xysoc). The subscription is free, and enterprise support
is available from [Upbound](TODO://upbound support).

Click on the Accept Terms.

```shell
up uxp install
```

You can check the state of the upbound deployment

```shell
kubectl get all -n upbound-system
```

### TODO: Install the configuration

The Spinnaker Cluster has been packaged as a Crossplane Composition

```shell
kubectl crossplane install configuration TODO
```
For now we will install the AWS provider.

```shell
kubectl crossplane install provider crossplane/provider-aws:v0.18.1
```

### Set up the AWS Provider

Ensure that your have your `aws` installation set up correctly by running the 
following command:

```shell
aws configure
```

Once the `aws` CLI is configured, set up a credentias secret for Crossplane to use for
provisioning and managing AWS resources:

```shell
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > creds.conf

```

The `creds.conf` file should look like:

```ini
[default]
aws_access_key_id = <id redacted>
aws_secret_access_key = <key redacted>
```

Create the provider secret:

```shell
kubectl create secret generic aws-creds -n upbound-system --from-file=creds=./creds.conf
```

Finaly, configure the `ProviderConfiguration`. We will configure a `default` provider that
the AWS Crossplane controller will use to authenticate to AWS. Note that the ProviderConfiguration
references the `aws-creds` scecret we created in the previous step.

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

```shell
kubectl apply -f examples/providerconfig.yaml
```


## Cleaning Up


Finally, delete the kind cluster used to provison the AWS environment:

```shell
kind delete cluster --name spinnaker-demo
```
