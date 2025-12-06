# Copilot Instructions for Coding Guidelines Repository

This repository is a comprehensive catalog of C# coding patterns and philosophies, focusing on Domain-Driven Design (DDD), Type-Driven Development, and Functional Programming concepts applied to C#.

## Repository Purpose

This is a **documentation repository** that teaches advanced C# patterns at the intersection of:
- Domain-Driven Design (Tactical Patterns)
- Type-Driven Development (TyDD)
- Algebraic Data Types (ADTs)
- Functional Programming (FP)
- Correctness by Construction
- Security by Construction
- Zero-Trust Intra-Code

## Repository Structure

```
/csharp/
  philosophy.md       # Core philosophies behind the patterns
  style.md            # C# coding style conventions
  /patterns/          # Individual pattern documentation
    __index__.md      # Catalog of all patterns
    *.md              # Individual pattern files
```

## Content Guidelines

### Philosophy and Approach

All patterns and documentation in this repository follow these core principles:

1. **Make Illegal States Unrepresentable** - Design types so invalid states cannot exist in code
2. **Parse, Don't Validate** - Transform data into trusted types rather than returning booleans
3. **Honest Functions** - Function signatures should reveal all outcomes (use Result types)
4. **Correctness by Construction** - Rely on compiler to catch errors before production
5. **Security by Construction** - Make unauthorized actions impossible to express in the type system
6. **Zero-Trust Intra-Code** - Every function call is a trust boundary

### Documentation Style

When creating or editing pattern documentation:

- **Structure**: Each pattern should have:
  - Clear title and problem statement
  - Code examples showing "before" (❌) and "after" (✅)
  - Explanation of benefits
  - Related patterns with links
  - Implementation details with C# code

- **Code Examples**: 
  - Use C# 12+ syntax
  - Prefer `record` types for value objects
  - Use pattern matching and switch expressions
  - Show immutable designs with `readonly` and `with` expressions
  - Demonstrate nullable reference types properly

- **Tone**: 
  - Educational but pragmatic
  - Focus on "why" before "how"
  - Use concrete examples from real-world scenarios
  - Avoid academic jargon; prefer clarity

### C# Coding Style

Follow these conventions in all code examples:

- Separate multi-line statements from preceding and following statements with empty lines
- Don't prefix fields with `_`
- Use `this.` when referencing fields
- Place static members above instance members in the type
- Use braces even for single line blocks
- Don't include explicit `internal` or `private` access modifiers when they would be redundant

### Pattern Categories

Patterns are organized into these categories:

1. **Project Configuration** - Foundation settings (nullable reference types, assemblies, configuration)
2. **Code Smells** - Problems to recognize and fix (primitive obsession, data clumps, flag arguments)
3. **Refactorings** - Transformations to improve code (enum to class hierarchy, honest functions)
4. **Type Safety** - Patterns that leverage the type system (strongly typed IDs, value semantics)
5. **Security** - Patterns for secure code (capability security, assembly isolation)

## Working with This Repository

### Adding New Patterns

When adding a new pattern:

1. Create a new `.md` file in `/csharp/patterns/`
2. Use kebab-case for filenames (e.g., `my-new-pattern.md`)
3. Follow the structure of existing patterns
4. Add the pattern to `__index__.md` under the appropriate category
5. Reference related patterns using relative links
6. Update `philosophy.md` if the pattern introduces a new concept

### Editing Existing Patterns

- Make surgical changes - preserve the existing structure and style
- Ensure code examples compile and demonstrate the point clearly
- Update cross-references if changing pattern names or structure
- Keep examples focused - don't add unrelated complexity

### Key Concepts to Maintain

- **Value Objects**: Types defined by their value, not identity
- **Algebraic Data Types**: Product types (AND) and Sum types (OR)
- **Capability Security**: Authorization via unforgeable tokens
- **Static Factory Methods**: Construction that enforces invariants
- **Result Types**: Explicit error handling without exceptions

## Testing and Validation

This is a documentation repository with **no code to build or test**. When making changes:

1. Verify all internal links work (check references between patterns)
2. Ensure code examples are syntactically correct C# (visually inspect)
3. Check that new patterns fit the established philosophical framework
4. Maintain consistency with existing documentation style

## References to Other Documentation

Key documents to reference when creating content:
- `/csharp/philosophy.md` - The "why" behind all patterns
- `/csharp/style.md` - C# code style conventions
- `/csharp/patterns/__index__.md` - The master catalog

## Common Tasks

### Adding a New Pattern
1. Study existing patterns in `/csharp/patterns/` for structure and style
2. Create new pattern file with clear problem/solution
3. Add entry to `__index__.md`
4. Cross-reference related patterns
5. Consider if `philosophy.md` needs updates

### Improving Existing Documentation
1. Maintain consistency with the established voice and structure
2. Improve code examples for clarity
3. Add missing cross-references
4. Ensure C# syntax follows modern best practices

### Reorganizing Content
1. Preserve all existing content - only restructure
2. Update all internal links after moving files
3. Verify the catalog (`__index__.md`) reflects changes
4. Keep the philosophical framework intact

## What NOT to Do

- Don't add build tools, test frameworks, or executable code
- Don't change the core philosophical principles without careful consideration
- Don't introduce patterns that contradict the DDD/Type-Driven approach
- Don't remove or significantly alter working patterns without good reason
- Don't add dependencies or package files

## Summary

This repository teaches advanced C# developers how to write safer, more maintainable code through type-driven design and functional programming concepts. When contributing, focus on clarity, concrete examples, and maintaining the philosophical consistency that makes this catalog cohesive.
