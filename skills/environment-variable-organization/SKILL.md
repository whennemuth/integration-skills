---
category: Configuration Management
description: Organize environment variables in .env files and example-env.md templates with proper grouping, naming conventions, and exemption rules.
---

# Environment Variable Organization & Grouping

## Overview
Proper .env organization is critical for harness maintainability. This skill covers:
- File structure and grouping strategies
- Naming conventions (prefixed vs unprefixed)
- Exemption rules (shared variables)
- Sanitization for documentation templates
- Discovery and validation processes

## .env File Structure

### Standard Layout

```bash
# ========================================
# SHARED VARIABLES (No Prefix)
# ========================================
# These are shared across ALL harnesses

DATASOURCE_BASE_URL=https://api-source.example.com
DATASOURCE_API_KEY=<your-api-key>
DATASOURCE_TIMEOUT=30000

DATATARGET_BASE_URL=https://hrs-target.example.com
DATATARGET_USER_ID=admin-user
DATATARGET_PASSWORD=<password>
DATATARGET_AUTH_TOKEN=<jwt-token>
DATATARGET_LOGIN_SVC_PATH=/loginsvc/api/v1/token/

# ========================================
# HARNESS GROUPS (Prefixed Variables)
# ========================================
# Organized by harness or pipeline phase

# ---------- Use these for src/data-mapper/DataMapper.ts ---------- #
DATA_MAPPER_TIMEOUT=10000
DATA_MAPPER_BATCH_SIZE=100
DATA_MAPPER_LOG_LEVEL=info

# ---------- Use these for src/data-source/PersonDataSource.ts ---------- #
PERSON_DATA_SOURCE_RETRY_COUNT=3
PERSON_DATA_SOURCE_CHUNK_SIZE=500

# ---------- Use these for src/data-target/crud/ReadPerson.ts ---------- #
READ_PERSON_TIMEOUT=15000
READ_PERSON_CACHE_ENABLED=true

# ... more harness groups ...
```

## Naming Conventions

### Shared Variables (NEVER PREFIX THESE)

```bash
# Data Source Credentials & Endpoints
DATASOURCE_BASE_URL=...
DATASOURCE_API_KEY=...
DATASOURCE_TIMEOUT=...

# Data Target Credentials & Endpoints
DATATARGET_BASE_URL=...
DATATARGET_USER_ID=...
DATATARGET_PASSWORD=...
DATATARGET_AUTH_TOKEN=...
DATATARGET_LOGIN_SVC_PATH=...
```

**Why Unprefixed?**
- Used by multiple harnesses (duplication prevention)
- Represent shared infrastructure (single source of truth)
- Change together (credential rotation, endpoint migration)
- Easier to find and update

### Harness-Specific Variables (ALWAYS PREFIX THESE)

```bash
# Pattern: PREFIX_DESCRIPTIVE_NAME
READ_PERSON_TIMEOUT=15000           # READ_PERSON is the harness
SYNC_PEOPLE_BATCH_SIZE=200          # SYNC_PEOPLE is the harness
CHUNK_FROM_API_RETRY_COUNT=5        # CHUNK_FROM_API is the harness
DOCKER_PROCESSOR_MAX_WORKERS=4      # DOCKER_PROCESSOR is the harness
```

**Why Prefixed?**
- Scoped to specific harness (clarity)
- Prevents naming collisions (multiple harnesses might need TIMEOUT)
- Enables harness-specific configuration (overrides for testing)
- Groups variables by concern (visible in .env)

### Naming Best Practices

1. **Use SCREAMING_SNAKE_CASE** for all environment variables
2. **Prefix = Harness identifier** (match file path where possible)
3. **Descriptor = Clear purpose**
   - ✅ `SYNC_PEOPLE_BATCH_SIZE` (clear)
   - ❌ `SYNC_PEOPLE_BS` (unclear abbreviation)
4. **Prefer compound names over suffixes**
   - ✅ `READ_PERSON_CACHE_ENABLED`
   - ❌ `READ_PERSON_CACHE_E`

## Exemption Rule Documentation

