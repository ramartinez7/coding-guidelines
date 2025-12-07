# C# Coding Guidelines

> Advanced C# patterns at the intersection of Domain-Driven Design, Type-Driven Development, and Functional Programming.

## What Is This?

This repository is a comprehensive catalog of C# coding patterns that help you write:

- **Safer Code** - Make illegal states unrepresentable
- **More Maintainable Code** - Compress knowledge into types
- **Secure Code** - Security by construction, not policy
- **Correct Code** - Catch errors at compile-time, not runtime

## Philosophy

These patterns draw from multiple disciplines:

- **Domain-Driven Design** - Code that matches business reality
- **Type-Driven Development** - If it compiles, it probably works
- **Algebraic Data Types** - Compose types precisely (AND/OR)
- **Functional Programming** - Honest functions and immutability
- **Correctness by Construction** - Rely on the compiler, not tests alone

Read the full [Philosophy](./csharp/philosophy.md) to understand the "why" behind these patterns.

## Quick Start

### Browse the Catalog

Start with the [Pattern Index](./csharp/patterns/__index__.md) to see all available patterns organized by category:

- **Code Smells** - Recognize and fix common problems
- **Refactorings** - Transform code to be safer
- **Type Safety** - Leverage the type system
- **Security** - Build secure code by design
- **Performance** - Optimize without sacrificing safety

### Popular Patterns

New to these concepts? Start here:

1. **[Primitive Obsession](./csharp/patterns/primitive-obsession.md)** - Why `string` and `int` aren't always the answer
2. **[Honest Functions](./csharp/patterns/honest-functions.md)** - Functions that tell the truth in their signature
3. **[Strongly Typed IDs](./csharp/patterns/strongly-typed-ids.md)** - Never swap `CustomerId` and `OrderId` again
4. **[Value Semantics](./csharp/patterns/value-semantics.md)** - Immutable value objects with C# records
5. **[Capability Security](./csharp/patterns/capability-security.md)** - Authorization via types, not checks

### Understand the Style

All code examples follow the conventions in [Style Guide](./csharp/style.md).

## Contributing

**We welcome contributions!** Whether you want to:

- Add a new pattern
- Improve existing documentation
- Fix typos or errors
- Suggest better examples

Please read [CONTRIBUTING.md](./CONTRIBUTING.md) for our collaboration strategy. It includes:

- **Conflict Prevention** - How to avoid merge conflicts when multiple people contribute
- **Pattern Template** - Structure for new patterns
- **Review Process** - What to expect
- **Style Guidelines** - Keep documentation consistent

### Quick Contribution Guide

1. **Check [open PRs](../../pulls)** to avoid duplicate work
2. **Open a draft PR early** to claim your work
3. **One pattern per PR** - keeps reviews focused
4. **Follow the template** - maintains consistency
5. **Link related patterns** - help readers discover connections

See [CONTRIBUTING.md](./CONTRIBUTING.md) for full details.

## Repository Structure

```
/csharp/
  philosophy.md           # Core philosophies behind the patterns
  style.md                # C# coding style conventions
  /patterns/
    __index__.md          # Catalog of all patterns
    *.md                  # Individual pattern files
```

## Who Is This For?

These patterns are for C# developers who want to:

- Write code that's **harder to use incorrectly**
- Shift errors **from runtime to compile-time**
- Apply **functional programming** concepts in C#
- Understand **Domain-Driven Design** tactical patterns
- Build **secure systems** with type-level guarantees

**Level**: Intermediate to Advanced. These patterns assume familiarity with C# basics.

## Examples

### Before: Primitive Obsession

```csharp
public void ProcessPayment(string customerId, string orderId, decimal amount)
{
    // Can accidentally swap customerId and orderId
    // No validation, amount could be negative
}
```

### After: Strongly Typed

```csharp
public Result<Payment, PaymentError> ProcessPayment(
    CustomerId customerId,
    OrderId orderId,
    Money amount)
{
    // Can't swap IDs - compiler error
    // Money enforces positive amounts
    // Result makes failure explicit
}
```

See [Strongly Typed IDs](./csharp/patterns/strongly-typed-ids.md) and [Honest Functions](./csharp/patterns/honest-functions.md) for full explanations.

## Key Principles

| Principle | Description |
|-----------|-------------|
| **Make Illegal States Unrepresentable** | Design types so invalid states cannot exist |
| **Parse, Don't Validate** | Transform data into trusted types |
| **Honest Functions** | Signatures reveal all possible outcomes |
| **Correctness by Construction** | Compiler catches errors before production |
| **Security by Construction** | Unauthorized actions are impossible to express |
| **Zero-Trust Intra-Code** | Every function call is a trust boundary |

Read more in [Philosophy](./csharp/philosophy.md).

## Pattern Categories

### 🏗️ Project Configuration
Foundation settings that enable type safety across your codebase.

### 🔍 Code Smells
Problems to recognize in existing code.

### ♻️ Refactorings
Transformations that improve code safety and clarity.

### 🛡️ Type Safety
Patterns that leverage the C# type system for correctness.

### 🔐 Security
Patterns for building secure code by design.

### ⚡ Performance
Optimize without sacrificing type safety.

### 🎯 API Design
Type-safe patterns for web APIs.

See the full catalog: [Pattern Index](./csharp/patterns/__index__.md)

## Learning Path

**Beginner** (New to Type-Driven Development):
1. [Primitive Obsession](./csharp/patterns/primitive-obsession.md)
2. [Strongly Typed IDs](./csharp/patterns/strongly-typed-ids.md)
3. [Value Semantics](./csharp/patterns/value-semantics.md)
4. [Static Factory Methods](./csharp/patterns/static-factory-methods.md)

**Intermediate** (Comfortable with types):
1. [Honest Functions](./csharp/patterns/honest-functions.md)
2. [Enum to Class Hierarchy](./csharp/patterns/enum-to-class-hierarchy.md)
3. [Data Clumps](./csharp/patterns/data-clump.md)
4. [Boolean Blindness](./csharp/patterns/boolean-blindness.md)

**Advanced** (Ready for deep patterns):
1. [Capability Security](./csharp/patterns/capability-security.md)
2. [Phantom Types](./csharp/patterns/phantom-types.md)
3. [Enforcing Call Order](./csharp/patterns/enforcing-call-order.md)
4. [Assembly Isolation](./csharp/patterns/assembly-isolation.md)

**Hands-On Exercise**:
- [Subscriptions Mini-Domain](./csharp/patterns/exercise-subscriptions.md) - Apply multiple patterns to a realistic scenario

## Resources

- **[Philosophy](./csharp/philosophy.md)** - Understand the foundations
- **[Style Guide](./csharp/style.md)** - C# code conventions
- **[Pattern Index](./csharp/patterns/__index__.md)** - Complete catalog
- **[Contributing](./CONTRIBUTING.md)** - Join the effort

## Feedback

Found something unclear? Have suggestions? Please:

- **Open an issue** - Report problems or suggest improvements
- **Start a discussion** - Ask questions or share ideas
- **Submit a PR** - Contribute directly

## License

This is a documentation repository. All content is provided for educational purposes.

---

**Start exploring**: Check out the [Pattern Index](./csharp/patterns/__index__.md) or read about the [Philosophy](./csharp/philosophy.md) behind these patterns.
