# Type-Safe Pipeline Builder (Data Transformation Pipelines)

> Chaining transformations with generic `Func<>` delegates loses type information and allows incompatible steps—use typed pipeline builders to make invalid pipelines uncompilable.

## Problem

Data processing pipelines built with collections of delegates or method references don't enforce that each step's output type matches the next step's input type. Invalid pipeline configurations compile successfully but fail at runtime.

## Example

### ❌ Before

```csharp
public class Pipeline
{
    private readonly List<Func<object, object>> _steps = new();
    
    public Pipeline AddStep(Func<object, object> transform)
    {
        _steps.Add(transform);
        return this;
    }
    
    public object Execute(object input)
    {
        object current = input;
        foreach (var step in _steps)
        {
            current = step(current);  // Runtime type checking
        }
        return current;
    }
}

// Usage: No compile-time safety
var pipeline = new Pipeline()
    .AddStep(input => ((string)input).ToUpper())  // Expects string
    .AddStep(input => ((int)input) * 2);  // Expects int—runtime error!

var result = pipeline.Execute("hello");  // Crashes: can't cast string to int
```

### ✅ After

```csharp
/// <summary>
/// Type-safe pipeline that ensures each step's output matches next step's input.
/// </summary>
public interface IPipeline<TInput, TOutput>
{
    TOutput Execute(TInput input);
}

public sealed class Pipeline<TInput, TOutput> : IPipeline<TInput, TOutput>
{
    private readonly Func<TInput, TOutput> _transform;
    
    private Pipeline(Func<TInput, TOutput> transform)
    {
        _transform = transform;
    }
    
    public static Pipeline<TInput, TOutput> Create(Func<TInput, TOutput> transform)
        => new(transform);
    
    public Pipeline<TInput, TNext> Then<TNext>(Func<TOutput, TNext> next)
    {
        return new Pipeline<TInput, TNext>(input =>
        {
            var intermediate = _transform(input);
            return next(intermediate);
        });
    }
    
    public Pipeline<TInput, TNext> Then<TNext>(IPipeline<TOutput, TNext> next)
    {
        return new Pipeline<TInput, TNext>(input =>
        {
            var intermediate = _transform(input);
            return next.Execute(intermediate);
        });
    }
    
    public TOutput Execute(TInput input)
    {
        return _transform(input);
    }
}

// Usage: Type-safe pipeline composition
var pipeline = Pipeline<string, string>
    .Create(s => s.ToUpper())
    .Then(s => s.Replace(" ", "-"))
    .Then(s => s.Length);  // Returns Pipeline<string, int>

var result = pipeline.Execute("hello world");  // Returns 11

// Won't compile:
// var bad = Pipeline<string, string>
//     .Create(s => s.ToUpper())
//     .Then((int x) => x * 2);  // Error: Can't convert string to int
```

## Async Pipeline

```csharp
public sealed class AsyncPipeline<TInput, TOutput>
{
    private readonly Func<TInput, Task<TOutput>> _transform;
    
    private AsyncPipeline(Func<TInput, Task<TOutput>> transform)
    {
        _transform = transform;
    }
    
    public static AsyncPipeline<TInput, TOutput> Create(
        Func<TInput, Task<TOutput>> transform)
        => new(transform);
    
    public static AsyncPipeline<TInput, TOutput> Create(
        Func<TInput, TOutput> transform)
        => new(input => Task.FromResult(transform(input)));
    
    public AsyncPipeline<TInput, TNext> Then<TNext>(
        Func<TOutput, Task<TNext>> next)
    {
        return new AsyncPipeline<TInput, TNext>(async input =>
        {
            var intermediate = await _transform(input);
            return await next(intermediate);
        });
    }
    
    public AsyncPipeline<TInput, TNext> Then<TNext>(
        Func<TOutput, TNext> next)
    {
        return new AsyncPipeline<TInput, TNext>(async input =>
        {
            var intermediate = await _transform(input);
            return next(intermediate);
        });
    }
    
    public Task<TOutput> ExecuteAsync(TInput input)
    {
        return _transform(input);
    }
}

// Usage: Async transformations
var pipeline = AsyncPipeline<string, string>
    .Create(async s => await FetchDataAsync(s))
    .Then(async data => await ProcessAsync(data))
    .Then(result => result.Length);

var count = await pipeline.ExecuteAsync("user123");
```

## Pipeline with Error Handling

```csharp
public sealed class ResultPipeline<TInput, TOutput, TError>
{
    private readonly Func<TInput, Result<TOutput, TError>> _transform;
    
    private ResultPipeline(Func<TInput, Result<TOutput, TError>> transform)
    {
        _transform = transform;
    }
    
    public static ResultPipeline<TInput, TOutput, TError> Create(
        Func<TInput, Result<TOutput, TError>> transform)
        => new(transform);
    
    public ResultPipeline<TInput, TNext, TError> Then<TNext>(
        Func<TOutput, Result<TNext, TError>> next)
    {
        return new ResultPipeline<TInput, TNext, TError>(input =>
        {
            var result = _transform(input);
            return result.Match(
                success => next(success),
                error => Result<TNext, TError>.Failure(error));
        });
    }
    
    public Result<TOutput, TError> Execute(TInput input)
    {
        return _transform(input);
    }
}

// Usage: Pipeline that short-circuits on error
var pipeline = ResultPipeline<string, UserId, ValidationError>
    .Create(s => ValidateEmail(s))
    .Then(email => LookupUser(email))
    .Then(user => GetUserId(user));

var result = pipeline.Execute("john@example.com");
// Stops at first error
```

