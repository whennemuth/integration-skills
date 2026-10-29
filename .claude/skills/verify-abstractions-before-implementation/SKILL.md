---
description: Verify the actual behavior of unfamiliar abstractions before implementing code that depends on them
globs:
  - "**/*.ts"
  - "**/*.js"
---

# Verify Abstractions Before Implementation Skill

## CRITICAL: Never Implement on Assumptions

When implementing code that depends on unfamiliar abstractions, control flow directives, or domain-specific patterns, you MUST verify their actual behavior before proceeding.

## Mandatory Verification Rules

### 1. Identify High-Risk Abstractions

**Always verify before using:**
- Control flow directives (e.g., `__arrayFieldOperations`, `__metadata`)
- Update semantics (append vs replace, merge vs overwrite)
- Authentication/authorization patterns
- State management abstractions
- Data transformation pipelines
- Sync/async coordination primitives

**Warning signs that verification is needed:**
- Variable names starting with `__` (double underscore)
- Names suggesting behavior (e.g., "Operations", "Directives", "Strategy")
- No obvious implementation in the current file
- Pattern you haven't seen before in this codebase

### 2. Search for Definitions and Usage

**NEVER** assume behavior from names alone.

**BAD:**
```typescript
// Seeing __arrayFieldOperations and inferring it means "append mode"
return { __arrayFieldOperations: { append: roles } };
```

**GOOD:**
```typescript
// First searching for where __arrayFieldOperations is processed
grep_search({ query: "__arrayFieldOperations", isRegexp: false })
// Reading the consumer code to understand actual semantics
// THEN implementing based on verified behavior
```

**Rule:** When you encounter an unfamiliar abstraction that controls critical behavior, IMMEDIATELY search for its definition and usage before making assumptions.

### 3. Verify Through Multiple Sources

