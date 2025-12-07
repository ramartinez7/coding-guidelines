# HATEOAS Links (Type-Safe Hypermedia)

> Building URLs with string concatenation and magic strings—use typed link relations to make APIs discoverable and self-documenting.

## Problem

REST APIs should be navigable through hyperlinks (HATEOAS), but implementations often use string-based URLs, hardcoded paths, and manual link construction. This makes APIs brittle, difficult to version, and prone to broken links.

## Example

### ❌ Before

```csharp
public class OrderDto
{
    public int Id { get; set; }
    public string Status { get; set; }
    
    // Manually constructed links as strings
    public Dictionary<string, string>? Links { get; set; }
}

[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetOrder(int id)
    {
        var order = _repository.GetById(id);
        
        // Manual string concatenation—error-prone
        var dto = new OrderDto
        {
            Id = order.Id,
            Status = order.Status.ToString(),
            Links = new Dictionary<string, string>
            {
                ["self"] = $"/api/orders/{id}",  // Hardcoded path
                ["cancel"] = $"/api/orders/{id}/cancel",  // Easy to get wrong
                ["items"] = $"/api/orders/{id}/items"
            }
        };
        
        // No type safety—typos only caught at runtime
        if (order.Status == OrderStatus.Pending)
        {
            dto.Links["pay"] = $"/api/orders/{id}/payment";  // What if route changes?
        }
        
        return Ok(dto);
    }
}
```

**Problems:**
- String-based link construction
- Hardcoded paths break when routes change
- No compile-time verification
- Links don't indicate which HTTP methods are available
- No way to know which links are valid for current state

### ✅ After

