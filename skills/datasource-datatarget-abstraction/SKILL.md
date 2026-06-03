---
category: Architecture & Patterns
description: Implement DataSource and DataTarget abstractions with proper authentication patterns (API Key vs JWT Token), error handling, and test harness structure.
---

# DataSource & DataTarget Abstraction Patterns

## Overview
DataSource and DataTarget are abstract base classes from `integration-core` that define how to:
- **DataSource**: Consume data (fetch persons from APIs, files, databases)
- **DataTarget**: Persist data (write persons to target systems)

Both manage authentication, error handling, and provide testable interfaces via harnesses.

## DataSource Pattern

### Purpose
Abstract the complexity of data ingestion across multiple sources:
- REST APIs (Huron IRB)
- S3 file drops
- Databases
- Legacy systems

### Base Class Interface

```typescript
// integration-core/src/DataSource.ts
export abstract class DataSource {
  abstract fetch(): Promise<Person[]>;
  abstract validateCredentials(): Promise<boolean>;
  abstract getSourceMetadata(): SourceMetadata;
}
```

### Implementation Pattern: API-Based DataSource

```typescript
// src/data-source/PersonDataSource.ts
import { DataSource } from 'integration-core';
import axios, { AxiosInstance } from 'axios';

export class PersonDataSource extends DataSource {
  private httpClient: AxiosInstance;
  private baseUrl: string;
  private apiKey: string;

  constructor(baseUrl: string, apiKey: string) {
    super();
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
    this.httpClient = axios.create({
      baseURL: baseUrl,
      headers: {
        'x-api-key': apiKey,
        'Content-Type': 'application/json'
      },
      timeout: 30000
    });
  }

  async fetch(): Promise<Person[]> {
    try {
      console.log('Fetching persons from API...');
      
      const response = await this.httpClient.get('/api/persons', {
        params: { limit: 10000, offset: 0 }
      });
      
      console.log(`Fetched ${response.data.length} persons`);
      return response.data;
      
    } catch (error) {
      throw new DataSourceError(`Failed to fetch persons: ${error.message}`);
    }
  }

  async validateCredentials(): Promise<boolean> {
    try {
      const response = await this.httpClient.get('/api/health');
      return response.status === 200;
    } catch (error) {
      console.error('Credential validation failed:', error.message);
      return false;
    }
  }

  getSourceMetadata() {
    return {
      name: 'PersonDataSource',
      type: 'REST_API',
      endpoint: this.baseUrl
    };
  }
}
```

### Implementation Pattern: File-Based DataSource

```typescript
// src/data-source/PeopleS3DataSource.ts
import { DataSource } from 'integration-core';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { readableToString } from '@aws-sdk/util-stream-node';

export class PeopleS3DataSource extends DataSource {
  private s3Client: S3Client;
  private bucket: string;
  private key: string;

  constructor(bucket: string, key: string, region = 'us-east-2') {
    super();
    this.bucket = bucket;
    this.key = key;
    this.s3Client = new S3Client({ region });
  }

  async fetch(): Promise<Person[]> {
    try {
      console.log(`Fetching persons from S3: s3://${this.bucket}/${this.key}`);
      
      const command = new GetObjectCommand({
        Bucket: this.bucket,
        Key: this.key
      });
      
      const response = await this.s3Client.send(command);
      const body = await readableToString(response.Body as any);
      
      // Parse based on file format (JSON, NDJSON, CSV)
      const persons = this.parseFormat(body);
      
      console.log(`Fetched ${persons.length} persons from S3`);
      return persons;
      
    } catch (error) {
      throw new DataSourceError(`Failed to fetch from S3: ${error.message}`);
    }
  }

  async validateCredentials(): Promise<boolean> {
    try {
      // List objects to verify S3 access
      const command = new ListObjectsV2Command({
        Bucket: this.bucket,
        MaxKeys: 1
      });
      await this.s3Client.send(command);
      return true;
    } catch (error) {
      console.error('S3 access validation failed:', error.message);
      return false;
    }
  }

  private parseFormat(content: string): Person[] {
    // Support multiple formats
    if (this.key.endsWith('.ndjson')) {
      return content.split('\n')
        .filter(line => line.trim())
        .map(line => JSON.parse(line));
    } else if (this.key.endsWith('.json')) {
      return JSON.parse(content);
    }
    throw new Error(`Unsupported format: ${this.key}`);
  }

  getSourceMetadata() {
    return {
      name: 'PeopleS3DataSource',
      type: 'S3_FILE',
      location: `s3://${this.bucket}/${this.key}`
    };
  }
}
```

### DataSource Test Harness

```typescript
// src/data-source/PersonDataSource.ts (harness block)
if (require.main === module) {
  const env = new TestEnvironment('PERSON_DATA_SOURCE');
  
  const baseUrl = env.getVar('DATASOURCE_BASE_URL') || 'https://api.example.com';
  const apiKey = env.getVar('DATASOURCE_API_KEY');
  
  async function main() {
    if (!apiKey) {
      throw new Error('DATASOURCE_API_KEY not set in .env');
    }
    
    const source = new PersonDataSource(baseUrl, apiKey);
    
    console.log('Validating credentials...');
    const valid = await source.validateCredentials();
    console.log(`Credentials valid: ${valid}`);
    
    if (valid) {
      console.log('Fetching persons...');
      const persons = await source.fetch();
      console.log(`Fetched ${persons.length} persons`);
      
      // Display sample
      if (persons.length > 0) {
        console.log('Sample person:', JSON.stringify(persons[0], null, 2));
      }
    }
  }
  
  main().catch(console.error);
}
```

## DataTarget Pattern

### Purpose
Abstract writing data back to target systems:
- Target HRS system
- Update organizational structures
- Manage person deactivations
- Handle authentication & token refresh

### Base Class Interface

```typescript
// integration-core/src/DataTarget.ts
export abstract class DataTarget {
  abstract read(id: string): Promise<Person | null>;
  abstract write(person: Person): Promise<void>;
  abstract deactivate(id: string, reason?: string): Promise<void>;
  abstract validateCredentials(): Promise<boolean>;
}
```

### Implementation Pattern: JWT Token Authentication

```typescript
// src/data-target/crud/ReadPerson.ts
import { DataTarget } from 'integration-core';
import axios, { AxiosInstance } from 'axios';

