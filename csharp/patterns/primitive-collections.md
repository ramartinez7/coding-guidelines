# Primitive Collections

> Using raw collections (`List<T>`, `Dictionary<K,V>`) instead of domain-specific collection types.

## Problem

Raw collections expose implementation details and scatter business logic across consuming code. When multiple collections must stay synchronized, or when collection operations have domain meaning, using primitives leads to bugs and duplication.

## Example

### ❌ Before

```csharp
public class CardGame
{
    Queue<Card> availableCards = GenerateCards();
    HashSet<Card> dealtCards = new();

    static Queue<Card> GenerateCards() { /* ... */ }

    public Card DealCard()
    {
        var card = availableCards.Dequeue();
        dealtCards.Add(card);
        return card;
    }

    public void ReturnCard(Card card)
    {
        if (!dealtCards.Contains(card)) throw new CheaterException();

        dealtCards.Remove(card);
        availableCards.Enqueue(card);
    }
}
```

### ✅ After

```csharp
public sealed class Deck
{
    private readonly Queue<Card> _available;
    private readonly HashSet<Card> _dealt = new();

    private Deck(Queue<Card> cards) => _available = cards;

    public static Deck CreateStandard52() => new(GenerateCards());

    public Card Deal()
    {
        var card = _available.Dequeue();
        _dealt.Add(card);
        return card;
    }

    public void Return(Card card)
    {
        if (!_dealt.Contains(card)) throw new CheaterException();

        _dealt.Remove(card);
        _available.Enqueue(card);
    }

    public bool IsDealt(Card card) => _dealt.Contains(card);
    public int Remaining => _available.Count;

    private static Queue<Card> GenerateCards() { /* ... */ }
}

public class CardGame
{
    private readonly Deck _deck = Deck.CreateStandard52();

    public Card DealCard() => _deck.Deal();
    public void ReturnCard(Card card) => _deck.Return(card);
}
```

## Symptoms

- Business logic mixed with collection manipulation code
- Mismatched abstraction levels: `list.Insert(0, card)` vs `deck.Return(card)`
- Multiple collections that must be updated together
- Values that must change when a collection changes
- LINQ chains repeated in multiple places with the same filtering/transformation

## Benefits

- **Encapsulated invariants**: Synchronized collections stay consistent
- **Domain vocabulary**: `deck.Deal()` is clearer than `queue.Dequeue()`
- **Reusability**: The `Deck` type can be used in other card games
- **Testability**: Collection logic can be unit tested in isolation

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Data Clumps](./data-clump.md)
