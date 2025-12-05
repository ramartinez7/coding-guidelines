# Specification Pattern (Logic as Types)

> Business rules buried in scattered LINQ lambdas—encapsulate rules as named, composable Specification types.

## Problem

Business rules often get buried inside anonymous LINQ lambdas scattered across Controllers, Services, and Repositories. This makes them hard to test in isolation, impossible to reuse, and invisible to the domain model.

## Example

### ❌ Before

```csharp
public class MovieController
{
    public IActionResult GetGreatMovies()
    {
        // What defines "great"? This logic has no name.
        var movies = _db.Movies
            .Where(m => m.Rating > 4.0 && m.ReviewCount > 100)
            .ToList();
        return Ok(movies);
    }
}

public class RecommendationService
{
    public List<Movie> GetRecommendations(User user)
    {
        // Same logic, duplicated—is it the same definition?
        var greatMovies = _db.Movies
            .Where(m => m.Rating > 4.0 && m.ReviewCount > 100)
            .Where(m => !user.WatchedMovieIds.Contains(m.Id))
            .ToList();
        return greatMovies;
    }
}

public class MovieRepository
{
    public List<Movie> GetFeaturedMovies()
    {
        // Slightly different—is this intentional or a bug?
        return _db.Movies
            .Where(m => m.Rating > 4.0)  // Missing ReviewCount check!
            .Where(m => m.ReleaseDate > DateTime.Now.AddYears(-2))
            .ToList();
    }
}
```

**Problems:**
- **Anonymous logic**: The rule `Rating > 4.0 && ReviewCount > 100` has no name
- **DRY violation**: If "great movie" changes to `Rating > 4.5`, you hunt through every lambda
- **Untestable**: Can't unit test the business rule without a database
- **Invisible domain**: The concept of "GreatMovie" exists only in developers' heads

### ✅ After

```csharp
// The Specification abstraction
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
    {
        return ToExpression().Compile()(entity);
    }

    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);

    public Specification<T> Not()
        => new NotSpecification<T>(this);
}

// Named business rules as types
public sealed class GreatMovieSpecification : Specification<Movie>
{
    public override Expression<Func<Movie, bool>> ToExpression()
        => movie => movie.Rating > 4.0m && movie.ReviewCount > 100;
}

public sealed class RecentMovieSpecification : Specification<Movie>
{
    private readonly TimeSpan _recency;

    public RecentMovieSpecification(TimeSpan recency) => _recency = recency;

    public override Expression<Func<Movie, bool>> ToExpression()
    {
        var cutoff = DateTime.UtcNow - _recency;
        return movie => movie.ReleaseDate > cutoff;
    }
}

public sealed class UnwatchedByUserSpecification : Specification<Movie>
{
    private readonly IReadOnlySet<MovieId> _watchedIds;

    public UnwatchedByUserSpecification(User user)
        => _watchedIds = user.WatchedMovieIds.ToHashSet();

    public override Expression<Func<Movie, bool>> ToExpression()
        => movie => !_watchedIds.Contains(movie.Id);
}
```

**Usage—composable and readable:**

```csharp
public class MovieController
{
    public IActionResult GetGreatMovies()
    {
        var spec = new GreatMovieSpecification();
        var movies = _db.Movies.Where(spec.ToExpression()).ToList();
        return Ok(movies);
    }
}

public class RecommendationService
{
    public List<Movie> GetRecommendations(User user)
    {
        var spec = new GreatMovieSpecification()
            .And(new UnwatchedByUserSpecification(user));

        return _db.Movies.Where(spec.ToExpression()).ToList();
    }
}

public class MovieRepository
{
    public List<Movie> GetFeaturedMovies()
    {
        var spec = new GreatMovieSpecification()
            .And(new RecentMovieSpecification(TimeSpan.FromDays(730)));

        return _db.Movies.Where(spec.ToExpression()).ToList();
    }
}
```

## Specification Combinators

```csharp
internal sealed class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();

        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));

        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

internal sealed class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public OrSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();

        var param = Expression.Parameter(typeof(T));
        var body = Expression.OrElse(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));

        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

internal sealed class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _spec;

    public NotSpecification(Specification<T> spec) => _spec = spec;

    public override Expression<Func<T, bool>> ToExpression()
    {
        var expr = _spec.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.Not(Expression.Invoke(expr, param));

        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}
```

## Why It's a Problem

1. **Anonymous logic**: Rules like `Rating > 4.0` have no identity—you can't discuss "the great movie rule" in code reviews.

2. **DRY violations**: The same rule duplicated across files diverges over time.

3. **Untestable in isolation**: Testing `m => m.Rating > 4.0` requires instantiating the entire query context.

4. **Hidden domain concepts**: Business stakeholders talk about "great movies" but the code only has anonymous predicates.

5. **Composition friction**: Combining rules requires copy-pasting lambda bodies.

## Symptoms

- LINQ `Where` clauses repeated across multiple files
- Business rule changes requiring "find in files" across the codebase
- Unit tests that need database setup just to test filtering logic
- Code reviews asking "is this the same rule as the one in X?"
- Comments explaining what a lambda "means"

## Benefits

- **Named domain concepts**: `GreatMovieSpecification` is discoverable and discussable
- **Single source of truth**: Change the rule in one place
- **Unit testable**: Test specifications with in-memory objects
- **Composable**: Build complex rules from simple ones with `And`, `Or`, `Not`
- **EF Core compatible**: `ToExpression()` translates to SQL

## Testing Specifications

```csharp
public class GreatMovieSpecificationTests
{
    [Fact]
    public void IsSatisfiedBy_HighRatingAndManyReviews_ReturnsTrue()
    {
        var spec = new GreatMovieSpecification();
        var movie = new Movie { Rating = 4.5m, ReviewCount = 200 };

        Assert.True(spec.IsSatisfiedBy(movie));
    }

    [Fact]
    public void IsSatisfiedBy_HighRatingButFewReviews_ReturnsFalse()
    {
        var spec = new GreatMovieSpecification();
        var movie = new Movie { Rating = 4.5m, ReviewCount = 50 };

        Assert.False(spec.IsSatisfiedBy(movie));
    }

    [Fact]
    public void And_CombinesTwoSpecifications()
    {
        var great = new GreatMovieSpecification();
        var recent = new RecentMovieSpecification(TimeSpan.FromDays(365));
        var combined = great.And(recent);

        var oldGreatMovie = new Movie 
        { 
            Rating = 4.5m, 
            ReviewCount = 200,
            ReleaseDate = DateTime.UtcNow.AddYears(-5)
        };

        Assert.False(combined.IsSatisfiedBy(oldGreatMovie));
    }
}
```

## Variation: Interface-Based Specification

For simpler scenarios without expression tree composition:

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
}

public sealed class GreatMovieSpecification : ISpecification<Movie>
{
    public bool IsSatisfiedBy(Movie movie)
        => movie.Rating > 4.0m && movie.ReviewCount > 100;
}

// Usage with in-memory filtering
var greatMovies = movies.Where(spec.IsSatisfiedBy).ToList();
```

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Boolean Blindness](./boolean-blindness.md)
- [Value Semantics](./value-semantics.md)