export class PersonDataTarget extends DataTarget {
  private httpClient: AxiosInstance;
  private baseUrl: string;
  private userId: string;
  private password: string;
  private token: string | null = null;
  private tokenExpiresAt: Date | null = null;

  constructor(baseUrl: string, userId: string, password: string) {
    super();
    this.baseUrl = baseUrl;
    this.userId = userId;
    this.password = password;
    this.httpClient = axios.create({ baseURL: baseUrl });
  }

  async validateCredentials(): Promise<boolean> {
    try {
      await this.ensureValidToken();
      return true;
    } catch (error) {
      console.error('Token validation failed:', error.message);
      return false;
    }
  }

  async read(id: string): Promise<Person | null> {
    try {
      await this.ensureValidToken();
      const response = await this.httpClient.get(`/api/v2/persons/${id}`, {
        headers: { 'Authorization': `Bearer ${this.token}` }
      });
      return response.data;
    } catch (error: any) {
      if (error.response?.status === 404) {
        return null;  // Person not found
      }
      throw new DataTargetError(`Failed to read person ${id}: ${error.message}`);
    }
  }

  async write(person: Person): Promise<void> {
    try {
      await this.ensureValidToken();
      
      // Determine if create or update based on ID
      if (person.id) {
        await this.httpClient.put(`/api/v2/persons/${person.id}`, person, {
          headers: { 'Authorization': `Bearer ${this.token}` }
        });
        console.log(`Updated person ${person.id}`);
      } else {
        const response = await this.httpClient.post('/api/v2/persons', person, {
          headers: { 'Authorization': `Bearer ${this.token}` }
        });
        console.log(`Created person with ID ${response.data.id}`);
      }
    } catch (error) {
      throw new DataTargetError(`Failed to write person: ${error.message}`);
    }
  }

  async deactivate(id: string, reason?: string): Promise<void> {
    try {
      await this.ensureValidToken();
      await this.httpClient.patch(`/api/v2/persons/${id}/deactivate`, 
        { reason },
        { headers: { 'Authorization': `Bearer ${this.token}` } }
      );
      console.log(`Deactivated person ${id}`);
    } catch (error) {
      throw new DataTargetError(`Failed to deactivate person ${id}: ${error.message}`);
    }
  }

