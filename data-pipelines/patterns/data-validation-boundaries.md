# Data Validation at Boundaries

> Validate and parse data at ingestion points—transform raw input into trusted types before downstream processing.

## Problem

Validation scattered throughout the pipeline leads to repeated checks, inconsistent error handling, and wasted processing on invalid data. When validation happens deep in the pipeline, you've already paid the cost of ingestion, transformation, and storage before discovering the data is unusable.

## Philosophy: Parse, Don't Validate

Once data is parsed into a trusted type at the boundary, downstream components can trust it's valid—no repeated validation needed.

## Example

### ❌ Before (Validation Everywhere)

```python
# Stage 1: Ingestion
def ingest_order(raw_data):
    # No validation, pass through
    kafka_producer.send("orders", raw_data)

# Stage 2: Enrichment
def enrich_order(message):
    data = json.loads(message)
    
    # ❌ Validation scattered
    if not data.get("order_id"):
        logger.error("Missing order_id")
        return
    
    if data.get("total", 0) < 0:
        logger.error("Negative total")
        return
    
    # Process...
    enriched = add_customer_info(data)
    kafka_producer.send("enriched_orders", enriched)

# Stage 3: Storage
def store_order(message):
    data = json.loads(message)
    
    # ❌ Validating again!
    if not data.get("order_id"):
        raise ValueError("Missing order_id")
    
    if not data.get("customer_id"):
        raise ValueError("Missing customer_id")
    
    db.insert(data)

# Invalid data wastes resources through multiple stages
```

### ✅ After (Validate at Boundary)

```python
from dataclasses import dataclass
from typing import Optional
from decimal import Decimal
from datetime import datetime

# Trusted domain type
@dataclass(frozen=True)
class Order:
    order_id: str
    customer_id: str
    total: Decimal
    items: list
    created_at: datetime
    
    @staticmethod
    def parse(raw_data: dict) -> "Order":
        """Parse and validate at boundary. Raises if invalid."""
        # Required fields
        if not raw_data.get("order_id"):
            raise ValueError("order_id is required")
        
        if not raw_data.get("customer_id"):
            raise ValueError("customer_id is required")
        
        # Type validation
        try:
            total = Decimal(str(raw_data["total"]))
        except (KeyError, ValueError):
            raise ValueError("total must be a valid number")
        
        # Business rules
        if total < 0:
            raise ValueError("total cannot be negative")
        
        if not raw_data.get("items"):
            raise ValueError("items cannot be empty")
        
        # Parse timestamp
        try:
            created_at = datetime.fromisoformat(raw_data["created_at"])
        except (KeyError, ValueError):
            raise ValueError("created_at must be valid ISO timestamp")
        
        return Order(
            order_id=raw_data["order_id"],
            customer_id=raw_data["customer_id"],
            total=total,
            items=raw_data["items"],
            created_at=created_at
        )

# Stage 1: Validate at boundary
def ingest_order(raw_data: dict):
    try:
        # ✅ Parse once at ingestion
        order = Order.parse(raw_data)
        
        # Serialize trusted type
        kafka_producer.send("orders", order.to_json())
        
    except ValueError as e:
        # Invalid data goes to dead letter queue
        dead_letter_queue.send({
            "data": raw_data,
            "error": str(e),
            "timestamp": datetime.now()
        })

# Stage 2: Enrichment trusts the type
def enrich_order(message: str):
    # ✅ No validation needed - data already trusted
    order = Order.from_json(message)
    
    enriched = add_customer_info(order)
    kafka_producer.send("enriched_orders", enriched.to_json())

# Stage 3: Storage trusts the type
def store_order(message: str):
    # ✅ No validation needed - data already trusted
    order = Order.from_json(message)
    db.insert(order)
```

## Multi-Layer Validation

Organize validation into layers:

```python
from typing import List, NewType
from pydantic import BaseModel, validator, Field

# Layer 1: Syntactic validation (schema, types)
class OrderDTO(BaseModel):
    order_id: str = Field(..., min_length=1)
    customer_id: str = Field(..., min_length=1) 
    total: Decimal = Field(..., ge=0)  # >= 0
    items: List[dict] = Field(..., min_items=1)
    created_at: datetime
    
    @validator('order_id')
    def validate_order_id_format(cls, v):
        if not v.startswith('ORD-'):
            raise ValueError('order_id must start with ORD-')
        return v

# Layer 2: Semantic validation (business rules)
class ValidatedOrder:
    def __init__(self, dto: OrderDTO):
        self.order_id = dto.order_id
        self.customer_id = dto.customer_id
        self.total = dto.total
        self.items = dto.items
        self.created_at = dto.created_at
        
        # Business rule: total must match sum of items
        items_total = sum(Decimal(str(item['price'])) * item['quantity'] 
                         for item in self.items)
        
        if abs(self.total - items_total) > Decimal('0.01'):
            raise ValueError(
                f"Order total {self.total} doesn't match items {items_total}"
            )
        
        # Business rule: order date cannot be in future
        if self.created_at > datetime.now():
            raise ValueError("Order date cannot be in future")
    
    @staticmethod
    def parse(raw_data: dict) -> "ValidatedOrder":
        # Layer 1: Schema validation
        dto = OrderDTO(**raw_data)
        
        # Layer 2: Business validation
        return ValidatedOrder(dto)

# Usage
def ingest_order(raw_data: dict):
    try:
        order = ValidatedOrder.parse(raw_data)
        # Downstream trusts both schema and business rules
        
    except ValueError as e:
        dead_letter_queue.send(raw_data, str(e))
```

