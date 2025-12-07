# Angular Patterns

A catalog of common patterns and best practices for Angular applications.

## Project Configuration

### [TypeScript Strict Mode](./typescript-strict-mode.md)

Enable TypeScript's strict mode for maximum type safety and early error detection.

### [Template Type Checking](./template-type-checking.md)

Enable Angular's strict template type checking to catch template errors at compile time.

### [Standalone Components](./standalone-components.md)

Use standalone components to simplify module management and improve tree-shaking.

## Component Patterns

### [Smart and Presentational Components](./smart-presentational-components.md)

Separate components that manage state from those that only display data.

### [OnPush Change Detection](./onpush-change-detection.md)

Use OnPush strategy to optimize change detection and improve performance.

### [Input/Output Best Practices](./input-output-best-practices.md)

Follow best practices for component communication through inputs and outputs.

### [Component Composition](./component-composition.md)

Build complex UIs by composing smaller, focused components.

### [Content Projection](./content-projection.md)

Use `ng-content` to create flexible, reusable component templates.

## Reactive Patterns

### [Async Pipe Over Subscribe](./async-pipe-over-subscribe.md)

Prefer the async pipe in templates over manual subscriptions to prevent memory leaks.

### [Declarative Data Loading](./declarative-data-loading.md)

Use observables and the async pipe for declarative, reactive data loading.

### [RxJS Error Handling](./rxjs-error-handling.md)

Handle errors in observable streams properly using catchError and retry operators.

### [Signals for Component State](./signals-component-state.md)

Use Angular Signals for simpler, more efficient component-local state management.

### [Signals vs RxJS](./signals-vs-rxjs.md)

Understand when to use Signals versus RxJS Observables for different use cases.

### [Computed Values](./computed-values.md)

Use computed signals to derive state efficiently from other signals.

## Form Patterns

### [Type-Safe Forms](./type-safe-forms.md)

Use typed forms (Angular 14+) for compile-time form validation and type safety.

### [Reactive Forms Over Template Forms](./reactive-forms-over-template.md)

Prefer reactive forms for complex forms and better testability.

### [Custom Form Controls](./custom-form-controls.md)

Implement ControlValueAccessor to create reusable form controls that integrate with Angular forms.

### [Form Validation Patterns](./form-validation-patterns.md)

Create reusable validators and handle validation errors consistently.

## Service Patterns

### [Service Abstractions](./service-abstractions.md)

Define service interfaces for better testability and loose coupling.

### [Injection Tokens](./injection-tokens.md)

Use InjectionToken for non-class dependencies and configuration.

### [Service State Management](./service-state-management.md)

Manage application state in services using BehaviorSubject or Signals.

### [HTTP Interceptors](./http-interceptors.md)

Use interceptors for cross-cutting concerns like authentication, logging, and error handling.

## Performance Patterns

### [TrackBy Functions](./trackby-functions.md)

Always provide trackBy functions for *ngFor to optimize list rendering.

### [Virtual Scrolling](./virtual-scrolling.md)

Use CDK virtual scrolling for efficiently rendering large lists.

### [Lazy Loading Strategies](./lazy-loading-strategies.md)

Implement lazy loading for feature modules to reduce initial bundle size.

### [Immutable Data Patterns](./immutable-data-patterns.md)

Use immutable data patterns to optimize change detection and prevent bugs.

### [Detach Change Detector](./detach-change-detector.md)

Manually control change detection for performance-critical components.

## Module Organization

### [Feature Module Structure](./feature-module-structure.md)

Organize code into feature modules for better maintainability and lazy loading.

### [Core vs Shared Modules](./core-shared-modules.md)

Understand the difference between Core (singleton services) and Shared (reusable components) modules.

### [Module Providers](./module-providers.md)

Properly configure service providers in modules to control singleton vs multiple instances.

## Routing Patterns

### [Route Guards](./route-guards.md)

Use route guards for authentication, authorization, and navigation control.

### [Route Resolvers](./route-resolvers.md)

Pre-load data before activating routes using resolvers.

### [Lazy Loading Guards](./lazy-loading-guards.md)

Implement guards at the lazy-loaded module level for better security.

## Testing Patterns

### [Testing with Mock Services](./testing-mock-services.md)

Create mock services for isolated component testing.

### [TestBed Configuration](./testbed-configuration.md)

Configure TestBed efficiently for component and service tests.

### [Testing Observables](./testing-observables.md)

Test asynchronous observable streams using testing utilities.

### [Testing Forms](./testing-forms.md)

Test reactive forms, validators, and form interactions.

## State Management

### [Component State Patterns](./component-state-patterns.md)

Manage component-local state effectively using Signals or RxJS.

### [Service-Based State](./service-based-state.md)

Centralize application state in services for cross-component communication.

### [State Management Libraries](./state-management-libraries.md)

When and how to use NgRx, Akita, or other state management libraries.

## Directive Patterns

### [Attribute Directives](./attribute-directives.md)

Create reusable attribute directives for cross-cutting DOM manipulation.

### [Structural Directives](./structural-directives.md)

Build custom structural directives for template logic.

### [Directive Composition](./directive-composition-api.md)

Use the Directive Composition API (Angular 15+) to compose directive behaviors.

## Pipe Patterns

### [Pure vs Impure Pipes](./pure-impure-pipes.md)

Understand when to use pure versus impure pipes for optimal performance.

### [Custom Pipes](./custom-pipes.md)

Create reusable custom pipes for template transformations.

### [Async Pipe Best Practices](./async-pipe-best-practices.md)

Use the async pipe effectively with multiple subscriptions and error handling.

## Security Patterns

### [Sanitization](./sanitization.md)

Properly sanitize user input to prevent XSS attacks.

### [Content Security Policy](./content-security-policy.md)

Implement CSP headers and nonce-based script loading.

### [Route Guards for Authorization](./route-guards-authorization.md)

Implement role-based and permission-based route guards.

## Accessibility Patterns

### [ARIA Best Practices](./aria-best-practices.md)

Implement proper ARIA attributes for accessible components.

### [Keyboard Navigation](./keyboard-navigation.md)

Ensure all interactive elements are keyboard accessible.

### [Focus Management](./focus-management.md)

Manage focus appropriately for modals, route changes, and dynamic content.
