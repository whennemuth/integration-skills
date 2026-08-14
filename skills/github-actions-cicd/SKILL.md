# GitHub Actions CI/CD Patterns

## Skill Overview

This skill provides comprehensive guidance for implementing GitHub Actions CI/CD pipelines across the integration projects. It covers workflow design, testing integration, build strategies, publishing patterns, Docker workflows, CDK deployments, and cross-project dependency coordination.

## When to Use This Skill

Use this skill when:
- Creating or modifying GitHub Actions workflows (`.github/workflows/*.yml`)
- Setting up CI/CD pipelines for TypeScript npm packages
- Implementing automated testing, building, and publishing
- Configuring Docker image builds and ECR publishing
- Setting up CDK infrastructure deployment automation
- Managing cross-project dependencies in CI/CD
- Troubleshooting workflow failures
- Migrating from S3-based package hosting to npm registries
- Implementing OIDC authentication for AWS access

## Project Context

The integration workspace consists of:
- **integration-core**: Foundation library (shared abstractions)
- **integration-huron-person**: Business logic implementation
- **integration-huron-person-fargate**: CDK infrastructure & serverless deployment

Each project is an independently versioned npm package in its own Git repository.

### Current Publishing Strategy
- **Approach**: Fixed versioning (1.0.0) with S3 bucket publishing
- **Bucket**: `simple-npm-package-bucket` (us-east-1)
- **Method**: "Paving over" - each publish replaces the previous artifact
- **Consumer Pattern**: Downstream projects uninstall and reinstall to get latest
- **Rationale**: Simplifies rapid development iteration for single developer

### Future Publishing Strategy
- **Approach**: Semantic versioning with npm registry
- **Registry Options**: Public npm, GitHub Packages, AWS CodeArtifact
- **Method**: Version-tagged releases with git tags
- **Consumer Pattern**: Standard npm dependency resolution with version constraints
- **Triggers**: Tag-based GitHub Actions (e.g., `v1.2.3` tags)

## Workflow Anatomy

### Basic Structure

```yaml
name: CI/CD Pipeline
on:
  workflow_dispatch:  # Manual trigger
    inputs:
      environment:
        description: 'Deployment environment'
        type: choice
        options: [dev, staging, prod]
        default: dev
      publish_destination:
        description: 'Publish destination'
        type: choice
        options: [s3, npm]
        default: s3

permissions:
  id-token: write   # Required for OIDC
  contents: read    # Required for checkout

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  publish:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      # Publishing steps here
```

### Key Components

**Triggers**: 
- `workflow_dispatch`: Manual trigger (recommended for initial setup)
- `push`: Automatic on branch commits
- `pull_request`: Automatic on PR creation/updates
- `release`: Automatic on GitHub releases
- `workflow_call`: Callable from other workflows (for reusable workflows)

**Permissions**:
- `id-token: write`: Required for OIDC authentication with AWS
- `contents: read`: Standard permission for checking out code
- `contents: write`: Required for creating releases or pushing tags
- `packages: write`: Required for publishing to GitHub Packages

**Runner**:
- `ubuntu-latest`: Recommended for Node.js projects (pre-installed Node, Docker, AWS CLI)
- `ubuntu-22.04`: Use for pinned LTS version
- `windows-latest`: Use only if Windows-specific testing required
- `macos-latest`: Use only if macOS-specific testing required

## Testing Integration

### Jest Exit Codes

Jest automatically exits with code `1` on test failure. GitHub Actions detects non-zero exit codes and fails the workflow.

```yaml
- name: Run tests
  run: npm run test  # Exits with code 1 if any test fails
```

**No additional configuration required** - npm scripts propagate exit codes correctly.

### Test Isolation

**Requirements**:
- Tests must run without external API dependencies
- Use mocks for AWS services, HTTP clients, databases
- Tests should be idempotent (no side effects)
- jest.setup.js should configure all mocks

**Example setup** (from integration-huron-person):
```javascript
// jest.setup.js
jest.mock('@nodable/entities', () => ({
  EntityDecoder: class EntityDecoder {},
  // ... other mocks
}), { virtual: true });
```

### Coverage Reporting (Optional)

```yaml
- name: Run tests with coverage
  run: npm run test -- --coverage
- uses: actions/upload-artifact@v4
  with:
    name: coverage-report
    path: coverage/
```

