# Type-Safe Graph Traversal

> Graph traversals with generic node types and unsafe casting—use typed graph structures to make traversal paths compile-time safe.

## Problem

Graph data structures using generic `Node<T>` or `object`-based nodes require runtime type checking and casting. Graph traversal algorithms can't verify that visited nodes match expected types, leading to invalid cast exceptions at runtime.

## Example

### ❌ Before

```csharp
public class Graph
{
    public Dictionary<string, List<object>> Edges { get; } = new();
    
    public void AddEdge(string from, string to, object edge)
    {
        if (!Edges.ContainsKey(from))
            Edges[from] = new List<object>();
        
        Edges[from].Add(edge);
    }
    
    public List<object> Traverse(string start)
    {
        var visited = new List<object>();
        var queue = new Queue<string>();
        queue.Enqueue(start);
        
        while (queue.Count > 0)
        {
            var current = queue.Dequeue();
            if (Edges.ContainsKey(current))
            {
                foreach (var edge in Edges[current])
                {
                    visited.Add(edge);  // Loses type information
                }
            }
        }
        
        return visited;
    }
}
```

### ✅ After

```csharp
public interface IGraphNode<TNodeId>
{
    TNodeId Id { get; }
}

public sealed record User(UserId Id, string Name) : IGraphNode<UserId>;
public sealed record Post(PostId Id, string Content) : IGraphNode<PostId>;

public interface IEdge<TFrom, TTo>
    where TFrom : IGraphNode<object>
    where TTo : IGraphNode<object>
{
    TFrom Source { get; }
    TTo Target { get; }
}

public sealed record UserFollowsEdge(User Source, User Target) 
    : IEdge<User, User>;

public sealed record UserCreatedPostEdge(User Source, Post Target) 
    : IEdge<User, Post>;

public sealed class TypedGraph<TNode> where TNode : IGraphNode<object>
{
    private readonly Dictionary<object, List<TNode>> _adjacencyList = new();
    
    public void AddNode(TNode node)
    {
        if (!_adjacencyList.ContainsKey(node.Id))
            _adjacencyList[node.Id] = new List<TNode>();
    }
    
    public void AddEdge<TTarget>(
        TNode source,
        IEdge<TNode, TTarget> edge)
        where TTarget : IGraphNode<object>
    {
        if (!_adjacencyList.ContainsKey(source.Id))
            _adjacencyList[source.Id] = new List<TNode>();
        
        // Type-safe edge addition
    }
    
    public IEnumerable<TNode> BreadthFirstTraversal(TNode start)
    {
        var visited = new HashSet<object>();
        var queue = new Queue<TNode>();
        queue.Enqueue(start);
        
        while (queue.Count > 0)
        {
            var current = queue.Dequeue();
            
            if (visited.Contains(current.Id))
                continue;
            
            visited.Add(current.Id);
            yield return current;
            
            if (_adjacencyList.TryGetValue(current.Id, out var neighbors))
            {
                foreach (var neighbor in neighbors)
                {
                    if (!visited.Contains(neighbor.Id))
                        queue.Enqueue(neighbor);
                }
            }
        }
    }
}

// Usage: Type-safe graph operations
var userGraph = new TypedGraph<User>();
var user1 = new User(UserId.New(), "Alice");
var user2 = new User(UserId.New(), "Bob");

userGraph.AddNode(user1);
userGraph.AddNode(user2);
userGraph.AddEdge(user1, new UserFollowsEdge(user1, user2));

var reachableUsers = userGraph.BreadthFirstTraversal(user1);
// All returned nodes are guaranteed to be User type
```

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md)
- [Discriminated Unions](./discriminated-unions.md)
- [Phantom Types](./phantom-types.md)
