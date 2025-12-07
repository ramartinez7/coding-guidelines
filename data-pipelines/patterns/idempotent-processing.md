# Idempotent Processing

> Design pipeline stages to be safely repeatable—same input produces same output, retries don't create duplicates.

## Problem

Networks fail, processes crash, and infrastructure experiences transient errors. Non-idempotent pipelines produce incorrect results when operations are retried—duplicating records, double-counting metrics, or corrupting aggregate state.

## Example

### ❌ Before (Non-Idempotent)

```python
# Non-idempotent: retry creates duplicate records
def process_order(order):
    # Transform
    transformed = transform_order(order)
    
    # Insert into destination
    db.execute("""
        INSERT INTO orders (order_id, customer_id, amount, processed_at)
        VALUES (?, ?, ?, ?)
    """, transformed.order_id, transformed.customer_id, 
         transformed.amount, datetime.now())
    
    # If process crashes after INSERT but before checkpoint,
    # retry will attempt to insert again -> duplicate or primary key error

# Non-idempotent: increments are not repeatable
def update_daily_metrics(date, amount):
    db.execute("""
        UPDATE daily_sales 
        SET total = total + ?
        WHERE date = ?
    """, amount, date)
    
    # Retry doubles the amount added
```

### ✅ After (Idempotent)

```python
# Idempotent: retry produces same result
def process_order(order):
    # Transform (deterministic)
    transformed = transform_order(order)
    
    # Upsert instead of insert
    db.execute("""
        INSERT INTO orders (order_id, customer_id, amount, processed_at)
        VALUES (?, ?, ?, ?)
        ON CONFLICT (order_id) DO UPDATE SET
            customer_id = EXCLUDED.customer_id,
            amount = EXCLUDED.amount,
            processed_at = EXCLUDED.processed_at
    """, transformed.order_id, transformed.customer_id,
         transformed.amount, transformed.processed_at)
    
    # Retry is safe: same order_id updates existing record

# Idempotent: set to absolute value
def update_daily_metrics(date, amount):
    # First, read current value
    current = db.query("""
        SELECT total FROM daily_sales WHERE date = ?
    """, date)
    
    # Calculate new value
    new_total = current.total + amount if current else amount
    
    # Set to absolute value with conditional update
    db.execute("""
        INSERT INTO daily_sales (date, total, version)
        VALUES (?, ?, 1)
        ON CONFLICT (date) DO UPDATE SET
            total = ?,
            version = daily_sales.version + 1
        WHERE daily_sales.version = ?
    """, date, new_total, new_total, current.version if current else 0)
```

### ✅ Alternative: Idempotency Keys

```python
# Use client-provided idempotency key
def process_order_with_key(order, idempotency_key):
    # Check if already processed
    existing = db.query("""
        SELECT * FROM orders WHERE idempotency_key = ?
    """, idempotency_key)
    
    if existing:
        # Already processed, return existing result
        return existing
    
    # Not processed yet
    transformed = transform_order(order)
    
    db.execute("""
        INSERT INTO orders (order_id, customer_id, amount, 
                          processed_at, idempotency_key)
        VALUES (?, ?, ?, ?, ?)
    """, transformed.order_id, transformed.customer_id,
         transformed.amount, transformed.processed_at, idempotency_key)
    
    return transformed

# Caller generates idempotency key once
import uuid
key = str(uuid.uuid4())

# First call processes order
result = process_order_with_key(order, key)

# Retry returns same result without duplicate processing
result = process_order_with_key(order, key)
```

## Streaming Example

### ❌ Non-Idempotent Stream Processing

```python
# Using Kafka consumer (pseudocode)
def consume_events():
    consumer = KafkaConsumer('orders')
    
    for message in consumer:
        order = parse_json(message.value)
        
        # Process order
        db.insert(order)  # ❌ Not idempotent
        
        # Commit offset
        consumer.commit()
        
        # If crash happens between insert and commit,
        # reprocessing creates duplicate
```

### ✅ Idempotent Stream Processing

