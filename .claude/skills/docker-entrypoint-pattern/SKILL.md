---
category: Docker & Infrastructure
description: Structure and test Docker entry points for ECS Fargate tasks with proper environment loading, credential management, and error handling.
---

# Docker Entry Point Pattern for ECS Fargate

## Overview
Docker entry points (`docker/processor.ts`, `docker/chunker.ts`, `docker/post-processor.ts`) are the executable interfaces between CDK infrastructure and application code. They:
- Load environment configuration
- Initialize dependencies
- Manage lifecycle
- Handle errors gracefully
- Report status back to ECS

## Entry Point Structure

### Standard Template

```typescript
// docker/processor.ts
import { ConfigManager, TestEnvironment } from 'integration-core';
import { ProcessorService } from '../src/ProcessorService';

interface ProcessorConfig {
  s3Bucket: string;
  s3Region: string;
  sqsQueueUrl: string;
  maxWorkers: number;
  timeout: number;
}

async function loadConfiguration(): Promise<ProcessorConfig> {
  // Use ConfigManager for three-tier priority
  const config = new ConfigManager()
    .fromJsonString(process.env.TASK_SECRETS)    // ECS TaskDef
    .fromSecretManager()                          // AWS Secrets
    .fromEnvironment();                           // .env for testing

  return {
    s3Bucket: config.getVariable('S3_BUCKET') || process.env.S3_BUCKET,
    s3Region: config.getVariable('AWS_REGION') || 'us-east-2',
    sqsQueueUrl: config.getVariable('SQS_QUEUE_URL'),
    maxWorkers: parseInt(config.getVariable('MAX_WORKERS') || '4'),
    timeout: parseInt(config.getVariable('TIMEOUT') || '300000')
  };
}

async function main() {
  try {
    console.log('Processor starting...');
    
    // 1. Load configuration
    const config = await loadConfiguration();
    console.log(`Config loaded: bucket=${config.s3Bucket}, workers=${config.maxWorkers}`);
    
    // 2. Initialize service
    const processor = new ProcessorService(config);
    await processor.initialize();
    
    // 3. Run processor
    console.log('Processing chunks...');
    const result = await processor.run();
    
    // 4. Report success
    console.log(`Processing complete: ${result.processed} processed, ${result.errors} errors`);
    process.exit(0);  // Success
    
  } catch (error) {
    console.error('Processor failed:', error);
    process.exit(1);  // Failure
  }
}

if (require.main === module) {
  main();
}

export { main };
```

## Key Patterns

### 1. Configuration Loading Hierarchy

```typescript
// 1st priority: ECS TaskDef secrets (base64 JSON)
// Injected as environment variable by ECS
process.env.TASK_SECRETS = '{"...":"..."}';

// 2nd priority: AWS Secrets Manager
// IAM role grants access to secret
// Loaded by ConfigManager.fromSecretManager()

// 3rd priority: Environment variables
// From .env file for local testing
// Loaded by ConfigManager.fromEnvironment()

const config = new ConfigManager()
  .fromJsonString(process.env.TASK_SECRETS)
  .fromSecretManager()
  .fromEnvironment();
```

### 2. Error Handling & Exit Codes

```typescript
async function main() {
  try {
    // Main logic
    await processor.run();
    process.exit(0);  // Success
  } catch (error) {
    if (error instanceof ValidationError) {
      console.error('Config validation failed:', error.message);
      process.exit(1);  // Configuration error
    } else if (error instanceof S3Error) {
      console.error('S3 access failed:', error.message);
      process.exit(2);  // Infrastructure error
    } else {
      console.error('Processor failed:', error);
      process.exit(1);  // General failure
    }
  }
}
```

### 3. Logging for ECS CloudWatch

```typescript
async function main() {
  // ECS CloudWatch captures stdout/stderr
  const timestamp = new Date().toISOString();
  
  console.log(`[${timestamp}] Processor starting`);
  console.log(`[${timestamp}] Configuration loaded`);
  console.log(`[${timestamp}] Processing ${chunks.length} chunks`);
  
  for (const chunk of chunks) {
    console.log(`[${timestamp}] Processing ${chunk.key}`);
    try {
      await processor.process(chunk);
      console.log(`[${timestamp}] ✓ ${chunk.key}`);
    } catch (error) {
      console.error(`[${timestamp}] ✗ ${chunk.key}: ${error.message}`);
    }
  }
  
  console.log(`[${timestamp}] Processor complete`);
}
```

### 4. Health Checks

```typescript
async function main() {
  // Pre-flight checks before processing
  const healthChecks = [
    { name: 'S3 access', check: () => processor.testS3Connection() },
    { name: 'SQS access', check: () => processor.testSQSConnection() },
    { name: 'Secrets Manager', check: () => processor.testSecretsAccess() }
  ];
  
  for (const health of healthChecks) {
    console.log(`Checking ${health.name}...`);
    try {
      await health.check();
      console.log(`✓ ${health.name} OK`);
    } catch (error) {
      console.error(`✗ ${health.name} FAILED: ${error.message}`);
      process.exit(1);
    }
  }
  
  // Proceed only if all checks pass
  await processor.run();
}
```

