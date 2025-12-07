# TypeScript Patterns

A catalog of common code smells and refactoring patterns for TypeScript.

## Project Configuration

### [Strict Type Checking](./strict-type-checking.md)

Enable strict mode in TypeScript—foundation for compile-time type safety.

### [Module Organization](./module-organization.md)

Organize code into modules with clear boundaries and dependencies.

## Code Smells

### [Primitive Obsession](./primitive-obsession.md)

Over-use of "primitive" built-in types instead of domain-specific types.

### [Strongly Typed IDs](./strongly-typed-ids.md)

Multiple ID parameters of the same type (string, number)—wrap each in a distinct type to prevent swapping.

### [Data Clumps](./data-clump.md)

Values that always travel together appear as separate parameters.

### [Flag Arguments](./flag-arguments.md)

Boolean or enum parameters that control which code path a function takes—use polymorphism instead.

### [Boolean Blindness](./boolean-blindness.md)

Functions returning `boolean` where `true`/`false` meanings are unclear—use descriptive types instead.

### [Discriminated Unions](./discriminated-unions.md)

Using multiple optional properties where only certain combinations are valid—model variants as a discriminated union instead.

### [Enforcing Call Order](./enforcing-call-order.md)

Runtime guards for function call sequences—use types to make invalid orderings uncompilable.

### [Honest Functions](./honest-functions.md)

Functions that lie about their return type (undefined, exceptions)—use Result types to make failure explicit.

### [Value Semantics](./value-semantics.md)

Mutable objects for value concepts—use readonly properties and immutable patterns instead.

### [Readonly Types](./readonly-types.md)

Using mutable types where immutability is intended—leverage TypeScript's `readonly` and `Readonly<T>` utility types.

### [Capability Security](./capability-security.md)

Checking authorization separately from actions—use capability tokens to make authorization proof part of the type system.

### [Boundary Enforcement](./boundary-enforcement.md)

Using the same interface for API serialization and domain logic—separate DTOs from domain models with explicit mappers.

## Creational Patterns

### [Static Factory Methods](./static-factory-methods.md)

Use static methods instead of constructors to create instances with clear, descriptive names.

### [Builder Pattern](./builder-pattern.md)

Builder patterns that allow construction before required fields are set—use types to enforce the sequence.

## Behavioral Patterns

### [Specification Pattern](./specification-pattern.md)

Business rules buried in scattered predicates—encapsulate rules as named, composable Specification types.

### [Domain Events](./domain-events.md)

Functions with core action plus many side effects—return domain events as types to decouple and announce changes.

## API Patterns

### [Result Types](./result-types.md)

Using exceptions or undefined for error handling—use explicit Result types to make errors type-safe and visible.

### [Opaque Types](./opaque-types.md)

Using type aliases that provide no nominal distinction—create branded types for compile-time safety.

## Type-Driven Development Patterns

### [Phantom Types](./phantom-types.md)

Using runtime checks or comments to track object state—use phantom type parameters to encode state in the type system.

### [Branded Types](./branded-types.md)

Type aliases that provide no type safety—create branded types to distinguish structurally identical but semantically different values.

## Domain-Driven Design Patterns

### [Aggregate Roots](./aggregate-roots.md)

Entities with direct public setters allow invariants to be broken—use aggregate roots to enforce consistency boundaries.

### [Ubiquitous Language](./ubiquitous-language.md)

Technical jargon in code disconnected from business vocabulary—use types that mirror the language domain experts use.
