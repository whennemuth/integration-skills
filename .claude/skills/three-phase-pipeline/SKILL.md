---
category: Architecture & Patterns
description: Understand the three-phase streaming pipeline architecture (chunking, processing, post-processing) and how to test each phase in isolation.
---

# Three-Phase Pipeline Architecture

## Overview
The three-phase pipeline processes large datasets by breaking them into chunks, processing in parallel, and consolidating results. This architecture enables:
- **Parallelism**: Process multiple chunks simultaneously
- **Resilience**: Failed chunks don't block pipeline
- **Scalability**: Add workers without redesigning
- **Observability**: Track progress through each phase

## Phase Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: CHUNKING                                           │
│ Purpose: Stream large payload → NDJSON chunks              │
│ Location: integration-huron-person-fargate/src/chunking/   │
├─────────────────────────────────────────────────────────────┤
│ Input:  Large JSON array or API stream                     │
│ Output: S3/chunks/chunk-0001.ndjson → chunk-NNNN.ndjson   │
│ Signal: SQS message → "chunks ready"                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: PROCESSING (Parallel ECS Fargate Tasks)           │
│ Purpose: Apply transformations to chunk batches            │
│ Location: docker/processor.ts (containerized)              │
├─────────────────────────────────────────────────────────────┤
│ Input:  S3 chunks + SQS message                            │
│ Output: Processed chunks → S3/processed/                   │
│ Signal: SQS message → "processing complete"                │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │ (Parallel Tasks) │ (Parallel Tasks) │
        ↓                  ↓                  ↓
      Task 1            Task 2             Task N
└──────────────────────────┬──────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ PHASE 3: POST-PROCESSING                                    │
│ Purpose: Consolidate results + write delta storage         │
│ Location: docker/post-processor.ts (containerized)         │
├─────────────────────────────────────────────────────────────┤
│ Input:  Processed chunks from S3                           │
│ Output: Delta records → data/delta-storage/TIMESTAMP/      │
│ Side Effects: Update statistics, coordinate soft deletes   │
└─────────────────────────────────────────────────────────────┘
```

## Phase 1: Chunking

### Purpose
Convert large source data (JSON arrays, API streams) into sized chunks that can be processed in parallel.

### Components
- **ChunkerService**: Lambda orchestrator (AWS CDK)
- **ChunkFromAPI**: Fetch data via REST API
- **ChunkFromS3**: Load from S3 file drop
- **BigJsonFetch**: Stream API responses
- **BigJsonFile**: Stream file operations
- **PersonArrayWrapper**: Array → chunks logic

### Implementation Details

**Chunking Process**:
```
Input: [person1, person2, person3, ... person100000]
       ↓
Split into chunks of size N (default 100)
       ↓
Output Files:
  chunk-0001.ndjson: [person1, person2, ... person100]
  chunk-0002.ndjson: [person101, person102, ... person200]
  chunk-0003.ndjson: [person201, person202, ... person300]
  ...
```

**NDJSON Format** (Newline-Delimited JSON):
```json
{"id":"1","name":"Alice","email":"alice@example.com"}
{"id":"2","name":"Bob","email":"bob@example.com"}
{"id":"3","name":"Charlie","email":"charlie@example.com"}
```

**Advantages of NDJSON**:
- Streamable (no need to load entire file)
- One record per line (easy to process)
- Splittable for parallel processing
- Language-agnostic

### Testing Phase 1

**Harness**: `src/chunking/fetch/ChunkFromAPI.ts`

```bash
# Test chunking with .env configuration
npx ts-node src/chunking/fetch/ChunkFromAPI.ts

# Verify:
# 1. API reachable with DATASOURCE_API_KEY
# 2. Chunks created in expected S3 location
# 3. NDJSON format valid (one JSON object per line)
# 4. All records present (no data loss)
```

**Validation Checklist**:
- [ ] Correct number of chunks for payload size
- [ ] Each chunk ≤ configured chunk size
- [ ] No duplicate records across chunks
- [ ] NDJSON format valid (parseable line-by-line)
- [ ] S3 file paths consistent with downstream expectations
- [ ] Metrics logged (total records, chunk count)

### Environment Variables
```
DATASOURCE_BASE_URL=<api-url>           # Shared
DATASOURCE_API_KEY=<api-key>            # Shared
CHUNK_FROM_API_TIMEOUT=30000            # Harness-specific
CHUNK_FROM_API_CHUNK_SIZE=100           # Harness-specific
CHUNK_FROM_API_RETRY_COUNT=3            # Harness-specific
CHUNKER_SERVICE_S3_BUCKET=<bucket>      # Service config
```

## Phase 2: Processing

### Purpose
Apply business logic transformations to each chunk in parallel. Examples:
- Data mapping (field transformation)
- Validation
- Deduplication
- Enrichment

### Parallelism
**Strategy**: One ECS Fargate task per chunk (or batch of chunks)

```
Phase 1 Outputs:
chunk-0001.ndjson
chunk-0002.ndjson
chunk-0003.ndjson
        ↓
Phase 2 Parallel Processing:
Task-1 processes chunk-0001 → processed-0001.ndjson
Task-2 processes chunk-0002 → processed-0002.ndjson
Task-3 processes chunk-0003 → processed-0003.ndjson
        ↓ (All complete)
