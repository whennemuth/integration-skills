---
category: Configuration Management
description: Implement multi-tier credential and configuration resolution using ConfigManager chain pattern for flexibility across local dev, staging, and production environments.
---

# ConfigManager Chain Pattern

## Overview
ConfigManager provides a priority-based credential and configuration loading mechanism that enables the same code to work across:
- Local development (`.env` files)
- ECS Fargate staging (AWS Secrets Manager)
- Production Lambda (TaskDef secrets)

The **chain pattern** defines a priority order for where credentials are loaded from, with fallback to the next tier if not found.

## The Chain: Priority Order

```
1. TaskDef Secrets (fromJsonString)
   ↓ (if not found)
2. AWS Secrets Manager (fromSecretManager)
   ↓ (if not found)
3. Environment Variables (fromEnvironment)
   ↓ (if not found)
4. Filesystem Config (fromFileSystem)
   ↓ (if not found)
   FAIL: Throw error or use default
```

## Tier Explanations

### Tier 1: TaskDef Secrets (fromJsonString)
**When Used**: ECS Fargate task execution

**Format**: Base64-encoded JSON in ECS TaskDefinition

```json
{
  "containerDefinitions": [{
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "{...base64-encoded-json...}"
      }
    ]
  }]
}
```

**Code**:
```typescript
const config = new ConfigManager()
  .fromJsonString(process.env.ECS_TASK_DEF_SECRETS);
```

**Priority**: **HIGHEST** (most specific to runtime environment)

**Use Case**: Production ECS tasks with injected secrets

### Tier 2: AWS Secrets Manager (fromSecretManager)
**When Used**: Production Lambda, ECS with IAM role

**Mechanism**: IAM role grants access to Secrets Manager secret

```bash
# Create secret
aws secretsmanager create-secret \
  --name huron/person/credentials \
  --secret-string '{"username":"admin","password":"secret"}'

# IAM role must have secretsmanager:GetSecretValue permission
```

**Code**:
```typescript
const config = new ConfigManager()
  .fromSecretManager('huron/person/credentials');
```

**Priority**: **HIGH** (production-grade security)

**Use Case**: Production environments where IAM controls access

### Tier 3: Environment Variables (fromEnvironment)
**When Used**: Local development, CI/CD pipelines

**Source**: `.env` file or shell environment

```bash
# .env or shell
DB_PASSWORD=my-dev-password
API_KEY=dev-key-12345
```

**Code**:
```typescript
const config = new ConfigManager()
  .fromEnvironment();
```

**Priority**: **MEDIUM** (convenient for dev, NOT for production)

**Use Case**: Local `.env` files, CI/CD with exported variables

### Tier 4: Filesystem Config (fromFileSystem)
**When Used**: Legacy systems, complex configurations

**Format**: JSON file on disk

```json
{
  "database": { "host": "localhost", "port": 5432 },
  "api": { "timeout": 30000 }
}
```

**Code**:
```typescript
const config = new ConfigManager()
  .fromFileSystem('/etc/app/config.json');
```

**Priority**: **LOW** (file system access not guaranteed)

**Use Case**: Fallback for complex configs, legacy integrations

## Using the Chain Pattern

### Basic Implementation

```typescript
import { ConfigManager } from 'integration-core';

async function main() {
  const config = new ConfigManager()
    .fromTaskDef()           // Try TaskDef secrets first
    .fromSecretManager()     // Fall back to Secrets Manager
    .fromEnvironment()       // Fall back to .env
    .fromFileSystem();       // Last resort

  const dbPassword = config.getVariable('DB_PASSWORD');
  const apiKey = config.getVariable('API_KEY');
  
  // Use credentials
  connectToDatabase(dbPassword);
  callApi(apiKey);
}
```

### In DataTarget (Authentication)

```typescript
export class PersonDataTarget extends DataTarget {
  private config: ConfigManager;
  private authToken: string;

  constructor() {
    this.config = new ConfigManager()
      .fromSecretManager()     // Production: Secrets Manager
      .fromEnvironment()       // Dev: .env file
      .fromFileSystem();       // Fallback

    this.initializeAuth();
  }

  private async initializeAuth() {
    const username = this.config.getVariable('DATATARGET_USER_ID');
    const password = this.config.getVariable('DATATARGET_PASSWORD');
    
    this.authToken = await this.obtainToken(username, password);
  }

  async refreshToken() {
    // Automatically called when token expires
    const username = this.config.getVariable('DATATARGET_USER_ID');
    const password = this.config.getVariable('DATATARGET_PASSWORD');
    this.authToken = await this.obtainToken(username, password);
  }
}
```

### In Lambda Handler

```typescript
// AWS Lambda with Secrets Manager
export async function handler(event: any) {
  const config = new ConfigManager()
    .fromSecretManager('huron/lambda-secrets')
    .fromEnvironment();  // Fallback if not in Secrets Manager

  const apiKey = config.getVariable('DATASOURCE_API_KEY');
  const targetUrl = config.getVariable('DATATARGET_BASE_URL');

  // Process event with credentials
  return await processEvent(event, apiKey, targetUrl);
}
```

### In ECS Fargate Task

```typescript
// docker/processor.ts with ECS TaskDef secrets
export async function main() {
  const config = new ConfigManager()
    .fromJsonString(process.env.TASK_SECRETS)  // Injected by ECS
    .fromSecretManager()                        // Fallback
    .fromEnvironment();                         // Dev fallback

  const credentials = {
    sourceUrl: config.getVariable('DATASOURCE_BASE_URL'),
    sourceKey: config.getVariable('DATASOURCE_API_KEY'),
    targetUrl: config.getVariable('DATATARGET_BASE_URL'),
    targetUser: config.getVariable('DATATARGET_USER_ID')
  };

  await processChunks(credentials);
}
```

