# Schema Evolution

> Handle schema changes gracefully with versioning and compatibility checks—add fields without breaking existing pipelines.

## Problem

Data schemas change over time as business requirements evolve. Pipelines that expect fixed schemas break when fields are added, removed, or have their types changed. Without careful schema management, you must stop all pipelines, update code, and redeploy simultaneously—causing downtime and deployment coordination nightmares.

## Example

### ❌ Before (Brittle Schema)

```python
# Producer writes data
def publish_order(order):
    message = {
        "order_id": order.id,
        "customer_id": order.customer_id,
        "total": order.total
    }
    kafka_producer.send("orders", json.dumps(message))

# Consumer expects exact schema
def process_order(message):
    data = json.loads(message)
    
    # ❌ Fails if new fields added or fields missing
    order_id = data["order_id"]
    customer_id = data["customer_id"] 
    total = data["total"]
    
    # Later, business adds "discount" field
    # Old consumers crash on KeyError
    discount = data["discount"]  # ❌ KeyError!
```

### ✅ After (Forward-Compatible Schema)

```python
# Producer: Add new fields with backward compatibility
def publish_order(order):
    message = {
        "order_id": order.id,
        "customer_id": order.customer_id,
        "total": order.total,
        "discount": order.discount if hasattr(order, "discount") else 0.0  # ✅ Default
    }
    kafka_producer.send("orders", json.dumps(message))

# Consumer: Handle missing fields gracefully
def process_order(message):
    data = json.loads(message)
    
    # ✅ Required fields
    order_id = data["order_id"]
    customer_id = data["customer_id"]
    total = data["total"]
    
    # ✅ Optional fields with defaults
    discount = data.get("discount", 0.0)
    shipping = data.get("shipping", 0.0)
    tax = data.get("tax", 0.0)
    
    # Works with old and new messages
```

## Schema Evolution with Schema Registry

### Using Avro with Schema Registry

```python
from confluent_kafka.avro import AvroProducer, AvroConsumer

# Version 1 schema
schema_v1 = {
    "type": "record",
    "name": "Order",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "string"},
        {"name": "total", "type": "double"}
    ]
}

# Version 2 schema (backward compatible)
schema_v2 = {
    "type": "record",
    "name": "Order",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "string"},
        {"name": "total", "type": "double"},
        # ✅ New field with default value
        {"name": "discount", "type": "double", "default": 0.0},
        {"name": "created_at", "type": "long", "default": 0}
    ]
}

# Producer (new version)
producer = AvroProducer({
    'bootstrap.servers': 'localhost:9092',
    'schema.registry.url': 'http://localhost:8081'
}, default_value_schema=schema_v2)

producer.produce(topic='orders', value={
    "order_id": "123",
    "customer_id": "456",
    "total": 99.99,
    "discount": 10.0,
    "created_at": 1234567890
})

# Consumer (old version) - still works!
consumer = AvroConsumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processor',
    'schema.registry.url': 'http://localhost:8081'
}, schema=schema_v1)

# Schema registry automatically handles conversion
# New fields are ignored by old consumer
```

## Compatibility Types

### Backward Compatibility

New schema can read data written by old schema:

```python
# Old schema
{
    "order_id": "string",
    "total": "double"
}

# New schema (backward compatible)
{
    "order_id": "string",
    "total": "double",
    "discount": "double" = 0.0  # ✅ Has default
}

# New consumer can read old data (uses default for missing fields)
```

### Forward Compatibility

Old schema can read data written by new schema:

```python
# Old consumer reading new data ignores unknown fields
# Works only if new fields are not required
```

### Full Compatibility

Both backward and forward compatible:

```python
# Can add optional fields (with defaults)
# Can remove optional fields
# Cannot change field types
# Cannot rename fields (without aliases)
```

## Breaking vs. Non-Breaking Changes

### ✅ Non-Breaking (Safe)

```python
# Add optional field with default
"new_field": {"type": "string", "default": ""}

# Add new variant to union type
"status": {"type": ["string", "null"], "default": null}

# Promote field from required to optional
# Before: "email": "string"
# After:  "email": {"type": ["string", "null"], "default": null}
```

### ❌ Breaking (Unsafe)