Always include this comment at the top of .env:

```bash
# ========================================
# EXEMPTION RULE
# ========================================
# DATASOURCE_* and DATATARGET_* variables are ALWAYS unprefixed.
# These represent shared infrastructure used across multiple harnesses.
# Never create HARNESS_PREFIX_DATASOURCE_* or HARNESS_PREFIX_DATATARGET_*.
# ========================================
```

## .env Grouping Strategy

### By Harness (Recommended for most projects)

```bash
# Each harness gets its own group section
# ---------- Use these for path/to/ModuleA.ts ---------- #
MODULE_A_VAR1=...
MODULE_A_VAR2=...

# ---------- Use these for path/to/ModuleB.ts ---------- #
MODULE_B_VAR1=...
MODULE_B_VAR2=...
```

**Pros**: Easy to find all variables for a harness
**Cons**: Many sections if many harnesses

### By Pipeline Phase (Fargate approach)

```bash
# Group by logical phase
# ---------- PHASE 1: CHUNKING ---------- #
CHUNK_FROM_API_VAR1=...
CHUNK_FROM_S3_VAR1=...
BIG_JSON_FETCH_VAR1=...

# ---------- PHASE 2: PROCESSING ---------- #
DOCKER_PROCESSOR_VAR1=...
DATA_MAPPER_VAR1=...

# ---------- PHASE 3: MERGING ---------- #
DOCKER_MERGER_VAR1=...
DEFERRED_DELETE_HANDLER_VAR1=...
```

**Pros**: Shows pipeline flow, logical grouping
**Cons**: Less obvious which file needs which variable

### By Category (Advanced)

```bash
# Group by variable type
# ---------- TIMEOUTS & PERFORMANCE ---------- #
CHUNK_FROM_API_TIMEOUT=30000
DOCKER_PROCESSOR_MAX_WORKERS=4
SYNC_PEOPLE_BATCH_SIZE=200

# ---------- CREDENTIALS & AUTHENTICATION ---------- #
DATASOURCE_API_KEY=...
DATATARGET_AUTH_TOKEN=...

# ---------- PATHS & ENDPOINTS ---------- #
DATASOURCE_BASE_URL=...
DATATARGET_BASE_URL=...
```

**Pros**: Related variables grouped together
**Cons**: Harder to find all variables for a harness

## Discovery Process: Finding All Environment Variables

When adding a new harness or expanding existing ones:

### Step 1: Identify Direct Variables
```bash
# What does the harness code directly access?
grep -r "process\.env\." src/my-harness.ts
grep -r "env\.getVar(" src/my-harness.ts
```

### Step 2: Trace Downstream Dependencies
```bash
# What does the harness call?
# Check those modules for their env variable usage

# Example: ReadPerson calls DataTarget
# Check DataTarget.ts for env.getVar() calls
# Add those to .env under READ_PERSON harness group
```

### Step 3: Identify Shared Variables
```bash
# Does the downstream dependency use DATASOURCE_* or DATATARGET_*?
# These should stay unprefixed (exemption rule)

# Does the downstream dependency use a variable needed by other harnesses?
# If yes, it should be unprefixed (shared)
# If no, it should be prefixed (harness-specific)
```

### Step 4: Document in .env and example-env.md

```bash
# .env
# ---------- Use these for src/my-harness/MyModule.ts ---------- #
MY_HARNESS_TIMEOUT=30000
MY_HARNESS_BATCH_SIZE=200
# (DATASOURCE_API_KEY stays unprefixed above)
```

## example-env.md: Sanitized Documentation

### Purpose
Provide developers with a template for setting up .env without exposing real credentials.

### Sanitization Rules

**Replace all sensitive values with angle-bracket placeholders:**

```markdown
# ❌ Wrong (exposes real credentials)
DATASOURCE_API_KEY=sk-1234567890abcdef
DATATARGET_PASSWORD=MyActualPassword123!

# ✅ Correct (sanitized placeholders)
DATASOURCE_API_KEY=<your-api-key>
DATATARGET_PASSWORD=<target-system-password>
```

### Placeholder Naming Conventions

