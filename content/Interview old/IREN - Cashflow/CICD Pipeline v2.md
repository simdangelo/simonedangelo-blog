---
modified: 2026-01-22T22:55:01+01:00
---
## What is CI/CD and Why Does It Matter?
Before diving into the specifics of this system, let's establish what CI/CD actually means. CI/CD stands for **Continuous Integration** and **Continuous Deployment** (or Continuous Delivery). These are practices that automate the process of taking code written by developers and safely deploying it to production environments.

> **Continuous Integration** means that every time a developer writes code and pushes it to the repository, that code is automatically validated—checked for errors, tested, and built. The idea is to catch problems early, when they're easier to fix.

> **Continuous Deployment** takes it further: once code passes all validation, it's automatically deployed to the target environment (like a test server or production). The human doesn't have to manually copy files, run commands, or configure servers. The pipeline does it all.

In traditional software development, deployment was a manual, error-prone process. Someone would download code, run build commands, copy files to a server, restart services, and hope nothing broke. This approach doesn't scale—it's slow, inconsistent, and risky. CI/CD solves this by making deployments **automated, repeatable, and auditable**.
## The Big Picture: What Happens When Code Changes?
Let me walk you through the complete journey of a code change, from the moment a developer writes a line of code to when users see that change in production. Understanding this flow is essential before diving into technical details.

**The Story of a Feature:**
1. A developer writes new code on their local machine
2. They push that code to a central repository (**CodeCommit**)
3. The push triggers an automated pipeline (**CodePipeline**)
4. The pipeline downloads the code and runs validation (**CodeBuild**)
5. If validation passes, the pipeline deploys the changes (**CDK deploy**)
6. Users see the new feature in the application

This entire process can happen automatically, or it can pause for human approval at key points. In this system:
- **dev** environment **deploys automatically**
- but **test** and **production** require **explicit approval**.
## The AWS Services: What Each Tool Does
This system uses several AWS services that work together. Let me explain each one in plain terms.
### CodeCommit: Where the Code Lives
**CodeCommit** is AWS's version control service—think of it as AWS's version of GitHub or GitLab. It stores all the project's code in a central location where:
- Multiple developers can collaborate
- Every change is tracked with history
- Different versions (branches) can exist simultaneously
- Changes trigger automated pipelines

When a developer runs `git push`, the code goes to CodeCommit. CodeCommit then notifies other services that new code is available.
### CodePipeline: The Orchestrator
**CodePipeline** is the conductor of the deployment orchestra. It doesn't do any actual work itself—instead, it coordinates other services in a sequence of steps called "stages." Think of it like a recipe:
1. First, get the ingredients (download code from CodeCommit)
2. Then, prepare the ingredients (run validation in CodeBuild)
3. Wait for the chef to approve (manual approval gate)
4. Finally, cook the meal (deploy with CodeBuild)

Each pipeline in this system has a specific name format:
```
devops-cash-flow-simulator-MERCATO-PROD-v0-7-5
       └──────────┬─────────┘ └──┬──┘└─┬─┘└──┬──┘
            Project name     Business  Env   Version
                              Unit
```

This naming tells you exactly what the pipeline does: it deploys version 0.7.5 of the cash-flow-simulator project to the PROD environment.
### CodeBuild: The Worker
**CodeBuild** is where actual work happens. It's a service that:
1. Spins up a temporary Linux machine
2. Downloads your code
3. Runs whatever commands you specify
4. Shuts down when done

You don't manage servers with CodeBuild—AWS handles that. You just tell it what commands to run, and it executes them in a clean, isolated environment.