  private async ensureValidToken(): Promise<void> {
    if (this.token && this.tokenExpiresAt && new Date() < this.tokenExpiresAt) {
      return;  // Token still valid
    }
    
    console.log('Obtaining new token...');
    const credentials = Buffer.from(`${this.userId}:${this.password}`).toString('base64');
    
    const response = await this.httpClient.post('/loginsvc/api/v1/token/', {}, {
      headers: { 'Authorization': `Basic ${credentials}` }
    });
    
    this.token = response.data.access_token;
    // Assume token valid for 1 hour
    this.tokenExpiresAt = new Date(Date.now() + 60 * 60 * 1000);
    
    console.log(`Token obtained, expires at ${this.tokenExpiresAt.toISOString()}`);
  }
}
```

### DataTarget Test Harness

```typescript
// src/data-target/crud/ReadPerson.ts (harness block)
if (require.main === module) {
  const env = new TestEnvironment('READ_PERSON');
  
  const targetUrl = env.getVar('DATATARGET_BASE_URL');
  const userId = env.getVar('DATATARGET_USER_ID');
  const password = env.getVar('DATATARGET_PASSWORD');
  
  async function main() {
    if (!targetUrl || !userId || !password) {
      throw new Error('Missing target configuration in .env');
    }
    
    const target = new PersonDataTarget(targetUrl, userId, password);
    
    console.log('Validating credentials...');
    const valid = await target.validateCredentials();
    console.log(`Credentials valid: ${valid}`);
    
    if (valid) {
      // Test read
      console.log('\nTesting read operation...');
      const person = await target.read('12345');  // Test ID
      if (person) {
        console.log('Sample person:', JSON.stringify(person, null, 2));
      } else {
        console.log('Person not found (expected for test ID)');
      }
    }
  }
  
  main().catch(console.error);
}
```

## Authentication Pattern Comparison

### Pattern 1: API Key Authentication
**When to Use**: Simple REST APIs with stateless auth

```typescript
const headers = {
  'x-api-key': apiKey,
  'Content-Type': 'application/json'
};
// No token refresh needed
```

**Pros**: Simple, no state management
**Cons**: Limited security, no expiration

### Pattern 2: JWT Token with Refresh
**When to Use**: Complex systems requiring token management

```typescript
// Obtain token
const token = await obtainToken(userId, password);

// Use token
const headers = {
  'Authorization': `Bearer ${token}`,
  'Content-Type': 'application/json'
};

// Refresh when expired
if (tokenExpired(token)) {
  const newToken = await refreshToken();
}
```

**Pros**: Better security, token expiration
**Cons**: State management complexity, refresh logic needed

### Pattern 3: OAuth 2.0
**When to Use**: Multi-system federated auth

```typescript
const token = await oauth2.getToken({
  client_id: clientId,
  client_secret: clientSecret
});
```

**Pros**: Most secure, industry standard
**Cons**: Most complex, requires OAuth provider

## Adding a New DataSource

1. **Extend DataSource**:
   ```typescript
   export class MyDataSource extends DataSource {
     async fetch(): Promise<Person[]> { }
     async validateCredentials(): Promise<boolean> { }
     getSourceMetadata() { }
   }
   ```

2. **Add harness block**:
   ```typescript
   if (require.main === module) {
     const testEnvironment = TestEnvironment('MY_DATA_SOURCE');
     [
       'API_URL',
       'TIMEOUT',
       'BATCH_SIZE'
     ].forEach(testEnvironment.getVarOrEmptyString);
     main();
   }
   ```

3. **Add environment variables**:
   ```bash
   MY_DATA_SOURCE_TIMEOUT=30000
   MY_DATA_SOURCE_RETRY_COUNT=3
   ```

4. **Add to example-env.md**
5. **Add test harness to README**

## Error Handling

### DataSource Errors
```typescript
class DataSourceError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DataSourceError';
  }
}

// Throw for connection failures, invalid data
throw new DataSourceError(`API returned invalid data: ${error.message}`);
```

### DataTarget Errors
```typescript
class DataTargetError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DataTargetError';
  }
}

// Throw for write failures, auth issues
throw new DataTargetError(`Failed to write person: ${error.message}`);
```

## Testing DataSources & DataTargets

### Unit Test
```typescript
describe('PersonDataSource', () => {
  it('should fetch persons from API', async () => {
    const mockHttp = jest.spyOn(axios, 'get')
      .mockResolvedValueOnce({ data: [{ id: '1', name: 'Test' }] });
    
    const source = new PersonDataSource('http://api', 'key');
    const persons = await source.fetch();
    
    expect(persons).toHaveLength(1);
    expect(persons[0].name).toBe('Test');
  });
});
```

### Integration Test (Harness)
```bash
npx ts-node src/data-source/PersonDataSource.ts
npx ts-node src/data-target/crud/ReadPerson.ts
```