## Build Strategies

### TypeScript Compilation

The integration projects use dual-module builds (ESM + CJS):

```json
{
  "scripts": {
    "build": "rm -rf dist/ && tsc",
    "build:esm": "tsc",
    "build:cjs": "tsc --module CommonJS --outdir dist/cjs"
  }
}
```

**Workflow implementation**:
```yaml
- name: Build TypeScript
  run: npm run build

- name: Verify build outputs
  run: |
    test -d dist/esm || exit 1
    test -d dist/cjs || exit 1
    test -f dist/types/bin/index.d.ts || exit 1
```

### Build Artifacts

**Upload for downstream jobs**:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: |
      dist/
      package.json
      package-lock.json
    retention-days: 7
```

**Download in publish job**:
```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: .
```

### Caching Dependencies

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # Automatically caches ~/.npm
```

**Advanced caching**:
```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

## Publishing Patterns

### S3 Bucket Publishing (Current)

**Configuration**:
- Bucket: `simple-npm-package-bucket`
- Region: `us-east-1`
- Artifact: `<package-name>-<version>.tgz`
- Method: `npm pack && aws s3 cp`

**Workflow implementation**:
```yaml
publish-s3:
  needs: build
  runs-on: ubuntu-latest
  if: inputs.publish_destination == 's3'
  steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - run: npm ci
    
    - uses: actions/download-artifact@v4
      with:
        name: build-output
        path: .
    
    - name: Create package tarball
      run: npm pack
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        role-session-name: GitHubActions-${{ github.repository_owner }}-${{ github.event.repository.name }}
        aws-region: ${{ secrets.AWS_REGION }}
    
    - name: Upload to S3
      run: |
        PACKAGE_NAME=$(node -p "require('./package.json').name")
        PACKAGE_VERSION=$(node -p "require('./package.json').version")
        aws s3 cp ${PACKAGE_NAME}-${PACKAGE_VERSION}.tgz s3://${{ secrets.S3_BUCKET_NAME }}/ \
          --metadata git-sha=${{ github.sha }},workflow-run=${{ github.run_id }}
```

### NPM Registry Publishing (Future)

**For public npm registry**:
```yaml
publish-npm:
  needs: build
  runs-on: ubuntu-latest
  if: inputs.publish_destination == 'npm'
  steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        registry-url: 'https://registry.npmjs.org'
    
    - run: npm ci
    
    - uses: actions/download-artifact@v4
      with:
        name: build-output
        path: .
    
    - name: Publish to npm
      run: npm publish --access public
      env:
        NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**For GitHub Packages**:
```yaml
publish-npm:
  needs: build
  runs-on: ubuntu-latest
  if: inputs.publish_destination == 'npm'
  permissions:
    packages: write
  steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        registry-url: 'https://npm.pkg.github.com'
        scope: '@bu-ist'
    
    - run: npm ci
    
    - uses: actions/download-artifact@v4
      with:
        name: build-output
        path: .
    
    - name: Publish to GitHub Packages
      run: npm publish
      env:
        NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Version Management

**Current (fixed version)**:
```yaml
- name: Get package info
  run: |
    echo "PACKAGE_NAME=$(node -p "require('./package.json').name")" >> $GITHUB_ENV
    echo "PACKAGE_VERSION=$(node -p "require('./package.json').version")" >> $GITHUB_ENV
    echo "GIT_SHA=${{ github.sha }}" >> $GITHUB_ENV
