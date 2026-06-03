---
category: Architecture & Patterns
description: Implement the standard harness initialization pattern with explicit environment variable declarations for test harnesses.
---

# Harness Initialization Pattern

## Overview
All test harnesses in the integration projects follow a consistent initialization approach that provides:
- Self-documenting code that explicitly declares environment dependencies
- Independent modules without centralized configuration lists
- Clear visibility into what each harness requires
- Flexibility to add or remove variables without affecting other harnesses

## The Pattern

```
Does the harness need environment configuration?
└─ YES → Use Explicit Environment Variable Arrays
```

## The Pattern: Explicit Environment Variable Arrays

**Used by**: All integration projects (huron-person, fargate, core)

**Characteristics**:
- Each harness explicitly declares which environment variables it uses
- No centralized key list to maintain
- Self-documenting: code shows exactly what each harness needs
- Independent: Easy to add/remove variables without affecting other harnesses

### Implementation

```typescript
// In any harness module
import { TestEnvironment } from 'integration-core';

if (require.main === module) {
  const testEnvironment = TestEnvironment('HARNESS_PREFIX');
  
  [
    'TIMEOUT',
    'BATCH_SIZE',
    'RETRY_COUNT',
    'LOG_LEVEL'
  ].forEach(testEnvironment.getVarOrEmptyString);
  
  main();
}
```

### Benefits
- **Self-documenting**: Harness explicitly lists all required variables
- **No central maintenance**: No COMMON_*_KEYS list to update
- **Independent modules**: Changes don't affect other harnesses
- **Clear visibility**: Read the harness to know what it needs
- **Flexible**: Easy to add/remove variables

### Adding a New Harness

1. **Create harness with explicit array**:
   ```typescript
   import { TestEnvironment } from 'integration-core';
   
   if (require.main === module) {
     const testEnvironment = TestEnvironment('MY_HARNESS');
     [
       'TIMEOUT',
       'BATCH_SIZE',
       'SPECIFIC_CONFIG'
     ].forEach(testEnvironment.getVarOrEmptyString);
     main();
   }
   ```

2. **Add to .env**:
   ```
   # ---------- Use these for path/to/MyModule.ts ---------- #
   MY_HARNESS_TIMEOUT=30000
   MY_HARNESS_BATCH_SIZE=200
   MY_HARNESS_SPECIFIC_CONFIG=value
   ```

### Maintenance Benefits
- No centralized list to keep in sync
- Each harness is self-contained
- Clear what each harness needs at a glance
- Easy onboarding for new team members

## Template: Standard Implementation

```typescript
// In any harness module
import { TestEnvironment } from 'integration-core';

async function main() {
  // Implementation
}

if (require.main === module) {
  const testEnvironment = TestEnvironment('HARNESS_PREFIX');
  
  [
    'REQUIRED_VAR',
    'OPTIONAL_VAR',
    'TIMEOUT',
    'BATCH_SIZE'
  ].forEach(testEnvironment.getVarOrEmptyString);
  
  main().catch(console.error);
}
```

async function main() {
  const env = new TestEnvironment('HARNESS_PREFIX');
  
  // Load required variables
  const requiredVar = env.getVar('REQUIRED_VAR');
  if (!requiredVar) {
    throw new Error('HARNESS_PREFIX_REQUIRED_VAR not set in .env');
  }
  
  // Load optional variables with defaults
  const optionalVar = env.getVar('OPTIONAL_VAR') || 'default-value';
  
  // ... implementation ...
}

if (require.main === module) {
  main().catch(console.error);
}
```

