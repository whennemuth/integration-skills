---
category: Architecture & Patterns
description: Apply TestEnvironment pattern for prefix-based environment variable isolation in test harnesses and configuration management.
---

# TestEnvironment Pattern & Application

## Overview
TestEnvironment is a utility from `integration-core` that enables prefix-based environment variable isolation. Each test harness loads its own namespaced variables with fallback chain support, eliminating scattered configuration and enabling clear separation of concerns.

## When to Use

**Use TestEnvironment when**:
- Creating a new test harness module
- Adding environment-dependent functionality to a component
- Building isolated test utilities that need external configuration
- Implementing configuration scoping across multiple harness modules

**Do NOT use when**:
- Building production runtime code (use ConfigManager instead for three-tier credential resolution)
- Environment variables are already loaded by application bootstrap
- Configuration is static/compile-time (use configuration objects)

## Import & Basic Usage

### From integration-core (Recommended)
```typescript
import { TestEnvironment } from 'integration-core';

const env = new TestEnvironment('MY_HARNESS');
const timeout = env.getVar('TIMEOUT');        // Loads MY_HARNESS_TIMEOUT
const apiKey = env.getVar('API_KEY');         // Loads MY_HARNESS_API_KEY or fallback to API_KEY
```

### Constructor Options
```typescript
// Basic: Use default prefix with fallback
const env = new TestEnvironment('PREFIX');

// With custom fallback values
const env = new TestEnvironment('PREFIX', {
  TIMEOUT: '30000',
  BATCH_SIZE: '200'
});
```

## Exemption Rule: Shared Variables

**DATASOURCE_* and DATATARGET_* variables are ALWAYS unprefixed:**

```
# ✅ Correct: Shared across all harnesses
DATASOURCE_BASE_URL=https://api.example.com
DATASOURCE_API_KEY=your-key-here
DATATARGET_BASE_URL=https://hrs.example.com
DATATARGET_USER_ID=admin

# ❌ Wrong: Never prefix these
HARNESS_DATASOURCE_BASE_URL=...  # DON'T DO THIS
MY_PREFIX_API_KEY=...              # Use DATASOURCE_API_KEY instead
```

**Why**: These credentials and endpoints are shared across multiple harnesses. Prefixing creates duplication and maintenance burden.

## Key Normalization: getVarOrEmptyString Method

**Purpose**: Load environment variable and ensure it's accessed for initialization validation

**Common Usage in Harness Blocks**:
```typescript
if (require.main === module) {
  const testEnvironment = TestEnvironment('MY_HARNESS');
  
  // Explicitly declare which variables this harness uses
  [
    'TIMEOUT',
    'BATCH_SIZE',
    'TASK_NAME'
  ].forEach(testEnvironment.getVarOrEmptyString);
  
  main();
}
```

**Why Use This Pattern**:
1. **Self-documenting**: Code explicitly shows which env vars the harness needs
2. **Validation**: Ensures all variables are accessed at startup
3. **Clear dependencies**: Anyone reading the harness knows its configuration requirements
4. **No external lists**: Each harness is independent

## Environment Variable Fallback Chain

```
Priority 1: HARNESS_PREFIX_VARNAME (direct lookup)
            ↓ (not found)
Priority 2: VARNAME (fallback, no prefix)
            ↓ (not found)
Priority 3: Constructor custom value if provided
            ↓ (not found)
Priority 4: Return undefined (caller must handle)
```

**Example**:
```typescript
const env = new TestEnvironment('READ_PERSON', { TIMEOUT: '30000' });

env.getVar('TIMEOUT');      // Returns: READ_PERSON_TIMEOUT if exists, else TIMEOUT if exists, else '30000'
env.getVar('API_KEY');      // Returns: READ_PERSON_API_KEY if exists, else API_KEY if exists, else undefined
```

## The Harness Pattern

**Standard approach for all test harnesses**: Explicit environment variable arrays

```typescript
// In harness module:
import { TestEnvironment } from 'integration-core';

if (require.main === module) {
  const testEnvironment = TestEnvironment('MY_HARNESS');
  
  [
    'TIMEOUT',
    'BATCH_SIZE',
    'RETRY_COUNT',
    'SPECIFIC_CONFIG'
  ].forEach(testEnvironment.getVarOrEmptyString);
  
  main();
}
```

**Benefits**:
- **Self-documenting**: Harness explicitly declares its dependencies
- **Independent**: No centralized list to maintain
- **Clear**: Anyone can see what variables are needed
- **Flexible**: Easy to add/remove variables

## Extending to New Modules

### Adding a New Harness

1. **Identify the harness prefix** (all-caps, module name derivative):
   ```
   ReadPerson → READ_PERSON
   ChunkFromAPI → CHUNK_FROM_API
   PersonCache → PERSON_CACHE
   ```

2. **Determine downstream keys** (what env vars does the harness need?):
   - API endpoints
   - Timeouts
   - Authentication
   - Batch sizes
   - Retry policies

3. **Create harness block with explicit array**:
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

4. **Add to .env**:
   ```
   # ---------- Use these for path/to/MyModule.ts ---------- #
   MY_HARNESS_TIMEOUT=30000
   MY_HARNESS_BATCH_SIZE=200
   ```

5. **Document in example-env.md**:
   ```markdown
   MY_HARNESS_TIMEOUT=<timeout-in-ms>
   MY_HARNESS_BATCH_SIZE=<number-of-records>
   MY_HARNESS_SPECIFIC_CONFIG=<config-value>
   ```

### Adding a New Downstream Variable

When a harness requires a new environment variable:

1. **Add to the harness's explicit array**:
   ```typescript
   [
     'EXISTING_VAR',
     'NEW_VARIABLE'  // Add here
   ].forEach(testEnvironment.getVarOrEmptyString);
   ```

2. **Add to .env** under the harness group:
   ```
   MY_HARNESS_NEW_VARIABLE=value
   ```

3. **Document in example-env.md**:
   ```markdown
   MY_HARNESS_NEW_VARIABLE=<placeholder-value>
   ```

## Troubleshooting

### "Variable not found" Error
**Checklist**:
1. Is it in `.env`? Check: `grep MY_HARNESS_VARNAME .env`
2. Is the prefix correct? `TestEnvironment('MY_HARNESS')` vs `MY_HARNESS_`
3. Did you use `getVar('VARNAME')` without the prefix?
4. Is it a DATASOURCE_* or DATATARGET_* variable? These should be unprefixed

### Explicit Array Implementation
**Symptom**: Not seeing expected environment variables loaded

**Solution**: 
- Ensure array is inside the `if (require.main === module)` block
- Check that `testEnvironment` uses correct prefix
- Verify `.env` has matching `PREFIX_VARNAME` entries

### Shared Variable Conflicts
**Symptom**: Different harnesses need different values for same variable

**Solution**:
- If truly harness-specific: Add prefix and include in harness array
- If shared across harnesses: Keep unprefixed (like DATASOURCE_API_KEY)
- If some harnesses need override: Use ConfigManager chain pattern in production

## Best Practices

1. **Document the exemption rule** in every `.env` file with a comment explaining DATASOURCE_* and DATATARGET_*
2. **Use descriptive harness prefixes** that map clearly to file paths
3. **Use explicit environment variable arrays** in harness blocks for self-documentation
4. **List all harnesses and their prefixes** in project README
5. **Use sanitized placeholders** in example-env.md (never real credentials)