## Branching Pipelines

```csharp
public sealed class ConditionalPipeline<TInput, TOutput>
{
    private readonly Func<TInput, TOutput> _transform;
    
    private ConditionalPipeline(Func<TInput, TOutput> transform)
    {
        _transform = transform;
    }
    
    public static ConditionalPipeline<TInput, TOutput> Create(
        Func<TInput, TOutput> transform)
        => new(transform);
    
    public ConditionalPipeline<TInput, TOutput> Branch(
        Func<TInput, bool> predicate,
        Func<TOutput, TOutput> whenTrue,
        Func<TOutput, TOutput> whenFalse)
    {
        return new ConditionalPipeline<TInput, TOutput>(input =>
        {
            var intermediate = _transform(input);
            return predicate(input)
                ? whenTrue(intermediate)
                : whenFalse(intermediate);
        });
    }
    
    public TOutput Execute(TInput input) => _transform(input);
}

// Usage: Conditional transformations
var pipeline = ConditionalPipeline<int, int>
    .Create(x => x * 2)
    .Branch(
        x => x > 10,
        whenTrue: x => x + 100,
        whenFalse: x => x - 100);

var result = pipeline.Execute(7);  // (7 * 2) - 100 = -86
```

## Parallel Pipeline

```csharp
public sealed class ParallelPipeline<TInput, TOutput>
{
    private readonly Func<TInput, Task<TOutput>>[] _transforms;
    
    private ParallelPipeline(params Func<TInput, Task<TOutput>>[] transforms)
    {
        _transforms = transforms;
    }
    
    public static ParallelPipeline<TInput, TOutput> Create(
        params Func<TInput, Task<TOutput>>[] transforms)
        => new(transforms);
    
    public async Task<TOutput[]> ExecuteAllAsync(TInput input)
    {
        var tasks = _transforms.Select(t => t(input));
        return await Task.WhenAll(tasks);
    }
    
    public async Task<TOutput> ExecuteFirstAsync(TInput input)
    {
        var tasks = _transforms.Select(t => t(input));
        return await Task.WhenAny(tasks).Unwrap();
    }
}

// Usage: Parallel execution
var pipeline = ParallelPipeline<string, UserData>.Create(
    async id => await FetchFromCache(id),
    async id => await FetchFromDatabase(id),
    async id => await FetchFromApi(id));

var results = await pipeline.ExecuteAllAsync("user123");
```

## Validation Pipeline

```csharp
public sealed class ValidationPipeline<T>
{
    private readonly List<Func<T, Option<string>>> _validators = new();
    
    public ValidationPipeline<T> AddRule(
        Func<T, bool> predicate,
        string errorMessage)
    {
        _validators.Add(value =>
            predicate(value)
                ? Option<string>.None()
                : Option<string>.Some(errorMessage));
        return this;
    }
    
    public Result<T, string[]> Validate(T value)
    {
        var errors = _validators
            .Select(v => v(value))
            .Where(o => o.IsSome)
            .Select(o => o.Value)
            .ToArray();
        
        return errors.Length == 0
            ? Result<T, string[]>.Success(value)
            : Result<T, string[]>.Failure(errors);
    }
}

// Usage: Declarative validation
var validator = new ValidationPipeline<User>()
    .AddRule(u => !string.IsNullOrEmpty(u.Email), "Email required")
    .AddRule(u => u.Email.Contains("@"), "Email must contain @")
    .AddRule(u => u.Age >= 18, "Must be 18 or older");

var result = validator.Validate(user);
```

## Why It's a Problem

1. **Type mismatches**: Pipeline steps with incompatible types fail at runtime
2. **No composition**: Can't combine or reuse pipeline segments safely
3. **Error handling**: Exceptions are the only failure mechanism
4. **No parallelism**: Sequential execution only

## Symptoms

- Runtime type cast exceptions in pipeline code
- Generic `object` types everywhere
- Manual type checking before each step
- Try-catch blocks around pipeline execution

## Benefits

- **Type safety**: Incompatible steps don't compile
- **Composition**: Pipelines compose type-safely
- **Reusability**: Pipeline segments are reusable
- **Error handling**: Result types for expected failures
- **Testability**: Each step testable independently

## See Also

- [Method Chaining](./method-chaining.md) — fluent interfaces
- [Result Monad](./result-monad.md) — error handling
- [Type-Safe Builder](./type-safe-builder.md) — step-wise construction
- [Async Patterns](./async-patterns.md) — async operations
