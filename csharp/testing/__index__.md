# C# Testing Practices

A catalog of testing patterns and best practices for C#, focusing on type-safe testing, domain-driven design testing patterns, and functional testing approaches.

## Test Structure and Organization

### [Arrange-Act-Assert (AAA) Pattern](./arrange-act-assert.md)

Structure tests in three distinct phases—setup, execution, and verification—for clarity and maintainability.

### [Test Naming Conventions](./test-naming-conventions.md)

Name tests to reveal their purpose—what is being tested, under what conditions, and what is expected.

### [One Assertion Per Test](./one-assertion-per-test.md)

Each test should verify a single logical concept—multiple physical assertions are fine if they verify the same behavior.

### [Test Isolation and Independence](./test-isolation-independence.md)

Tests should not depend on each other—run in any order with consistent results.

### [AAA Pattern with Clear Boundaries](./aaa-clear-boundaries.md)

Use whitespace and comments to visually separate test phases—make the structure obvious at a glance.

## Test Data Management

### [Builder Pattern for Test Data](./test-data-builders.md)

Build complex test objects fluently—specify only what matters for each test, use sensible defaults for the rest.

### [Object Mother Pattern](./object-mother-pattern.md)

Centralize creation of standard test fixtures—reusable, named configurations of test objects.

### [Test Data Builders with Fluent API](./fluent-test-builders.md)

Combine builders with fluent interfaces—expressive, maintainable test data setup.

## Test Doubles and Isolation

### [Test Doubles: Mocks, Stubs, Fakes](./test-doubles.md)

Understand when to use mocks (behavior verification), stubs (state setup), and fakes (working implementations).

### [Testing Side Effects](./testing-side-effects.md)

Verify effects on the outside world—use test doubles to capture and verify interactions.

## Advanced Testing Patterns

### [Property-Based Testing](./property-based-testing.md)

Test invariants across many generated inputs—let the framework discover edge cases you didn't think of.

### [Parameterized Tests](./parameterized-tests.md)

Run the same test logic with different inputs—reduce duplication while covering more scenarios.

### [Snapshot Testing](./snapshot-testing.md)

Capture and verify complex outputs—compare against approved baselines to detect unintended changes.

## Testing Async and Exceptions

### [Testing Async Code](./testing-async-code.md)

Properly test asynchronous operations—await async methods and verify timing-dependent behavior.

### [Testing Exceptions](./testing-exceptions.md)

Verify error conditions—test both that exceptions are thrown and that they contain the right information.

## Domain-Driven Design Testing

### [Testing Domain Invariants](./testing-domain-invariants.md)

Verify business rules are enforced—test that invalid states cannot be created or reached.

### [Testing Value Objects](./testing-value-objects.md)

Test value semantics—equality, immutability, and validation in domain value types.

### [Testing State Machines](./testing-state-machines.md)

Verify state transitions—test valid transitions succeed and invalid ones fail.

### [Testing Discriminated Unions](./testing-discriminated-unions.md)

Exhaustively test all variants—ensure pattern matching handles every case correctly.

## Integration Testing

### [Integration Test Patterns](./integration-test-patterns.md)

Test components working together—balance speed, reliability, and realistic scenarios.