The commands CodeBuild runs are defined in files called **buildspecs** (we'll explore these in detail). In this system, there are two buildspecs:
- `buildspec-synth.yml` — For **validating and previewing changes**
- `buildspec-deploy.yml` — For actually **deploying change**
### SSM Parameter Store: The Configuration Database
**SSM Parameter Store** is a simple key-value database where configuration values are stored. Instead of hardcoding values like "which AWS account is production?" or "what version of Python should we use?", these values live in Parameter Store.

Why is this useful? Because:
- The same code can deploy to different environments by reading different parameters
- You can change configuration without changing code
- Secrets (like passwords) can be stored securely
- Changes are auditable—you can see who changed what and when

For example, this system stores parameters like:
```
/MERCATO/devops-cash-flow-simulator/PROD/versions/CDK_VERSION → "2.149.0"
/MERCATO/devops-cash-flow-simulator/PROD/CA_ROLE → "advana-ca-codebuild-execution"
/GLOBAL/MERCATO/PROD/account → "238788445602"
```
### CDK: Infrastructure as Code
**CDK (Cloud Development Kit)** is a tool that lets you define AWS infrastructure using programming languages like Python. Instead of clicking through the AWS Console to create services, you write Python code that describes what you want, and CDK creates it for you.

For example, instead of manually creating an S3 bucket through the AWS website, you write:
```python
bucket = s3.Bucket(self, "MyBucket", bucket_name="my-data-bucket")
```

CDK then translates this into CloudFormation templates (AWS's native infrastructure language) and creates the bucket. The beauty is that this code is version-controlled, reviewable, and reproducible.
### ECR: Docker Image Storage
**ECR (Elastic Container Registry)** is where Docker images are stored. Think of Docker images as packages containing your application and everything it needs to run. ECR is like a warehouse for these packages—you push images there, and later, services like ECS pull them to run your application.
## The Multi-Account Strategy: Why Three AWS Accounts?
This system doesn't deploy everything to a single AWS account. Instead, it uses **four separate AWS accounts**:

| Account          | ID           | Purpose                  |
| ---------------- | ------------ | ------------------------ |
| advana (central) | 656332797039 | Runs the CI/CD pipelines |
| dev              | 732789817868 | Development environment  |
| test             | 643356557982 | Testing environment      |
| prod             | 238788445602 | Production environment   |
**Why separate accounts?**
Imagine you're building a house. You wouldn't want the construction equipment in your living room, and you definitely wouldn't want workers testing electrical wiring while you're sleeping. Similarly:
1. **Isolation**: If something goes wrong in dev, it can't affect production. They're completely separate AWS accounts with no shared resources.
2. **Security**: Production data and credentials are only accessible from the production account. A developer with dev access can't accidentally (or maliciously) access production data.
3. **Cost Tracking**: You can see exactly how much each environment costs. Maybe dev is using expensive resources unnecessarily.
4. **Compliance**: Auditors can verify that production changes go through proper approval processes.

The **central "advana" account** is special—it's where the pipelines run. This account doesn't host any application resources (no databases, no servers). It only orchestrates deployments to the other accounts.
## Cross-Account Deployment: How the Pipeline Reaches Target Accounts
Here's a crucial question: if the pipeline runs in the advana account (656332797039), how does it create resources in the production account (238788445602)?

The answer is **STS AssumeRole**—a mechanism where one AWS identity temporarily "becomes" another identity with different permissions.
### How STS AssumeRole Works
1. The pipeline runs in the advana account with its own identity
2. When it needs to deploy to production, it calls AWS STS (Security Token Service)
3. STS checks if the advana account is allowed to assume a role in the production account
4. If allowed, STS returns temporary credentials (valid for a limited time)
5. The pipeline uses these temporary credentials for all subsequent operations

The `codebuild-helpers.sh` file contains the function that does this:
```bash
function do_assume_cross_account_role() {
    set -e
    local CdkOp="${1^}"  # "Deploy" or "Synth"
    local RoleArn="arn:aws:iam::${TARGET_ACCOUNT}:role/${CA_ROLE}"
    local DateNow=$(date '+%y%m%d%H%M%S%5N')
    local RoleSessName="CaCBuild${CdkOp}-${STAGE}-${BRANCH_NAME_FOR_ID}-$DateNow"
    
    export ASSUMED_ROLE=$(
        aws sts assume-role                    \
            --role-arn $RoleArn                \
            --role-session-name $RoleSessName  \
            --endpoint-url https://sts.${AWS_DEFAULT_REGION}.amazonaws.com
    )
    
    export AWS_ACCESS_KEY_ID=$(echo "${ASSUMED_ROLE}" | jq -r '.Credentials.AccessKeyId')
    export AWS_SECRET_ACCESS_KEY=$(echo "${ASSUMED_ROLE}" | jq -r '.Credentials.SecretAccessKey')
    export AWS_SESSION_TOKEN=$(echo "${ASSUMED_ROLE}" | jq -r '.Credentials.SessionToken')
}
```

Let me explain what this code does line by line:
1. `local RoleArn="arn:aws:iam::${TARGET_ACCOUNT}:role/${CA_ROLE}"` — Constructs the full identifier of the role to assume. `TARGET_ACCOUNT` might be `238788445602` (prod), and `CA_ROLE` is `advana-ca-codebuild-execution`.
2. `local RoleSessName="CaCBuild${CdkOp}-${STAGE}-${BRANCH_NAME_FOR_ID}-$DateNow"` — Creates a unique session name like `CaCBuildDeploy-PROD-v0-7-5-2601201430123456789`. This name appears in AWS CloudTrail logs, so if you're investigating "who made this change?", you can trace it back to exactly which pipeline execution.
3. `aws sts assume-role` — Calls the AWS Security Token Service to request temporary credentials.
4. The `export` commands extract the access key, secret key, and session token from the response, making them available to subsequent commands.

After this function runs, all AWS commands (like `cdk deploy`) will execute as if they're running in the target account, with the permissions of the assumed role.
### The Role Name: advana-ca-codebuild-execution
The role `advana-ca-codebuild-execution` exists in each target account (dev, test, prod). This role:
- Trusts the advana account to assume it
- Has permissions to create CloudFormation stacks
- Can push Docker images to ECR
- Can read/write to S3 and other services

This role was created by the platform team and is shared across all projects deploying from the central advana account.
## The Buildspec Files: Detailed Walkthrough
A **buildspec** is a YAML file that tells CodeBuild exactly what to do. Let's examine `buildspec-synth.yml` piece by piece.
### The Header Section
```yaml
version: 0.2
run-as: root
env:
  git-credential-helper: yes
  shell: bash
  variables:
    SOURCE_BASEDIR: "."
    BASH_ENV: "$CODEBUILD_SRC_DIR/BUILD/codebuild-helpers.sh"
```

- `version: 0.2` — Tells CodeBuild which buildspec format we're using
- `run-as: root` — Commands run as the root user (necessary for installing packages)
- `git-credential-helper: yes` — CodeBuild automatically handles Git authentication to CodeCommit
- `BASH_ENV: codebuild-helpers.sh"` — This is clever: by setting BASH_ENV, bash automatically loads this script, making all functions defined in it available throughout the build
### Parameter Store Section
```yaml
  parameter-store:
    PYTHON_RUNTIME: "/$PREFIX/${BUSINESS_UNIT}/${PROJECT_NAME}/${STAGE}/versions/PYTHON_RUNTIME"
    CDK_VERSION: "/$PREFIX/${BUSINESS_UNIT}/${PROJECT_NAME}/${STAGE}/versions/CDK_VERSION"
    TARGET_ACCOUNT: "/GLOBAL/${BUSINESS_UNIT}/${STAGE}/account"
    CA_ROLE: "/$PREFIX/${BUSINESS_UNIT}/${PROJECT_NAME}/${STAGE}/CA_ROLE"
    CODEARTIFACT_DOMAIN: "/$PREFIX/${BUSINESS_UNIT}/${PROJECT_NAME}/CODE_ARTIFACT_DOMAIN"
    VPC_ID: "/GLOBAL/${BUSINESS_UNIT}/${STAGE}/vpc/default_vpc"
    # ... more parameters
```

This section tells CodeBuild to fetch values **from SSM Parameter Store** and set them as environment variables. The path uses variables like `${STAGE}` which are set by the pipeline when it invokes CodeBuild.

For a production deployment, `${STAGE}` would be `PROD`, so the parameter path becomes:
```
/devops/MERCATO/devops-cash-flow-simulator/PROD/versions/CDK_VERSION
```

And CodeBuild fetches whatever value is stored there (e.g., "2.149.0") and sets it as the `CDK_VERSION` environment variable.
### The Install Phase
```yaml
phases:
  install:
    on-failure: ABORT
    runtime-versions:
      python: $PYTHON_RUNTIME
      nodejs: $NODEJS_RUNTIME
    commands:
      - echo "Checking Environment Variables..."
      - do_check_environment_variables
      - echo "Performing Codeartifact Login..."
      - do_login_code_artifact
      - echo "Installing required packages..."
      - do_install_dependencies
```

The install phase sets up the environment:
1. `runtime-versions` — CodeBuild machines come with multiple Python and Node.js versions installed. This specifies which to use. The value comes from Parameter Store, allowing different environments to use different versions without code changes.
2. `do_check_environment_variables` — This function (from `codebuild-helpers.sh`) prints all environment variables. It's not strictly necessary, but it's invaluable for debugging—if something fails, the logs show exactly what configuration was used.
3. `do_login_code_artifact` — This authenticates to AWS CodeArtifact, a private package repository. Let me explain why this matters.
### Understanding CodeArtifact
When you run `pip install pandas`, pip downloads the package from PyPI (Python Package Index), a public repository. But organizations often have internal packages that shouldn't be public. **CodeArtifact** is AWS's private package repository.

This system has an internal library called `advanalib` that contains shared code. It can't be on PyPI because it contains organization-specific code. Instead, it's published to CodeArtifact.

The `do_login_code_artifact` function:
```bash
function do_login_code_artifact() {
    set -e
    export CODEARTIFACT_AUTH_TOKEN=$(
        aws codeartifact get-authorization-token  \
            --domain ${CODEARTIFACT_DOMAIN}       \
            --domain-owner ${ACCOUNT_ID}          \
            --query authorizationToken            \
            --output text
    )
    local pip_url="${CODEARTIFACT_AUTH_TOKEN}@${CODEARTIFACT_DOMAIN}-${ACCOUNT_ID}.d.codeartifact.eu-central-1.amazonaws.com"
    export CODEARTIFACT_PIP_URL="https://aws:$pip_url/pypi/${CODEARTIFACT_REGISTRY}/simple/"
    pip config set global.index-url $CODEARTIFACT_PIP_URL
}
```

This:
1. Gets an authentication token from CodeArtifact (valid for 12 hours)
2. Constructs a URL that includes the token
3. Configures pip to use this URL instead of PyPI

After this runs, `pip install advanalib` works because pip knows where to find the private package and has credentials to access it.
### The Pre-Build Phase
```yaml
  pre_build:
    on-failure: ABORT
    commands:
      - echo "Checking installed software..."
      - node --version
      - python --version
      - aws --version
      - echo "Checking AWS CLI Credentials..."
      - aws sts get-caller-identity
      - echo "Assuming Cross Account Role..."
      - do_assume_cross_account_role synth
      - echo "Checking Assumed Role Credentials..."
      - aws sts get-caller-identity
```

This phase prepares for the actual build:
1. Version checks — Logs which versions are installed for debugging
2. `aws sts get-caller-identity` — Shows who the build is running as (the CodeBuild service role)
3. `do_assume_cross_account_role synth` — Assumes the role in the target account
4. Second `aws sts get-caller-identity` — Confirms we're now operating as the assumed role

The second identity check is important. It verifies the role assumption worked. If you see the same identity twice, something went wrong.
### The Build Phase (Synth)
```yaml
  build:
    on-failure: ABORT
    commands:
      - echo "Logging into ECR..."
      - do_login_ecr
      - echo "Checking Stacks Difference..."
      - do_cdk_operation synth
      - echo "Producing Diff File..."
      - do_cdk_operation diff
      - echo "Committing Diff File..."
      - git config --global user.email "codebuild-${STAGE}-${BRANCH}@IREN.it"
      - git checkout ${BRANCH_NAME}
      - git branch --set-upstream-to=origin/${BRANCH_NAME} ${BRANCH_NAME}
      - git add ./Diff_files/cdk-diff-${BUSINESS_UNIT}-$STAGE.txt
      - git commit -m "(CodeBuild) Commit of diff file ($STAGE)"
      - git config pull.rebase false
      - git pull --ff --no-edit
      - git push -u origin ${BRANCH_NAME}
```

This is where the actual work happens. Let me explain each step:
####  1. `do_login_ecr`
ECR requires authentication before you can push or pull images. This function:
```bash
function do_login_ecr() {
    aws ecr get-login-password --region=${AWS_REGION} \
    | docker login --username AWS --password-stdin \
        ${TARGET_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
}
```

It gets a password from ECR and pipes it to `docker login`. After this, Docker commands can pull/push images from/to ECR.
#### 2. `do_cdk_operation synth`
This runs CDK synthesis. Let's understand what that means.

CDK works in two steps:
1. **Synthesis** — Your Python code runs, and CDK generates CloudFormation templates
2. **Deploy** — CloudFormation applies those templates to create/update resources

Synthesis is like creating a blueprint. No actual resources are created—it's just generating the plan. This is safe to run because nothing changes in AWS yet.

The `do_cdk_operation` function (relevant parts):
```bash
function do_cdk_operation() {
    local CdkOp="${1}"  # deploy, synth or diff
    local CdkParams="                                              \
        -c REGION=${AWS_DEFAULT_REGION}                            \
        -c project_name=${PROJECT_NAME}                            \
        -c business_unit=${BUSINESS_UNIT}                          \
        -c STAGE=${STAGE}                                          \
        -c TARGET_ACCOUNT=${TARGET_ACCOUNT}                        \
        -c CODEARTIFACT_AUTH_TOKEN=${CODEARTIFACT_AUTH_TOKEN}      \
        -c VPC_ID=${VPC_ID}                                        \
        # ... more context variables
    "
    
    cd infra
    
    case $CdkOp in
        synth)
            npx cdk $CdkOp $CdkParams
            ;;
        # ...
    esac
}
```

The `-c` flags pass **context variables** to CDK. These are values your Python code can access. For example, in app.py:
```python
stage = app.node.try_get_context("STAGE").lower()
```

This retrieves the `STAGE` context variable that was passed with `-c STAGE=${STAGE}`.
#### 3. `do_cdk_operation diff`
This compares the synthesized templates with what's currently deployed and produces a human-readable diff. The function saves this to a file:
```bash
diff)
    echo $(date +%Y-%m-%d-%H-%M-%S-%N) > $DiffFile
    npx cdk $CdkOp $CdkParams --no-color &>> $DiffFile
    ;;
```
#### 4. Git Operations
The next several commands commit the diff file back to the repository:
```yaml
- git config --global user.email "codebuild-${STAGE}-${BRANCH}@IREN.it"
- git checkout ${BRANCH_NAME}
- git add ./Diff_files/cdk-diff-${BUSINESS_UNIT}-$STAGE.txt
- git commit -m "(CodeBuild) Commit of diff file ($STAGE)"
- git push -u origin ${BRANCH_NAME}
```

This is a powerful governance feature. The diff file becomes part of the repository's history. Before approving a deployment, a reviewer can look at exactly what will change. After deployment, there's a permanent record of what was proposed.
## The Diff File: Seeing What Will Change
Let's examine an actual diff file from this system. The file cdk-diff-MERCATO-PROD.txt shows what would change if we deployed to production:
```
Stack cash-flow-simulator-frontend-stack
[~] Custom::CDKECRDeployment ...
 └─ [~] SrcImage (may cause replacement)
     └─ [~] .Fn::Join:
         └─ @@ -3,7 +3,7 @@
            [-]       "Fn::Sub": "238788445602.dkr.ecr....:7929dc2f6997..."
            [+]       "Fn::Sub": "238788445602.dkr.ecr....:dadbac13eba5..."
```

This tells us:
- The frontend stack has changes
- The Docker image will be replaced (a new version will be deployed)
- The old image hash was `7929dc2f6997...`, the new one is `dadbac13eba5...`

Another change:
```
[~] AWS::ECS::TaskDefinition ...
 └─ [~] ContainerDefinitions (requires replacement)
     └─ @@ -47,7 +47,7 @@
        [-]     "Value": "86c8b246ce910299f1abacf819f567335b040a21"
        [+]     "Value": "unknown"
```

This shows the `COMMIT_ID` environment variable changing. The `-` line shows the previous commit, the `+` line shows the new value.

**Why is this valuable?**

Before clicking "approve" on a production deployment, a reviewer can verify:
- Only expected stacks have changes
- Changes are what they should be (new code, not infrastructure modifications)
- No unexpected resources are being deleted
## The Deploy Buildspec: Making It Real
After the synth phase completes and someone approves the deployment, the deploy buildspec runs. Looking at `buildspec-deploy.yml`, the key difference is in the build phase:
```yaml
  build:
    on-failure: ABORT
    commands:
      - echo "Logging into ECR..."
      - do_login_ecr
      - export COMMIT_ID=${CODEBUILD_RESOLVED_SOURCE_VERSION}
      - echo "Deploying Stacks..."
      - do_cdk_operation deploy
```

#### 1. `export COMMIT_ID=${CODEBUILD_RESOLVED_SOURCE_VERSIO`**
`CODEBUILD_RESOLVED_SOURCE_VERSION` is a special variable CodeBuild sets automatically. It contains the exact Git commit hash being deployed. By exporting it as `COMMIT_ID`, it becomes available to CDK.

In the CDK code, this value gets embedded in the ECS task definition as an environment variable. When the container runs, it knows exactly which commit it came from. This is invaluable for debugging—if a user reports a bug, you can see exactly which version they're running.
#### 2. `do_cdk_operation deploy`
This is where actual changes happen. The function:
```bash
deploy)
    # Setup Docker BuildX 
    export DOCKER_BUILDKIT=1
    docker buildx install || echo "ERROR: Could not install buildx."
    docker buildx create --name container --driver=docker-container \
        --driver-opt "default-load=true" || echo "ERROR: Could not create container driver."
    docker buildx use container || echo "ERROR: Could not use container driver."
    # 
    npx cdk $CdkOp --all -v --require-approval never $CdkParams
    ;;
```

Let me explain each part:

**Docker BuildKit Setup**

Docker BuildKit is an improved Docker build system with features like:
- Better caching
- Parallel builds
- Secret handling (so credentials don't end up in image layers)

The commands set up BuildKit's "container" driver, which provides the most features.

**The CDK Deploy Command**

`npx cdk deploy --all -v --require-approval never $CdkParams`

- `deploy` — Actually create/update resources
- `--all` — Deploy all stacks in the project
- `-v` — Verbose output (helpful for debugging)
- `--require-approval never` — Don't wait for terminal confirmation

The `--require-approval never` flag is important. Normally, CDK prompts "Are you sure?" for changes involving security (like IAM). In a CI/CD pipeline, there's no human to answer, so we skip this prompt. The human approval happened earlier, in the pipeline itself.
## Docker Image Building During Deploy
When CDK deploys, it needs to build Docker images. This is handled by the `AutoDeployingCachedECR` construct in `ecr.py`.

Here's what happens:
1. CDK detects that a Docker image needs building
2. It calls Docker with the Dockerfile
3. Docker builds the image, using the `--mount=type=secret` feature to pass CodeArtifact credentials securely
4. The built image is pushed to a temporary CDK asset repository
5. A custom resource copies the image to the project's ECR repository with the `:latest` tag

The caching mechanism is particularly sophisticated:
```python
cache_from=[
    aws_ecr_assets.DockerCacheOption(
        type="registry",
        params={"ref": repo_tag_cache_url},  # e.g., "account.ecr.region/repo:cache"
    )
],
cache_to=aws_ecr_assets.DockerCacheOption(
    type="registry",
    params={
        "mode": "max",             # Cache ALL layers, not just final
        "image-manifest": "true",  # ECR compatibility
        "oci-mediatypes": "true",  # ECR compatibility
        "ignore-error": "true",    # Don't fail if cache doesn't exist yet
        "compression": "zstd",     # Fast compression
        "ref": repo_tag_cache_url,
    },
),
```

**What does this mean in practice?**

Docker images are built in layers. If you have a Dockerfile like:
```dockerfile
FROM python:3.11
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY ./app /app/app
```

Docker caches each step. If you only changed application code (the last line), Docker reuses the cached layers from previous steps. The `pip install` doesn't run again—it's cached.

The `registry` cache type stores these layers in ECR under a `:cache` tag. On subsequent builds:
1. Docker fetches the cache from ECR
2. Checks if any layers can be reused
3. Only rebuilds changed layers

This dramatically speeds up builds. A full build might take 8 minutes; with caching, an incremental build (only code changes) takes 2 minutes.
## Local Development Parity
Developers often need to deploy from their local machine, not through the pipeline. The infra.sh script provides this capability while maintaining the same behavior as the CI/CD pipeline.

Let's trace through what happens when a developer runs:
```bash
./scripts/infra.sh --stage test --deploy
```
#### 1. Argument Parsing
The script parses arguments to determine what to do:
```bash
Stage=test
DoInfraDeploy=1
```
#### 2. SSM Parameter Retrieval
The script fetches the same parameters that CodeBuild would fetch:
```bash
AccountNumber=$(_get_ssm_param /GLOBAL/${BusinessUnit^^}/${Stage^^}/account)
ProjectVPC=$(_get_ssm_param /GLOBAL/${BusinessUnit^^}/${Stage^^}/vpc/default_vpc)
CodeArtifactToken=$(_get_codeartifact_token)
```

The function `_get_ssm_param` is simple:
```bash
function _get_ssm_param() {
    local param_name="$1"
    aws ssm get-parameter --name $param_name --profile saml-advana \
    | jq -r .Parameter.Value
}
```

It uses the AWS CLI to fetch parameter values. Note the `--profile saml-advana`—this requires the developer to have valid credentials for the advana account (where parameters are stored).
#### 3. CodeArtifact Authentication
Just like in CodeBuild:
```bash
CodeArtifactToken=$(_get_codeartifact_token)
CodeArtifactPipUrl="https://aws:${CodeArtifactToken}@${CodeArtifactDomain}-${CodeArtifactAccount}.d.codeartifact.eu-central-1.amazonaws.com/pypi/${CodeArtifactRegistry}/simple/"
```
#### 4. CDK Execution
The script runs CDK with the same context variables as the pipeline:
```bash
cdk $CdkCommand $Stack                                \
    -c REGION=$Region                                 \
    -c project_name=devops-$ProjectName               \
    -c business_unit=${BusinessUnit^^}                \
    -c STAGE=${Stage^^}                               \
    -c TARGET_ACCOUNT=$AccountNumber                  \
    -c CODEARTIFACT_AUTH_TOKEN=${CodeArtifactToken}   \
    # ... more context variables
    --profile $Profile
```

The `--profile $Profile` is the key difference from CI/CD. Instead of assuming a cross-account role, the developer's AWS credentials (configured locally with their own IAM role) are used directly.
#### 5. Release Mode
The script also supports triggering the CI/CD pipeline:
```bash
if [[ $DoReleaseInfra -eq 1 ]]; then
    PipelineName="devops-${ProjectName}-${BusinessUnit^^}-${Stage^^}-v${VersionToRelease//./-}"
    aws codepipeline start-pipeline-execution --name $PipelineName --profile saml-advana
fi
```

Running `.infra.sh --stage prod --release 0.7.5` would trigger the `devops-cash-flow-simulator-MERCATO-PROD-v0-7-5` pipeline.

Importantly, there's a safeguard:
```bash
if [[ $DoReleaseInfra -eq 1 ]] && [[ $Stage == "dev" ]]; then
    _fatal "Cannot start CodePipeline for 'dev' account. Push on branch 'main'." 1
fi
```

You can't manually trigger a dev release—it should auto-deploy on push. This prevents confusion about which deployment method was used.
## Environment-Specific Configuration
Different environments have different configurations. Looking at parameter-devops-mercato.json:
```json
{
    "/MERCATO/devops-cash-flow-simulator/DEV/AUTOSTART": true,
    "/MERCATO/devops-cash-flow-simulator/TEST/AUTOSTART": false,
    "/MERCATO/devops-cash-flow-simulator/PROD/AUTOSTART": false
}
```

**AUTOSTART** controls whether the pipeline runs automatically when code is pushed.
- **Dev (AUTOSTART: true)**: Push code → Pipeline runs immediately. Developers see their changes quickly.
- **Test/Prod (AUTOSTART: false)**: Push code → Nothing happens. Someone must manually start the pipeline. This is a governance control—test and production deployments are deliberate decisions.

Other environment-specific settings:
```json
{
    "/MERCATO/devops-cash-flow-simulator/DEV/CodeBuild/compute_type": "MEDIUM",
    "/MERCATO/devops-cash-flow-simulator/DEV/versions/CDK_VERSION": "2.149.0",
    "/MERCATO/devops-cash-flow-simulator/DEV/versions/PYTHON_RUNTIME": "3.11"
}
```

All environments currently use the same compute type and versions, but they could differ. For example, you might test a new CDK version in dev before using it in production.
## The Complete Flow: From Git Push to Running Application
Let me trace through a complete production deployment, step by step:

**Day 1: Developer Writes Code**
1. Developer creates a feature branch: `git checkout -b feature/new-report`
2. Writes code, tests locally
3. Pushes to CodeCommit: `git push origin feature/new-report`
4. Creates a pull request for review
5. After review, merges to main branch

**Day 2: Version Tagging**
1. Team decides to release
2. Creates a version tag: `git tag v0.7.5` and `git push --tags`
3. This creates a specific version snapshot

**Day 3: Production Deployment**
1. Someone runs: `.infra.sh --stage prod --release 0.7.5`
2. This calls: `aws codepipeline start-pipeline-execution --name devops-cash-flow-simulator-MERCATO-PROD-v0-7-5`

**The Pipeline Executes:**

**Stage 1: Source**
- CodePipeline downloads the code from CodeCommit (the v0.7.5 tag)
- Outputs: Source artifact (a zip of the code)

**Stage 2A: Synth Build**
- CodeBuild starts a build machine
- Downloads the source artifact
- Runs buildspec-synth.yml:
  - Installs Python, Node.js, dependencies
  - Assumes role in production account
  - Runs `cdk synth` (generates CloudFormation templates)
  - Runs `cdk diff` (compares to current state)
  - Commits diff file to repository
- Outputs: Synthesized templates, diff file

**Manual Approval Gate**
- Pipeline pauses
- Sends notification email to approvers
- Approver reviews the diff file
- Approver clicks "Approve" in CodePipeline console (or CLI)

**Stage 2B: Deploy Build**
- CodeBuild starts a new build machine
- Downloads the source artifact
- Runs buildspec-deploy.yml:
  - Installs dependencies
  - Assumes role in production account
  - Sets up Docker BuildKit
  - Logs into ECR
  - Runs `cdk deploy`:
    - Builds Docker images (using cache when possible)
    - Pushes images to ECR
    - Creates/updates CloudFormation stacks
    - Creates/updates all AWS resources

**Result:**
- New Docker images in ECR with `:latest` tag
- Updated ECS task definitions pointing to new images
- ECS service restarted with new containers
- Users access the new version

---

## Rollback and Recovery

What if something goes wrong? Several mechanisms enable recovery:

**1. CloudFormation Rollback**

If a stack update fails, CloudFormation automatically rolls back to the previous state. Resources revert to their prior configuration.

**2. Previous Image Tags**

ECR keeps old images (until lifecycle rules delete them). You can manually update an ECS task definition to use a previous image tag.

**3. Version-Based Pipelines**

Each version has its own pipeline. If v0.7.5 is broken, you can run the v0.7.4 pipeline to redeploy the previous version.

**4. Diff File History**

The committed diff files provide a history of what changed in each deployment, helping diagnose what might have caused issues.

---

## Summary: The Architecture's Strengths

This CI/CD architecture embodies several important principles:

**Separation of Concerns**: The central account manages pipelines; target accounts host applications. Clear boundaries between environments.

**Auditability**: Every change is traceable. Diff files are committed to Git. CloudTrail records who assumed what role when. COMMIT_ID flows into running containers.

**Safety Gates**: Manual approval for test/prod deployments. Diff preview before deploy. AUTOSTART only for dev.

**Performance**: Docker layer caching in ECR. BuildKit for efficient builds. Parallel operations where possible.

**Consistency**: Local development uses the same tools and configuration as CI/CD. The script mirrors the buildspec behavior.

**Flexibility**: Environment-specific configuration via SSM parameters. Easy to change versions, settings, or behavior without code changes.

This pipeline represents a production-grade implementation of CI/CD that balances automation with governance, speed with safety, and simplicity with power. Understanding each component—why it exists and how it works—prepares you to operate, debug, and improve it.