```csharp
/// <summary>
/// Strongly typed link relation.
/// </summary>
public abstract record LinkRelation
{
    public required string Rel { get; init; }
    public required string Title { get; init; }
    
    private LinkRelation() { }
    
    // Standard IANA link relations
    public sealed record Self : LinkRelation;
    public sealed record Next : LinkRelation;
    public sealed record Previous : LinkRelation;
    public sealed record First : LinkRelation;
    public sealed record Last : LinkRelation;
    
    // Custom domain-specific relations
    public sealed record Cancel : LinkRelation;
    public sealed record Pay : LinkRelation;
    public sealed record Ship : LinkRelation;
    public sealed record Items : LinkRelation;
    public sealed record Customer : LinkRelation;
}

/// <summary>
/// HTTP method for a link.
/// </summary>
public enum HttpMethod
{
    GET,
    POST,
    PUT,
    PATCH,
    DELETE
}

/// <summary>
/// Typed hyperlink with metadata.
/// </summary>
public sealed record Link
{
    public required LinkRelation Relation { get; init; }
    public required Uri Href { get; init; }
    public required HttpMethod Method { get; init; }
    public Option<string> Type { get; init; } = Option<string>.None;
    public bool Templated { get; init; } = false;
    
    /// <summary>
    /// Creates a link for a GET operation.
    /// </summary>
    public static Link Get(LinkRelation relation, Uri href, Option<string> type = default)
    {
        return new Link
        {
            Relation = relation,
            Href = href,
            Method = HttpMethod.GET,
            Type = type
        };
    }
    
    /// <summary>
    /// Creates a link for a POST operation.
    /// </summary>
    public static Link Post(LinkRelation relation, Uri href, string type)
    {
        return new Link
        {
            Relation = relation,
            Href = href,
            Method = HttpMethod.POST,
            Type = Option<string>.Some(type)
        };
    }
    
    /// <summary>
    /// Creates a link for a DELETE operation.
    /// </summary>
    public static Link Delete(LinkRelation relation, Uri href)
    {
        return new Link
        {
            Relation = relation,
            Href = href,
            Method = HttpMethod.DELETE
        };
    }
}

/// <summary>
/// Resource with hypermedia links.
/// </summary>
public interface IHypermediaResource
{
    IReadOnlyList<Link> Links { get; }
}

/// <summary>
/// Builder for creating links based on state.
/// </summary>
public interface ILinkBuilder
{
    Uri BuildUri(string routeName, object routeValues);
}

public class LinkBuilder : ILinkBuilder
{
    private readonly IUrlHelper _urlHelper;
    
    public LinkBuilder(IUrlHelper urlHelper)
    {
        _urlHelper = urlHelper;
    }
    
    public Uri BuildUri(string routeName, object routeValues)
    {
        var url = _urlHelper.Link(routeName, routeValues);
        
        if (url == null)
            throw new InvalidOperationException($"Could not generate URL for route '{routeName}'");
        
        return new Uri(url);
    }
}

/// <summary>
/// Order resource with state-dependent links.
/// </summary>
public sealed record OrderResource : IHypermediaResource
{
    public required int Id { get; init; }
    public required string Status { get; init; }
    public required decimal Total { get; init; }
    public required IReadOnlyList<Link> Links { get; init; }
    
    /// <summary>
    /// Creates order resource with appropriate links based on state.
    /// </summary>
    public static OrderResource From(Order order, ILinkBuilder linkBuilder)
    {
        var links = new List<Link>();
        
        // Self link always present
        links.Add(Link.Get(
            new LinkRelation.Self { Rel = "self", Title = "Order Details" },
            linkBuilder.BuildUri("GetOrder", new { id = order.Id })));
        
        // Items link always present
        links.Add(Link.Get(
            new LinkRelation.Items { Rel = "items", Title = "Order Items" },
            linkBuilder.BuildUri("GetOrderItems", new { id = order.Id })));
        
        // Customer link always present
        links.Add(Link.Get(
            new LinkRelation.Customer { Rel = "customer", Title = "Customer Details" },
            linkBuilder.BuildUri("GetCustomer", new { id = order.CustomerId })));
        
        // State-dependent links
        switch (order.Status)
        {
            case OrderStatus.Pending:
                // Can pay or cancel
                links.Add(Link.Post(
                    new LinkRelation.Pay { Rel = "pay", Title = "Pay for Order" },
                    linkBuilder.BuildUri("PayOrder", new { id = order.Id }),
                    "application/json"));
                
                links.Add(Link.Delete(
                    new LinkRelation.Cancel { Rel = "cancel", Title = "Cancel Order" },
                    linkBuilder.BuildUri("CancelOrder", new { id = order.Id })));
                break;
            
            case OrderStatus.Paid:
                // Can ship
                links.Add(Link.Post(
                    new LinkRelation.Ship { Rel = "ship", Title = "Ship Order" },
                    linkBuilder.BuildUri("ShipOrder", new { id = order.Id }),
                    "application/json"));
                break;
            
            case OrderStatus.Shipped:
                // No actions available
                break;
            
            case OrderStatus.Cancelled:
                // No actions available
                break;
        }
        
        return new OrderResource
        {
            Id = order.Id,
            Status = order.Status.ToString(),
            Total = order.Total,
            Links = links
        };
    }
}

[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _repository;
    private readonly ILinkBuilder _linkBuilder;
    
    public OrdersController(IOrderRepository repository, ILinkBuilder linkBuilder)
    {
        _repository = repository;
        _linkBuilder = linkBuilder;
    }
    
    [HttpGet("{id}", Name = "GetOrder")]
    public IActionResult GetOrder(int id)
    {
        var order = _repository.GetById(OrderId.From(id));
        
        // Links are automatically generated based on state
        var resource = OrderResource.From(order, _linkBuilder);
        
        return Ok(resource);
    }
    
    [HttpGet("{id}/items", Name = "GetOrderItems")]
    public IActionResult GetOrderItems(int id)
    {
        var items = _repository.GetItems(OrderId.From(id));
        return Ok(items);
    }
    
    [HttpPost("{id}/payment", Name = "PayOrder")]
    public IActionResult PayOrder(int id, [FromBody] PaymentDto payment)
    {
        var order = _repository.GetById(OrderId.From(id));
        
        // Only valid if order is pending
        if (order.Status != OrderStatus.Pending)
        {
            return BadRequest(new
            {
                error = "invalid_state",
                message = $"Cannot pay for order in {order.Status} status"
            });
        }
        
        order.Pay(payment);
        _repository.Save(order);
        
        // Return updated resource with new links
        var resource = OrderResource.From(order, _linkBuilder);
        return Ok(resource);
    }
    
    [HttpPost("{id}/ship", Name = "ShipOrder")]
    public IActionResult ShipOrder(int id, [FromBody] ShipmentDto shipment)
    {
        var order = _repository.GetById(OrderId.From(id));
        
        if (order.Status != OrderStatus.Paid)
        {
            return BadRequest(new
            {
                error = "invalid_state",
                message = $"Cannot ship order in {order.Status} status"
            });
        }
        
        order.Ship(shipment);
        _repository.Save(order);
        
        var resource = OrderResource.From(order, _linkBuilder);
        return Ok(resource);
    }
    
    [HttpDelete("{id}", Name = "CancelOrder")]
    public IActionResult CancelOrder(int id)
    {
        var order = _repository.GetById(OrderId.From(id));
        
        if (order.Status != OrderStatus.Pending)
        {
            return BadRequest(new
            {
                error = "invalid_state",
                message = $"Cannot cancel order in {order.Status} status"
            });
        }
        
        order.Cancel();
        _repository.Save(order);
        
        return NoContent();
    }
}
```

