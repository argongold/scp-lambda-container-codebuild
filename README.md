# Lambda Container CodeBuild — Service Catalog Product

A CloudFormation template (Service Catalog product) that provisions an AWS CodeBuild project to build a Lambda container image with [aws-nuke](https://github.com/ekristen/aws-nuke) and push it to Amazon ECR.

## Architecture

```
CloudFormation Stack
├── S3 Bucket          (stores aws-nuke compressed package)
├── ECR Repository     (stores built container images)
├── CodeBuild Project  (builds Docker image with inline buildspec)
├── IAM Role           (CodeBuild service role)
└── CloudWatch Logs    (build logs)
```

## How It Works

1. The template creates all infrastructure (S3, ECR, CodeBuild, IAM, Logs)
2. Upload the aws-nuke release tarball to S3: `s3://<bucket>/<version>/aws-nuke-<version>-linux-amd64.tar.gz`
3. Trigger CodeBuild manually — it downloads aws-nuke from S3, builds a Lambda container image, and pushes to ECR
4. The container image is tagged with the aws-nuke version (e.g., `v3.65.0`)

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ProjectName` | `aws-nuke-container` | CodeBuild project name |
| `ECRRepositoryName` | `aws-nuke` | ECR repository name |
| `ComputeType` | `BUILD_GENERAL1_SMALL` | CodeBuild compute (2 vCPU, 4 GiB) |
| `BuildImage` | `aws/codebuild/amazonlinux2-x86_64-standard:6.0` | CodeBuild managed image |
| `BuildTimeout` | `60` | Timeout in minutes |
| `AwsNukeVersion` | `v3.65.0` | aws-nuke version (also used as image tag) |
| `ServiceCatalogAccountId` | *(required)* | AWS Account ID (used for S3 bucket naming) |

## Deployment

### Deploy via CloudFormation (testing)

```bash
aws cloudformation deploy \
  --template-file product.template.yaml \
  --stack-name test-lambda-container-codebuild \
  --parameter-overrides ProjectName=aws-nuke-container AwsNukeVersion=v3.65.0 ECRRepositoryName=aws-nuke ServiceCatalogAccountId=123456789012 \
  --capabilities CAPABILITY_IAM
```

### Upload aws-nuke package to S3

```bash
aws s3 cp aws-nuke-v3.65.0-linux-amd64.tar.gz s3://123456789012-aws-nuke-artifacts/v3.65.0/aws-nuke-v3.65.0-linux-amd64.tar.gz
```

### Trigger CodeBuild

```bash
aws codebuild start-build --project-name aws-nuke-container
```

## Updating aws-nuke Version

1. Update the `AwsNukeVersion` parameter and redeploy the stack
2. Upload the new tarball to S3: `s3://<bucket>/<new-version>/aws-nuke-<new-version>-linux-amd64.tar.gz`
3. Trigger CodeBuild — the new image will be tagged with the new version

## Outputs

| Output | Description |
|--------|-------------|
| `CodeBuildProjectName` | CodeBuild project name |
| `CodeBuildProjectArn` | CodeBuild project ARN |
| `ECRRepositoryUri` | ECR repository URI |
| `S3BucketName` | S3 bucket for aws-nuke package |
| `IAMRoleArn` | CodeBuild service role ARN |
| `CloudWatchLogGroupName` | Build logs location |
