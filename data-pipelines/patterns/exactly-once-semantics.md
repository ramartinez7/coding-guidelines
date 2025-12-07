# Exactly-Once Semantics

> Guarantee each record is processed exactly once—neither duplicated nor lost—using transactional commits and deduplication.

## Problem

Distributed systems experience failures, retries, and network partitions. Without careful design, records are either lost (at-most-once) or duplicated (at-least-once). Business logic that requires exact counts, balances, or state must guarantee exactly-once processing.

## Delivery Guarantees

### At-Most-Once

Record may be lost but never duplicated:

```python
def process_at_most_once(message):
    # Commit offset BEFORE processing
    consumer.commit()
    
    try:
        process_message(message)
    except Exception:
        # ❌ Message lost if processing fails
        pass

# Use when: Data loss acceptable, duplicates unacceptable
# Example: Logging, metrics (approximations OK)
```

### At-Least-Once

Record may be duplicated but never lost:

```python
def process_at_least_once(message):
    try:
        process_message(message)
    except Exception:
        # ❌ Retry causes duplicate
        raise
    
    # Commit offset AFTER processing
    consumer.commit()

# Use when: Duplicates acceptable (with idempotent processing)
# Example: Most streaming systems default to this
```

### Exactly-Once

Record is neither lost nor duplicated:

```python
def process_exactly_once(message):
    # ✅ Transactional processing + deduplication
    with transaction():
        # Check if already processed
        if not is_processed(message.id):
            process_message(message)
            mark_processed(message.id)
        
        # Commit offset atomically with processing
        commit_offset(message.offset)

# Use when: Correctness critical (financial, inventory)
# Example: Payment processing, account balances
```

## Implementation Strategies

### Transactional Processing

```python
def process_with_transaction(message):
    """
    Atomic commit of:
    1. Processing result
    2. Deduplication record
    3. Offset checkpoint
    """
    
    with db.transaction():
        # Check deduplication
        message_id = message.key.decode()
        
        existing = db.query_one("""
            SELECT id FROM processed_messages
            WHERE message_id = ?
        """, message_id)
        
        if existing:
            # Already processed, skip
            logger.info(f"Skipping duplicate: {message_id}")
            return
        
        # Process message
        order = json.loads(message.value)
        db.execute("""
            INSERT INTO orders (order_id, customer_id, amount)
            VALUES (?, ?, ?)
        """, order['order_id'], order['customer_id'], order['amount'])
        
        # Record that we processed this message
        db.execute("""
            INSERT INTO processed_messages (message_id, processed_at)
            VALUES (?, ?)
        """, message_id, datetime.now())
        
        # Update offset checkpoint
        db.execute("""
            INSERT INTO kafka_offsets (topic, partition, offset)
            VALUES (?, ?, ?)
            ON CONFLICT (topic, partition) DO UPDATE SET
                offset = EXCLUDED.offset
        """, message.topic, message.partition, message.offset)
    
    # Transaction committed: processing, dedup, and offset are atomic
    # If crash happens, transaction rolls back and we retry
```

### Kafka Transactions

Kafka provides built-in transactional support:

```python
from kafka import KafkaConsumer, KafkaProducer

def process_with_kafka_transactions():
    consumer = KafkaConsumer(
        'orders',
        group_id='processor',
        enable_auto_commit=False,
        isolation_level='read_committed'  # ✅ Only read committed messages
    )
    
    producer = KafkaProducer(
        transactional_id='order-processor-0',  # ✅ Enable transactions
        enable_idempotence=True
    )
    
    # Initialize transactions
    producer.init_transactions()
    
    for message in consumer:
        try:
            # Begin transaction
            producer.begin_transaction()
            
            # Process
            order = json.loads(message.value)
            transformed = transform_order(order)
            
            # Produce output
            producer.send('processed_orders', 
                         key=order['order_id'],
                         value=json.dumps(transformed))
            
            # ✅ Atomically commit: output + offset
            producer.send_offsets_to_transaction(
                {
                    TopicPartition(message.topic, message.partition): 
                        OffsetAndMetadata(message.offset + 1, None)
                },
                consumer.config['group_id']
            )
            
            # Commit transaction
            producer.commit_transaction()
            
        except Exception as e:
            # Abort transaction on error
            producer.abort_transaction()
            logger.error(f"Transaction aborted: {e}")

# ✅ Exactly-once: output and offset committed together
# If crash happens, transaction aborts and we retry from last committed offset
```