```

**Future (semantic versioning)**:
```yaml
- name: Extract version from tag
  if: startsWith(github.ref, 'refs/tags/v')
  run: |
    VERSION=${GITHUB_REF#refs/tags/v}
    echo "PACKAGE_VERSION=${VERSION}" >> $GITHUB_ENV
    npm version ${VERSION} --no-git-tag-version

- name: Verify version
  run: |
    CURRENT=$(node -p "require('./package.json').version")
    if [ "$CURRENT" != "$PACKAGE_VERSION" ]; then
      echo "Version mismatch: $CURRENT != $PACKAGE_VERSION"
      exit 1
    fi
```

### Parameterization Strategy

**Workflow inputs**:
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        type: choice
        options: [dev, staging, prod]
        default: dev
        required: true
      
      publish_destination:
        description: 'Publish destination'
        type: choice
        options: [s3, npm]
        default: s3
        required: true
      
      version_override:
        description: 'Version override (leave empty for package.json version)'
        type: string
        required: false
      
      skip_tests:
        description: 'Skip tests (use with caution)'
        type: boolean
        default: false
        required: false
```

**Usage in jobs**:
```yaml
jobs:
  test:
    if: ${{ !inputs.skip_tests }}
    # ...
  
  publish-s3:
    if: inputs.publish_destination == 's3'
    # ...
  
  publish-npm:
    if: inputs.publish_destination == 'npm'
    # ...
```

## AWS Authentication

### OIDC Authentication (Recommended)

OIDC eliminates the need for static AWS credentials by using GitHub's identity provider.

**Prerequisites** (one-time setup):
1. Create OIDC identity provider in AWS IAM
2. Create IAM role with trust policy for GitHub
3. Attach permissions policy to role

**Trust policy example**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": [
            "repo:OWNER/REPO:ref:refs/heads/main",
            "repo:OWNER/REPO:ref:refs/heads/cicd"
          ]
        }
      }
    }
  ]
}
```

**Permissions policy** (S3 publishing):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::simple-npm-package-bucket",
        "arn:aws:s3:::simple-npm-package-bucket/*"
      ]
    }
  ]
}
```

**Workflow usage**:
```yaml
permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  publish:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          role-session-name: GitHubActions-${{ github.run_id }}
          aws-region: ${{ secrets.AWS_REGION }}
      
      - name: Verify AWS credentials
        run: aws sts get-caller-identity
```

**Required GitHub Secrets**:
- `AWS_ROLE_ARN`: IAM role ARN (e.g., `arn:aws:iam::123456789012:role/GitHubActionsRole`)
- `AWS_REGION`: AWS region (e.g., `us-east-1`)
- `S3_BUCKET_NAME`: S3 bucket name (e.g., `simple-npm-package-bucket`)

### Static Credentials (Not Recommended)

Only use if OIDC setup is not feasible.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```

**Security risks**:
- Credentials can be leaked or compromised
- Requires regular rotation
- Broad permissions if not scoped properly
- No automatic expiration

## Docker Workflows (Fargate)

### Build and Push to ECR

**Prerequisites**:
- ECR repository exists
- IAM role has ECR permissions (`ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, etc.)

**Workflow implementation**:
```yaml
docker-build-push:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        aws-region: ${{ secrets.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2
    
    - name: Extract metadata
      id: meta
      run: |
        echo "IMAGE_TAG=${{ github.sha }}" >> $GITHUB_OUTPUT
        echo "BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_OUTPUT
    
    - name: Build Docker image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY_NAME }}
        IMAGE_TAG: ${{ steps.meta.outputs.IMAGE_TAG }}
      run: |
        docker build \
          --platform linux/arm64 \
          --build-arg BUILD_DATE=${{ steps.meta.outputs.BUILD_DATE }} \
          --build-arg GIT_SHA=${{ github.sha }} \
          -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} \
          -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest \
          -f docker/Dockerfile \
          .
    
    - name: Push to ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY_NAME }}
        IMAGE_TAG: ${{ steps.meta.outputs.IMAGE_TAG }}
      run: |
        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
```

### Image Tagging Strategy

**Current (latest only)**:
- Single tag: `latest`
- Overwrites previous image
- Simple but no rollback capability

**Future (semantic + SHA)**:
- Version tag: `v1.2.3` (matches npm package version)
- SHA tag: `sha-abc1234` (exact codebase reference)
- Latest tag: `latest` (convenience alias)
- Example: `770203350335.dkr.ecr.us-east-2.amazonaws.com/huron-person-integration:v1.2.3`

**Implementation**:
```yaml
- name: Determine tags
  id: tags
  run: |
    TAGS="${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"
    TAGS="${TAGS},${ECR_REGISTRY}/${ECR_REPOSITORY}:sha-${GITHUB_SHA::7}"
    if [[ "${{ github.ref }}" =~ ^refs/tags/v[0-9] ]]; then
      VERSION=${GITHUB_REF#refs/tags/}
      TAGS="${TAGS},${ECR_REGISTRY}/${ECR_REPOSITORY}:${VERSION}"
    fi
    echo "TAGS=${TAGS}" >> $GITHUB_OUTPUT

- name: Build and push
  run: |
    IFS=',' read -ra TAG_ARRAY <<< "${{ steps.tags.outputs.TAGS }}"
    BUILD_ARGS=""
    for tag in "${TAG_ARRAY[@]}"; do
      BUILD_ARGS="${BUILD_ARGS} -t ${tag}"
    done
    docker build ${BUILD_ARGS} -f docker/Dockerfile .
    for tag in "${TAG_ARRAY[@]}"; do
      docker push ${tag}
    done
```

