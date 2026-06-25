# Work Plan: Lambda Container CodeBuild — Service Catalog Product

## Configurations for `product.template.yaml`

### 1. Template Metadata & Parameters

- **AWSTemplateFormatVersion** / **Description**
- **Parameters** to make the product configurable:
  - `ProjectName` — CodeBuild project name → `aws-nuke-container`
  - `ECRRepositoryName` — target ECR repo for the built image → `aws-nuke`
  - `ComputeType` — CodeBuild compute type → `BUILD_GENERAL1_SMALL` (2 vCPU, 4 GiB)
  - `BuildImage` — CodeBuild managed image (e.g., `aws/codebuild/amazonlinux2-x86_64-standard:6.0`)
  - `BuildTimeout` — timeout in minutes → `60`
  - `LambdaFunctionName` — (optional) Lambda function to update after build
  - `EnvironmentVariables` — any extra env vars for the build
  - `AwsNukeVersion` — version of aws-nuke to download; also used as the container image tag → `v3.65.0`

> **Note:** No source repository parameters (`SourceRepository`, `SourceBranch`) are needed. The Dockerfile, bootstrap, and buildspec are not sourced from an external repo. The buildspec will be defined **inline** within the CodeBuild project resource.

### 2. IAM Role for CodeBuild

- **CodeBuildServiceRole** (`AWS::IAM::Role`):
  - Trust policy allowing `codebuild.amazonaws.com`
  - Compute type: EC2
  - Permissions for:
    - ECR: `GetAuthorizationToken`, `BatchCheckLayerAvailability`, `PutImage`, `InitiateLayerUpload`, `UploadLayerPart`, `CompleteLayerUpload`
    - CloudWatch Logs: `CreateLogGroup`, `CreateLogStream`, `PutLogEvents`
    - S3: `GetObject` — to download aws-nuke package
    - SSM Parameter Store: `GetParameter`, `GetParameters` — to retrieve build configuration/secrets
    - Lambda: `UpdateFunctionCode` (if auto-deploying)

### 3. ECR Repository

- **ECRRepository** (`AWS::ECR::Repository`):
  - Repository name → `aws-nuke`
  - Image scanning configuration → default (scan on push)
  - Image tag mutability → `MUTABLE`
  - Lifecycle policy (to limit untagged image retention)
  - Encryption configuration → default (AES-256)
  - Repository policy → allow `lambda.amazonaws.com` to pull images

### 4. CodeBuild Project

- **CodeBuildProject** (`AWS::CodeBuild::Project`):
  - **Source**: type `NO_SOURCE`, buildspec inline
  - **Environment**:
    - Type: `LINUX_CONTAINER`
    - ComputeType
    - Image (managed or custom)
    - `PrivilegedMode: true` (required for Docker builds)
    - Environment variables (ECR URI, AWS account ID, region, image tag)
  - **Artifacts**: `NO_ARTIFACTS` (since output goes to ECR)
  - **ServiceRole**: reference to the IAM role
  - **LogsConfig**: CloudWatch log group/stream
  - **TimeoutInMinutes**
  - **VpcConfig** (optional — if source or base images are in a VPC)
  - **Cache** (optional — local Docker layer caching for faster builds)

### 5. Buildspec Configuration (inline in CloudFormation template)

The buildspec is defined inline in the CodeBuild project's `BuildSpec` property. It generates the `Dockerfile` and `bootstrap` script dynamically using heredocs, then builds and pushes the container image.

Build phases:
- **pre_build**:
  - ECR login (`aws ecr get-login-password`)
  - Generate `Dockerfile` via heredoc
  - Generate `bootstrap` script via heredoc
  - Download aws-nuke package from S3
- **build**: `docker build -t $ECR_URI:$AWS_NUKE_VERSION .`
- **post_build**: `docker push $ECR_URI:$AWS_NUKE_VERSION`, optionally `aws lambda update-function-code`

> **Note:** Use quoted heredocs (`<<'EOF'`) to avoid variable expansion issues inside file content.

### 6. Supporting Resources

- **CloudWatch Log Group** (`AWS::Logs::LogGroup`) — with retention policy
- **S3 Bucket** (`AWS::S3::Bucket`) — stores the aws-nuke package
- **Custom Resource (crhelper)** — Lambda-backed custom resource that downloads aws-nuke from GitHub releases and uploads to the S3 bucket on stack create/update
- **Lambda Function for Custom Resource** (`AWS::Lambda::Function`) — runs the crhelper logic (download aws-nuke, upload to S3)
- **IAM Role for Custom Resource Lambda** — permissions for S3 PutObject and internet access (to download from GitHub)
- **SNS Topic** — (optional) for build notifications (success/failure)
- **Lambda Permission** — (optional) if CodeBuild will invoke Lambda update

### 7. Outputs

- CodeBuild project name/ARN
- ECR repository URI
- IAM role ARN
- CloudWatch log group name

---

## Key Design Decisions