Before implementing, verify abstraction behavior through:
- ✅ Definition/interface files (types, contracts)
- ✅ Consumer code (where it's processed/interpreted)  
- ✅ Test files (expected behavior examples)
- ✅ Similar usage patterns elsewhere in codebase
- ✅ Documentation or comments

**Do NOT:**
- ❌ Infer from variable names alone
- ❌ Assume based on "similar patterns" from other projects
- ❌ Proceed with "educated guesses"

### 4. State Your Uncertainty

When you cannot fully verify an abstraction's behavior, you MUST:

**Explicitly state what you don't know:**
```
"I see `__arrayFieldOperations` here. Based on the name, I'm inferring 
it controls append vs replace behavior, but I haven't verified how it's 
actually processed by the target system. Should I search for its 
implementation first?"
```

**Ask for guidance:**
```
"I need to understand how `__arrayFieldOperations` works before 
implementing the role modification logic. Let me search for where 
this directive is consumed."
```

**Do NOT:**
- ❌ Implement silently based on assumptions
- ❌ Present inferred behavior as fact
- ❌ Skip verification to avoid "wasting time"

### 5. Think Through Interactions

Even after verifying individual abstractions, think through their interactions:

**Example: Array Operations**
```typescript
// If __arrayFieldOperations controls append/replace at the target,
// and I'm sending a roles array, what happens if I send:
// - All roles (existing + new) with append directive?
//   → Potentially adds duplicates
// - Only delta roles with append directive?
//   → Correctly adds only new roles
// - Remaining roles with append directive (for "remove" operation)?
//   → WRONG - appends remaining instead of replacing
```

**Checklist for interaction thinking:**
- [ ] What data am I sending?
- [ ] What directive am I using?
- [ ] What will the target system do with this combination?
- [ ] Are there edge cases (empty arrays, duplicates, etc.)?

## Example: The __arrayFieldOperations Bug

**Scenario:** Implementing 'append' and 'remove' operations for role management.

**Wrong Approach (What Happened):**
```typescript
// 'append' operation: Send ALL roles (existing + custom, deduplicated)
const allHrns = [...existingHrns, ...this.roleHrns];
const uniqueHrns = Array.from(new Set(allHrns));
return { roles: uniqueHrns.map(hrn => ({ hrn })) };

// 'remove' operation: Send remaining roles with append directive
const remainingHrns = existingHrns.filter(hrn => !customHrnSet.has(hrn));
return { roles: remainingHrns.map(hrn => ({ hrn })) };
// Later: Keep append directive for 'append' and 'remove' ← BUG
```

**Problems:**
1. 'append' sends roles that may already exist → inefficient, risky
2. 'remove' uses append directive → adds remaining roles instead of replacing → broken functionality

**Correct Approach:**
```typescript
// FIRST: Search for __arrayFieldOperations usage
grep_search({ query: "__arrayFieldOperations", isRegexp: false })

// SECOND: Read how it's processed - discover:
// - Empty {} means "replace mode" (default)
// - Populated { append: field } means "append to existing"

// THIRD: Think through interactions:
// - 'append' + delta roles + append directive → adds only new roles ✓
// - 'remove' + remaining roles + append directive → wrong, appends ✗
// - 'remove' + remaining roles + replace directive → correct ✓

// FOURTH: Implement correctly
if (this.operation === 'append') {
  const deltaHrns = this.roleHrns.filter(hrn => !existingHrnSet.has(hrn));
  return { roles: deltaHrns.map(hrn => ({ hrn })) }; // Uses append directive
} else if (this.operation === 'remove') {
  const remainingHrns = existingHrns.filter(hrn => !customHrnSet.has(hrn));
  return { roles: remainingHrns.map(hrn => ({ hrn })) }; // Uses replace directive
}
```

## Verification Workflow

When implementing code that depends on an unfamiliar abstraction:

1. **Identify the abstraction** - Name, purpose, where it appears
2. **Search for definition** - Find interface, type, or contract
3. **Search for consumers** - Find where it's processed/interpreted
4. **Read usage examples** - Look at tests, similar patterns
5. **Verify your understanding** - State what you think it does
6. **Think through interactions** - Consider edge cases and combinations
7. **Implement with confidence** - Now you know it's correct

## Search Patterns

### Finding Abstraction Definitions
```typescript
// Type definitions
grep_search({ query: "type.*__arrayFieldOperations", isRegexp: true })
grep_search({ query: "interface.*Operations", isRegexp: true })

// Usage/consumers
grep_search({ query: "__arrayFieldOperations", isRegexp: false })
semantic_search({ query: "array field operations processing" })
```

### Finding Similar Patterns
```typescript
// Find other uses of the same pattern
grep_search({ query: "pattern_name", isRegexp: false })

// Find test cases
file_search({ query: "**/*test*.ts" })
grep_search({ query: "pattern_name.*test", isRegexp: true })
```

## Common Patterns to Recognize

### Pattern: "Magic Metadata Fields"
Fields starting with `__` often carry special meaning:
- `__arrayFieldOperations` - Controls array merge behavior
- `__metadata` - Carries processing instructions
- `__type` - Controls polymorphic behavior

**Action:** Always search for how these are consumed before using them.

### Pattern: "Behavioral Enums/Strings"
Values that control behavior:
- "append" vs "replace"
- "merge" vs "overwrite"  
- "create" vs "update" vs "upsert"

**Action:** Verify what each value actually causes to happen.

### Pattern: "Strategy/Factory Patterns"
Abstractions that switch implementations:
- `getStrategy(type)` - Which strategy is actually used?
- `createProcessor(config)` - What processor type is created?

**Action:** Trace through to see which concrete implementation you're actually calling.

## Checklist Before Implementing

Before implementing code that depends on an unfamiliar abstraction, verify you have:

- [ ] Searched for the abstraction's definition/interface
- [ ] Found and read the code that processes/consumes it
- [ ] Looked at test cases or usage examples
- [ ] Stated your understanding explicitly (to yourself or user)
- [ ] Thought through the interactions and edge cases
- [ ] Considered whether your approach makes logical sense
- [ ] Not proceeded on "educated guesses" or assumptions

## Remember

**Naming is not documentation.** Just because something is called `__arrayFieldOperations` doesn't mean you understand what operations it supports or how they work.

**Critical bugs come from assumptions.** The 'remove' operation bug was critical precisely because it was built on an untested assumption about directive behavior.

**You have search tools.** Use grep_search, semantic_search, and file_search liberally. It takes seconds to verify and hours to fix bugs from assumptions.

**Verification is not wasteful.** It's wasteful to implement incorrectly, have the user catch it, and then fix it. Verify first, implement once.