## CDK Deployment (Fargate)

### Deployment Workflow

```yaml
deploy-cdk:
  needs: docker-build-push
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - run: npm ci
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        aws-region: ${{ secrets.AWS_REGION }}
    
    - name: Install AWS CDK
      run: npm install -g aws-cdk
    
    - name: Verify CDK bootstrap
      run: |
        BOOTSTRAP_VERSION=$(aws ssm get-parameter \
          --name /cdk-bootstrap/$(aws sts get-caller-identity --query Account --output text)/version \
          --query Parameter.Value --output text 2>/dev/null || echo "0")
        if [ "$BOOTSTRAP_VERSION" -lt "6" ]; then
          echo "CDK bootstrap required or outdated"
          exit 1
        fi
    
    - name: CDK synth
      run: cdk synth
    
    - name: CDK deploy
      run: |
        STACK_NAME=$(node -e "console.log(require('./context/context.json').STACK_ID)")
        cdk deploy ${STACK_NAME} \
          --require-approval never \
          --context landscape=${{ inputs.environment }}
```

### Context Overrides

The fargate project uses `context/context.json` with landscape-specific settings (dev/staging/prod).

**Current approach**: Context file stored in git (no secrets)

**Environment-specific deployment**:
```yaml
- name: Override context for environment
  run: |
    cat > cdk.context.json <<EOF
    {
      "landscape": "${{ inputs.environment }}",
      "git-sha": "${{ github.sha }}",
      "workflow-run": "${{ github.run_id }}"
    }
    EOF

- name: CDK deploy with context
  run: |
    cdk deploy \
      --context landscape=${{ inputs.environment }} \
      --require-approval never
```

**Tag-based versioning**:
```yaml
- name: Extract version from tag
  if: startsWith(github.ref, 'refs/tags/v')
  run: |
    VERSION=${GITHUB_REF#refs/tags/v}
    echo "DEPLOY_VERSION=${VERSION}" >> $GITHUB_ENV

- name: Update image tag in CDK context
  if: env.DEPLOY_VERSION
  run: |
    jq --arg tag "${{ env.DEPLOY_VERSION }}" \
      '.ECR.imageTag = $tag' \
      context/context.json > context/context.json.tmp
    mv context/context.json.tmp context/context.json
```

### Rollback Strategy

```yaml
rollback-cdk:
  runs-on: ubuntu-latest
  steps:
    - name: Get previous deployment
      run: |
        STACK_NAME="${{ secrets.CDK_STACK_NAME }}"
        PREVIOUS_IMAGE=$(aws cloudformation describe-stacks \
          --stack-name ${STACK_NAME} \
          --query 'Stacks[0].Parameters[?ParameterKey==`ImageTag`].ParameterValue' \
          --output text)
        echo "ROLLBACK_IMAGE=${PREVIOUS_IMAGE}" >> $GITHUB_ENV
    
    - name: Deploy with previous image
      run: |
        cdk deploy \
          --context imageTag=${{ env.ROLLBACK_IMAGE }} \
          --require-approval never
```

## Trigger Strategies

### Manual Triggers (Current)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        type: choice
        options: [dev, staging, prod]
        default: dev
```

**Usage**: GitHub repository → Actions → Select workflow → Run workflow

**Benefits**:
- Full control over when deployments occur
- Test changes before automation
- Suitable for early development phases

**Drawbacks**:
- Requires manual intervention
- Easy to forget to trigger dependent builds

### Push-Based Triggers (Future Option 1)

```yaml
on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
```

**Benefits**:
- Automatic on every commit
- Fast feedback loop
- Good for continuous integration

**Drawbacks**:
- May trigger too frequently
- Every commit = deployment (might be wasteful)
- Harder to control production deployments

**Recommendation**: Use for core and huron-person (library packages) on main branch only

### Tag-Based Triggers (Future Option 2)

```yaml
on:
  push:
    tags:
      - 'v*.*.*'  # Matches v1.0.0, v2.1.3, etc.