### Idempotency Keys

Use client-provided keys for deduplication:

```python
class ExactlyOnceProcessor:
    def __init__(self):
        self.processed_keys = set()  # In production: use Redis/DB
    
    def process(self, message, idempotency_key):
        """
        Process message with idempotency key.
        Returns: (success, already_processed)
        """
        
        # Check if already processed
        if idempotency_key in self.processed_keys:
            return (True, True)  # Already done
        
        try:
            # Process
            result = process_message(message)
            
            # Record key
            self.processed_keys.add(idempotency_key)
            
            return (True, False)  # Success, not duplicate
            
        except Exception as e:
            logger.error(f"Processing failed: {e}")
            return (False, False)  # Failed

# Usage
processor = ExactlyOnceProcessor()

for message in consumer:
    # Use message offset as idempotency key
    idempotency_key = f"{message.topic}:{message.partition}:{message.offset}"
    
    success, was_duplicate = processor.process(message, idempotency_key)
    
    if success:
        consumer.commit()
```

### Distributed Deduplication

Use external store for deduplication across instances:

```python
import redis

class DistributedDeduplicator:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def is_processed(self, message_id, ttl_seconds=86400):
        """Check if message already processed."""
        key = f"processed:{message_id}"
        return self.redis.exists(key)
    
    def mark_processed(self, message_id, ttl_seconds=86400):
        """Mark message as processed with TTL."""
        key = f"processed:{message_id}"
        self.redis.setex(key, ttl_seconds, "1")

# Usage
redis_client = redis.Redis(host='localhost', port=6379)
dedup = DistributedDeduplicator(redis_client)

def process_exactly_once(message):
    message_id = message.key.decode()
    
    # Check if processed
    if dedup.is_processed(message_id):
        logger.info(f"Skipping duplicate: {message_id}")
        consumer.commit()
        return
    
    # Process
    try:
        process_message(message)
        
        # Mark as processed
        dedup.mark_processed(message_id)
        
        # Commit
        consumer.commit()
        
    except Exception as e:
        logger.error(f"Processing failed: {e}")
        # Don't mark as processed, will retry
```

## Handling Zombie Writers

Prevent duplicate processing from multiple instances:

```python
import time
import uuid

class FencedProcessor:
    """Prevent zombie writers with fencing tokens."""
    
    def __init__(self, redis_client, processor_id=None):
        self.redis = redis_client
        self.processor_id = processor_id or str(uuid.uuid4())
        self.fence_token = None
    
    def acquire_lock(self, partition, timeout=30):
        """Acquire exclusive lock on partition."""
        lock_key = f"partition_lock:{partition}"
        
        # Try to acquire lock
        acquired = self.redis.set(
            lock_key,
            self.processor_id,
            nx=True,  # Only if not exists
            ex=timeout  # Expires in 30 seconds
        )
        
        if acquired:
            self.fence_token = time.time()
            return True
        
        return False
    
    def renew_lock(self, partition, timeout=30):
        """Renew lock before expiry."""
        lock_key = f"partition_lock:{partition}"
        
        # Only renew if we still hold it
        current_owner = self.redis.get(lock_key)
        if current_owner and current_owner.decode() == self.processor_id:
            self.redis.expire(lock_key, timeout)
            return True
        
        return False
    
    def process_with_fence(self, partition, message):
        """Process only if we hold the lock."""
        lock_key = f"partition_lock:{partition}"
        
        # Verify we still hold lock
        current_owner = self.redis.get(lock_key)
        if not current_owner or current_owner.decode() != self.processor_id:
            raise Exception(f"Lost lock on partition {partition}")
        
        # Process
        process_message(message)

# Usage
processor = FencedProcessor(redis_client)

partition = message.partition

if processor.acquire_lock(partition):
    try:
        # Renew lock periodically
        last_renew = time.time()
        
        for message in consumer:
            # Renew every 10 seconds
            if time.time() - last_renew > 10:
                if not processor.renew_lock(partition):
                    raise Exception("Failed to renew lock")
                last_renew = time.time()
            
            # Process with fencing
            processor.process_with_fence(partition, message)
            consumer.commit()
            
    except Exception as e:
        logger.error(f"Processing error: {e}")
```