## Batch Validation

Validate entire batches efficiently:

```python
def validate_batch(records: List[dict]) -> tuple[List[Order], List[dict]]:
    """
    Validate batch of records.
    Returns (valid_orders, invalid_records_with_errors)
    """
    valid = []
    invalid = []
    
    for record in records:
        try:
            order = Order.parse(record)
            valid.append(order)
            
        except ValueError as e:
            invalid.append({
                "record": record,
                "error": str(e),
                "error_timestamp": datetime.now()
            })
    
    return valid, invalid

# Process batch
def process_batch(raw_records: List[dict]):
    valid_orders, invalid_records = validate_batch(raw_records)
    
    # Process valid orders
    if valid_orders:
        kafka_producer.send_batch("orders", 
                                 [o.to_json() for o in valid_orders])
    
    # Send invalid to DLQ
    if invalid_records:
        dead_letter_queue.send_batch(invalid_records)
    
    # Metrics
    logger.info(f"Validated: {len(valid_orders)} valid, "
               f"{len(invalid_records)} invalid")
```

## Validation with Context

Sometimes validation requires external context:

```python
class OrderValidator:
    def __init__(self, customer_service, inventory_service):
        self.customer_service = customer_service
        self.inventory_service = inventory_service
    
    def validate(self, raw_data: dict) -> Order:
        # Basic validation
        order = Order.parse(raw_data)
        
        # Contextual validation: customer exists?
        if not self.customer_service.exists(order.customer_id):
            raise ValueError(f"Customer {order.customer_id} not found")
        
        # Contextual validation: items in stock?
        for item in order.items:
            available = self.inventory_service.get_stock(item['sku'])
            if available < item['quantity']:
                raise ValueError(
                    f"Insufficient stock for {item['sku']}: "
                    f"need {item['quantity']}, have {available}"
                )
        
        return order

# Use in pipeline
validator = OrderValidator(customer_service, inventory_service)

def ingest_order(raw_data: dict):
    try:
        order = validator.validate(raw_data)
        kafka_producer.send("orders", order.to_json())
        
    except ValueError as e:
        dead_letter_queue.send(raw_data, str(e))
```

## Monitoring Validation Failures

Track validation metrics to detect data quality issues:

```python
from prometheus_client import Counter, Histogram

validation_failures = Counter(
    'validation_failures_total',
    'Total validation failures',
    ['error_type']
)

validation_duration = Histogram(
    'validation_duration_seconds',
    'Time spent validating records'
)

def ingest_order(raw_data: dict):
    with validation_duration.time():
        try:
            order = Order.parse(raw_data)
            kafka_producer.send("orders", order.to_json())
            
        except ValueError as e:
            # Track failure by error type
            error_type = type(e).__name__
            validation_failures.labels(error_type=error_type).inc()
            
            dead_letter_queue.send(raw_data, str(e))
            
            # Alert if failure rate exceeds threshold
            if validation_failures._metrics:
                total = sum(m._value.get() 
                           for m in validation_failures._metrics.values())
                if total > 1000:  # Alert threshold
                    alert_service.send(
                        f"High validation failure rate: {total}"
                    )
```

## Why It's a Problem

1. **Wasted resources**: Invalid data consumes compute, storage, and network
2. **Repeated validation**: Every stage validates the same data
3. **Inconsistent rules**: Validation logic diverges across stages
4. **Late failures**: Errors discovered hours after ingestion

## Symptoms

- High CPU usage processing invalid data
- Same validation errors logged at multiple stages
- Different stages rejecting data for different reasons
- Large volumes of data in storage that can't be used

## Benefits

- **Early failure**: Reject bad data before wasting resources
- **Single source of truth**: One place to update validation rules
- **Type safety**: Downstream code works with trusted types
- **Better metrics**: Know exactly where data fails validation

## Best Practices

1. **Validate at ingestion**: Parse raw input into domain types at boundaries
2. **Use typed languages**: Leverage type systems to prevent invalid states
3. **Layer validation**: Separate syntactic (schema) from semantic (business) rules
4. **Dead letter queues**: Preserve invalid data for inspection
5. **Monitor failures**: Track validation error rates and types

## See Also

- [Dead Letter Queues](./dead-letter-queues.md)
- [Type-Safe Transformations](./type-safe-transformations.md)
- [Schema Evolution](./schema-evolution.md)
- [Data Quality Monitoring](./data-quality-monitoring.md)
