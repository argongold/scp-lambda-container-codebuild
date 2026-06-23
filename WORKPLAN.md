# Work Plan: Lambda Container CodeBuild — Service Catalog Product

## Configurations for `product.template.yaml`

### 1. Template Metadata & Parameters

- **AWSTemplateFormatVersion** / **Description**
- **Parameters** to make the product configurable:
  - `ProjectName` — CodeBuild project name
  - `SourceRepository` — Git repo URL (CodeCommit or GitHub) containing the Dockerfile
  - `SourceBranch` — branch to build from
  - `ECRRepositoryName` — target ECR repo for the built image
  - `ImageTag` — tag strategy (e.g., `latest`, commit SHA)
  - `ComputeType` — CodeBuild compute size (`BUILD_GENERAL1_SMALL`, `MEDIUM`, `LARGE`)
  - `BuildImage` — CodeBuild managed image (e.g., `aws/codebuild/amazonlinux2-x86_64-standard:5.0`)
  - `BuildTimeout` — timeout in minutes
  - `LambdaFunctionName` — (optional) Lambda function to update after build
  - `EnvironmentVariables` — any extra env vars for the build

### 2. IAM Role for CodeBuild

- **CodeBuildServiceRole** (`AWS::IAM::Role`):
  - Trust policy allowing `codebuild.amazonaws.com`
  - Permissions for:
    - ECR: `GetAuthorizationToken`, `BatchCheckLayerAvailability`, `PutImage`, `InitiateLayerUpload`, `UploadLayerPart`, `CompleteLayerUpload`
    - CloudWatch Logs: `CreateLogGroup`, `CreateLogStream`, `PutLogEvents`
    - S3: access to artifacts bucket (if needed)
    - CodeCommit or connection: `GitPull` permissions
    - Lambda: `UpdateFunctionCode` (if auto-deploying)
    - (Optional) Secrets Manager / SSM for build secrets

### 3. ECR Repository

- **ECRRepository** (`AWS::ECR::Repository`):
  - Repository name
  - Image scanning configuration
  - Lifecycle policy (to limit untagged image retention)
  - Encryption configuration
  - Repository policy (who can pull the image)

### 4. CodeBuild Project

- **CodeBuildProject** (`AWS::CodeBuild::Project`):
  - **Source**: type (CODECOMMIT, GITHUB, S3), location, buildspec (inline or file reference)
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

### 5. Buildspec Configuration (inline or referenced)

Define the build phases:
- **pre_build**: ECR login (`aws ecr get-login-password`)
- **build**: `docker build -t $IMAGE_URI .`
- **post_build**: `docker push $IMAGE_URI`, optionally `aws lambda update-function-code`

### 6. (Optional) Supporting Resources

- **CloudWatch Log Group** (`AWS::Logs::LogGroup`) — with retention policy
- **S3 Bucket** — if using S3 source or caching
- **EventBridge Rule / Trigger** — if you want builds triggered on code push
- **SNS Topic** — for build notifications (success/failure)
- **Lambda Permission** — if CodeBuild will invoke Lambda update

### 7. Outputs

- CodeBuild project name/ARN
- ECR repository URI
- IAM role ARN
- CloudWatch log group name

---

## Key Design Decisions

| Decision | Options |
|----------|---------|
| Source type | CodeCommit vs GitHub (via connection) vs S3 |
| Trigger mechanism | Manual, EventBridge on push, or scheduled |
| Image tagging | `latest` only, commit SHA, semantic versioning |
| Auto-deploy to Lambda | Yes (add `update-function-code`) or No (just push to ECR) |
| VPC access | Needed if pulling private base images or accessing internal resources |
| Multi-arch builds | Single platform or `docker buildx` for arm64 + x86_64 |

---

## Implementation Checklist

| # | Section | Item | Status |
|---|---------|------|--------|
| 1 | Template Metadata & Parameters | AWSTemplateFormatVersion & Description | ☐ |
| 1 | Template Metadata & Parameters | ProjectName parameter | ☐ |
| 1 | Template Metadata & Parameters | SourceRepository parameter | ☐ |
| 1 | Template Metadata & Parameters | SourceBranch parameter | ☐ |
| 1 | Template Metadata & Parameters | ECRRepositoryName parameter | ☐ |
| 1 | Template Metadata & Parameters | ImageTag parameter | ☐ |
| 1 | Template Metadata & Parameters | ComputeType parameter | ☐ |
| 1 | Template Metadata & Parameters | BuildImage parameter | ☐ |
| 1 | Template Metadata & Parameters | BuildTimeout parameter | ☐ |
| 1 | Template Metadata & Parameters | LambdaFunctionName parameter | ☐ |
| 1 | Template Metadata & Parameters | EnvironmentVariables parameter | ☐ |
| 2 | IAM Role for CodeBuild | CodeBuildServiceRole resource | ☐ |
| 2 | IAM Role for CodeBuild | Trust policy for codebuild.amazonaws.com | ☐ |
| 2 | IAM Role for CodeBuild | ECR permissions | ☐ |
| 2 | IAM Role for CodeBuild | CloudWatch Logs permissions | ☐ |
| 2 | IAM Role for CodeBuild | S3 permissions (if needed) | ☐ |
| 2 | IAM Role for CodeBuild | Source repo permissions (CodeCommit/GitHub) | ☐ |
| 2 | IAM Role for CodeBuild | Lambda UpdateFunctionCode permission (optional) | ☐ |
| 2 | IAM Role for CodeBuild | Secrets Manager / SSM permissions (optional) | ☐ |
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
| 5 | Buildspec Configuration | build: docker build | ☐ |
| 5 | Buildspec Configuration | post_build: docker push | ☐ |
| 5 | Buildspec Configuration | post_build: Lambda update (optional) | ☐ |
| 6 | Supporting Resources | CloudWatch Log Group | ☐ |
| 6 | Supporting Resources | S3 Bucket (optional) | ☐ |
| 6 | Supporting Resources | EventBridge Rule / Trigger (optional) | ☐ |
| 6 | Supporting Resources | SNS Topic for notifications (optional) | ☐ |
| 6 | Supporting Resources | Lambda Permission (optional) | ☐ |
| 7 | Outputs | CodeBuild project name/ARN | ☐ |
| 7 | Outputs | ECR repository URI | ☐ |
| 7 | Outputs | IAM role ARN | ☐ |
| 7 | Outputs | CloudWatch log group name | ☐ |
