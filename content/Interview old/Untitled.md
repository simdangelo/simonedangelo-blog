---
modified: 2026-01-18T19:31:05+01:00
---

# Cloud Provider: AWS (Amazon Web Services)
This project is entirely built on AWS infrastructure, deployed in the **eu-central-1** (Frankfurt) region.
## Cloud Services Used
### Compute Services
| Service                  | Usage                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **AWS Lambda**           | Serverless functions for simulation tasks, image updates, and background jobs. Uses container images from ECR (simulation.lambda.dockerfile) |
| **Amazon ECS (Fargate)** | Hosts the Streamlit frontend application as a containerized service (frontend.py)                                                            |
| **ECS Tasks**            | Long-running simulation jobs via standalone ECS tasks (simulation.ecs.dockerfile)                                                            |

### Storage Services
| Service             | Usage                                                                                                                                              |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Amazon S3**       | Primary data storage for datasets, simulation results, and intermediate files. Uses encrypted buckets with KMS (storage.py, simulation_storage.py) |
| **Amazon DynamoDB** | Stores simulation run metadata and job tracking (delete_simulations.py)                                                                            |
### Analytics & Query Services
| Service           | Usage                                                                             |
| ----------------- | --------------------------------------------------------------------------------- |
| **Amazon Athena** | SQL queries over S3 data using Parquet format with SNAPPY compression (athena.py) |
| **AWS Glue**      | Data catalog for Athena tables (referenced via `aws_glue_alpha` in CDK)           |
### Networking & Security
| Service                             | Usage                                                                                 |
| ----------------------------------- | ------------------------------------------------------------------------------------- |
| **Amazon VPC**                      | Private networking with multiple subnets across availability zones (cdk.context.json) |
| **Application Load Balancer (ALB)** | Routes traffic to Fargate frontend with TLS certificates                              |
| **AWS Certificate Manager (ACM)**   | TLS certificates per environment (cdk.json)                                           |
| **Security Groups**                 | Network-level access control                                                          |
| **AWS KMS**                         | Encryption keys for S3, Athena results                                                |
### Identity & Access Management
| Service                     | Usage                                                                                  |
| --------------------------- | -------------------------------------------------------------------------------------- |
| **AWS IAM**                 | Roles for ECS tasks, Lambda functions, cross-account access (roles.py)                 |
| **AWS STS**                 | Role assumption for cross-account deployments and EC2 role assumption (assume_role.py) |
| **AWS SSM Parameter Store** | Configuration storage for project parameters (parameter-devops-mercato.json)           |
### Container Services
| Service            | Usage                                                                          |
| ------------------ | ------------------------------------------------------------------------------ |
| **Amazon ECR**     | Container image registry with lifecycle policies (ecr.py)                      |
| **ECR Deployment** | Custom CDK construct for automated image deployment using `cdk-ecr-deployment` |
### Messaging & Notifications
| Service        | Usage                                        |
| -------------- | -------------------------------------------- |
| **Amazon SNS** | Failure notifications and alerts (shared.py) |
### CI/CD Services
| Service              | Usage                                                                      |
| -------------------- | -------------------------------------------------------------------------- |
| **AWS CodeBuild**    | Build and deployment pipelines (buildspec-synth.yml, buildspec-deploy.yml) |
| **AWS CodePipeline** | Release orchestration for versioned deployments                            |
| **AWS CodeArtifact** | Private Python/npm package registry                                        |
### Monitoring
| Service                    | Usage                                       |
| -------------------------- | ------------------------------------------- |
| **Amazon CloudWatch Logs** | Log aggregation for Lambda, ECS, and Athena |
| **CloudWatch Metrics**     | Athena workgroup metrics publishing         |
## Infrastructure as Code (IaC)
### AWS CDK (Cloud Development Kit)
The entire infrastructure is defined using **AWS CDK v2** with Python bindings.
**Stack Structure** (app.py):
```

┌─────────────────────────────────────────────────────────────┐
│                        CDK App                               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │   athena    │  │  frontend   │  │      roles       │    │
│  └─────────────┘  └─────────────┘  └──────────────────┘    │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │   shared    │  │  storage    │  │ simulation-storage│    │
│  └─────────────┘  └─────────────┘  └──────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    simulation                        │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Stack Dependencies**:
```python
stacks["frontend"].add_dependency(stacks["roles"])
stacks["simulation"].add_dependency(stacks["roles"])
stacks["simulation"].add_dependency(stacks["shared"])
```
### Custom CDK Constructs
**`AutoDeployingCachedECR`** - A reusable construct that:
1. Creates an ECR repository with lifecycle rules
2. Builds Docker images using BuildKit with registry caching
3. Automatically deploys images to ECR on stack deployment
4. Manages IAM policies for CDK's image publishing role
```python
class AutoDeployingCachedECR(Construct):
    def create(self) -> None:
        self.create_ecr_repository()
        self.create_cdk_image_deploy_policy()
        self.create_docker_image_asset()
        self.create_docker_image_deploy_and_dependency_chain()
```
## Deployment Pipeline
### Multi-Environment Strategy
Three environments with separate AWS accounts:

| Stage | Account ID   | Profile             |
| ----- | ------------ | ------------------- |
| dev   | 732789817868 | `saml-mercato-dev`  |
| test  | 643356557982 | `saml-mercato-test` |
| prod  | 238788445602 | `saml-mercato-prod` |
### CI/CD Flow
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   CodeCommit │────▶│  CodeBuild   │────▶│ CodePipeline │
│   (Source)   │     │  (Synth)     │     │  (Release)   │
└──────────────┘     └──────────────┘     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │  CodeBuild   │
                     │  (Deploy)    │
                     └──────────────┘
```

### Build Phases (codebuild-helpers.sh)
1. **Install Phase**:
   - Install Python/Node.js runtimes
   - Login to CodeArtifact for private packages
   - Install CDK and dependencies
1. **Pre-Build Phase**:
   - Assume cross-account IAM role via STS
   - Login to target account's ECR
1. **Build Phase**:
   - **Synth**: Generate CloudFormation templates
   - **Diff**: Compare with deployed stacks, commit diff file
   - **Deploy**: Build Docker images with BuildKit, deploy stacks
### Docker Build Optimization
Uses **Docker BuildKit** with registry-based caching:
```python
cache_from=[
    aws_ecr_assets.DockerCacheOption(
        type="registry",
        params={"ref": repo_tag_cache_url},
    )
],

cache_to=aws_ecr_assets.DockerCacheOption(
    type="registry",
    params={"ref": repo_tag_cache_url, "mode": "max"},
)
```
### Local Deployment Script
infra.sh enables local CDK operations:
- `--synth`: Generate CloudFormation
- `--deploy`: Deploy to AWS
- `--diff`: Compare changes
- `--release <version>`: Trigger CodePipeline
## Container Architecture
### Three Docker Images
| Image             | Base                                | Purpose                    | Location                     |
| ----------------- | ----------------------------------- | -------------------------- | ---------------------------- |
| Frontend          | `amazonlinux:2023`                  | Streamlit web app          | frontend.dockerfile          |
| Simulation Lambda | `public.ecr.aws/lambda/python:3.11` | Serverless simulation jobs | simulation.lambda.dockerfile |
| Simulation ECS    | `python:3.11`                       | Long-running ECS tasks     | simulation.ecs.dockerfile    |
### Common Build Pattern
All Dockerfiles use BuildKit secrets for CodeArtifact authentication:
```dockerfile
RUN --mount=type=secret,id=CODEARTIFACT_PIP_URL \
    python3.11 -m pip config set \
    global.index-url $(cat /run/secrets/CODEARTIFACT_PIP_URL) \
    && python3.11 -m pip install -r requirements.txt
```
## Database Architecture
### Athena Tables Schema
Tables are defined as SQLAlchemy models with Athena-specific metadata:
```python
class CicloAttivo(TableBaseModel):
    __tablename__ = 'ciclo_attivo'
    __table_args__ = dict(
        schema="cash_flow_simulator_simulations",
        awsathena_location=f'{BUCKET_BASE_PATH_SIMULATIONS}/dataset/ciclo_attivo',
        awsathena_file_format='PARQUET',
        awsathena_compression='SNAPPY'
    )
```
### Athena Workgroup Configuration (athena.py)
```python
work_group_configuration=aws_athena.CfnWorkGroup.WorkGroupConfigurationProperty(
    bytes_scanned_cutoff_per_query=200_000_000_000,  # 200GB limit
    engine_version="Athena engine version 3",
    publish_cloud_watch_metrics_enabled=True, result_configuration=aws_athena.CfnWorkGroup.ResultConfigurationProperty(
    encryption_configuration=aws_athena.CfnWorkGroup.EncryptionConfigurationProperty(
            encryption_option="SSE_KMS",
        ),
    ),
)
```
### Data Flow
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   S3 Input   │────▶│  Simulation  │────▶│  S3 Output   │
│   Bucket     │     │  (Lambda/ECS)│     │   Bucket     │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │    Athena    │
                                          │   (Query)    │
                                          └──────────────┘
```
## Application Architecture
### Frontend Stack (app.py)
**Streamlit** application with modular sections:
```python
def main():
    is_auth = check_auth()
    curr_section = navigation_sidebar.load()
    SECTIONS_MAP[curr_section]["component"].load()
```
### ECS Fargate Configuration (frontend.py)
```python
StreamlitHostingApplicationLoadBalancedFargateService(
    ecs_task_cpu=2048,        # 2 vCPU
    ecs_task_memory_mib=8192,  # 8 GB
    ecs_container_environment={
        "AWS_DEFAULT_REGION": self.region,
        "STAGE": self.stage,
        "AWS_ATHENA_WORKGROUP": "cash-flow-simulator-athena-workgroup",
        "STREAMLIT_SERVER_MAX_UPLOAD_SIZE": '1000',
    },
)
```
### Execution Strategies Pattern
The codebase uses a Strategy pattern for job execution:
```python
class ExecutionStrategy(ABC):
    pass
  
class ForecastExecutionStrategy(ExecutionStrategy):
    pass
  
class ConsuntivazioneExecutionStrategy(ExecutionStrategy):
    pass
```

With runtime selection:
```python
if run_mode == "LOCAL":
    strategy = LocalInvokeStrategy(local_function_name=start_function)
else:
    strategy = LambdaInvokeStrategy(lambda_function_name=ctx.simulation_starter_lambda)
```
## Security Architecture
### Encryption
- **S3**: Server-side encryption with KMS (SSE-KMS)
- **Athena Results**: KMS encrypted
- **ECR**: Default encryption
### IAM Role Strategy
Environment-specific roles with least-privilege:
```python
def get_iren_de_roles(self) -> List[aws_iam.ArnPrincipal]:
    match self.stage:
        case "dev":
            iren_de_role_names = ["saml-adfs-iren-de-dev", "AWSReservedSSO_ADVANA-DataEngineer_..."]
        case "test":
            # Different roles for test
        case "prod":
            # Production roles with restricted access
```
### Cross-Account Access
Uses STS AssumeRole for:
1. CodeBuild deployments to target accounts
2. Local development with EC2 role assumption
```python
def assume_ec2_role():
    if os.environ.get("EC2_ROLE_ARN"):
        sts = boto3.client("sts")
        response = sts.assume_role(RoleArn=role_arn, RoleSessionName="ec2-test-session")
        boto3.setup_default_session(...)