```python
# Transactional processing with idempotency
def consume_events():
    consumer = KafkaConsumer('orders')
    
    for message in consumer:
        order = parse_json(message.value)
        
        # Use message offset + partition as idempotency key
        idempotency_key = f"{message.partition}:{message.offset}"
        
        with db.transaction():
            # Check if already processed
            if not db.exists("processed_offsets", idempotency_key):
                # Process order (idempotent upsert)
                db.upsert("orders", order)
                
                # Record that we processed this offset
                db.insert("processed_offsets", {
                    "key": idempotency_key,
                    "processed_at": datetime.now()
                })
            
            # Commit transaction (atomic)
        
        # Commit offset after successful transaction
        consumer.commit()
```

## Deterministic Transformations

Ensure transformations themselves are idempotent:

```python
# ❌ Non-deterministic transformation
def transform_order(order):
    return {
        "order_id": order["id"],
        "processed_id": str(uuid.uuid4()),  # ❌ Different every time!
        "processed_at": datetime.now(),     # ❌ Different every time!
        "amount": order["amount"]
    }

# ✅ Deterministic transformation
def transform_order(order):
    return {
        "order_id": order["id"],
        "processed_id": order["id"],  # ✅ Use business key
        "processed_at": order["created_at"],  # ✅ Use original timestamp
        "amount": order["amount"]
    }

# ✅ Alternative: Generate once at source
def ingest_order(raw_order):
    # Generate at ingestion time, pass through pipeline
    return {
        "order_id": raw_order["id"],
        "processed_id": str(uuid.uuid4()),  # Generated once
        "processed_at": datetime.now(),     # Generated once
        "amount": raw_order["amount"]
    }
```

## Aggregations

Make aggregations idempotent by storing individual contributions:

```python
# ❌ Non-idempotent aggregation
def aggregate_daily_sales(date):
    # Incremental update
    for order in get_orders_for_date(date):
        db.execute("""
            UPDATE daily_sales 
            SET total = total + ?
            WHERE date = ?
        """, order.amount, date)
    # ❌ Running twice doubles the totals

# ✅ Idempotent aggregation: full recomputation
def aggregate_daily_sales(date):
    # Query all orders for date
    orders = get_orders_for_date(date)
    total = sum(order.amount for order in orders)
    
    # Set to computed value (idempotent)
    db.execute("""
        INSERT INTO daily_sales (date, total)
        VALUES (?, ?)
        ON CONFLICT (date) DO UPDATE SET
            total = EXCLUDED.total
    """, date, total)

# ✅ Alternative: Store individual contributions
def aggregate_daily_sales(date):
    for order in get_orders_for_date(date):
        # Upsert individual contribution
        db.execute("""
            INSERT INTO order_contributions (date, order_id, amount)
            VALUES (?, ?, ?)
            ON CONFLICT (date, order_id) DO UPDATE SET
                amount = EXCLUDED.amount
        """, date, order.order_id, order.amount)
    
    # Recompute aggregate from contributions
    db.execute("""
        INSERT INTO daily_sales (date, total)
        SELECT date, SUM(amount)
        FROM order_contributions
        WHERE date = ?
        GROUP BY date
        ON CONFLICT (date) DO UPDATE SET
            total = EXCLUDED.total
    """, date)
```

## Why It's a Problem

1. **Duplicate data**: Retries create multiple copies of the same record
2. **Incorrect aggregates**: Counters and sums are incremented multiple times
3. **Cascading errors**: Non-idempotent stages amplify errors downstream
4. **Difficult debugging**: Issues only appear under retry conditions

## Symptoms

- Duplicate records with slightly different timestamps
- Inflated metrics and aggregates (2x, 3x actual values)
- Primary key violations during retries
- Inconsistent state after failures and recovery

## Benefits

- **Safe retries**: Operations can be repeated without side effects
- **Simpler recovery**: Just reprocess without worrying about duplicates
- **Distributed systems**: Essential for reliable event-driven architectures
- **Operational confidence**: Infrastructure failures don't corrupt data

## Implementation Strategies

1. **Natural keys**: Use business identifiers (order ID, user ID + timestamp)
2. **Idempotency keys**: Client-generated UUIDs stored alongside data
3. **Conditional updates**: Use version numbers or check-then-set patterns
4. **Transactional commits**: Atomically commit data + processing state
5. **Deterministic transformations**: Same input always produces same output

## See Also

- [Exactly-Once Semantics](./exactly-once-semantics.md)
- [Checkpointing and Recovery](./checkpointing-recovery.md)
- [State Management](./state-management.md)