## Implementation Patterns

### Pattern 1: Centralized Config (Recommended)

```typescript
// src/config.ts
import { ConfigManager } from 'integration-core';

export const appConfig = new ConfigManager()
  .fromSecretManager('huron/app-secrets')  // Prod
  .fromEnvironment()                        // Dev
  .fromFileSystem();                        // Fallback

// In any module:
import { appConfig } from './config';
const apiKey = appConfig.getVariable('API_KEY');
```

**Pros**: Single initialization, consistent across app
**Cons**: Module dependency on centralized config

### Pattern 2: Per-Component Config

```typescript
// In DataTarget implementation
class PersonDataTarget extends DataTarget {
  constructor() {
    this.config = new ConfigManager()
      .fromSecretManager()
      .fromEnvironment();
  }

  async authenticate() {
    const password = this.config.getVariable('DATATARGET_PASSWORD');
    // ...
  }
}
```

**Pros**: Self-contained, easier to test
**Cons**: Multiple ConfigManager instances

### Pattern 3: Lazy Initialization

```typescript
let _config: ConfigManager | null = null;

function getConfig(): ConfigManager {
  if (!_config) {
    _config = new ConfigManager()
      .fromSecretManager()
      .fromEnvironment();
  }
  return _config;
}

export const config = getConfig();
```

**Pros**: Deferred initialization, supports runtime changes
**Cons**: Slightly more complex

## Error Handling

### Variable Not Found

```typescript
try {
  const password = config.getVariable('DATABASE_PASSWORD');
  if (!password) {
    throw new Error('DATABASE_PASSWORD not found in any tier');
  }
} catch (error) {
  console.error('Config error:', error);
  // Fail fast - don't continue without critical credentials
  process.exit(1);
}
```

### Secrets Manager Unavailable

```typescript
const config = new ConfigManager()
  .fromSecretManager()  // Might fail in dev without AWS creds
  .fromEnvironment()    // Fallback to .env
  .fromFileSystem();    // Last resort

// If Secrets Manager fails, chain automatically tries next tier
const apiKey = config.getVariable('API_KEY'); // Won't throw, uses fallback
```

## Environment-Specific Setup

### Local Development
```bash
# .env file
DATASOURCE_API_KEY=dev-key-123
DATATARGET_PASSWORD=dev-password
```

```typescript
const config = new ConfigManager().fromEnvironment();
```

### Staging (ECS with Secrets Manager)
```bash
# ECR task definition includes:
# - IAM role with secretsmanager:GetSecretValue
# - Secret name: huron/staging-secrets
```

```typescript
const config = new ConfigManager()
  .fromSecretManager('huron/staging-secrets')
  .fromEnvironment(); // Fallback for local ECS testing
```

### Production (Lambda with ECS)
```bash
# Lambda environment:
# - ECS_TASK_DEF_SECRETS env var (base64 JSON)
# - Or direct Secrets Manager with production secret name
```

```typescript
const config = new ConfigManager()
  .fromJsonString(process.env.ECS_TASK_DEF_SECRETS)
  .fromSecretManager('huron/prod-secrets');
```

## Testing with ConfigManager

### Unit Test (Mock Config)

```typescript
describe('PersonDataTarget', () => {
  it('should authenticate with provided credentials', async () => {
    const testConfig = new ConfigManager().fromEnvironment({
      'DATATARGET_USER_ID': 'test-user',
      'DATATARGET_PASSWORD': 'test-pass'
    });

    const target = new PersonDataTarget(testConfig);
    const token = await target.authenticate();
    expect(token).toBeDefined();
  });
});
```

### Integration Test (Real Secrets Manager)

```typescript
describe('PersonDataTarget with Secrets Manager', () => {
  it('should load credentials from Secrets Manager', async () => {
    const config = new ConfigManager()
      .fromSecretManager('test/credentials');

    const password = config.getVariable('DATATARGET_PASSWORD');
    expect(password).toBeDefined();
  });
});
```

### Local Testing (Harness)

```typescript
// src/data-target/crud/ReadPerson.ts harness
if (require.main === module) {
  const env = new TestEnvironment('READ_PERSON');
  
  // Test with .env variables
  const userId = env.getVar('DATATARGET_USER_ID');
  const password = env.getVar('DATATARGET_PASSWORD');
  
  const config = new ConfigManager().fromEnvironment();
  main(config);
}
```

## Best Practices

1. **Always include fallback tiers**: Never rely on single source
2. **Fail fast**: Throw errors on missing critical variables
3. **Document which tier is used**: Comments showing environment context
4. **Test each tier independently**: Mock per-tier behavior
5. **Rotate secrets regularly**: Use Secrets Manager for non-dev
6. **Never log credentials**: Even in errors
7. **Use IAM for Secrets Manager access**: Not API keys
8. **Keep .env in .gitignore**: Always

## Troubleshooting

### "Variable not found across all tiers"
1. Check .env file exists and has variable
2. Verify TaskDef or Secrets Manager contains variable
3. Ensure IAM role has secretsmanager:GetSecretValue
4. Test each tier independently: `config.fromEnvironment().getVariable('X')`

### "Secrets Manager access denied"
1. Verify IAM role attached to ECS task/Lambda
2. Check secret name matches code (case-sensitive)
3. Verify secret exists in correct AWS account/region
4. Test with AWS CLI: `aws secretsmanager get-secret-value --secret-id huron/secrets`

### "Wrong credential used in production"
1. Check chain order (should be .fromSecretManager() before .fromEnvironment())
2. Verify ECS TaskDef secrets have highest priority
3. Don't set environment variables in production (remove from Dockerfile)

