# ECR images cleaner github action

A github action to help with the cleanup of old ECR images, supporting some parametrisation.

The action will call a bash script that performs the following:

First, it will retrieve all images declared in the replica set history for the `kube-namespace` input provided.   
This will usually be production as these are the images that you would want to protect from being deleted.

So for example if you have a deployment with `revisionHistoryLimit: 5` in their spec, this action will protect up to 
6 image tags (the one in the currently active deployment plus the previous 5), no matter how old they are.

Then it will get the details of all images stored in the ECR, filtering out the above replica set images, any image 
that matches the regex `additional-tags-regex` and any image pushed less than `days-to-keep-old-images` ago.  
The default for this variable is 30 days but can be configured.

Finally, we sort the resulting images by their `imagePushedAt` date, from older to newer, and proceed to apply a 
safety buffer `max-old-images-to-keep` that we will maintain, even knowing that there are old images.  

So for example if the resulting set of images to delete is 75 after applying all the above filters, without the buffer 
we would delete all of them. With a buffer of 25 (this is the default value), we will only delete the oldest 50, 
and leave the most recent 25 images. This buffer can be configured too.

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
  uses: ministryofjustice/ecr-images-cleaner-action@v1.0.1
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
