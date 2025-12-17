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

### [Test Categories and Traits](./testing-test-categories.md)

Organize tests with categories and traits—run specific subsets of tests based on speed, environment, or purpose.

## Test Data Management

### [Data-Driven Testing](./testing-data-driven-tests.md)

Load test data from external sources—CSV files, JSON, databases—to run comprehensive test suites without hardcoding values.

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

### [Mutation Testing](./testing-mutation-testing.md)

Using mutation testing to verify test quality—ensure your tests actually catch bugs by introducing deliberate defects.

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

## MSTest and FluentAssertions Patterns

### [Custom Assertions](./testing-custom-assertions.md)

Create domain-specific assertions for clearer, more maintainable tests—express business validations in the language of your domain.

### [Assertion Scopes](./testing-assertion-scopes.md)

Use FluentAssertions assertion scopes to report all failures at once—see all assertion failures in a single test run.

### [Testing Object Graphs](./testing-object-graphs.md)

Test complex object graphs and deep equality—verify nested objects, circular references, and structural equivalence.

### [Testing Record Types](./testing-record-types.md)

Verify record type behavior—equality, immutability, with-expressions, and deconstruction.

### [Testing Equality and Comparison](./testing-equality-comparison.md)

Verify equality operators, IEquatable<T>, and comparison logic—ensure consistent behavior across all equality methods.

### [Testing Collection Assertions](./testing-collection-assertions.md)

Verify collection contents, order, and properties using FluentAssertions' powerful collection testing capabilities.

### [Testing String Assertions](./testing-string-assertions.md)

Verify string content, format, and patterns using FluentAssertions' comprehensive string testing capabilities.

### [Testing Numeric Assertions](./testing-numeric-assertions.md)

Verify numeric values, ranges, precision, and arithmetic operations using FluentAssertions' numeric testing capabilities.

### [Testing Boolean Logic](./testing-boolean-logic.md)

Verify boolean values, conditions, and logical operations using clear and expressive assertions.

### [Testing Reference Types and Null](./testing-reference-types-null.md)

Verify reference type behavior, null handling, and object identity using FluentAssertions.

### [Testing Nullable Value Types](./testing-nullable-value-types.md)

Verify nullable value type behavior, HasValue/Value properties, and null handling with GetValueOrDefault.

### [Testing LINQ Queries](./testing-linq-queries.md)

Verify LINQ query results, transformations, filtering, and projections using FluentAssertions.

### [Testing Event Raising](./testing-event-raising.md)

Verify that events are raised correctly, with proper event arguments and timing using FluentAssertions event monitoring.

### [Testing Timeouts and Delays](./testing-timeouts-delays.md)

Verify asynchronous operations complete within expected time limits and test timeout behavior using FluentAssertions.

### [Testing Concurrency](./testing-concurrency.md)

Verify thread-safe operations, concurrent access patterns, and race conditions using MSTest and FluentAssertions.

### [Testing Disposable Resources](./testing-disposable-resources.md)

Verify proper disposal of IDisposable resources, using statements, and resource cleanup using FluentAssertions.

### [Testing Factory Methods](./testing-factory-methods.md)

Verify factory method behavior, object creation, validation, and error handling using MSTest and FluentAssertions.

### [Testing Guard Clauses](./testing-guard-clauses.md)

Verify input validation, precondition checks, and defensive programming using MSTest and FluentAssertions.

### [Testing Result Patterns](./testing-result-patterns.md)

Verify Result<T> monadic error handling, success/failure cases, and composition using FluentAssertions.

### [Testing Option Patterns](./testing-option-patterns.md)

Verify Option<T> monadic handling of optional values, Some/None cases, and mapping operations using FluentAssertions.

### [Testing Refinement Types](./testing-refinement-types.md)

Verify constrained types, validation rules, and type-level guarantees using MSTest and FluentAssertions.

### [Testing Smart Constructors](./testing-smart-constructors.md)

Verify validated object construction, factory methods that enforce invariants, and constructor guards using FluentAssertions.

### [Testing Type-Safe Builders](./testing-type-safe-builders.md)

Verify builder pattern implementation, fluent APIs, immutability, and build validation using MSTest and FluentAssertions.

## Modern C# Features Testing

### [Testing Inheritance and Polymorphism](./testing-inheritance-hierarchies.md)

Test inheritance hierarchies and polymorphic behavior—verify correct dispatch, override behavior, and substitutability.

### [Testing Extension Methods](./testing-extension-methods.md)

Test extension methods effectively—verify they enhance existing types without modifying them, handle edge cases, and compose correctly.

### [Testing Static Methods](./testing-static-methods.md)

Test static methods effectively—verify pure functions, manage static state, and handle singleton patterns.

### [Testing Readonly Collections](./testing-readonly-collections.md)

Test readonly and immutable collections—verify immutability guarantees, prevent modifications, and ensure thread safety.

### [Testing Lazy Initialization](./testing-lazy-initialization.md)

Test lazy-loaded values and Lazy<T>—verify deferred initialization, thread safety, and memoization.

### [Testing Operator Overloading](./testing-operator-overloading.md)

Test custom operators—verify arithmetic, comparison, equality, and conversion operators work correctly.

### [Testing Indexers and Properties](./testing-indexers-properties.md)

Test indexers and computed properties—verify getter/setter behavior, bounds checking, and calculations.

### [Testing Enumerators and Yield](./testing-enumerators-yield.md)

Test custom enumerators and yield return—verify iteration, lazy evaluation, and state management.

### [Testing Pattern Matching](./testing-pattern-matching.md)

Test pattern matching expressions—verify type patterns, property patterns, and exhaustiveness.

### [Testing Switch Expressions](./testing-switch-expressions.md)

Test switch expressions and exhaustiveness—verify all cases are handled and return correct values.

### [Testing Record With-Expressions](./testing-record-with-expressions.md)

Test record with-expressions and cloning—verify immutable updates and shallow vs deep copying.

### [Testing Init-Only Setters](./testing-init-only-setters.md)

Test init-only properties and immutability—verify properties can only be set during initialization.

## Advanced Memory Management

### [Testing Weak References](./testing-weak-references.md)

Test memory management with weak references—verify objects can be collected when only weakly referenced.

### [Testing Finalizers and Destructors](./testing-finalizers-destructors.md)

Test finalizers and cleanup patterns—verify resources are released correctly.