Use descriptive placeholders that indicate the type of value:

```bash
# API Keys
DATASOURCE_API_KEY=<your-api-key>
DATASOURCE_API_KEY=<snaplogic-api-key>

# Usernames & Passwords
DATATARGET_USER_ID=<target-system-username>
DATATARGET_PASSWORD=<target-system-password>

# Email addresses
USER_EMAIL=<user-email@bu.edu>

# URLs & Endpoints
DATASOURCE_BASE_URL=<datasource-api-url>
DATATARGET_BASE_URL=<target-system-url>

# AWS Resources
S3_BUCKET_NAME=<your-s3-bucket-name>
S3_REGION=<aws-region>

# AWS ARNs & Secrets Manager paths
SECRET_ARN=<arn:aws:secretsmanager:region:account:secret:path-xxxxx>
QUEUE_URL=<sqs-queue-url>

# IDs & Numbers
BATCH_SIZE=<batch-size-number>
TIMEOUT=<timeout-in-milliseconds>
ACCOUNT_ID=<aws-account-id>
```

### example-env.md Template Structure

```markdown
# Integration Environment Configuration Template

Copy this file to `.env` and replace all placeholders with your actual values.

## Shared Variables (No Prefix)

### Data Source Configuration
DATASOURCE_BASE_URL=<datasource-api-url>
DATASOURCE_API_KEY=<your-api-key>
DATASOURCE_TIMEOUT=30000

### Data Target Configuration
DATATARGET_BASE_URL=<target-system-url>
DATATARGET_USER_ID=<target-system-username>
DATATARGET_PASSWORD=<target-system-password>
DATATARGET_LOGIN_SVC_PATH=/loginsvc/api/v1/token/

## Harness-Specific Variables

### Data Mapping Harnesses
DATA_MAPPER_TIMEOUT=10000
DATA_MAPPER_BATCH_SIZE=100
...
```

## Maintenance Checklist

### When Adding a New Harness:
- [ ] Identify all environment variables the harness needs
- [ ] Trace dependencies for downstream variables
- [ ] Check if any are DATASOURCE_* or DATATARGET_* (keep unprefixed)
- [ ] Add to .env under new harness group
- [ ] Add to example-env.md with sanitized placeholders
- [ ] Update COMMON_TEST_HARNESS_ENV_KEYS if using wrapper pattern
- [ ] Update project README with new harness list

### When Adding a New Environment Variable:
- [ ] Determine if it's shared (should be unprefixed)
- [ ] Add to .env under correct harness group
- [ ] Add to example-env.md with placeholder
- [ ] Add variable to harness explicit array in direct-run block
- [ ] Document in harness section of README

### When Rotating Credentials:
- [ ] Update all instances in .env
- [ ] Do NOT update example-env.md (keep as template)
- [ ] Verify ConfigManager chain loads new credentials
- [ ] Test harnesses with new credentials

### Quarterly Review:
- [ ] Compare COMMON_TEST_HARNESS_ENV_KEYS with actual harness usage
- [ ] Remove unused variables
- [ ] Consolidate duplicated variable purposes
- [ ] Update comments/documentation

## Common Mistakes

❌ **Mistake 1**: Prefixing DATASOURCE_* variables
```bash
HARNESS_DATASOURCE_API_KEY=...  # Wrong!
```

✅ **Correct**: Keep unprefixed
```bash
DATASOURCE_API_KEY=...  # Right
```

❌ **Mistake 2**: Inconsistent naming conventions
```bash
READPERSON_TIMEOUT=...      # No underscore after prefix
read_person_timeout=...     # Wrong case
READ_PERSON_TO=...          # Abbreviated
```

✅ **Correct**: Consistent naming
```bash
READ_PERSON_TIMEOUT=...     # Correct
```

❌ **Mistake 3**: Exposing credentials in example-env.md
```markdown
DATASOURCE_API_KEY=sk-1234567890abcdef  # Wrong!
```

✅ **Correct**: Use placeholder
```markdown
DATASOURCE_API_KEY=<your-api-key>  # Right
```