## Local Testing (Without Docker)

### Test Entry Point with .env

```bash
# 1. Set up .env with test configuration
cat > .env << EOF
DOCKER_PROCESSOR_S3_BUCKET=test-bucket
DOCKER_PROCESSOR_MAX_WORKERS=2
DOCKER_PROCESSOR_TIMEOUT=30000
EOF

# 2. Run entry point locally
npx ts-node docker/processor.ts

# 3. Verify:
# - Configuration loaded correctly
# - Service initialized
# - Processing executed
# - Output as expected
```

### Using TestEnvironment in Entry Point (Optional)

```typescript
// docker/processor.ts - with harness support
if (require.main === module) {
  // Load from TestEnvironment if called as harness
  const env = new TestEnvironment('DOCKER_PROCESSOR');
  
  const s3Bucket = env.getVar('S3_BUCKET') || 'default-bucket';
  const maxWorkers = parseInt(env.getVar('MAX_WORKERS') || '4');
  
  main({ s3Bucket, maxWorkers });
}
```

### Interactive Testing

```bash
# Load .env and run entry point with custom values
export S3_BUCKET=my-test-bucket
export MAX_WORKERS=2
npx ts-node docker/processor.ts
```

## Docker Build & Runtime

### Dockerfile Pattern

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy source code
COPY package*.json ./
RUN npm install --production

COPY src ./src
COPY docker ./docker
COPY tsconfig.json .

# Entry point
CMD ["npx", "ts-node", "docker/processor.ts"]
```

### ECS Task Definition

```json
{
  "family": "huron-processor",
  "containerDefinitions": [{
    "name": "processor",
    "image": "123456789.dkr.ecr.us-east-2.amazonaws.com/huron-processor:latest",
    "essential": true,
    "environment": [
      { "name": "AWS_REGION", "value": "us-east-2" },
      { "name": "NODE_ENV", "value": "production" }
    ],
    "secrets": [
      {
        "name": "TASK_SECRETS",
        "valueFrom": "arn:aws:secretsmanager:us-east-2:123456789:secret:huron/processor-xxxxx"
      }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/huron-processor",
        "awslogs-region": "us-east-2",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/huronProcessorTaskRole"
}
```

## Testing Entry Points Locally

### Unit Test Entry Point

```typescript
describe('Processor Entry Point', () => {
  it('should load configuration from .env', async () => {
    process.env.S3_BUCKET = 'test-bucket';
    process.env.MAX_WORKERS = '2';
    
    const config = await loadConfiguration();
    
    expect(config.s3Bucket).toBe('test-bucket');
    expect(config.maxWorkers).toBe(2);
  });

  it('should handle missing configuration gracefully', async () => {
    delete process.env.S3_BUCKET;  // Required!
    
    await expect(loadConfiguration()).rejects.toThrow('S3_BUCKET not found');
  });
});
```

### Integration Test Entry Point

```typescript
describe('Processor Entry Point Integration', () => {
  it('should process chunks end-to-end', async () => {
    // Set up test environment
    process.env.S3_BUCKET = 'test-bucket';
    process.env.SQS_QUEUE_URL = 'http://localstack:4566/queues/test';
    
    // Mock S3 and SQS
    const mockS3 = jest.spyOn(S3, 'getObject');
    const mockSQS = jest.spyOn(SQS, 'receiveMessage');
    
    // Run entry point
    const exitCode = await main();
    
    // Verify
    expect(exitCode).toBe(0);
    expect(mockS3).toHaveBeenCalled();
    expect(mockSQS).toHaveBeenCalled();
  });
});
```

## Deployment Workflow

### 1. Test Locally
```bash
npm run test:entry-points
npm run test:docker:build
npm run test:docker:run
```

### 2. Build Docker Image
```bash
docker build -t huron-processor:latest -f docker/Dockerfile .
docker tag huron-processor:latest 123456789.dkr.ecr.us-east-2.amazonaws.com/huron-processor:latest
docker push 123456789.dkr.ecr.us-east-2.amazonaws.com/huron-processor:latest
```

### 3. Deploy with CDK
```bash
npm run cdk:deploy -- --require-approval=never
```

### 4. Monitor in CloudWatch
```bash
# Tail logs
aws logs tail /ecs/huron-processor --follow

# Check task status
aws ecs describe-tasks --cluster huron-cluster --tasks task-id
```

## Troubleshooting

### Container won't start
1. Check Docker build: `docker build -t test .`
2. Check entry point: `docker run test`
3. Verify `package.json` has correct scripts

### Configuration not found
1. Check .env file exists: `ls -la .env`
2. Check variables loaded: `grep PROCESSOR .env`
3. Verify ConfigManager order (TaskDef → Secrets → .env)

### ECS task fails with status 1
1. Check CloudWatch logs: `aws logs get-log-events ...`
2. Check task CPU/memory: `aws ecs describe-tasks ...`
3. Verify IAM role permissions

### Secrets Manager access denied
1. Check task role has `secretsmanager:GetSecretValue`
2. Verify secret exists and name matches
3. Test access: `aws secretsmanager get-secret-value --secret-id <name>`

