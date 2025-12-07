# Variance Patterns (Covariance and Contravariance)

> Ignoring variance in generic types leads to runtime casts and lost type safety—use `in` and `out` keywords to express variance constraints at compile time.

## Problem

Generic types in C# are invariant by default: `IEnumerable<Dog>` is not assignable to `IEnumerable<Animal>` even though `Dog` inherits from `Animal`. This forces runtime casting, breaks polymorphism, and loses type safety. Understanding variance enables type-safe generic APIs.

## Core Concepts

### Covariance (`out T`)

A type is **covariant** in `T` when it only *produces* values of type `T` (returns them), never consumes them (accepts as input).

**Rule:** If `Dog : Animal`, then `IProducer<Dog>` can be treated as `IProducer<Animal>`.

### Contravariance (`in T`)

A type is **contravariant** in `T` when it only *consumes* values of type `T` (accepts them), never produces them (returns them).

**Rule:** If `Dog : Animal`, then `IConsumer<Animal>` can be treated as `IConsumer<Dog>`.

### Invariance (no modifier)

A type is **invariant** in `T` when it both produces and consumes values of type `T`.

**Rule:** `IStorage<Dog>` is *only* compatible with `IStorage<Dog>`, never `IStorage<Animal>`.

## Example

### ❌ Before (Invariant, Requires Casts)

```csharp
public abstract class Animal
{
    public abstract string Name { get; }
}

public sealed class Dog : Animal
{
    public override string Name => "Dog";
    public void Bark() => Console.WriteLine("Woof!");
}

public sealed class Cat : Animal
{
    public override string Name => "Cat";
    public void Meow() => Console.WriteLine("Meow!");
}

// Invariant generic interface
public interface IAnimalRepository
{
    List<Animal> GetAll();
    void Add(Animal animal);
}

public class DogRepository : IAnimalRepository
{
    List<Dog> dogs = new();
    
    // Must return List<Animal>, even though we have List<Dog>
    public List<Animal> GetAll() => dogs.Cast<Animal>().ToList();
    
    public void Add(Animal animal)
    {
        // Runtime cast required—throws if not a Dog
        if (animal is Dog dog)
        {
            dogs.Add(dog);
        }
        else
        {
            throw new ArgumentException("Expected a Dog");
        }
    }
}

// Problem: Can't use Dog collection where Animal collection expected
void ProcessAnimals(List<Animal> animals)
{
    foreach (var animal in animals)
    {
        Console.WriteLine(animal.Name);
    }
}

List<Dog> dogs = new() { new Dog(), new Dog() };
// ProcessAnimals(dogs);  // ❌ Compile error: Cannot convert List<Dog> to List<Animal>

// Must create new list with cast
ProcessAnimals(dogs.Cast<Animal>().ToList());  // Runtime overhead
```

### ✅ After (With Variance)

```csharp
/// <summary>
/// Covariant interface: only produces T, never consumes it.
/// </summary>
public interface IReadOnlyRepository<out T>
{
    IEnumerable<T> GetAll();
    T GetById(int id);
    int Count { get; }
}

/// <summary>
/// Contravariant interface: only consumes T, never produces it.
/// </summary>
public interface IWriter<in T>
{
    void Add(T item);
    void Remove(T item);
    void Update(T item);
}

/// <summary>
/// Invariant interface: both produces AND consumes T.
/// </summary>
public interface IRepository<T> : IReadOnlyRepository<T>, IWriter<T>
{
    // Combines read (covariant) and write (contravariant) operations
    // Therefore must be invariant
}

public sealed class DogRepository : IRepository<Dog>
{
    List<Dog> dogs = new();
    
    public IEnumerable<Dog> GetAll() => dogs;
    public Dog GetById(int id) => dogs[id];
    public int Count => dogs.Count;
    
    public void Add(Dog dog) => dogs.Add(dog);
    public void Remove(Dog dog) => dogs.Remove(dog);
    public void Update(Dog dog) { /* ... */ }
}

// Covariance: Dog repository can be used as Animal repository (read-only)
IReadOnlyRepository<Dog> dogRepo = new DogRepository();
IReadOnlyRepository<Animal> animalRepo = dogRepo;  // ✅ Allowed: covariance

foreach (var animal in animalRepo.GetAll())
{
    Console.WriteLine(animal.Name);  // Works without casting
}

// Contravariance: Animal writer can write Dogs
IWriter<Animal> animalWriter = GetAnimalWriter();
IWriter<Dog> dogWriter = animalWriter;  // ✅ Allowed: contravariance

dogWriter.Add(new Dog());  // Safe: Dog is an Animal

// Type-safe processing with covariance
void ProcessAnimals(IReadOnlyRepository<Animal> animals)
{
    foreach (var animal in animals.GetAll())
    {
        Console.WriteLine(animal.Name);
    }
}

IReadOnlyRepository<Dog> dogs = new DogRepository();
ProcessAnimals(dogs);  // ✅ Works! No casting needed.
```