```

**Benefits**:
- Explicit version control
- Aligns with semantic versioning
- Production-safe (deliberate action required)
- Git tags provide audit trail

**Drawbacks**:
- Requires manual tag creation
- Extra step in release process

**Recommendation**: Use for production deployments, especially fargate infrastructure

**Tag creation workflow**:
```bash
# Local development
git tag -a v1.2.3 -m "Release version 1.2.3"
git push origin v1.2.3
```

### Combined Strategy (Recommended Future)

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'package.json'
  push:
    tags:
      - 'v*.*.*'
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, staging, prod]

jobs:
  publish:
    steps:
      - name: Determine environment
        run: |
          if [[ "${{ github.ref }}" =~ ^refs/tags/v ]]; then
            echo "ENVIRONMENT=prod" >> $GITHUB_ENV
          elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "ENVIRONMENT=dev" >> $GITHUB_ENV
          else
            echo "ENVIRONMENT=${{ inputs.environment }}" >> $GITHUB_ENV
          fi
```

## Cross-Project Dependency Coordination

### Dependency Chain

```
integration-core (package)
    ↓ (depends on)
integration-huron-person (package)
    ↓ (depends on)
integration-huron-person-fargate (infrastructure)
```

### Manual Coordination (Current)

1. Publish core: `npm run publish-s3` (manual or GitHub Actions)
2. Update huron-person: `npm run reinstall-tgz-clean` (manual)
3. Publish huron-person: `npm run publish-s3` (manual or GitHub Actions)
4. Update fargate: `npm run reinstall-tgz-clean` (manual)
5. Deploy fargate: `npm run deploy` (manual or GitHub Actions)

### Automated Coordination (Future)

**Option A: Repository Dispatch**

Trigger downstream builds from upstream workflows:

```yaml
# In integration-core/.github/workflows/ci-cd.yml
jobs:
  publish:
    steps:
      # ... publish steps ...
      
      - name: Trigger downstream builds
        run: |
          curl -X POST \
            -H "Accept: application/vnd.github.v3+json" \
            -H "Authorization: token ${{ secrets.REPO_DISPATCH_TOKEN }}" \
            https://api.github.com/repos/OWNER/integration-huron-person/dispatches \
            -d '{"event_type":"dependency-updated","client_payload":{"package":"integration-core","version":"1.0.0","sha":"${{ github.sha }}"}}'
```

```yaml
# In integration-huron-person/.github/workflows/ci-cd.yml
on:
  repository_dispatch:
    types: [dependency-updated]

jobs:
  update-and-publish:
    if: github.event.client_payload.package == 'integration-core'
    steps:
      - name: Update dependency
        run: npm run reinstall-tgz-clean
      
      # ... build and publish steps ...
```

**Option B: workflow_run**

Trigger workflows when other workflows complete:

```yaml
# In integration-huron-person/.github/workflows/ci-cd.yml
on:
  workflow_run:
    workflows: ["Core CI/CD"]
    types:
      - completed
    branches:
      - main

jobs:
  update-and-publish:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    # ...
```

**Option C: Monorepo Tooling**

Consider tools like Nx or Turborepo for automatic dependency detection (requires restructuring to monorepo).

### Semantic Versioning Coordination (Future)

When semantic versioning is introduced:

```yaml
# Automatically update package.json dependency versions
- name: Update dependency version
  run: |
    NEW_VERSION="${{ github.event.client_payload.version }}"
    npm install integration-core@${NEW_VERSION}
    
- name: Commit updated package-lock.json
  run: |
    git config user.name "GitHub Actions"
    git config user.email "actions@github.com"
    git add package.json package-lock.json
    git commit -m "chore: update integration-core to ${NEW_VERSION}"
    git push
```

## Troubleshooting

### Common Failures

**Test failures**:
- **Symptom**: Workflow fails at test job
- **Diagnosis**: Check test logs for specific failures
- **Fix**: Run tests locally with `npm test`, fix failing tests

**Build failures**:
- **Symptom**: TypeScript compilation errors
- **Diagnosis**: Check for type errors or missing dependencies
- **Fix**: Run `npm run build` locally, resolve type errors