```python
# Remove field without default
# Before: {"name": "email", "type": "string"}
# After:  (field removed)

# Change field type
# Before: "total": "int"
# After:  "total": "double"

# Rename field
# Before: "user_id": "string"
# After:  "customer_id": "string"  # Different name

# Make optional field required
# Before: "email": {"type": ["string", "null"]}
# After:  "email": "string"
```

## Versioning Strategies

### Explicit Version Field

```python
# Include schema version in every message
def publish_order(order):
    message = {
        "__schema_version__": 2,  # ✅ Explicit version
        "order_id": order.id,
        "customer_id": order.customer_id,
        "total": order.total,
        "discount": order.discount
    }
    kafka_producer.send("orders", json.dumps(message))

# Consumer handles multiple versions
def process_order(message):
    data = json.loads(message)
    version = data.get("__schema_version__", 1)  # Default to v1
    
    if version == 1:
        return process_order_v1(data)
    elif version == 2:
        return process_order_v2(data)
    else:
        raise ValueError(f"Unknown schema version: {version}")

def process_order_v1(data):
    return {
        "order_id": data["order_id"],
        "customer_id": data["customer_id"],
        "total": data["total"],
        "discount": 0.0  # V1 had no discount
    }

def process_order_v2(data):
    return {
        "order_id": data["order_id"],
        "customer_id": data["customer_id"],
        "total": data["total"],
        "discount": data.get("discount", 0.0)
    }
```

### Separate Topics/Tables per Version

```python
# When breaking changes are unavoidable
kafka_producer.send("orders_v1", message_v1)
kafka_producer.send("orders_v2", message_v2)

# Run both consumers in parallel during migration
consumer_v1 = Consumer("orders_v1")
consumer_v2 = Consumer("orders_v2")

# Eventually deprecate v1
```

## Migration Strategies

### Dual Writing

```python
# During migration: write to both old and new schemas
def publish_order(order):
    # Old schema (for legacy consumers)
    message_v1 = {
        "order_id": order.id,
        "total": order.total
    }
    kafka_producer.send("orders", json.dumps(message_v1))
    
    # New schema (for new consumers)
    message_v2 = {
        "order_id": order.id,
        "total": order.total,
        "discount": order.discount,
        "tax": order.tax
    }
    kafka_producer.send("orders_v2", json.dumps(message_v2))

# Migration steps:
# 1. Start dual writing
# 2. Deploy new consumers reading from orders_v2
# 3. Verify new consumers work correctly
# 4. Stop old consumers
# 5. Stop writing to old topic
# 6. Deprecate old schema
```

### Backfill Historical Data

```python
# When changing storage schema
def backfill_new_fields():
    # Read old format
    for record in db.query("SELECT * FROM orders_old"):
        # Transform to new format
        new_record = {
            "order_id": record["order_id"],
            "total": record["total"],
            "discount": 0.0,  # Default for old records
            "created_at": record["timestamp"]
        }
        
        # Write to new table
        db.insert("orders_new", new_record)
    
    # Switch applications to new table
    # Eventually drop old table
```

## Why It's a Problem

1. **Breaking deployments**: Schema changes require coordinated updates across all services
2. **Downtime**: Can't safely deploy new schema versions without stopping pipelines
3. **Data loss**: Incompatible schema changes can cause data to be rejected or lost
4. **Testing difficulty**: Hard to test schema migrations in production-like conditions

## Symptoms

- Pipeline failures after schema changes
- Missing or corrupted data in destination
- Deployment rollback due to schema incompatibilities
- Coordination overhead for simultaneous deployments

## Benefits

- **Independent deployments**: Producers and consumers can update independently
- **Zero downtime**: Schema changes don't require stopping pipelines
- **Gradual migrations**: Roll out changes incrementally with parallel versions
- **Data integrity**: Schema validation prevents incompatible data

## Best Practices

1. **Always use defaults**: New fields should have sensible default values
2. **Schema registry**: Use a centralized registry to enforce compatibility
3. **Test compatibility**: Validate changes against compatibility rules before deployment
4. **Version explicitly**: Include schema version in messages or use schema registry
5. **Document changes**: Maintain changelog of schema evolution
6. **Monitor**: Alert on schema validation failures

## See Also

- [Data Validation at Boundaries](./data-validation-boundaries.md)
- [Type-Safe Transformations](./type-safe-transformations.md)
- [Dead Letter Queues](./dead-letter-queues.md)