## Real-World Patterns

### Event Handlers (Contravariance)

```csharp
/// <summary>
/// Event handler is contravariant in event args.
/// A handler that accepts any EventArgs can handle specific ones.
/// </summary>
public interface IEventHandler<in TEventArgs> where TEventArgs : EventArgs
{
    void Handle(TEventArgs eventArgs);
}

public class LoggingEventHandler : IEventHandler<EventArgs>
{
    public void Handle(EventArgs eventArgs)
    {
        Console.WriteLine($"Event: {eventArgs}");
    }
}

public sealed class UserCreatedEventArgs : EventArgs
{
    public string UserId { get; init; }
}

// Contravariance in action
IEventHandler<EventArgs> generalHandler = new LoggingEventHandler();
IEventHandler<UserCreatedEventArgs> specificHandler = generalHandler;  // ✅ Valid

specificHandler.Handle(new UserCreatedEventArgs { UserId = "123" });

// Helper to demonstrate the pattern
IEventHandler<Animal> GetAnimalEventHandler() => new LoggingEventHandler();
```

### Factories (Covariance)

```csharp
/// <summary>
/// Factory is covariant: produces T.
/// </summary>
public interface IFactory<out T>
{
    T Create();
}

public sealed class DogFactory : IFactory<Dog>
{
    public Dog Create() => new Dog();
}

// Covariance allows substitution
void CreateAnimals(IFactory<Animal> factory, int count)
{
    for (int i = 0; i < count; i++)
    {
        var animal = factory.Create();
        Console.WriteLine(animal.Name);
    }
}

IFactory<Dog> dogFactory = new DogFactory();
CreateAnimals(dogFactory, 5);  // ✅ Works: IFactory<Dog> → IFactory<Animal>
```

### Comparers (Contravariance)

```csharp
/// <summary>
/// Comparer is contravariant: consumes T for comparison.
/// </summary>
public interface IComparer<in T>
{
    int Compare(T x, T y);
}

public sealed class AnimalNameComparer : IComparer<Animal>
{
    public int Compare(Animal x, Animal y)
        => string.Compare(x.Name, y.Name, StringComparison.Ordinal);
}

// Sort dogs using animal comparer
void SortAnimals<T>(List<T> animals, IComparer<T> comparer) where T : Animal
{
    animals.Sort((x, y) => comparer.Compare(x, y));
}

List<Dog> dogs = new() { /* ... */ };
IComparer<Animal> animalComparer = new AnimalNameComparer();
IComparer<Dog> dogComparer = animalComparer;  // ✅ Contravariance

SortAnimals(dogs, dogComparer);
```

### Validators (Contravariance)

```csharp
/// <summary>
/// Validator consumes T to validate it.
/// </summary>
public interface IValidator<in T>
{
    bool IsValid(T item);
}

public sealed class AnimalValidator : IValidator<Animal>
{
    public bool IsValid(Animal animal) => !string.IsNullOrEmpty(animal.Name);
}

// Use general validator for specific type
void ValidatePets<T>(IEnumerable<T> pets, IValidator<T> validator) 
    where T : Animal
{
    foreach (var pet in pets)
    {
        if (!validator.IsValid(pet))
        {
            Console.WriteLine($"Invalid: {pet.Name}");
        }
    }
}

IValidator<Animal> animalValidator = new AnimalValidator();
IValidator<Dog> dogValidator = animalValidator;  // ✅ Contravariance

List<Dog> dogs = new() { /* ... */ };
ValidatePets(dogs, dogValidator);
```

## Combining Variance in Complex Types