```
## Development Tools & Dependencies
### Infrastructure Dependencies (requirements.cdk.txt)
| Package                  | Version   | Purpose                  |
| ------------------------ | --------- | ------------------------ |
| `aws-cdk-lib`            | 2.143.1   | Core CDK library         |
| `aws-cdk.aws-glue-alpha` | 2.143.1a0 | Glue constructs (alpha)  |
| `cdk-ecr-deployment`     | 4.0.1     | ECR image deployment     |
| `cdk-ec2-key-pair`       | 4.0.1     | EC2 key management       |
| `advanalib`              | 0.5.2     | Internal utility library |
### Application Dependencies (requirements.txt)
| Package          | Purpose                     |
| ---------------- | --------------------------- |
| `streamlit`      | Web UI framework            |
| `boto3/botocore` | AWS SDK                     |
| `awswrangler`    | Pandas integration with AWS |
| `pyathena`       | Athena SQLAlchemy dialect   |
| `sqlalchemy`     | ORM for Athena              |
| `pandas/numpy`   | Data processing             |
| `pyarrow`        | Parquet file handling       |
| `s3fs`           | S3 filesystem interface     |
### Build Tools
| Tool                    | Purpose                                 |
| ----------------------- | --------------------------------------- |
| `uv`                    | Python package management (pip-compile) |
| `npm`                   | CDK CLI installation                    |
| `Docker BuildKit`       | Optimized image builds                  |
| `git-remote-codecommit` | CodeCommit Git helper                   |
## Configuration Management
### Environment Variables (.env.template)
```
AWS_DEFAULT_REGION=eu-central-1
PROJECT_NAME=
RUNNER_API_ENDPOINT=https://...execute-api.../prod
DEPLOY_TYPE=DEV
OVERRIDE_AUTH=ON
```
### SSM Parameters (parameter-devops-mercato.json)
Hierarchical parameter structure:
```
/MERCATO/devops-cash-flow-simulator/
├── notification_email
├── CODE_ARTIFACT_DOMAIN
├── DEV/
│   ├── versions/CDK_VERSION
│   ├── versions/PYTHON_RUNTIME
│   ├── CA_ROLE
│   └── CodeBuild/
├── TEST/
└── PROD/
```
### CDK Context (cdk.json)
```json
{
  "runner_api_endpoint": {
    "dev": "https://8hpmmddo24-vpce-....execute-api.eu-central-1.amazonaws.com/prod",
    "test": "https://...",
    "prod": "https://..."
  },
  "certificate_arn": {
    "dev": "arn:aws:acm:...",
    "test": "arn:aws:acm:...",
    "prod": "arn:aws:acm:..."
  }
}
```
## Network Architecture
### VPC Configuration (cdk.context.json)
- **CIDR Block**: 10.224.12.0/22
- **Subnet Types**: Private subnets only
- **Availability Zones**: eu-central-1a, eu-central-1b, eu-central-1c
- **VPC Endpoints**: Implied by VPCE references in API endpoints
### Connectivity Pattern
```
┌─────────────────────────────────────────────────────────────┐
│                          VPC                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Subnet 1a   │  │  Subnet 1b   │  │  Subnet 1c   │      │
│  │  (Private)   │  │  (Private)   │  │  (Private)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                 │                 │               │
│         └─────────────────┴─────────────────┘               │
│                           │                                  │
│                    ┌──────────────┐                         │
│                    │     ALB      │                         │
│                    └──────────────┘                         │
│                           │                                  │
│                    ┌──────────────┐                         │
│                    │   Fargate    │                         │
│                    └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
          │
          ▼ (VPC Endpoints)
    ┌──────────────┐
    │  S3, Athena  │
    │  DynamoDB    │
    └──────────────┘
```
## Versioning & Release Strategy
### Version-Based Releases
```bash
# Trigger release pipeline for version 1.0.0
./scripts/infra.sh --stage prod --release 1.0.0
```

Pipeline naming: `devops-cash-flow-simulator-MERCATO-PROD-v1-0-0`
### Diff Tracking
CDK diffs are automatically committed to the repository:
```yaml
# buildspec-synth.yml
- git add ./Diff_files/cdk-diff-${BUSINESS_UNIT}-$STAGE.txt
- git commit -m "(CodeBuild) Commit of diff file ($STAGE)"
- git push -u origin ${BRANCH_NAME}
```
  
# Introduction
I'd be happy to walk you through the infrastructure architecture I've been working on. This is a cloud-native application built entirely on AWS, and I'll explain how all the pieces fit together, from the moment a developer commits code to when a user interacts with the running application.

Let me break this down into logical sections, starting from the foundational infrastructure and working up to the deployment pipeline.
## The Cloud Foundation: AWS Services
### Compute Layer
We have a **hybrid compute architecture** that uses both **serverless** and containerized **workloads**, and the choice between them is driven by the nature of the workload.

**AWS Lambda** handles our short-lived, event-driven simulation tasks. Lambda is perfect here because simulations can be triggered on-demand, and we only pay for the actual compute time used. The Lambda functions run as container images rather than ZIP packages, which gives us more flexibility in terms of dependencies and runtime environment. The container images are stored in Amazon ECR and are built from a specific Dockerfile that uses the official AWS Lambda Python 3.11 base image.

**Amazon ECS with Fargate** serves two purposes in our architecture. First, it hosts our frontend application, which is a Streamlit web application. We chose Fargate over EC2-backed ECS because we didn't want to manage the underlying EC2 instances ourselves. Fargate abstracts that away completely—we just define the CPU and memory requirements (in our case, 2 vCPUs and 8 GB of memory), and AWS handles the rest.

Second, ECS is also used for long-running simulation jobs that would exceed Lambda's 15-minute timeout. These run as standalone ECS Tasks rather than as part of a Service. The distinction is important: a Service maintains a desired count of running tasks and handles load balancing, while a Task is a one-off execution that runs to completion and terminates.
### Storage Layer
**Amazon S3** is our primary data store. We have multiple S3 buckets serving different purposes: input data storage, simulation results, and intermediate processing files. All buckets are encrypted using AWS KMS (Key Management Service) with customer-managed keys, which gives us control over the encryption keys and allows us to define key policies and rotation schedules.

The data stored in S3 follows a specific format—we use **Parquet files with SNAPPY compression**. Parquet is a columnar storage format, which is highly efficient for analytical queries because you can read only the columns you need rather than entire rows. SNAPPY compression provides a good balance between compression ratio and decompression speed.

**Amazon DynamoDB** stores metadata about simulation runs. This includes job status, timestamps, configuration parameters, and references to the S3 locations where results are stored. DynamoDB is ideal for this use case because we need fast, consistent reads and writes for job tracking, and we don't need complex relational queries.
### Analytics Layer
**Amazon Athena** provides SQL query capabilities over our S3 data. This is where the Parquet format really shines—Athena can leverage the columnar structure to minimize the amount of data scanned, which directly reduces costs since Athena pricing is based on data scanned.

We've configured a dedicated Athena Workgroup with specific settings:
- A query scan limit of 200 GB to prevent runaway queries from scanning too much data
- Athena Engine Version 3, which includes performance improvement and additional SQL functions
- CloudWatch metrics publishing enabled for monitoring query performance
- Results stored in S3 with KMS encryption

**AWS Glue Data Catalog** serves as the **metadata repository for Athena**. It stores table definitions, column types, and partition information. We define our tables programmatically using SQLAlchemy models with custom Athena-specific metadata, which allows us to version-control our schema definitions alongside our application code. Here's an example of how a table is defined:
```python
class CicloAttivo(TableBaseModel):
    __tablename__ = 'ciclo_attivo'
    __table_args__ = dict(
        schema="cash_flow_simulator_simulations",
        awsathena_location=f'{BUCKET_BASE_PATH_SIMULATIONS}/dataset/ciclo_attivo',
        awsathena_file_format='PARQUET',
        awsathena_compression='SNAPPY'
    )