## Window Deduplication

For streaming windows, deduplicate within window:

```python
class WindowedDeduplicator:
    """Deduplicate within time windows."""
    
    def __init__(self, window_duration_seconds):
        self.window_duration = window_duration_seconds
        self.windows = {}  # window_start -> set of message_ids
    
    def is_duplicate(self, message_id, event_time):
        """Check if message is duplicate within its window."""
        window_start = self._get_window_start(event_time)
        
        if window_start not in self.windows:
            self.windows[window_start] = set()
        
        if message_id in self.windows[window_start]:
            return True
        
        # Not duplicate, add to window
        self.windows[window_start].add(message_id)
        
        # Clean old windows
        self._clean_old_windows(event_time)
        
        return False
    
    def _get_window_start(self, event_time):
        """Get window start for event time."""
        timestamp = event_time.timestamp()
        window_start = (timestamp // self.window_duration) * self.window_duration
        return datetime.fromtimestamp(window_start)
    
    def _clean_old_windows(self, current_time):
        """Remove windows older than 2x window duration."""
        cutoff = current_time - timedelta(seconds=self.window_duration * 2)
        cutoff_start = self._get_window_start(cutoff)
        
        to_remove = [w for w in self.windows if w < cutoff_start]
        for window in to_remove:
            del self.windows[window]

# Usage for hourly windows
dedup = WindowedDeduplicator(window_duration_seconds=3600)

for message in consumer:
    order = json.loads(message.value)
    message_id = order['order_id']
    event_time = datetime.fromisoformat(order['created_at'])
    
    if not dedup.is_duplicate(message_id, event_time):
        process_message(order)
        consumer.commit()
```

## Why It's a Problem

1. **Data correctness**: Duplicates corrupt balances, counts, and aggregates
2. **Compliance**: Financial systems require exactly-once guarantees
3. **Cascading errors**: Duplicates multiply through pipeline stages
4. **Customer impact**: Duplicate charges, emails, or notifications

## Symptoms

- Inflated metrics and aggregates
- Duplicate records with slightly different timestamps
- Account balances don't reconcile
- Customers receive duplicate notifications

## Benefits

- **Correctness**: Guaranteed accurate processing
- **Compliance**: Meet regulatory requirements
- **Data quality**: No duplicates or data loss
- **Trust**: Confidence in system behavior

## Trade-offs

- **Performance**: Extra coordination adds latency
- **Complexity**: Requires transactions or deduplication logic
- **Cost**: External stores (Redis, DB) for deduplication state
- **Availability**: Stronger guarantees may reduce availability

## Best Practices

1. **Use transactions**: Atomically commit processing + checkpoints
2. **Idempotency keys**: Unique identifiers for deduplication
3. **TTL on dedup records**: Clean up old deduplication state
4. **Fencing tokens**: Prevent zombie writers
5. **Combine with idempotency**: Defense in depth

## See Also

- [Idempotent Processing](./idempotent-processing.md)
- [Checkpointing and Recovery](./checkpointing-recovery.md)
- [State Management](./state-management.md)