```csharp
/// <summary>
/// Function type: contravariant in input, covariant in output.
/// </summary>
public interface IFunction<in TInput, out TOutput>
{
    TOutput Apply(TInput input);
}

public sealed class ToStringFunction : IFunction<object, string>
{
    public string Apply(object input) => input?.ToString() ?? "";
}

// Both variance directions work together
IFunction<object, string> objToStr = new ToStringFunction();

// Covariance in output: string → object
IFunction<object, object> objToObj = objToStr;  // ✅

// Contravariance in input: object → Dog
IFunction<Dog, string> dogToStr = objToStr;  // ✅

// Both: Dog → object
IFunction<Dog, object> dogToObj = objToStr;  // ✅
```

## Collections and Variance

```csharp
// BCL examples of variance:

// IEnumerable<out T> - Covariant (only produces values)
IEnumerable<Dog> dogs = GetDogs();
IEnumerable<Animal> animals = dogs;  // ✅ Covariance

// IEnumerator<out T> - Covariant
IEnumerator<Dog> dogEnumerator = dogs.GetEnumerator();
IEnumerator<Animal> animalEnumerator = dogEnumerator;  // ✅

// IReadOnlyList<out T> - Covariant
IReadOnlyList<Dog> readOnlyDogs = dogs.ToList();
IReadOnlyList<Animal> readOnlyAnimals = readOnlyDogs;  // ✅

// IList<T> - Invariant (both reads and writes)
List<Dog> dogList = new();
// List<Animal> animalList = dogList;  // ❌ Compile error: invariant

// Action<in T> - Contravariant (consumes T)
Action<Animal> animalAction = a => Console.WriteLine(a.Name);
Action<Dog> dogAction = animalAction;  // ✅ Contravariance

// Func<out T> - Covariant (produces T)
Func<Dog> getDog = () => new Dog();
Func<Animal> getAnimal = getDog;  // ✅ Covariance

// Func<in T, out TResult> - Both!
Func<Animal, string> animalToString = a => a.Name;
Func<Dog, object> dogToObject = animalToString;  // ✅ Both directions
```

## Why Variance Matters

### Without Variance

```csharp
// Need separate method for each type
void ProcessDogs(IEnumerable<Dog> dogs) { /* ... */ }
void ProcessCats(IEnumerable<Cat> cats) { /* ... */ }
void ProcessAnimals(IEnumerable<Animal> animals) { /* ... */ }

// Or use runtime casts everywhere
void Process(IEnumerable<object> items)
{
    foreach (var item in items)
    {
        if (item is Animal animal)
        {
            // Process animal
        }
    }
}
```

### With Variance

```csharp
// Single generic method works for all
void ProcessAnimals<T>(IEnumerable<T> animals) where T : Animal
{
    foreach (var animal in animals)
    {
        // Type-safe: T is known to be Animal
        Console.WriteLine(animal.Name);
    }
}

// Works with any animal subtype due to covariance
IEnumerable<Dog> dogs = GetDogs();
IEnumerable<Cat> cats = GetCats();

ProcessAnimals(dogs);  // ✅
ProcessAnimals(cats);  // ✅
```

## Rules of Thumb

| Pattern | Variance | Use Case |
|---------|----------|----------|
| Only returns `T` | `out T` (covariant) | Producers, factories, read-only collections |
| Only accepts `T` | `in T` (contravariant) | Consumers, validators, comparers, event handlers |
| Both returns and accepts | Invariant (no modifier) | Mutable collections, repositories |

## Common Mistakes

```csharp
// ❌ Wrong: Can't have covariant parameter
public interface IProcessor<out T>
{
    void Process(T item);  // Compile error: covariant type used in input position
}

// ❌ Wrong: Can't have contravariant return type
public interface IFactory<in T>
{
    T Create();  // Compile error: contravariant type used in output position
}

// ✅ Correct: Covariant only in output
public interface IReadable<out T>
{
    T Read();
    IEnumerable<T> ReadMany();
}

// ✅ Correct: Contravariant only in input
public interface IWritable<in T>
{
    void Write(T item);
    void WriteMany(IEnumerable<T> items);
}
```

## Benefits

- **Type safety**: No runtime casts required
- **Flexibility**: More reusable generic code
- **Polymorphism**: Generic types participate in inheritance
- **Performance**: Eliminates unnecessary allocations and casts

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md) — type safety for identifiers
- [Value Semantics](./value-semantics.md) — immutability enables covariance
- [Honest Functions](./honest-functions.md) — function signatures and variance