```

This approach ensures that our application code and database schema are always in sync.
## Infrastructure as Code with AWS CDK
### Why CDK?
We use **AWS CDK** (**Cloud Development Kit**) version 2 with Python bindings to define all our infrastructure. CDK is a significant step up from raw CloudFormation templates because we can use actual programming constructs—loops, conditionals, functions, classes—to define our infrastructure.

This matters because infrastructure code becomes much more maintainable. Instead of copying and pasting YAML blocks with slight variations, we can write a function that generates resources based on parameters. We can also create custom abstractions that encapsulate common patterns.
### Stack Architecture
Our CDK application is organized into multiple stacks, each representing a logical grouping of related resources:
1. **Storage Stack**: S3 buckets and their configurations
2. **Simulation Storage Stack**: S3 buckets and Glue tables specifically for simulation data
3. **Athena Stack**: Athena workgroup and query-related resources
4. **Roles Stack**: IAM roles and policies
5. **Shared Stack**: Resources used across multiple stacks, like SNS topics for notifications
6. **Frontend Stack**: ECS Fargate service, Application Load Balancer, and related networking
7. **Simulation Stack**: Lambda functions and ECS task definitions for simulation workloads

The stacks have explicit dependencies. For example, the Frontend stack depends on the Roles stack because the ECS tasks need IAM roles to access other AWS services. CDK manages these dependencies automatically—when you deploy, it determines the correct order based on the dependency graph.
### Custom Constructs
One of the most powerful features of CDK is the ability to create custom constructs—reusable components that encapsulate complex resource configurations.

We've created a construct called `AutoDeployingCachedECR` that handles a common pattern: creating an ECR repository and automatically deploying a Docker image to it during CDK deployment. Here's what it does:
1. **Creates an ECR repository** with lifecycle policies that automatically clean up old images to control storage costs
2. **Sets up IAM policies** that allow the CDK deployment process to push images to the repository
3. **Builds the Docker image** using BuildKit with registry-based caching (more on this later)
4. **Deploys the image** to ECR as part of the CDK deployment

This construct is used multiple times in our application—once for the frontend image, once for the Lambda simulation image, and once for the ECS simulation image. By encapsulating this logic in a construct, we avoid code duplication and ensure consistent configuration across all our container images.
## Networking Architecture
### VPC Configuration
The application runs within a Virtual Private Cloud (VPC) in the eu-central-1 (Frankfurt) region. The VPC has a CIDR block of 10.224.12.0/22, which gives us 1,024 IP addresses to work with.

We use **private subnets only**, spread across three Availability Zones (eu-central-1a, eu-central-1b, eu-central-1c). This is a deliberate security decision—none of our compute resources have public IP addresses or direct internet access.

So how does our application communicate with AWS services and the internet? Through **VPC Endpoints**. These are private connections to AWS services that don't traverse the public internet. We have endpoints for:
- S3 (Gateway endpoint)
- ECR (Interface endpoints for API and Docker registry)
- CloudWatch Logs
- Athena
- DynamoDB
- And others as needed

This architecture means that even if an attacker compromised one of our containers, they couldn't easily exfiltrate data over the internet because there's no route out of the VPC except through the endpoints we've explicitly configured.
### Load Balancing
The frontend application sits behind an **Application Load Balancer (ALB)**. The ALB:
- Terminates TLS connections using certificates from AWS Certificate Manager
- Distributes traffic across multiple Fargate tasks (if we scale horizontally)
- Performs health checks to ensure traffic only goes to healthy containers
- Provides a stable DNS name that doesn't change when containers are replaced
## Security Architecture
### Encryption
We follow a defense-in-depth approach to encryption:

**Data at rest:**
- S3 buckets use SSE-KMS (Server-Side Encryption with KMS) with customer-managed keys
- Athena query results are encrypted with KMS
- ECR images are encrypted with AWS-managed keys
- DynamoDB uses AWS-managed encryption

**Data in transit:**
- TLS 1.2+ for all external connections
- All AWS API calls use HTTPS
- VPC endpoints use private connectivity
### IAM Strategy
We follow the principle of least privilege. Each component has its own IAM role with only the permissions it needs:
- **ECS Task Roles**: Allow the frontend containers to read from S3, query Athena, write to DynamoDB, and invoke Lambda functions
- **Lambda Execution Roles**: Allow simulation functions to read/write S3, log to CloudWatch, and access DynamoDB
- **CodeBuild Roles**: Allow the CI/CD pipeline to deploy infrastructure and push container images

For different environments (dev, test, prod), we define different sets of IAM principals who can access resources. Production has more restricted access than development.
### Cross-Account Access
We operate in a multi-account setup, which is an AWS best practice for isolating workloads:

| Environment | AWS Account  |
| ----------- | ------------ |
| Development | 732789817868 |
| Test        | 643356557982 |
| Production  | 238788445602 |
The CI/CD pipeline runs in a central account but deploys to these target accounts using **STS AssumeRole**. Here's how it works:
1. CodeBuild has an IAM role in the central account
2. That role has permission to assume a specific role in each target account
3. Before deploying, CodeBuild calls `sts:AssumeRole` to get temporary credentials for the target account
4. Those credentials are used for all subsequent AWS API calls

The `do_assume_cross_account_role` function in our build scripts handles this:
```bash
function do_assume_cross_account_role() {
    local RoleArn="arn:aws:iam::${TARGET_ACCOUNT}:role/${CA_ROLE}"
    export ASSUMED_ROLE=$(
        aws sts assume-role \
            --role-arn $RoleArn \
            --role-session-name $RoleSessName
    )
    export AWS_ACCESS_KEY_ID=$(echo "${ASSUMED_ROLE}" | jq -r '.Credentials.AccessKeyId')
    export AWS_SECRET_ACCESS_KEY=$(echo "${ASSUMED_ROLE}" | jq -r '.Credentials.SecretAccessKey')
    export AWS_SESSION_TOKEN=$(echo "${ASSUMED_ROLE}" | jq -r '.Credentials.SessionToken')
}
```

This pattern provides strong security boundaries between environments while still allowing automated deployments.
## CI/CD Pipeline
### Pipeline Architecture
Our deployment pipeline uses AWS-native services:
1. **Source Stage**: Code is stored in AWS CodeCommit (a Git-compatible repository service)
2. **Build Stage**: AWS CodeBuild runs our build and deployment scripts
3. **Release Orchestration**: AWS CodePipeline coordinates versioned releases to production
### Build Process
The build process is defined in buildspec files (YAML configurations that tell CodeBuild what to do) and shell scripts that contain reusable functions.

**Phase 1: Install**
First, we set up the build environment:
```bash
function do_install_dependencies() {
    # Install CDK globally via npm
    npm install -g aws-cdk@${CDK_VERSION}
    # Install Python dependencies
    pip3 install --upgrade -r requirements.cdk.txt
    # Install CodeCommit Git helper for pushing changes back to the repo
    pip3 install --upgrade git-remote-codecommit
}
```

**Phase 2: Authenticate with CodeArtifact**
We use AWS CodeArtifact as a private package repository. This is important because:
- We have internal Python packages that can't be published to PyPI
- We can cache packages from PyPI, reducing external dependencies
- We can enforce security policies on what packages are allowed

The authentication function configures both pip (for Python) and npm (for Node.js) to use CodeArtifact:
```bash
function do_login_code_artifact() {
    # Get an auth token from CodeArtifact
    export CODEARTIFACT_AUTH_TOKEN=$(aws codeartifact get-authorization-token ...)
    # Configure pip to use CodeArtifact
    pip config set global.index-url $CODEARTIFACT_PIP_URL
    # Configure npm to use CodeArtifact
    npm config set registry=https://$npm_url/npm/${CODEARTIFACT_REGISTRY}/
}
```

**Phase 3: Cross-Account Authentication**
Before we can deploy, we need credentials for the target AWS account. We assume a cross-account role as described earlier.

**Phase 4: ECR Login**
We also need to authenticate with ECR in the target account so we can push container images:
```bash
function do_login_ecr() {
    aws ecr get-login-password --region=${AWS_REGION} \
    | docker login --username AWS --password-stdin \
        ${TARGET_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
}
```

**Phase 5: CDK Operations**
Finally, we run the actual CDK commands. The `do_cdk_operation` function supports three operations:
- **synth**: Generates CloudFormation templates without deploying
- **diff**: Compares the current state with what would be deployed
- **deploy**: Actually deploys the infrastructure

For deployment, we set up Docker BuildKit with a special configuration:
```bash
export DOCKER_BUILDKIT=1
docker buildx create --name container --driver=docker-container
docker buildx use container
```

This enables advanced Docker features like registry-based caching and better layer management.
### Docker Build Optimization
Building Docker images can be slow, especially in CI/CD where you start fresh each time. We use several techniques to speed this up:

**Registry-based caching**: Instead of storing build cache locally (which would be lost between CI runs), we push cache layers to ECR. On the next build, Docker pulls these cache layers and only rebuilds what changed.
```python
cache_from=[
    aws_ecr_assets.DockerCacheOption(
        type="registry",
        params={"ref": repo_tag_cache_url},
    )
],

cache_to=aws_ecr_assets.DockerCacheOption(
    type="registry",
    params={"ref": repo_tag_cache_url, "mode": "max"},
)
```

**BuildKit secrets**: Our Dockerfiles need to authenticate with CodeArtifact to install private packages. We use BuildKit secrets to pass credentials securely without baking them into image layers:
```dockerfile
RUN --mount=type=secret,id=CODEARTIFACT_PIP_URL \
    python3.11 -m pip config set \
    global.index-url $(cat /run/secrets/CODEARTIFACT_PIP_URL) \
    && python3.11 -m pip install -r requirements.txt
```

This is more secure than using ARG or ENV because secrets mounted this way never appear in the image history.
### Diff Tracking
One unique aspect of our pipeline is that we automatically track infrastructure changes. During the synth phase, we run `cdk diff` and commit the output to the repository:
```yaml
- git add ./Diff_files/cdk-diff-${BUSINESS_UNIT}-$STAGE.txt
- git commit -m "(CodeBuild) Commit of diff file ($STAGE)"
- git push -u origin ${BRANCH_NAME}
```

This creates an audit trail of what changed with each deployment. When reviewing a pull request, you can see not just the code changes but also the resulting infrastructure changes.
### Versioned Releases
For production deployments, we use a versioned release process. Instead of deploying directly, you trigger a release with a version number:
```bash
./scripts/infra.sh --stage prod --release 1.0.0
```

This creates a CodePipeline with the version number in its name (`devops-cash-flow-simulator-MERCATO-PROD-v1-0-0`). The pipeline pulls the specific version of the code and deploys it. This approach:
- Provides clear traceability of what's deployed
- Allows rollback to specific versions
- Separates the decision to release from the mechanics of deployment
## Container Architecture
We have three distinct Docker images, each optimized for its specific use case:
### Frontend Image
```dockerfile
FROM amazonlinux:2023
```

Built on Amazon Linux 2023, which is AWS's own Linux distribution optimized for cloud workloads. This image runs the Streamlit web application. We chose Amazon Linux because:
- It's well-tested on AWS infrastructure
- It receives long-term security updates
- It has predictable behavior on ECS Fargate
### Lambda Simulation Image
```dockerfile
FROM public.ecr.aws/lambda/python:3.11
```

Uses the official AWS Lambda Python base image. This image includes the Lambda runtime interface, which handles the communication between the Lambda service and our code. Using a container for Lambda (instead of a ZIP file) allows us to:
- Include dependencies that are too large for the 250 MB ZIP limit
- Use the same dependency management (requirements.txt) as our other containers
- Include native libraries that might be tricky to package in a ZIP
### ECS Simulation Image
```dockerfile
FROM python:3.11
```

A standard Python image for long-running simulation tasks. These tasks run on ECS Fargate and don't have the 15-minute timeout that Lambda has. We use this for simulations that might take hours to complete.
## Configuration Management
### Environment Variables
Runtime configuration is handled through environment variables. This follows the twelve-factor app methodology, which advocates for strict separation of config from code.

The ECS task definition specifies environment variables that the container receives at startup:
```python
ecs_container_environment={
    "AWS_DEFAULT_REGION": self.region,
    "STAGE": self.stage,
    "AWS_ATHENA_WORKGROUP": "cash-flow-simulator-athena-workgroup",
    "STREAMLIT_SERVER_MAX_UPLOAD_SIZE": '1000',
}
```

For local development, developers use a .env file that mirrors these settings.
### SSM Parameter Store
We use AWS Systems Manager Parameter Store for configuration that:
- Might change without a code deployment
- Contains sensitive values (using SecureString parameters)
- Is shared across multiple applications

Parameters are organized hierarchically:
```
/MERCATO/devops-cash-flow-simulator/
├── notification_email
├── CODE_ARTIFACT_DOMAIN
├── DEV/
│   ├── versions/CDK_VERSION
│   ├── versions/PYTHON_RUNTIME
│   └── CA_ROLE
├── TEST/
│   └── ...
└── PROD/
    └── ...
```

This structure allows environment-specific overrides while sharing common settings.
### CDK Context
CDK-specific configuration lives in `cdk.json` and `cdk.context.json`. These files contain:
- Feature flags that control CDK behavior
- Environment-specific values like certificate ARNs and API endpoints
- VPC information cached from lookups

The context mechanism allows CDK to "remember" values from previous runs, which speeds up synthesis and ensures consistency.
## Application Architecture Patterns
### Strategy Pattern for Job Execution
The application uses the Strategy pattern to abstract how simulation jobs are executed. This is elegant because it allows the same business logic to run in different contexts:
```python
class ExecutionStrategy(ABC):
    @abstractmethod
    def execute(self, params):
        pass

class LambdaInvokeStrategy(ExecutionStrategy):
    def execute(self, params):
        # Invoke via AWS Lambda
        lambda_client.invoke(FunctionName=self.function_name, Payload=params)
        
class LocalInvokeStrategy(ExecutionStrategy):
    def execute(self, params):
        # Run locally for development/testing
        local_function(params)
```

At runtime, the appropriate strategy is selected based on configuration:
```python
if run_mode == "LOCAL":
    strategy = LocalInvokeStrategy(local_function_name=start_function)
else:
    strategy = LambdaInvokeStrategy(lambda_function_name=ctx.simulation_starter_lambda)
```

This pattern makes testing easier (you can use a mock strategy) and allows developers to run simulations locally without deploying to AWS.
### Data Flow
The overall data flow follows a clear pattern:
1. **Input**: Data files are uploaded to S3 (either through the UI or batch processes)
2. **Processing**: Lambda or ECS tasks read input data, run simulations, and write results back to S3
3. **Storage**: Results are stored as Parquet files in a structured S3 path
4. **Query**: Athena provides SQL access to the results for analysis and reporting
5. **Visualization**: The Streamlit frontend queries Athena and displays results

This separation of concerns allows each component to scale independently and be modified without affecting others.
## Monitoring and Observability
### CloudWatch Integration
All components emit logs to CloudWatch Logs:
- Lambda functions automatically log to `/aws/lambda/function-name`
- ECS tasks log to custom log groups
- Athena query history is tracked in CloudWatch

We've enabled CloudWatch metrics publishing for the Athena workgroup, which gives us visibility into:
- Query execution times
- Data scanned per query
- Failed queries
### Alerting
An SNS topic is configured for failure notifications. When something goes wrong (a Lambda function fails, an ECS task crashes), an alert is sent to the subscribed email addresses. This is configured in the Shared stack, making it available to all other components.
## Summary
To summarize the key architectural decisions:
1. **Serverless-first compute**: Lambda for short tasks, Fargate for longer tasks—no servers to manage
2. **Private networking**: All resources in private subnets, accessed via VPC endpoints
3. **Infrastructure as code**: Everything defined in CDK, version-controlled, and reproducible
4. **Multi-account isolation**: Separate AWS accounts for dev, test, and prod
5. **Automated CI/CD**: CodeBuild and CodePipeline handle all deployments
6. **Columnar storage**: Parquet format with SNAPPY compression for cost-effective analytics
7. **Encryption everywhere**: KMS encryption for data at rest, TLS for data in transit

This architecture provides a solid foundation that balances security, cost efficiency, and developer productivity. The use of managed services reduces operational burden, while the infrastructure-as-code approach ensures consistency and enables rapid iteration.

# Comprehensive Architectural Deep Dive: Cash Flow Simulator Infrastructure
## Introduction and Context
Let me walk you through the complete infrastructure architecture of this project. I'll explain every component, every design decision, and how all the pieces work together. My goal is to give you such a thorough understanding that you could recreate this architecture from scratch or confidently discuss any aspect of it in detail.

This is a cloud-native application built entirely on Amazon Web Services (AWS), deployed in the eu-central-1 region (Frankfurt, Germany). The application follows modern cloud architecture principles: serverless where possible, containers where necessary, infrastructure as code for everything, and a strong emphasis on security and automation.
## Part 1: AWS Services - The Building Blocks
### 1.1 Compute Services
#### AWS Lambda
AWS Lambda is our **serverless compute service** for event-driven workloads. In this architecture, Lambda handles relatively short-lived simulation tasks (under 15 minutes) and can be triggered on-demand.

**Why Lambda for simulations?**
Lambda provides several advantages for this use case:
1. **Pay-per-use pricing**: We only pay for the actual compute time consumed. If no simulations are running, we pay nothing. This is dramatically different from running EC2 instances 24/7.
2. **Automatic scaling**: If 100 users trigger simulations simultaneously, Lambda automatically spins up 100 concurrent executions (subject to account limits). We don't need to provision capacity in advance.
3. **No server management**: AWS handles patching, security updates, and infrastructure maintenance. Our team focuses on application code, not operating systems.

**Container-based Lambda deployment**
Rather than packaging our Lambda function as a ZIP file (the traditional approach), we deploy it as a container image. This is defined in `simulation.lambda.dockerfile`:
```dockerfile
FROM public.ecr.aws/lambda/python:3.11
```

This base image comes from AWS's public ECR registry and includes the Lambda Runtime Interface Client (RIC), which handles the communication protocol between the Lambda service and our code.

We chose container-based deployment because:
1. **Larger size limit**: ZIP-based Lambdas are limited to 250 MB uncompressed. Container-based Lambdas can be up to 10 GB. Our simulation code has substantial dependencies (numerical libraries, data processing tools) that would be difficult to fit in 250 MB.
2. **Consistent tooling**: We use Docker for all our compute workloads. Having Lambda also use Docker means one set of skills, one set of tools, and consistent debugging approaches.
3. **Local testing**: We can run the exact same container locally that will run in Lambda, making development and debugging much easier.

**Lambda configuration details**
The Lambda functions are configured with:
- Memory allocation that scales CPU proportionally (Lambda allocates CPU power linearly with memory)
- Timeout settings appropriate for simulation workloads
- VPC configuration so the function can access resources in our private network
- IAM execution roles with least-privilege permissions
#### Amazon ECS with AWS Fargate
ECS (Elastic Container Service) is AWS's container orchestration platform, and Fargate is a serverless compute engine for containers. Together, they let us run Docker containers without managing the underlying EC2 instances.

**Frontend hosting on Fargate**
Our user-facing application is a Streamlit web application. Streamlit is a Python framework for building data applications with minimal frontend code. The application runs as a Docker container on Fargate.

Here's how the Fargate configuration looks in our CDK code:
```python
StreamlitHostingApplicationLoadBalancedFargateService(
    ecs_task_cpu=2048,        # 2 vCPU (1024 units = 1 vCPU)
    ecs_task_memory_mib=8192,  # 8 GB RAM
    ecs_container_environment={
        "AWS_DEFAULT_REGION": self.region,
        "STAGE": self.stage,
        "AWS_ATHENA_WORKGROUP": "cash-flow-simulator-athena-workgroup",
        "STREAMLIT_SERVER_MAX_UPLOAD_SIZE": '1000',  # MB
    },
)
```

Let me explain each setting:
- **ecs_task_cpu=2048**: This allocates 2 vCPUs to each task. In Fargate, CPU is measured in units where 1024 units equals 1 vCPU. We chose 2 vCPUs because Streamlit applications can be CPU-intensive when processing data and generating visualizations.
- **ecs_task_memory_mib=8192**: 8 GB of memory. This is necessary because the application loads datasets into memory for processing. Streamlit keeps session state in memory, and with multiple concurrent users, memory usage adds up.
- **Environment variables**: These are passed to the container at runtime. They configure the application to know which AWS region it's in, which environment (dev/test/prod) it's running in, which Athena workgroup to use for queries, and what the maximum file upload size should be.

**ECS Service vs. ECS Task**
It's important to understand the distinction between an ECS Service and an ECS Task:
- **ECS Service**: A long-running application that should always be available. The service maintains a desired count of tasks and replaces them if they fail. It integrates with load balancers for traffic distribution. Our frontend runs as a Service.
- **ECS Task**: A one-off execution that runs to completion and terminates. Think of it like a batch job. Our long-running simulations (those that exceed Lambda's 15-minute timeout) run as standalone Tasks.

**Why Fargate instead of EC2-backed ECS?**
With EC2-backed ECS, you manage a cluster of EC2 instances that run your containers. You're responsible for:
- Choosing instance types
- Managing cluster capacity
- Patching the EC2 instances
- Monitoring instance health

With Fargate, AWS manages all of this. You specify CPU and memory requirements, and AWS handles the rest. The tradeoff is slightly higher cost per unit of compute, but significantly lower operational burden.

For our use case, the reduced operational complexity is worth the cost premium. Our team is small, and we'd rather spend time on application features than infrastructure maintenance.
#### The Dockerfile Structure
We have three distinct Dockerfiles, each optimized for its specific runtime environment:

**1. Frontend Dockerfile (`frontend.dockerfile`)**
```dockerfile
FROM amazonlinux:2023
```

Amazon Linux 2023 is AWS's own Linux distribution. We chose it because:
- It's optimized for AWS infrastructure
- It receives long-term support and security updates from AWS
- Package versions are tested for compatibility
- It's the same OS that runs on many AWS services internally

The frontend container includes:
- Python 3.11 runtime
- Streamlit and its dependencies
- AWS SDK (boto3) for interacting with AWS services
- Data processing libraries (pandas, numpy, pyarrow)

**2. Lambda Simulation Dockerfile (`simulation.lambda.dockerfile`)**
```dockerfile
FROM public.ecr.aws/lambda/python:3.11
```

This uses AWS's official Lambda base image, which includes:
- The Lambda Runtime Interface Client (RIC)
- A stripped-down Amazon Linux environment
- Python 3.11 runtime

The image is designed to be minimal—it includes only what's necessary to run Lambda functions. This keeps the image size small, which improves cold start times (the time it takes to start a new Lambda execution environment).

**3. ECS Simulation Dockerfile (`simulation.ecs.dockerfile`)**
```dockerfile
FROM python:3.11
```

This uses the official Python image from Docker Hub. For ECS tasks, we don't need the Lambda-specific components, so a standard Python image works well. It's larger than the Lambda image but includes more tooling that can be useful for debugging long-running jobs.

**Common patterns across all Dockerfiles**
All three Dockerfiles share a common pattern for installing dependencies from our private package registry:
```dockerfile
RUN --mount=type=secret,id=CODEARTIFACT_PIP_URL \
    python3.11 -m pip config set \
    global.index-url $(cat /run/secrets/CODEARTIFACT_PIP_URL) \
    && python3.11 -m pip install -r requirements.txt
```

This uses Docker BuildKit's secrets feature. Let me explain why this matters.

When you install packages from a private registry, you need credentials. The naive approach would be to pass credentials as build arguments:
```dockerfile
ARG CODEARTIFACT_URL
RUN pip config set global.index-url $CODEARTIFACT_URL
```

But this is insecure! Build arguments are stored in the image's layer history. Anyone who pulls the image can run `docker history` and see the credentials.

BuildKit secrets solve this by mounting credentials as temporary files that exist only during the build step. They're never written to any image layer. The `--mount=type=secret,id=CODEARTIFACT_PIP_URL` syntax tells Docker to mount the secret, and `$(cat /run/secrets/CODEARTIFACT_PIP_URL)` reads its value.
### 1.2 Storage Services
#### Amazon S3 (Simple Storage Service)
S3 is our primary data store. It's an object storage service that offers virtually unlimited capacity, high durability (99.999999999% - eleven 9s), and integration with almost every other AWS service.

**Bucket organization**
We have multiple S3 buckets serving different purposes:
1. **Input data bucket**: Stores raw data files that feed into simulations
2. **Simulation results bucket**: Stores output from simulation runs
3. **Intermediate processing bucket**: Temporary storage for data transformations
4. **Athena results bucket**: Stores query results from Athena

**Why multiple buckets instead of one bucket with folders?**
While S3 doesn't have true folders (it's a flat object store where "/" in key names creates a folder-like illusion), using separate buckets provides:
1. **Different access controls**: We can give some systems read-only access to input data while allowing write access to results
2. **Different lifecycle policies**: Input data might be retained indefinitely, while intermediate files might be deleted after 30 days
3. **Cost allocation**: We can track storage costs per bucket
4. **Blast radius limitation**: A misconfigured deletion policy on one bucket doesn't affect others

**Data format: Parquet with SNAPPY compression**
All our analytical data is stored in Parquet format with SNAPPY compression. This is a crucial architectural decision that affects both cost and performance.

**What is Parquet?**
Parquet is a columnar storage format. In a traditional row-based format (like CSV or JSON), data is organized like this:
```
Row 1: name="Alice", age=30, city="NYC"
Row 2: name="Bob", age=25, city="LA"
Row 3: name="Carol", age=35, city="Chicago"
```

In columnar format, data is organized like this:
```
name column: ["Alice", "Bob", "Carol"]
age column: [30, 25, 35]
city column: ["NYC", "LA", "Chicago"]
```

Why does this matter? Consider a query like "What's the average age?" With row-based storage, you must read entire rows even though you only need the age column. With columnar storage, you read only the age column.

For our use case, this is transformative:
- Our tables have dozens of columns
- Most queries use only a few columns
- Columnar storage can reduce I/O by 90% or more

**What is SNAPPY compression?**
SNAPPY is a compression algorithm developed by Google. It prioritizes speed over compression ratio. Compared to alternatives:

| Algorithm | Compression Ratio | Speed     |
| --------- | ----------------- | --------- |
| GZIP      | High              | Slow      |
| SNAPPY    | Medium            | Fast      |
| LZ4       | Low               | Very Fast |
| ZSTD      | High              | Medium    |
  We chose SNAPPY because:
1. Athena can decompress it in parallel across many nodes
2. The decompression speed means queries start returning results quickly
3. The compression ratio is good enough to significantly reduce storage costs

**S3 encryption with KMS**
All our S3 buckets use server-side encryption with AWS KMS (Key Management Service). Here's how it works:
1. We create a Customer Managed Key (CMK) in KMS
2. When an object is uploaded to S3, S3 requests a data encryption key from KMS
3. KMS returns two versions of the key: plaintext and encrypted
4. S3 uses the plaintext key to encrypt the data
5. S3 stores the encrypted data key alongside the encrypted object
6. The plaintext key is discarded (never stored)
7. When reading, S3 asks KMS to decrypt the data key, then uses it to decrypt the object

Why use customer-managed keys instead of S3-managed keys (SSE-S3)?
1. **Audit trail**: All key usage is logged in CloudTrail. We can see who accessed what data and when.
2. **Key policies**: We can restrict which IAM principals can use the key
3. **Key rotation**: We control the rotation schedule
4. **Cross-account access**: We can grant other AWS accounts permission to use the key

**Bucket lifecycle policies**
We configure lifecycle policies to manage storage costs:
```python
lifecycle_rules=[
    s3.LifecycleRule(
        id="DeleteOldIntermediateFiles",
        prefix="intermediate/",
        expiration=Duration.days(30),
    ),
    s3.LifecycleRule(
        id="TransitionToGlacier",
        prefix="archive/",
        transitions=[
            s3.Transition(
                storage_class=s3.StorageClass.GLACIER,
                transition_after=Duration.days(90),
            )
        ],
    ),
]
```

This automatically:
- Deletes intermediate files after 30 days
- Moves archive data to Glacier (much cheaper storage) after 90 days
#### Amazon DynamoDB
DynamoDB is our NoSQL database for simulation metadata. It stores information about each simulation run:
- Run ID (partition key)
- Timestamp
- Status (pending, running, completed, failed)
- Configuration parameters
- S3 paths to input and output data
- Error messages (if failed)

**Why DynamoDB for metadata instead of a relational database?**
1. **Access patterns**: Our access is simple—we look up runs by ID or list recent runs. We don't need complex joins.
2. **Serverless**: DynamoDB is fully managed. No instances to provision, no maintenance windows.
3. **Performance**: Single-digit millisecond latency for key-value lookups
4. **Scalability**: Automatically scales to handle thousands of requests per second
5. **Cost**: Pay-per-request pricing for our workload is cheaper than running an RDS instance

**Why not store metadata in S3?**
S3 is optimized for large objects, not small lookups. Operations like "list all runs from the last hour" would require listing objects and parsing their metadata, which is slow and expensive at scale.
### 1.3 Analytics Services
#### Amazon Athena
Athena is a serverless query engine that lets us run SQL queries directly on data in S3. There's no infrastructure to manage—we just point Athena at our data and start querying.

**How Athena works under the hood**
Athena is based on Presto (now called Trino), an open-source distributed SQL query engine. When you submit a query:
1. Athena parses the SQL and creates an execution plan
2. It reads table metadata from the Glue Data Catalog
3. It identifies which S3 objects contain relevant data
4. It spins up a distributed cluster of workers (invisible to us)
5. Workers read data from S3 in parallel
6. Results are aggregated and written to the results bucket
7. The result location is returned to the caller

This happens in seconds for most queries, and we only pay for the data scanned.

**Athena Workgroup configuration**
We've created a dedicated Athena Workgroup with specific settings:
```python
work_group_configuration=aws_athena.CfnWorkGroup.WorkGroupConfigurationProperty(
    bytes_scanned_cutoff_per_query=200_000_000_000,  # 200 GB
    engine_version=aws_athena.CfnWorkGroup.EngineVersionProperty(
        selected_engine_version="Athena engine version 3"
    ),
    publish_cloud_watch_metrics_enabled=True,
    enforce_work_group_configuration=True,
    result_configuration=aws_athena.CfnWorkGroup.ResultConfigurationProperty(
        output_location=f"s3://{results_bucket.bucket_name}/athena-results/",
        encryption_configuration=aws_athena.CfnWorkGroup.EncryptionConfigurationProperty(
            encryption_option="SSE_KMS",
            kms_key=kms_key.key_arn,
        ),
    ),
)
```

Let me explain each setting.

**bytes_scanned_cutoff_per_query=200_000_000_000 (200 GB)**
This is a safety limit. Athena charges $5 per TB scanned. A runaway query scanning petabytes of data could generate enormous costs. This limit cancels any query that would scan more than 200 GB.

Why 200 GB? It's a balance between allowing legitimate large queries and preventing accidents. At $5/TB, a 200 GB query costs about $1.

**engine_version="Athena engine version 3"**
Athena Engine v3 includes significant improvements over v2:
- Better query performance (some queries are 10x faster)
- More SQL functions and features
- Improved handling of complex data types
- Better error messages

**publish_cloud_watch_metrics_enabled=True**
This publishes query metrics to CloudWatch:
- Number of queries
- Data scanned
- Query execution time
- Failed queries

We use these metrics for monitoring and alerting.

**enforce_work_group_configuration=True**
This ensures that all queries in the workgroup use the workgroup's settings, even if the user tries to override them. This prevents users from bypassing the scan limit or writing results to a different location.

**Result encryption with KMS**
Query results are encrypted with our KMS key. Even though results are typically short-lived, they may contain sensitive data, so encryption is essential.
#### AWS Glue Data Catalog
The Glue Data Catalog is a metadata repository that stores information about our data:
- Database and table definitions
- Column names and data types
- Partition information
- S3 locations

Think of it as a "table of contents" for our data lake. When Athena receives a query, it consults the Glue catalog to understand the data structure and location.

**Defining tables with SQLAlchemy**
Rather than creating tables manually in the AWS console or with raw DDL, we define them as SQLAlchemy models:
```python
class CicloAttivo(TableBaseModel):
    __tablename__ = 'ciclo_attivo'
    __table_args__ = dict(
        schema="cash_flow_simulator_simulations",
        awsathena_location=f'{BUCKET_BASE_PATH_SIMULATIONS}/dataset/ciclo_attivo',
        awsathena_file_format='PARQUET',
        awsathena_compression='SNAPPY'
    )
    id = Column(String, primary_key=True)
    timestamp = Column(DateTime)
    amount = Column(Float)
    category = Column(String)
    # ... more columns
```

This approach provides several benefits:
1. **Version control**: Table definitions are code, stored in Git alongside the application
2. **Type safety**: SQLAlchemy validates that our queries use columns that exist
3. **Documentation**: The model serves as documentation of the data structure
4. **Consistency**: The same model can be used for reading, writing, and schema creation

The `awsathena_*` attributes are custom extensions that tell our schema creation tool how to configure the Athena table.
### 1.4 Networking Services
#### Amazon VPC (Virtual Private Cloud)
A VPC is an isolated virtual network within AWS. Think of it as your own private data center in the cloud, with complete control over IP addressing, subnets, routing, and security.

**Our VPC configuration**
From the CDK context file, our VPC has:
```json
{
  "vpcId": "vpc-0e4d359a4c9ac1a5e",
  "vpcCidrBlock": "10.224.12.0/22",
  "availabilityZones": ["eu-central-1a", "eu-central-1b", "eu-central-1c"],
  "privateSubnetIds": [
    "subnet-0e4bcba46a52a5db8",
    "subnet-0c35a1b2f99deb1aa", 
    "subnet-009fbb290b2a02679"
  ],
  "privateSubnetRouteTableIds": [...]
}

```

**CIDR block: 10.224.12.0/22**
This defines our IP address range. The /22 means the first 22 bits are fixed, leaving 10 bits for host addresses. That's 2^10 = 1,024 IP addresses.

The address 10.224.12.0 is in the RFC 1918 private range, which means it's not routable on the public internet. This is standard for private networks.

Why this specific range? In enterprise environments, VPCs often need to connect to each other or to on-premises networks via VPN or Direct Connect. The IP range must not overlap with other networks. This address was assigned to us to avoid conflicts with other VPCs in the organization.

**Availability Zones**
We use three Availability Zones (AZs): eu-central-1a, eu-central-1b, and eu-central-1c.

An Availability Zone is one or more discrete data centers with redundant power, networking, and connectivity. AZs within a region are connected by high-bandwidth, low-latency networking, but they're physically separated so that a failure in one AZ (fire, flood, power outage) doesn't affect others.

By deploying across three AZs, our application remains available even if an entire data center goes offline.

**Private subnets only**
Notice that we only have private subnets—there are no public subnets. This is a security architecture decision.

In a typical VPC architecture, you'd have:
- **Public subnets**: Resources here have public IP addresses and can be reached from the internet
- **Private subnets**: Resources here only have private IP addresses and cannot be directly reached from the internet

We use only private subnets because:
1. None of our compute resources need to be directly accessible from the internet
2. It eliminates an entire class of security risks (direct attacks on public IPs)
3. All external access goes through the Application Load Balancer, which provides a single point of control

**How do private resources access the internet?**
Private resources often need to access the internet—to download packages, call external APIs, etc. In a traditional setup, you'd use a NAT Gateway.

However, based on the architecture (using VPC endpoints and private API endpoints), it appears this VPC may not have NAT Gateways. Instead, all necessary AWS service access happens through VPC Endpoints.
#### VPC Endpoints
VPC Endpoints provide private connectivity to AWS services without going over the internet.
**Types of VPC Endpoints**
1. **Gateway Endpoints** (S3, DynamoDB)
   - Free to use
   - Add routes to your route tables
   - Traffic stays within AWS network
2. **Interface Endpoints** (most other services)
   - Create an Elastic Network Interface (ENI) in your subnet
   - Cost: ~$0.01/hour plus data processing charges
   - Accessible via private DNS

**Why VPC Endpoints matter for security**
Without VPC Endpoints, traffic from a private subnet to S3 would need to:
1. Go through a NAT Gateway
2. Exit to the public internet
3. Enter AWS's public S3 endpoint
4. Return via the same path

With a VPC Endpoint:
1. Traffic goes directly to S3 via AWS's private network
2. It never touches the public internet
3. It's faster and more secure
  
  The API endpoint URLs in our configuration reveal the use of VPC Endpoints:
```json
"runner_api_endpoint": {
  "dev": "https://8hpmmddo24-vpce-0c5b5c7bf50bf2fc9-hv04ypip.execute-api.eu-central-1.amazonaws.com/prod"
}
```

The `vpce-0c5b5c7bf50bf2fc9` in the URL indicates this API is accessed through an Interface VPC Endpoint.
#### Application Load Balancer (ALB)
The ALB sits in front of our Fargate frontend service and handles:
1. **TLS termination**: The ALB holds our SSL certificate and handles HTTPS. Traffic between the ALB and Fargate containers can be HTTP (encrypted by VPC).
2. **Health checks**: The ALB periodically calls a health endpoint on each container. If a container fails health checks, the ALB stops sending it traffic and ECS replaces it.
3. **Traffic distribution**: If we have multiple Fargate tasks, the ALB distributes requests among them.
4. **Integration with AWS Certificate Manager**: Certificates are automatically renewed and deployed.

**ALB configuration in CDK**
```python
load_balanced_fargate_service = ecs_patterns.ApplicationLoadBalancedFargateService(
    self, "FrontendService",
    cluster=cluster,
    task_definition=task_definition,
    certificate=certificate,
    protocol=elbv2.ApplicationProtocol.HTTPS,
    redirect_http=True,  # Redirect HTTP to HTTPS
    public_load_balancer=True,  # ALB needs to be public to receive traffic
)
```

Note that while our Fargate tasks are in private subnets, the ALB itself is in public subnets (which must exist in the VPC for the ALB). The ALB is the only component with a public IP address.
#### AWS Certificate Manager (ACM)
ACM provides and manages SSL/TLS certificates. Our configuration includes certificate ARNs for each environment:
```json
"certificate_arn": {
  "dev": "arn:aws:acm:eu-central-1:732789817868:certificate/...",
  "test": "arn:aws:acm:eu-central-1:643356557982:certificate/...",
  "prod": "arn:aws:acm:eu-central-1:238788445602:certificate/..."
}
```

Each environment has its own certificate, likely for different domain names (e.g., dev.example.com, test.example.com, example.com).

ACM certificates are free and auto-renewing. ACM handles the certificate lifecycle—we never manually generate CSRs or install certificates.
### 1.5 Security Services
#### AWS KMS (Key Management Service)
KMS is a managed service for creating and controlling encryption keys. We use it extensively:

**Customer Managed Keys (CMKs)**
We create our own KMS keys rather than using AWS-managed keys. This gives us:
1. **Key policies**: Define exactly who can use, manage, and administer each key
2. **Audit logging**: All key usage is logged in CloudTrail
3. **Cross-account sharing**: Grant other accounts permission to use our keys
4. **Automatic rotation**: Keys can be automatically rotated annually

**Key usage in our architecture**
- **S3**: All buckets use KMS encryption for objects at rest
- **Athena**: Query results are encrypted with KMS
- **Secrets**: Any secrets stored in Systems Manager are encrypted with KMS
#### AWS IAM (Identity and Access Management)
IAM controls who can do what in AWS. Our IAM strategy follows the principle of least privilege—every identity gets only the permissions it needs.

**IAM Roles for compute services**
Each compute service has its own IAM role:
1. **ECS Task Role** (for the frontend):
   ```python
   # Permissions needed by the frontend application
   task_role.add_to_policy(iam.PolicyStatement(
       actions=["s3:GetObject", "s3:ListBucket"],
       resources=[input_bucket.bucket_arn, f"{input_bucket.bucket_arn}/*"]
   ))
   task_role.add_to_policy(iam.PolicyStatement(
       actions=["athena:StartQueryExecution", "athena:GetQueryResults"],
       resources=["*"]
   ))
   task_role.add_to_policy(iam.PolicyStatement(
       actions=["lambda:InvokeFunction"],
       resources=[simulation_lambda.function_arn]
   ))
   ```
2. **Lambda Execution Role** (for simulation functions):
   ```python
   # Permissions needed by simulation code
   execution_role.add_to_policy(iam.PolicyStatement(
       actions=["s3:GetObject", "s3:PutObject"],
       resources=[simulation_bucket.bucket_arn, f"{simulation_bucket.bucket_arn}/*"]
   ))
   execution_role.add_to_policy(iam.PolicyStatement(
       actions=["dynamodb:PutItem", "dynamodb:UpdateItem"],
       resources=[metadata_table.table_arn]
   ))
   ```

**IAM Roles for CI/CD**
The CI/CD pipeline uses cross-account role assumption. Here's how it works:
1. **CodeBuild Role** (in the central account):
	- Has permission to assume roles in target accounts
	- Has permission to read from CodeArtifact
	- Has permission to write logs to CloudWatch
2. **Cross-Account Deployment Role** (in each target account):
	   - Can be assumed by the CodeBuild role
	   - Has permission to deploy CloudFormation stacks
	   - Has permission to create/update all the resources we deploy

**Environment-specific role restrictions**
The `roles.py` stack defines different access levels per environment:
```python
def get_iren_de_roles(self) -> List[aws_iam.ArnPrincipal]:
    match self.stage:
        case "dev":
            # Development: broader access for experimentation
            return [
                iam.ArnPrincipal("arn:aws:iam::...:role/saml-adfs-iren-de-dev"),
                iam.ArnPrincipal("arn:aws:iam::...:role/AWSReservedSSO_ADVANA-DataEngineer_..."),
            ]
        case "test":
            # Test: similar to dev but different accounts
            return [...]
        case "prod":
            # Production: restricted to operations team only
            return [...]
```

This ensures that developers have broad access in development but limited access in production.
#### AWS STS (Security Token Service)
STS provides temporary, limited-privilege credentials. We use it for:
1. **Cross-account access**: The `do_assume_cross_account_role` function uses STS to get credentials for target accounts:
   ```bash
   aws sts assume-role \
       --role-arn arn:aws:iam::${TARGET_ACCOUNT}:role/${CA_ROLE} \
       --role-session-name $RoleSessName
   ```

	This returns temporary credentials (access key, secret key, session token) that are valid for a limited time (default: 1 hour).
2. **EC2 role assumption for local development**: Developers can assume roles via EC2 instance profiles:
   ```python
   def assume_ec2_role():
       if os.environ.get("EC2_ROLE_ARN"):
           sts = boto3.client("sts")
           response = sts.assume_role(
               RoleArn=role_arn, 
               RoleSessionName="ec2-test-session"
           )
           boto3.setup_default_session(
               aws_access_key_id=response['Credentials']['AccessKeyId'],
               aws_secret_access_key=response['Credentials']['SecretAccessKey'],
               aws_session_token=response['Credentials']['SessionToken']
           )
   ```
### 1.6 Container Services
#### Amazon ECR (Elastic Container Registry)
ECR is AWS's managed Docker registry. We store all our container images here.

**Repository per image**
We create a separate ECR repository for each type of image:
- `cash-flow-simulator-frontend`
- `cash-flow-simulator-simulation-lambda`
- `cash-flow-simulator-simulation-ecs`

**Lifecycle policies**
ECR can accumulate many images over time, increasing storage costs. We configure lifecycle policies to clean up:
```python
lifecycle_rules=[
    ecr.LifecycleRule(
        description="Keep only the last 10 images",
        max_image_count=10,
        rule_priority=1,
        tag_status=ecr.TagStatus.ANY,
    ),
]
```

This keeps only the 10 most recent images, automatically deleting older ones.

**ECR authentication**
Before pushing or pulling images, you must authenticate with ECR:
```bash
aws ecr get-login-password --region=${AWS_REGION} \
| docker login --username AWS --password-stdin \
    ${TARGET_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

This gets a temporary password from AWS and uses it to authenticate Docker with the ECR registry. The authentication is valid for 12 hours.
### 1.7 CI/CD Services
#### AWS CodeArtifact
CodeArtifact is a managed artifact repository that works with npm, pip, Maven, and other package managers.

**Why a private package registry?**
1. **Internal packages**: We have Python libraries that are used across multiple projects but shouldn't be published publicly.
2. **Caching**: CodeArtifact caches packages from public registries. If PyPI goes down, we can still install packages.
3. **Security scanning**: We can enforce policies about which packages are allowed.
4. **Audit trail**: We can see what packages are installed and who published what.

**Authentication with CodeArtifact**
The `do_login_code_artifact` function sets up authentication for both pip and npm:
```bash
function do_login_code_artifact() {
    # Get authorization token from CodeArtifact
    export CODEARTIFACT_AUTH_TOKEN=$(
        aws codeartifact get-authorization-token \
            --domain ${CODEARTIFACT_DOMAIN} \
            --domain-owner ${ACCOUNT_ID} \
            --query authorizationToken \
            --output text
    )
    # Configure pip
    export CODEARTIFACT_PIP_URL="https://aws:$pip_url/pypi/${CODEARTIFACT_REGISTRY}/simple/"
    pip config set global.index-url $CODEARTIFACT_PIP_URL
    # Configure npm
    npm config set registry=https://$npm_url/npm/${CODEARTIFACT_REGISTRY}/
    npm config set //$npm_url/npm/${CODEARTIFACT_REGISTRY}/:_authToken=${CODEARTIFACT_AUTH_TOKEN}
}
```

The token is valid for 12 hours and is used for all subsequent pip and npm operations.
#### AWS CodeBuild
CodeBuild is a managed build service that compiles code, runs tests, and produces deployable artifacts.

**How CodeBuild works**
1. CodeBuild spins up a fresh container for each build
2. It clones our source code
3. It runs commands defined in a buildspec file
4. It produces artifacts and reports

**Our buildspec files**
We have two buildspec files:
1. **`buildspec-synth.yml`**: Synthesizes CDK to CloudFormation and generates diffs
2. **`buildspec-deploy.yml`**: Actually deploys the infrastructure

**Build phases explained**
Each build goes through phases:
```yaml
phases:
  install:
    runtime-versions:
      python: 3.11
      nodejs: 18
    commands:
      - source BUILD/codebuild-helpers.sh
      - do_login_code_artifact
      - do_install_dependencies
  pre_build:
    commands:
      - do_assume_cross_account_role deploy
      - do_login_ecr
  build:
    commands:
      - do_cdk_operation deploy
```

**Install phase**: Sets up the build environment. We install:
- Python 3.11 and Node.js 18 runtimes
- CDK CLI via npm
- Python dependencies from requirements.cdk.txt
- git-remote-codecommit for pushing changes back to the repository

**Pre-build phase**: Prepares for deployment
- Assumes the cross-account IAM role to get credentials for the target account
- Authenticates Docker with ECR so we can push images

**Build phase**: Does the actual work
- For synth: Runs `cdk synth` to generate CloudFormation templates
- For deploy: Runs `cdk deploy` to deploy all stacks
#### AWS CodePipeline
CodePipeline orchestrates the release process. For production deployments, we use versioned pipelines.

When you run:
```bash
./scripts/infra.sh --stage prod --release 1.0.0
```

It creates (or triggers) a CodePipeline named `devops-cash-flow-simulator-MERCATO-PROD-v1-0-0`.

This pipeline:
1. Pulls the source code at the specified version
2. Runs the synth build
3. Waits for manual approval (if configured)
4. Runs the deploy build

The version number in the pipeline name provides traceability—you can always see exactly which version is deployed.
### 1.8 Messaging and Notification Services
#### Amazon SNS (Simple Notification Service)
SNS provides pub/sub messaging for notifications. We use it for:
1. **Failure alerts**: When a Lambda function fails or an ECS task crashes, an SNS message is sent
2. **Deployment notifications**: Team members can subscribe to be notified of deployments

**SNS Topic configuration**
```python
failure_topic = sns.Topic(
    self, "FailureNotificationTopic",
    display_name="Cash Flow Simulator Failures",
)

failure_topic.add_subscription(
    subscriptions.EmailSubscription(notification_email)
)
```

The email address comes from SSM Parameter Store, so it can be different per environment.
### 1.9 Monitoring Services
#### Amazon CloudWatch
CloudWatch is AWS's monitoring service. We use it for:

**CloudWatch Logs**
All our services write logs to CloudWatch:
- Lambda functions: `/aws/lambda/function-name`
- ECS tasks: Custom log groups defined in task definitions
- Athena queries: Query history and results

Logs are retained according to retention policies (e.g., 30 days for development, 365 days for production).

**CloudWatch Metrics**
We publish custom metrics and use AWS-provided metrics:
- Athena: Queries run, data scanned, execution time
- Lambda: Invocations, errors, duration, concurrent executions
- ECS: CPU utilization, memory utilization
- ALB: Request count, error rate, latency

**CloudWatch Alarms**
We can create alarms that trigger when metrics cross thresholds:
```python
alarm = cloudwatch.Alarm(
    self, "HighErrorRate",
    metric=lambda_function.metric_errors(),
    threshold=5,
    evaluation_periods=2,
    alarm_description="Lambda error rate is too high",
    actions=[sns_action],
)
```

This sends an SNS notification if the Lambda has more than 5 errors in two consecutive evaluation periods.
## Part 2: Infrastructure as Code with AWS CDK
### 2.1 Why CDK?
AWS CDK (Cloud Development Kit) is our infrastructure-as-code tool. It lets us define cloud resources using Python code instead of YAML/JSON templates.

**CDK vs. CloudFormation**
CloudFormation is AWS's native IaC service. You write YAML or JSON templates that describe your resources. CDK generates CloudFormation templates but lets you write in a real programming language.

Here's the same S3 bucket in CloudFormation YAML vs. CDK Python:

**CloudFormation:**
```yaml
Resources:
  DataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-data-bucket
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref MyKey
```

**CDK:**
```python
data_bucket = s3.Bucket(
    self, "DataBucket",
    bucket_name="my-data-bucket",
    encryption=s3.BucketEncryption.KMS,
    encryption_key=my_key,
)
```

The CDK version is more concise, but the real advantages come with complexity:

**Loops and conditionals:**
```python
for env in ["dev", "test", "prod"]:
    s3.Bucket(self, f"Bucket-{env}", bucket_name=f"data-{env}")
```

**Functions and abstraction:**
```python
def create_encrypted_bucket(scope, name, key):
    return s3.Bucket(scope, name, encryption=s3.BucketEncryption.KMS, encryption_key=key)
```

**Type checking and IDE support**: Your IDE understands the types and can provide autocomplete and error checking.
### 2.2 CDK Application Structure
Our CDK application is in the infra directory:
```
infra/
├── app.py              # Entry point, creates all stacks
├── cdk.json            # CDK configuration
├── cdk.context.json    # Cached context values
├── stacks/
│   ├── athena.py       # Athena workgroup
│   ├── frontend.py     # ECS Fargate frontend
│   ├── roles.py        # IAM roles
│   ├── shared.py       # Shared resources (SNS, etc.)
│   ├── simulation.py   # Lambda and ECS simulation
│   ├── simulation_storage.py  # Simulation S3 and Glue
│   └── storage.py      # Input/output S3 buckets
└── utils/
    └── ecr.py          # Custom ECR construct
```
### 2.3 The Entry Point: app.py
The app.py file is the entry point for CDK. Here's a simplified version of what it does:
```python
#!/usr/bin/env python3
import aws_cdk as cdk
from stacks.athena import AthenaStack
from stacks.frontend import FrontendStack
from stacks.roles import RolesStack
from stacks.shared import SharedStack
from stacks.simulation import SimulationStack
from stacks.simulation_storage import SimulationStorageStack
from stacks.storage import StorageStack

app = cdk.App()

# Read context values (passed via -c flags or cdk.json)
stage = app.node.try_get_context("STAGE")
region = app.node.try_get_context("REGION")
account = app.node.try_get_context("TARGET_ACCOUNT")

# Define environment
env = cdk.Environment(account=account, region=region)

# Create stacks
stacks = {}
stacks["storage"] = StorageStack(app, f"storage-{stage}", env=env)
stacks["simulation_storage"] = SimulationStorageStack(app, f"sim-storage-{stage}", env=env)
stacks["athena"] = AthenaStack(app, f"athena-{stage}", env=env)
stacks["roles"] = RolesStack(app, f"roles-{stage}", env=env)
stacks["shared"] = SharedStack(app, f"shared-{stage}", env=env)
stacks["frontend"] = FrontendStack(app, f"frontend-{stage}", env=env)
stacks["simulation"] = SimulationStack(app, f"simulation-{stage}", env=env)

# Define stack dependencies
stacks["frontend"].add_dependency(stacks["roles"])
stacks["simulation"].add_dependency(stacks["roles"])
stacks["simulation"].add_dependency(stacks["shared"])

# Synthesize the app
app.synth()
```

**Stack dependencies**
Dependencies ensure stacks are deployed in the correct order. When we say `frontend.add_dependency(roles)`, CDK knows that the roles stack must be deployed before the frontend stack.

This matters because the frontend stack references IAM roles created in the roles stack. If we tried to deploy frontend first, it would fail because the roles don't exist yet.
### 2.4 Custom Constructs: AutoDeployingCachedECR
One of our most sophisticated custom constructs is `AutoDeployingCachedECR`. Let me walk through what it does:
```python
class AutoDeployingCachedECR(Construct):
    def __init__(
        self,
        scope: Construct,
        id: str,
        ecr_repository_name: str,
        dockerfile_path: str,
        build_context_path: str,
        **kwargs
    ):
        super().__init__(scope, id, **kwargs)
        self.ecr_repository_name = ecr_repository_name
        self.dockerfile_path = dockerfile_path
        self.build_context_path = build_context_path
        self.create()
    def create(self) -> None:
        self.create_ecr_repository()
        self.create_cdk_image_deploy_policy()
        self.create_docker_image_asset()
        self.create_docker_image_deploy_and_dependency_chain()
```

**Step 1: Create ECR Repository**
```python
def create_ecr_repository(self):
    self.repository = ecr.Repository(
        self, "Repository",
        repository_name=self.ecr_repository_name,
        removal_policy=RemovalPolicy.DESTROY,  # Delete on stack deletion
        lifecycle_rules=[
            ecr.LifecycleRule(
                max_image_count=10,  # Keep only 10 images
            ),
        ],
    )
```

This creates an ECR repository with lifecycle rules to control costs.

**Step 2: Create IAM Policy for CDK's Image Deployment Role**
CDK's image deployment process runs in a separate role. We need to grant it permission to push images:
```python
def create_cdk_image_deploy_policy(self):
    # CDK uses a bootstrap role for image deployment
    cdk_image_deploy_role = iam.Role.from_role_arn(
        self, "CdkImageDeployRole",
        f"arn:aws:iam::{self.account}:role/cdk-{qualifier}-image-deploy-role-{self.account}-{self.region}"
    )
    # Grant push permissions
    self.repository.grant_pull_push(cdk_image_deploy_role)
```

**Step 3: Create Docker Image Asset**
```python
def create_docker_image_asset(self):
    repo_url = self.repository.repository_uri
    cache_url = f"{repo_url}:cache"
    self.docker_image = aws_ecr_assets.DockerImageAsset(
        self, "DockerImage",
        directory=self.build_context_path,
        file=self.dockerfile_path,
        platform=aws_ecr_assets.Platform.LINUX_AMD64,
        cache_from=[
            aws_ecr_assets.DockerCacheOption(
                type="registry",
                params={"ref": cache_url},
            )
        ],
        cache_to=aws_ecr_assets.DockerCacheOption(
            type="registry",
            params={"ref": cache_url, "mode": "max"},
        ),
        build_secrets={
            "CODEARTIFACT_PIP_URL": self.codeartifact_pip_url,
        },
    )
```

The key features here:

**Registry-based caching**: The `cache_from` and `cache_to` parameters tell Docker to use the ECR repository itself as a cache. When we build:
1. Docker checks if the cache tag exists in ECR
2. If it does, Docker pulls the cached layers
3. Docker only rebuilds layers that have changed
4. After building, Docker pushes the new cache

This dramatically speeds up builds. A full build might take 10 minutes, but a cached build (where only application code changed) might take 30 seconds.

**Build secrets**: The `build_secrets` parameter passes the CodeArtifact URL as a BuildKit secret. This is accessible in the Dockerfile via `--mount=type=secret,id=CODEARTIFACT_PIP_URL`.

**Step 4: Deploy Image to ECR**
```python
def create_docker_image_deploy_and_dependency_chain(self):
    self.image_deploy = ECRDeployment(
        self, "ImageDeploy",
        src=DockerImageName(self.docker_image.image_uri),
        dest=DockerImageName(f"{self.repository.repository_uri}:latest"),
    )
    # Ensure deployment happens after the image is built
    self.image_deploy.node.add_dependency(self.docker_image)
```

This uses the `cdk-ecr-deployment` library to copy the image from CDK's asset bucket to our ECR repository with the `latest` tag.
### 2.5 Stack Communication
CDK stacks often need to share information. For example, the simulation stack needs to know the S3 bucket names created by the storage stack.

**Cross-stack references**
CDK handles this automatically when you pass constructs between stacks:
```python
# In storage.py
class StorageStack(Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        self.input_bucket = s3.Bucket(self, "InputBucket")
        
# In simulation.py
class SimulationStack(Stack):
    def __init__(self, scope, id, input_bucket: s3.IBucket, **kwargs):
        super().__init__(scope, id, **kwargs)
        # Use the bucket from the other stack
        self.lambda_function.add_environment("INPUT_BUCKET", input_bucket.bucket_name)
```

When you reference `input_bucket.bucket_name` in a different stack, CDK automatically creates a CloudFormation Export in the storage stack and an Import in the simulation stack.

**Context values**
For configuration that isn't tied to specific resources, we use CDK context:
```python
# In app.py or via -c flag
stage = app.node.try_get_context("STAGE")

# In a stack
class MyStack(Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        stage = self.node.try_get_context("STAGE")
        if stage == "prod":
            # Production-specific configuration
            instance_count = 3
        else:
            instance_count = 1
```
### 2.6 CDK Configuration Files
**cdk.json**
This is the main CDK configuration file:
```json
{
  "app": "python app.py",
  "context": {
    "@aws-cdk/core:stackRelativeExports": true,
    "runner_api_endpoint": {
      "dev": "https://...",
      "test": "https://...",
      "prod": "https://..."
    },
    "certificate_arn": {
      "dev": "arn:aws:acm:...",
      "test": "arn:aws:acm:...",
      "prod": "arn:aws:acm:..."
    }
  }
}
```

The app field tells CDK how to run the app. The `context` field provides static configuration values that don't change based on deployed state.

**cdk.context.json**
This file caches values that CDK looks up from AWS:
```json
{
  "vpc-provider:account=732789817868:filter.vpc-id=vpc-0e4d359a4c9ac1a5e:region=eu-central-1": {
    "vpcId": "vpc-0e4d359a4c9ac1a5e",
    "vpcCidrBlock": "10.224.12.0/22",
    "availabilityZones": ["eu-central-1a", "eu-central-1b", "eu-central-1c"],
    ...
  }
}
```

When you use `ec2.Vpc.from_lookup()`, CDK queries AWS for VPC details. To avoid querying every time (which would be slow and require credentials during synthesis), CDK caches the results.

This file should be committed to version control so that CI/CD builds don't need to query AWS for these values.
## Part 3: The Deployment Pipeline in Detail
### 3.1 Pipeline Architecture
Our deployment pipeline follows this flow:
```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   Developer    │────▶│   CodeCommit   │────▶│   CodeBuild    │
│   commits code │     │   (Git repo)   │     │   (Synth)      │
└────────────────┘     └────────────────┘     └────────────────┘
                                                      │
                                                      ▼
                                              ┌────────────────┐
                                              │   Diff file    │
                                              │   committed    │
                                              └────────────────┘
                                                      │
                                                      ▼
                                              ┌────────────────┐
                                              │   CodeBuild    │
                                              │   (Deploy)     │
                                              └────────────────┘
                                                      │
                                                      ▼
                                              ┌────────────────┐
                                              │     AWS        │
                                              │   Resources    │
                                              └────────────────┘
```
### 3.2 The Build Scripts in Detail
The codebuild-helpers.sh file contains all the functions used during builds. Let me explain each one:

**Environment variable validation**
```bash
function do_check_environment_variables() {
    echo "BRANCH_NAME = ${BRANCH_NAME}"
    echo "BUSINESS_UNIT = ${BUSINESS_UNIT}"
    echo "PROJECT_NAME = ${PROJECT_NAME}"
    echo "STAGE = ${STAGE}"
    echo "PYTHON_RUNTIME = ${PYTHON_RUNTIME}"
    echo "NODEJS_RUNTIME = ${NODEJS_RUNTIME}"
    echo "CDK_VERSION = ${CDK_VERSION}" 
    echo "TARGET_ACCOUNT = ${TARGET_ACCOUNT}"
    echo "AWS_DEFAULT_REGION = ${AWS_DEFAULT_REGION}"
    echo "CA_ROLE = ${CA_ROLE}"
    echo "BRANCH_NAME_FOR_ID = ${BRANCH_NAME_FOR_ID}"
    echo "VPC_ID = ${VPC_ID}"
    echo "DEFAULT_SECURITY_GROUP_ID = ${DEFAULT_SECURITY_GROUP_ID}"
    echo "VPC_SUBNET_ID_1 = ${VPC_SUBNET_ID_1}"
    echo "VPC_SUBNET_ID_2 = ${VPC_SUBNET_ID_2}"
    echo "VPC_SUBNET_ID_3 = ${VPC_SUBNET_ID_3}"
    echo "WBS = ${WBS}"
}
```

This prints all environment variables for debugging. When a build fails, the logs show exactly what configuration was used.

**CodeArtifact authentication**
```bash
function do_login_code_artifact() {
    set -e  # Exit on any error
    # Get authorization token from CodeArtifact
    export CODEARTIFACT_AUTH_TOKEN=$(
        aws codeartifact get-authorization-token \
            --domain ${CODEARTIFACT_DOMAIN} \
            --domain-owner ${ACCOUNT_ID} \
            --query authorizationToken \
            --output text
    )
    # Build the pip and npm URLs
    local pip_url="${CODEARTIFACT_AUTH_TOKEN}@${CODEARTIFACT_DOMAIN}-${ACCOUNT_ID}.d.codeartifact.eu-central-1.amazonaws.com"
    local npm_url="${CODEARTIFACT_DOMAIN}-${ACCOUNT_ID}.d.codeartifact.eu-central-1.amazonaws.com"
    # Configure pip to use CodeArtifact
    export CODEARTIFACT_PIP_URL="https://aws:$pip_url/pypi/${CODEARTIFACT_REGISTRY}/simple/"
    pip config set global.index-url $CODEARTIFACT_PIP_URL
    # Configure npm to use CodeArtifact
    npm config set registry=https://$npm_url/npm/${CODEARTIFACT_REGISTRY}/
    npm config set //$npm_url/npm/${CODEARTIFACT_REGISTRY}/:_authToken=${CODEARTIFACT_AUTH_TOKEN}
}
```

Let me break down the pip URL format:
```
https://aws:$pip_url/pypi/${CODEARTIFACT_REGISTRY}/simple/
```
- `https://aws:` - The username is literally "aws" (it's a placeholder)
- `$pip_url` - Contains the auth token and domain
- `/pypi/` - Indicates this is a PyPI repository
- `${CODEARTIFACT_REGISTRY}` - The name of the repository within CodeArtifact
- `/simple/` - The PEP 503 simple API endpoint
  
**ECR authentication**
```bash
function do_login_ecr() {
    aws ecr get-login-password --region=${AWS_REGION} \
    | docker login \
        --username AWS \
        --password-stdin \
        ${TARGET_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
}
```

This pipes the ECR password directly into docker login via stdin, which is more secure than passing it as a command-line argument (which would be visible in process listings).

**Cross-account role assumption**
```bash
function do_assume_cross_account_role() {
    set -e
    local CdkOp="${1^}"  # Capitalize first letter (deploy -> Deploy)
    # Build the role ARN
    local RoleArn="arn:aws:iam::${TARGET_ACCOUNT}:role/${CA_ROLE}"
    # Create unique session name with timestamp
    local DateNow=$(date '+%y%m%d%H%M%S%5N')
    local RoleSessName="CaCBuild${CdkOp}-${STAGE}-${BRANCH_NAME_FOR_ID}-$DateNow"
    # Assume the role
    export ASSUMED_ROLE=$(
        aws sts assume-role \
            --role-arn $RoleArn \
            --role-session-name $RoleSessName \
            --endpoint-url https://sts.${AWS_DEFAULT_REGION}.amazonaws.com
    )
    # Extract credentials from the response
    export AWS_ACCESS_KEY_ID=$(
        echo "${ASSUMED_ROLE}" | jq -r '.Credentials.AccessKeyId'
    )
    export AWS_SECRET_ACCESS_KEY=$(
        echo "${ASSUMED_ROLE}" | jq -r '.Credentials.SecretAccessKey'
    )
    export AWS_SESSION_TOKEN=$(
        echo "${ASSUMED_ROLE}" | jq -r '.Credentials.SessionToken'
    )
    export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
}
```

Important details:
- **`${1^}`**: Bash syntax to capitalize the first letter. This makes log messages clearer.
- **Unique session name**: Includes timestamp with nanoseconds to ensure uniqueness. This is required by AWS and also helps with CloudTrail auditing.
- **Regional STS endpoint**: We specify `--endpoint-url https://sts.${AWS_DEFAULT_REGION}.amazonaws.com` to use the regional endpoint. This is faster and avoids the global endpoint in us-east-1.
- **`jq`**: A command-line JSON processor. It extracts specific fields from the STS response.

**Dependency installation**
```bash
function do_install_dependencies() {
    set -e
    # Install CDK globally
    echo "Installing aws-cdk with npm..."
    cd infra
    npm install -g aws-cdk@${CDK_VERSION}
    cd ..
    # Install Python dependencies
    echo "Installing dependencies with pip3..."
    pip3 install --upgrade --no-cache --no-cache-dir -r requirements.cdk.txt
    # Install CodeCommit helper
    pip3 install --upgrade git-remote-codecommit
}
```
- **`--no-cache --no-cache-dir`**: Don't use or create a pip cache. In CI, each build starts fresh, so caching locally is pointless and wastes disk space.
- **`git-remote-codecommit`**: A Git remote helper that allows Git to authenticate with CodeCommit using IAM credentials instead of HTTPS passwords.

**CDK operations**
```bash
function do_cdk_operation() {
    set -e
    local CdkOp="${1}"  # synth, deploy, or diff
    # Build the CDK parameters
    local CdkParams="
        -c REGION=${AWS_DEFAULT_REGION}
        -c project_name=${PROJECT_NAME}
        -c business_unit=${BUSINESS_UNIT}
        -c STAGE=${STAGE}
        -c TARGET_ACCOUNT=${TARGET_ACCOUNT}
        -c CODEARTIFACT_AUTH_TOKEN=${CODEARTIFACT_AUTH_TOKEN}
        -c CODEARTIFACT_DOMAIN=${CODEARTIFACT_DOMAIN}
        -c CODEARTIFACT_REGISTRY=${CODEARTIFACT_REGISTRY}
        -c CODEARTIFACT_ACCOUNT=${CODEARTIFACT_ACCOUNT}
        -c BRANCH_NAME=${BRANCH_NAME}
        -c VPC_ID=${VPC_ID}
        -c DEFAULT_SECURITY_GROUP_ID=${DEFAULT_SECURITY_GROUP_ID}
        -c VPC_SUBNET_ID_1=${VPC_SUBNET_ID_1}
        -c VPC_SUBNET_ID_2=${VPC_SUBNET_ID_2}
        -c VPC_SUBNET_ID_3=${VPC_SUBNET_ID_3}
        -c WBS=${WBS}
    "
    local DiffFile="../Diff_files/cdk-diff-${BUSINESS_UNIT}-${STAGE}.txt"
    cd infra
    npx cdk --version  # Log CDK version for debugging
    case $CdkOp in
        synth)
            npx cdk $CdkOp $CdkParams
            ;;
        deploy)
            # Setup Docker BuildKit for optimized builds
            export DOCKER_BUILDKIT=1
            docker buildx install || echo "ERROR: Could not install buildx."
            docker buildx create \
                --name container \
                --driver=docker-container \
                --driver-opt "default-load=true" \
            || echo "ERROR: Could not create container driver."
            docker buildx use container || echo "ERROR: Could not use container driver."
            # Deploy all stacks
            npx cdk $CdkOp --all -v --require-approval never $CdkParams
            ;;
        diff)
            # Record timestamp and diff output
            echo $(date +%Y-%m-%d-%H-%M-%S-%N) > $DiffFile
            npx cdk $CdkOp $CdkParams --no-color &>> $DiffFile
            ;;
    esac
    cd ..
}
```

**Docker BuildKit setup explained**:
1. **`export DOCKER_BUILDKIT=1`**: Enables BuildKit, Docker's next-generation build system with improved caching and parallel builds.
2. **`docker buildx install`**: Installs buildx as the default builder.
3. **`docker buildx create --driver=docker-container`**: Creates a builder that runs in a container. This is necessary for some advanced features like multi-platform builds and better cache handling.
4. **`--driver-opt "default-load=true"`**: Automatically loads built images into the local Docker daemon.

**CDK command flags**:
- **`--all`**: Deploy all stacks (not just one)
- **`-v`**: Verbose output
- **`--require-approval never`**: Don't prompt for approval (we're in CI/CD, no human to approve)
- **`--no-color`**: For diff output, don't use ANSI color codes (they look like garbage in log files)
### 3.3 Diff Tracking