## Paginated Collections with Links

```csharp
/// <summary>
/// Paginated collection resource with navigation links.
/// </summary>
public sealed record PagedResource<T> : IHypermediaResource where T : IHypermediaResource
{
    public required IReadOnlyList<T> Items { get; init; }
    public required int TotalCount { get; init; }
    public required int PageSize { get; init; }
    public required int PageNumber { get; init; }
    public required IReadOnlyList<Link> Links { get; init; }
    
    public static PagedResource<T> Create(
        IReadOnlyList<T> items,
        int totalCount,
        int pageSize,
        int pageNumber,
        ILinkBuilder linkBuilder,
        string routeName)
    {
        var links = new List<Link>();
        
        // Self link
        links.Add(Link.Get(
            new LinkRelation.Self { Rel = "self", Title = "Current Page" },
            linkBuilder.BuildUri(routeName, new { page = pageNumber, pageSize })));
        
        // First page
        links.Add(Link.Get(
            new LinkRelation.First { Rel = "first", Title = "First Page" },
            linkBuilder.BuildUri(routeName, new { page = 1, pageSize })));
        
        // Previous page (if not on first page)
        if (pageNumber > 1)
        {
            links.Add(Link.Get(
                new LinkRelation.Previous { Rel = "prev", Title = "Previous Page" },
                linkBuilder.BuildUri(routeName, new { page = pageNumber - 1, pageSize })));
        }
        
        // Next page (if not on last page)
        var totalPages = (int)Math.Ceiling((double)totalCount / pageSize);
        if (pageNumber < totalPages)
        {
            links.Add(Link.Get(
                new LinkRelation.Next { Rel = "next", Title = "Next Page" },
                linkBuilder.BuildUri(routeName, new { page = pageNumber + 1, pageSize })));
        }
        
        // Last page
        links.Add(Link.Get(
            new LinkRelation.Last { Rel = "last", Title = "Last Page" },
            linkBuilder.BuildUri(routeName, new { page = totalPages, pageSize })));
        
        return new PagedResource<T>
        {
            Items = items,
            TotalCount = totalCount,
            PageSize = pageSize,
            PageNumber = pageNumber,
            Links = links
        };
    }
}

[HttpGet(Name = "GetOrders")]
public IActionResult GetOrders(int page = 1, int pageSize = 20)
{
    var orders = _repository.GetPage(page, pageSize);
    var totalCount = _repository.GetTotalCount();
    
    var resources = orders.Select(o => OrderResource.From(o, _linkBuilder)).ToList();
    
    var pagedResource = PagedResource<OrderResource>.Create(
        resources,
        totalCount,
        pageSize,
        page,
        _linkBuilder,
        "GetOrders");
    
    return Ok(pagedResource);
}
```

## Link Templates

