---
category: Best Practices
description: Craft git commit messages that lead with the problem solved and explain the context before implementation details.
---

# Commit Message Conventions

## Overview
Commit messages in this workspace follow a problem-first pattern that provides readers with context about *why* changes were necessary, not just *what* changed. This enables faster code review, easier git history comprehension, and better architectural understanding for future developers.

## Pattern Structure

### Section 1: Problem Statement (Primary)
**Lead with the problem** being solved. This is the "why":
- What pain point or limitation is being addressed?
- What makes the current state insufficient?
- Be specific: "scattered configuration with no dedicated single-use prefix organization" is better than "configuration management"

**Example**: "Test harnesses had no dedicated environment variable grouping mechanism. Configuration was scattered across the codebase with no single-use prefix organization..."

### Section 2: Context (Optional but Recommended)
**Explain the concept** if the feature/pattern is non-obvious:
- What is a test harness? (isolated executable module for component validation)
- How does the current system work?
- What constraints or architectural patterns apply?

**When to include**: When the feature/system is domain-specific, newly introduced to the codebase, or when understanding it requires technical background.

**Example**: "Test harnesses are isolated executable modules that verify individual components. Each harness loads its own namespaced environment variables..."

### Section 3: Solution (Supporting)
**Brief explanation of the solution** that addresses the problem:
- What pattern or approach was chosen?
- How does it solve the stated problem?
- Why this solution over alternatives?

**Example**: "Introduce TestEnvironment utility supporting prefix-based environment variable isolation. Each harness can load its own namespaced variables, enabling clear separation of concerns..."

### Section 4: Changes (Technical Details)
**Bullet list of what was actually changed**:
- Implementation details
- File modifications
- New exports or patterns introduced

**Format**: Present as implementation details that directly support the solution.

## Examples from This Codebase

### ✅ Good: Problem-First Approach
```
Add test harness infrastructure with TestEnvironment utility

Problem: Test harnesses (isolated executable modules for component validation) had
no dedicated environment variable grouping mechanism. Configuration was scattered
across multiple harness files with no single-use prefix organization, making it
difficult to maintain consistent harness setup and debug environment issues.

Solution: Introduce TestEnvironment utility supporting prefix-based environment
variable isolation. Each harness can load its own namespaced variables, enabling
clear separation of concerns and simplified configuration management.

Changes:
- Export TestEnvironment utility for prefix-based environment variable isolation
- Add test harness documentation to README with environment variable configuration
- Provide dual execution methods: VS Code launch config (recommended) and npx CLI
```

### ❌ Avoid: Implementation-First Approach
```
Add test harness infrastructure

- Export TestEnvironment utility
- Update README
- Add environment variable configuration
- Support npx execution
```
(Lacks context—a reviewer doesn't understand why these changes solve a real problem)

## Key Principles

1. **Lead with why, not what** – The problem statement comes first, implementation details last
2. **Include domain context** – Briefly explain what you're building if it's non-obvious
3. **Multi-harness patterns** – When coordinating across multiple modules, highlight that scattering is the problem being solved
4. **Prefix-based organization** – For environment variable or configuration patterns, call out the "dedicated single-use" grouping principle
5. **Solution clarity** – Explain the architectural approach (centralization, prefix isolation, wrapper patterns, etc.)

## Referencing This Skill

When amending commit messages or reviewing commits:
1. Does the commit lead with a **problem statement**?
2. Is there **context** explaining what the feature/pattern does?
3. Are the **changes** listed as implementation details supporting the solution?
4. Would a reader understand *why* this commit was necessary from just the message?

## Related Patterns

- **TestEnvironment Pattern**: Environment variable isolation via prefix-based namespacing
- **Scattering Problem**: Configuration scattered across multiple files with no unified organization
- **Explicit Configuration**: Patterns (like explicit environment variable arrays in harness blocks) that document dependencies directly in code
- **Key Normalization**: Handling environment variable names that contain prefixes to prevent double-expansion