Phase 3 Post-processor consolidates all processed chunks
```

**Key Benefit**: If one task fails, others continue. Retry failed chunks independently.

### Components
- **docker/processor.ts**: Container entry point
- **DataMapper**: Field-level transformations
- **Validators**: Data quality checks
- **DeactivationStrategy**: Mark records for deletion

### Implementation Details

**Processing Workflow**:
```typescript
// docker/processor.ts
async function processChunk(chunkPath: string): Promise<void> {
  // 1. Fetch chunk from S3
  const chunk = await s3.getObject(chunkPath);
  
  // 2. Parse NDJSON (line by line)
  const records = parseNdjson(chunk);
  
  // 3. Transform each record
  const transformed = records.map(r => dataMapper.map(r));
  
  // 4. Validate
  const valid = transformed.filter(r => validator.isValid(r));
  
  // 5. Write to S3
  await s3.putObject(processedChunkPath, toNdjson(valid));
}
```

### Testing Phase 2

**Harness**: `docker/processor.ts`

```bash
# Test processor locally before ECS deployment
npx ts-node docker/processor.ts

# Verify:
# 1. Processor reads chunks from S3
# 2. Data mapping transforms records correctly
# 3. Invalid records filtered/logged
# 4. Output written to correct S3 location
# 5. Metrics captured (success/fail counts)
```

**Validation Checklist**:
- [ ] Input chunk readable from S3
- [ ] Data mapping produces correct output
- [ ] Validation filters applied
- [ ] Output NDJSON format valid
- [ ] All input records accounted for (success or failure log)
- [ ] Performance acceptable (< 5 min per chunk)

### Environment Variables
```
DATASOURCE_BASE_URL=<api-url>          # Shared
DATATARGET_BASE_URL=<target-url>       # Shared
DOCKER_PROCESSOR_TIMEOUT=300000        # Container config
DOCKER_PROCESSOR_MAX_WORKERS=4         # Parallelism
DATA_MAPPER_FIELD_MAP=<mapping-config> # Transformation
```

## Phase 3: Post-processing

### Purpose
Consolidate all processed chunks and write final results:
- Merge delta records (new/updated/deleted)
- Coordinate soft deletes (deactivations)
- Write to delta storage
- Publish statistics

### Components
- **docker/post-processor.ts**: Consolidation orchestrator
- **DeferredDeleteHandler**: Coordinate deletes
- **StatisticsTable**: Post-processing metrics
- **PersonCache**: Deduplication

### Implementation Details

**Post-processing Workflow**:
```
Phase 2 Outputs:
processed-0001.ndjson
processed-0002.ndjson
processed-0003.ndjson
        ↓
Post-processor Actions:
1. Read all processed chunks
2. Merge into single delta
3. Detect deletions (soft deletes)
4. Coordinate with any concurrent deletes
5. Write to data/delta-storage/{TIMESTAMP}/
6. Record statistics (counts, timing)
```

**Delta Storage Structure**:
```
data/delta-storage/
  20260531_143022/
    delta-persons.ndjson     # New/updated persons
    delta-deletions.ndjson   # Deactivated persons
    metadata.json            # Stats: {processed: 1000, created: 800, updated: 150, deleted: 50}
```

### Testing Phase 3

**Harness**: `docker/post-processor.ts`

```bash
# Test post-processor after processor completes
npx ts-node docker/post-processor.ts

# Verify:
# 1. Reads all processed chunks
# 2. Merges into coherent delta
# 3. Soft delete coordination works
# 4. Delta storage written correctly
# 5. Statistics accurate
```

**Validation Checklist**:
- [ ] All chunks processed before merge
- [ ] Delta output format correct (NDJSON)
- [ ] Deletion coordination prevents conflicts
- [ ] Statistics match actual records
- [ ] Timestamp directory created
- [ ] No data loss from merge

### Environment Variables
```
DATATARGET_BASE_URL=<target-url>           # Shared
DOCKER_POST_PROCESSOR_TIMEOUT=600000       # Long timeout
DEFERRED_DELETE_HANDLER_BATCH_SIZE=1000    # Delete batching
STATISTICS_TABLE_UPDATE_INTERVAL=5000      # Metrics update
```

## End-to-End Testing

### Local Validation (Before Deployment)

1. **Test Phase 1 (Chunking)**
   ```bash
   npx ts-node src/chunking/fetch/ChunkFromAPI.ts
   # Verify chunks in S3
   ```

2. **Test Phase 2 (Processing)**
   ```bash
   npx ts-node docker/processor.ts
   # Verify processed chunks in S3
   ```

3. **Test Phase 3 (Post-processing)**
   ```bash
   npx ts-node docker/post-processor.ts
   # Verify delta storage created
   ```

### Integration Testing

```bash
# Full pipeline (all 3 phases)
npm run test:integration

# Expectations:
# 1. Input data processed end-to-end
# 2. Delta storage contains correct records
# 3. Statistics match expectations
# 4. No data loss
# 5. Performance within SLA
```

## Common Issues & Debugging

### "Processor fails on chunk"
1. Check chunk format (valid NDJSON?)
2. Verify data mapping rules
3. Check error logs from processor task
4. Retry specific chunk independently

### "Post-processor doesn't find processed chunks"
1. Verify processor completed successfully
2. Check S3 path configuration
3. Verify IAM permissions for S3 access
4. Check post-processor S3 bucket/prefix configuration

### "Statistics don't match actual records"
1. Verify all chunks processed
2. Check deduplication logic
3. Review deletion coordination
4. Compare input vs output counts

### "Performance degradation"
1. Reduce chunk size (more parallelism)
2. Increase ECS task resources
3. Check S3 throughput limits
4. Optimize data mapping logic

## Scaling Considerations

- **Chunk Size**: Smaller chunks = more parallelism, higher overhead
- **Task Count**: More tasks = parallel throughput, higher AWS costs
- **S3 Throughput**: Consider regional limits
- **Network**: Data transfer between phases (chunker → processor → post-processor)

