# ECR images cleaner github action

A github action to help with the cleanup of old ECR images, supporting some parametrisation.

## Setup

To set this up, you require an IAM user with access to the ECR, as well as credentials to authenticate to the 
kubernetes cluster and namespace of your interest, usually `production`.

As a pre-requisite, you will need the following secrets setup in your GitHub project:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `KUBE_PROD_CERT`
- `KUBE_PROD_TOKEN`
- `KUBE_PROD_CLUSTER`
- `KUBE_PROD_NAMESPACE`

Your secrets may have other names, it doesn't matter, as you will pass them when using the action in your workflow.

Also, if instead of your production namespace, you prefer to use staging, or any other, the secrets must correspond to that 
namespace.

## ReplicaSet and kubernetes cluster

The `KUBE_*` secrets are needed because this action will retrieve your deployment replica set, in order to protect (i.e. 
do not delete) any image part of the replica set history, including those images currently deployed and active.

So for example if you have a deployment with `revisionHistoryLimit: 5` in their spec, this action will protect up to 
6 image tags (the one in the currently active deployment plus the previous 5).  
These images will not be deleted from the ECR even if they are older than the `days-to-keep-old-images`.

## Inputs

The following inputs can be passed to the action. Some are mandatory, some are optional.

- `aws-access-key-id` - Required - Access key for IAM User, needed to access the ECR

- `aws-secret-access-key` - Required - Secret access key for IAM User, needed to access the ECR

- `kube-cert` - Required - Credentials to authenticate to the kubernetes cluster

- `kube-token` - Required - Credentials to authenticate to the kubernetes cluster

- `kube-cluster` - Required - Kubernetes cluster

- `kube-namespace` - Required - Namespace to retrieve the replica sets

- `ecr-repo-name` - Required - Name of the ECR repository, usually is **team-name/app-name**

- `additional-tags-regex` - Optional - Additional image tags that should not be deleted (regex). Default: `.*main.*|.*latest.*`

- `days-to-keep-old-images` - Optional - Number of days to keep images, older will be purge (unless in use or in buffer). **Default: 30**.

- `max-old-images-to-keep` - Optional - Number of images to keep even if they are older than the cutoff date. **Default: 25**.

## Example of use

```yaml
- name: Run ECR cleanup script
  uses: ministryofjustice/ecr-images-cleaner-action@v1.0.0
  with:
    aws-access-key-id: ${{ secrets.ECR_AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.ECR_AWS_SECRET_ACCESS_KEY }}
    kube-cert: ${{ secrets.KUBE_PROD_CERT }}
    kube-token: ${{ secrets.KUBE_PROD_TOKEN }}
    kube-cluster: ${{ secrets.KUBE_PROD_CLUSTER }}
    kube-namespace: ${{ secrets.KUBE_PROD_NAMESPACE }}
    ecr-repo-name: family-justice/disclosure-checker
    max-old-images-to-keep: 75
```

Then you can use this job on a workflow that runs on a schedule, or with `workflow_dispatch` to run it manually.