**Authentication failures**:
- **Symptom**: `AssumeRoleWithWebIdentity` errors
- **Diagnosis**: OIDC trust policy mismatch or missing permissions
- **Fix**: Verify trust policy includes correct repository path, check role permissions

**S3 upload failures**:
- **Symptom**: `AccessDenied` or `NoSuchBucket` errors
- **Diagnosis**: IAM permissions insufficient or bucket name incorrect
- **Fix**: Verify role has S3 permissions, check bucket name in secrets

**Artifact not found**:
- **Symptom**: Download artifact step fails
- **Diagnosis**: Upload artifact didn't run or used different name
- **Fix**: Verify upload step succeeded, match artifact names exactly

**CDK deployment failures**:
- **Symptom**: CloudFormation stack update fails
- **Diagnosis**: Syntax errors, resource conflicts, or permissions issues
- **Fix**: Run `cdk synth` locally, check CloudFormation events in AWS console

### Debugging Techniques

**Enable debug logging**:
```yaml
- name: Debug step
  run: set -x; your-command-here
  env:
    ACTIONS_STEP_DEBUG: true
```

**Save logs as artifacts**:
```yaml
- name: Save logs
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: failure-logs
    path: |
      **/*.log
      cdk.out/
```

**Test workflow locally**:
```bash
# Using act (https://github.com/nektos/act)
act -j test -s GITHUB_TOKEN="$(gh auth token)"
```

**Dry run mode**:
```yaml
- name: Dry run publish
  run: |
    echo "Would publish to S3: s3://${S3_BUCKET}/${PACKAGE_NAME}-${VERSION}.tgz"
    echo "Would tag Docker image: ${ECR_REGISTRY}/${REPOSITORY}:${TAG}"
  if: inputs.dry_run
```

## Security Best Practices

1. **Use OIDC instead of static credentials**
2. **Scope IAM role permissions narrowly** (principle of least privilege)
3. **Distinguish secrets from variables** (use `vars` for non-sensitive config, `secrets` only for credentials)
4. **Don't log secrets** (GitHub automatically masks secrets, but be careful with derived values)
5. **Pin action versions** (use `@v4` not `@latest`)
6. **Review workflow permissions** (use minimal `permissions:` block)
7. **Use branch protection rules** (require PR reviews before merging to main)
8. **Enable workflow approval for production** (use environments with required reviewers)
9. **Scan Docker images** (use `docker scan` or third-party tools)
10. **Rotate secrets regularly** (even with OIDC, rotate tokens and credentials if compromised)
11. **Audit workflow runs** (review logs for suspicious activity)

## GitHub Configuration

GitHub Actions workflows require configuration values. Use **Variables** (`vars.NAME`) for non-sensitive configuration like bucket names, regions, and ARNs. Use **Secrets** (`secrets.NAME`) only for actual credentials like tokens and passwords. Variables are visible in logs for debugging; secrets are automatically masked.

**Example**:
```yaml
# Non-sensitive configuration (use vars)
role-to-assume: ${{ vars.AWS_ROLE_ARN }}
aws-region: ${{ vars.AWS_REGION }}

# Credentials (use secrets)
NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

Configuration is set in: Repository Settings → Secrets and variables → Actions (separate tabs for Variables and Secrets).

For complete setup instructions, see project-specific CLAUDE.md files in each repository's `.github/workflows/` directory.

## Best Practices

1. **Use manual triggers initially** - Build confidence before automating
2. **Test workflows on feature branches** - Use `workflow_dispatch` on any branch
3. **Keep workflows DRY** - Use reusable workflows or composite actions
4. **Fail fast** - Run tests before expensive build/deploy steps
5. **Cache aggressively** - npm dependencies, Docker layers, build outputs
6. **Parameterize everything** - Avoid hardcoded values, use variables for config and secrets for credentials (see "GitHub Configuration")
7. **Document workflow inputs** - Use clear descriptions for `workflow_dispatch` inputs
8. **Version your actions** - Pin to specific versions (`@v4`), not `@latest`
9. **Use job dependencies** - `needs:` ensures correct execution order
10. **Implement rollback procedures** - Have a plan for failed deployments
11. **Monitor workflow costs** - GitHub Actions minutes are limited on free plans
12. **Keep workflows focused** - Separate concerns (CI vs CD, test vs deploy)

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS Actions: Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [integration-core/docs/cicd.md](../../integration-core/docs/cicd.md) - OIDC setup precedent