| Decision | Options |
|----------|---------|
| Source type | No external repo — inline buildspec, no Dockerfile source |
| Trigger mechanism | Manual — start CodeBuild after successful stack update |
| Image tagging | Derived from `AwsNukeVersion` parameter (e.g., `v2.26.0`) |
| Auto-deploy to Lambda | Yes (add `update-function-code`) or No (just push to ECR) |
| VPC access | Needed if pulling private base images or accessing internal resources |
| Multi-arch builds | Single platform or `docker buildx` for arm64 + x86_64 |

---

## Testing Strategy

Deploy the template directly via CloudFormation (bypasses Service Catalog overhead for rapid iteration):

```bash
aws cloudformation deploy \
  --template-file product.template.yaml \
  --stack-name test-lambda-container-codebuild \
  --parameter-overrides ProjectName=aws-nuke-container AwsNukeVersion=v3.65.0 ECRRepositoryName=aws-nuke \
  --capabilities CAPABILITY_IAM
```

**Test workflow:**
1. Deploy stack → verify resources (S3 bucket, ECR repo, CodeBuild project, custom resource success)
2. Manually trigger CodeBuild → verify image built and pushed to ECR with correct tag
3. Iterate on template fixes → redeploy (`aws cloudformation deploy` handles updates)
4. Once stable → publish as Service Catalog product

---

## Implementation Checklist

| # | Section | Item | Status |
|---|---------|------|--------|
| 1 | Template Metadata & Parameters | AWSTemplateFormatVersion & Description | ☐ |
| 1 | Template Metadata & Parameters | ProjectName parameter | ☐ |
| 1 | Template Metadata & Parameters | ECRRepositoryName parameter | ☐ |
| 1 | Template Metadata & Parameters | ComputeType parameter | ☐ |
| 1 | Template Metadata & Parameters | BuildImage parameter | ☐ |
| 1 | Template Metadata & Parameters | BuildTimeout parameter | ☐ |
| 1 | Template Metadata & Parameters | LambdaFunctionName parameter | ☐ |
| 1 | Template Metadata & Parameters | EnvironmentVariables parameter | ☐ |
| 1 | Template Metadata & Parameters | AwsNukeVersion parameter | ☐ |
| 2 | IAM Role for CodeBuild | CodeBuildServiceRole resource | ☐ |
| 2 | IAM Role for CodeBuild | Trust policy for codebuild.amazonaws.com | ☐ |
| 2 | IAM Role for CodeBuild | ECR permissions | ☐ |
| 2 | IAM Role for CodeBuild | CloudWatch Logs permissions | ☐ |
| 2 | IAM Role for CodeBuild | S3 GetObject (download aws-nuke package) | ☐ |
| 2 | IAM Role for CodeBuild | SSM Parameter Store permissions | ☐ |
| 2 | IAM Role for CodeBuild | Lambda UpdateFunctionCode permission (optional) | ☐ |
| 3 | ECR Repository | ECRRepository resource | ☐ |
| 3 | ECR Repository | Image scanning configuration | ☐ |
| 3 | ECR Repository | Lifecycle policy | ☐ |
| 3 | ECR Repository | Encryption configuration | ☐ |
| 3 | ECR Repository | Repository policy | ☐ |
| 4 | CodeBuild Project | CodeBuildProject resource | ☐ |
| 4 | CodeBuild Project | Source configuration | ☐ |
| 4 | CodeBuild Project | Environment (type, compute, image) | ☐ |
| 4 | CodeBuild Project | PrivilegedMode: true | ☐ |
| 4 | CodeBuild Project | Environment variables | ☐ |
| 4 | CodeBuild Project | Artifacts: NO_ARTIFACTS | ☐ |
| 4 | CodeBuild Project | ServiceRole reference | ☐ |
| 4 | CodeBuild Project | LogsConfig | ☐ |
| 4 | CodeBuild Project | TimeoutInMinutes | ☐ |
| 4 | CodeBuild Project | VpcConfig (optional) | ☐ |
| 4 | CodeBuild Project | Cache configuration (optional) | ☐ |
| 5 | Buildspec Configuration | pre_build: ECR login | ☐ |
| 5 | Buildspec Configuration | pre_build: Generate Dockerfile via heredoc | ☐ |
| 5 | Buildspec Configuration | pre_build: Generate bootstrap script via heredoc | ☐ |
| 5 | Buildspec Configuration | pre_build: Download aws-nuke from S3 | ☐ |
| 5 | Buildspec Configuration | build: docker build | ☐ |
| 5 | Buildspec Configuration | post_build: docker push | ☐ |
| 5 | Buildspec Configuration | post_build: Lambda update (optional) | ☐ |
| 6 | Supporting Resources | CloudWatch Log Group | ☐ |
| 6 | Supporting Resources | S3 Bucket for aws-nuke package | ☐ |
| 6 | Supporting Resources | Custom Resource Lambda (crhelper) | ☐ |
| 6 | Supporting Resources | IAM Role for Custom Resource Lambda | ☐ |
| 6 | Supporting Resources | SNS Topic for notifications (optional) | ☐ |
| 6 | Supporting Resources | Lambda Permission (optional) | ☐ |
| 7 | Outputs | CodeBuild project name/ARN | ☐ |
| 7 | Outputs | ECR repository URI | ☐ |
| 7 | Outputs | IAM role ARN | ☐ |
| 7 | Outputs | CloudWatch log group name | ☐ |
