---
description: Systematic approach to debugging execution flow issues, especially when code appears to not execute despite passing tests
globs:
  - "**/*.ts"
  - "**/*.js"
---

# Debugging Execution Issues Skill

## CRITICAL: Never Stop at Abstraction Boundaries

When debugging execution issues where code appears to not run despite tests passing, you MUST follow the complete execution path through all called functions.

## Mandatory Debugging Rules

### 1. Always Read Called Function Implementations

**NEVER** assume a called function is "safe" or treat it as a black box when you have access to its source code.

**BAD:**
```typescript
// Seeing this and stopping investigation here
await chunker.runChunking(params);
console.log('✓ Chunker process completed successfully');
```

**GOOD:**
```typescript
// Seeing this and IMMEDIATELY reading the implementation of runChunking()
await chunker.runChunking(params);
```

**Rule:** When you see `await someFunction()` or `someMethod()`, IMMEDIATELY read that function's implementation before making ANY conclusions about execution flow.

### 2. Trace ALL Execution Paths First

Before concluding root cause, trace execution through:
- ✅ The entry point file
- ✅ All called functions/methods  
- ✅ All dependencies those functions use
- ✅ Error handlers (try/catch/finally)
- ✅ Early returns and exit conditions

**Do NOT:**
- ❌ Stop at the first file you read
- ❌ Assume called functions work correctly
- ❌ Skip reading implementations of dependencies

### 3. Search for Control Flow Terminators

When debugging "code not executing" issues, ALWAYS search for:

- `process.exit()` - Terminates Node.js process, bypassing finally blocks
- `throw new Error()` - May escape try/catch blocks
- `return` (early) - Exits function before later code runs
- `process.abort()` - Hard termination
- Unhandled promise rejections

**How to search:**
```typescript
// Use grep_search to find these patterns
grep_search({ query: "process\\.exit\\(", isRegexp: true })
```

### 4. Follow Dependencies Systematically

When you see code like:
```typescript
const chunker = await getChunkerInstance(config, chunkerQueue);
await chunker.runChunking(params);
```

Your checklist:
1. ✅ Read `getChunkerInstance()` implementation
2. ✅ Identify what type `chunker` actually is (ChunkFromAPI? ChunkFromS3?)
3. ✅ Read the `runChunking()` method for THAT specific type
4. ✅ Check for process.exit(), throw, or early returns in runChunking()
5. ✅ Only THEN conclude about execution flow

### 5. Understand Finally Block Bypass Conditions

A `finally` block will NOT execute if:
- `process.exit()` is called (immediately terminates)
- `process.abort()` is called
- Uncaught exception in a different execution context
- SIGKILL signal received

A `finally` block WILL execute if:
- Normal return from try block
- Exception caught and handled
- Exception propagates up the stack (finally runs before propagation)

## Example: The Process.Exit Bug

**Scenario:** Code in a finally block doesn't execute, but tests pass.

**Wrong Approach:**
```typescript
// Looking at chunker.ts and stopping here
finally {
  if(chunker) {
    if(!chunker.noMessagesFromQueue) {
      const message = chunker.getMessage();
      await chunkerQueue.deleteMessageFromQueue(message!); // ← "Why doesn't this run?"
    }
  }
}
```

**Correct Approach:**
```typescript
// 1. Read chunker.runChunking() implementation
// 2. Discover it calls process.exit(0) or process.exit(1)
// 3. Understand process.exit() bypasses finally blocks
// 4. Fix: Return error codes instead of calling process.exit()
```

## Debugging Workflow

When a user reports "line X doesn't execute":

1. **Read the entry point** - Understand the main execution flow
2. **Identify all function calls** - List every function/method called
3. **Read each implementation** - Don't skip any, read them all
4. **Search for terminators** - Look for process.exit(), throw, etc.
5. **Trace the path** - Follow execution from start to the problem line
6. **Verify with tests** - Confirm your hypothesis matches test behavior

## Common Patterns to Recognize

### Pattern: "Code after await doesn't run"
- Check if the awaited function calls process.exit()
- Check if it throws an unhandled exception
- Check if it has an infinite loop or hangs

### Pattern: "Finally block doesn't execute"
- Search for process.exit() calls in the try block
- Check for uncaught exceptions in async callbacks
- Look for SIGTERM/SIGKILL handlers

### Pattern: "Tests pass but production fails"
- Tests may mock the function that calls process.exit()
- Tests may stub out the actual execution path
- Production may have different code paths than tests

## Checklist Before Claiming "Can't Find Issue"

Before saying you cannot find a bug, verify you have:

- [ ] Read ALL files in the execution path, not just the entry point
- [ ] Searched for `process.exit` in all relevant files
- [ ] Traced execution through every called function
- [ ] Checked for early returns, throws, and process termination
- [ ] Understood the difference between test mocks and real execution
- [ ] Not stopped at any abstraction boundary without reading the implementation

## Remember

**You have full access to the source code.** There are NO black boxes except external libraries. If it's user code, READ IT.

**Process.exit() is invisible** if you don't look for it. It doesn't show up in stack traces or error messages. You must actively search for it.

**Tests can hide bugs** by mocking the problem code. A passing test doesn't mean the real code path works correctly.