```csharp
/// <summary>
/// URI template for parameterized links.
/// </summary>
public sealed record LinkTemplate
{
    public required string Template { get; init; }
    public required IReadOnlyList<TemplateParameter> Parameters { get; init; }
    
    /// <summary>
    /// Expands template with provided values.
    /// </summary>
    public Result<Uri, string> Expand(IDictionary<string, object> values)
    {
        var expanded = Template;
        
        foreach (var param in Parameters)
        {
            if (!values.TryGetValue(param.Name, out var value))
            {
                if (param.Required)
                    return Result<Uri, string>.Failure($"Required parameter '{param.Name}' not provided");
                
                continue;
            }
            
            expanded = expanded.Replace($"{{{param.Name}}}", value.ToString());
        }
        
        return Result<Uri, string>.Success(new Uri(expanded, UriKind.Relative));
    }
}

public sealed record TemplateParameter
{
    public required string Name { get; init; }
    public required bool Required { get; init; }
    public Option<string> Description { get; init; } = Option<string>.None;
}

// Usage: search link with query parameters
public sealed record SearchLink
{
    public static Link CreateSearchLink(ILinkBuilder linkBuilder)
    {
        var template = new LinkTemplate
        {
            Template = "/api/orders/search{?status,customerId,minTotal}",
            Parameters = new[]
            {
                new TemplateParameter { Name = "status", Required = false },
                new TemplateParameter { Name = "customerId", Required = false },
                new TemplateParameter { Name = "minTotal", Required = false }
            }
        };
        
        return new Link
        {
            Relation = new LinkRelation.Self { Rel = "search", Title = "Search Orders" },
            Href = new Uri("/api/orders/search", UriKind.Relative),
            Method = HttpMethod.GET,
            Templated = true
        };
    }
}
```

## Testing

```csharp
public class HypermediaTests
{
    [Fact]
    public void OrderResource_PendingStatus_IncludesPayAndCancelLinks()
    {
        var order = new Order { Id = OrderId.From(1), Status = OrderStatus.Pending };
        var resource = OrderResource.From(order, _linkBuilder);
        
        Assert.Contains(resource.Links, l => l.Relation is LinkRelation.Pay);
        Assert.Contains(resource.Links, l => l.Relation is LinkRelation.Cancel);
        Assert.DoesNotContain(resource.Links, l => l.Relation is LinkRelation.Ship);
    }
    
    [Fact]
    public void OrderResource_PaidStatus_IncludesShipLink()
    {
        var order = new Order { Id = OrderId.From(1), Status = OrderStatus.Paid };
        var resource = OrderResource.From(order, _linkBuilder);
        
        Assert.Contains(resource.Links, l => l.Relation is LinkRelation.Ship);
        Assert.DoesNotContain(resource.Links, l => l.Relation is LinkRelation.Pay);
        Assert.DoesNotContain(resource.Links, l => l.Relation is LinkRelation.Cancel);
    }
    
    [Fact]
    public void PagedResource_NotOnFirstPage_IncludesPreviousLink()
    {
        var pagedResource = PagedResource<OrderResource>.Create(
            items: new List<OrderResource>(),
            totalCount: 100,
            pageSize: 10,
            pageNumber: 2,
            _linkBuilder,
            "GetOrders");
        
        Assert.Contains(pagedResource.Links, l => l.Relation is LinkRelation.Previous);
    }
}
```

## Why It's a Problem

1. **Hardcoded URLs**: Routes in strings break when endpoints change
2. **No discoverability**: Clients must know all URLs upfront
3. **Brittle**: Changes to URL structure break clients
4. **State coupling**: Client must know which actions are valid
5. **Manual construction**: Error-prone link building

## Symptoms

- String concatenation for URLs
- Hardcoded paths throughout codebase
- Broken links when routes change
- Clients must maintain URL templates
- No indication of available actions

## Benefits

- **Type safety**: Links built from named routes
- **Discoverability**: Clients navigate through links
- **State-driven**: Links reflect current state
- **Resilient**: Route changes don't break clients
- **Self-documenting**: Links describe available operations

## See Also

- [Enum State Machine](./enum-state-machine.md) — state-dependent behavior
- [Versioned Endpoints](./versioned-endpoints.md) — API versioning
- [Pagination Cursors](./pagination-cursors.md) — pagination links
- [DTO vs. Domain Boundary](./dto-domain-boundary.md) — resource representation
