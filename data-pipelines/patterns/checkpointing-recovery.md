# Checkpointing and Recovery

> Save progress periodically to resume from last known state after failures—avoid reprocessing from the beginning.

## Problem

Pipelines fail due to infrastructure issues, bugs, or transient errors. Without checkpointing, they must restart from the beginning, wasting time and resources reprocessing data that was already handled successfully. Long-running pipelines may never complete if failures happen frequently enough.

## Example

### ❌ Before (No Checkpointing)

```python
def process_daily_batch(date):
    # Get all records for the day
    records = database.query(
        "SELECT * FROM orders WHERE date = ?", date
    )
    
    # Process each record
    for record in records:
        transform_and_load(record)
    
    # ❌ If crash happens midway, restart from beginning
    # All successfully processed records are reprocessed

# Run takes 4 hours
# Crash at hour 3
# Restart: reprocess all 3 hours of work
```

### ✅ After (With Checkpointing)

```python
def process_daily_batch(date):
    # Check last checkpoint
    checkpoint = load_checkpoint(date)
    last_processed_id = checkpoint.last_id if checkpoint else None
    
    # Get records after checkpoint
    query = """
        SELECT * FROM orders 
        WHERE date = ? 
        AND order_id > ?
        ORDER BY order_id
    """
    records = database.query(query, date, last_processed_id or 0)
    
    batch = []
    for record in records:
        batch.append(record)
        
        # Process in batches
        if len(batch) >= 1000:
            transform_and_load_batch(batch)
            
            # ✅ Save checkpoint after successful batch
            save_checkpoint(date, batch[-1].order_id)
            batch = []
    
    # Process remaining records
    if batch:
        transform_and_load_batch(batch)
        save_checkpoint(date, batch[-1].order_id)

def load_checkpoint(date):
    return database.query_one(
        "SELECT last_id FROM checkpoints WHERE date = ?", date
    )

def save_checkpoint(date, last_id):
    database.execute("""
        INSERT INTO checkpoints (date, last_id, updated_at)
        VALUES (?, ?, ?)
        ON CONFLICT (date) DO UPDATE SET
            last_id = EXCLUDED.last_id,
            updated_at = EXCLUDED.updated_at
    """, date, last_id, datetime.now())

# Run takes 4 hours
# Crash at hour 3
# Restart: resume from hour 3, only 1 hour of reprocessing
```

## Streaming Checkpoints

### Kafka Offset Management

```python
from kafka import KafkaConsumer

def process_stream():
    consumer = KafkaConsumer(
        'orders',
        group_id='order-processor',
        enable_auto_commit=False,  # ✅ Manual commit control
        auto_offset_reset='earliest'
    )
    
    batch = []
    
    for message in consumer:
        order = parse_order(message.value)
        batch.append(order)
        
        if len(batch) >= 100:
            # Process batch
            transform_and_load_batch(batch)
            
            # ✅ Commit offset after successful processing
            consumer.commit()
            batch = []

# Restart: Kafka remembers committed offset, resumes from there
```

### Transactional Checkpointing

```python
def process_stream_transactional():
    consumer = KafkaConsumer('orders', enable_auto_commit=False)
    
    for message in consumer:
        order = parse_order(message.value)
        
        # ✅ Atomically process + checkpoint in transaction
        with database.transaction():
            # Process order
            database.insert("processed_orders", order)
            
            # Save checkpoint (offset)
            database.execute("""
                INSERT INTO kafka_offsets 
                (topic, partition, offset, updated_at)
                VALUES (?, ?, ?, ?)
                ON CONFLICT (topic, partition) DO UPDATE SET
                    offset = EXCLUDED.offset,
                    updated_at = EXCLUDED.updated_at
            """, message.topic, message.partition, 
                 message.offset, datetime.now())
        
        # Commit Kafka offset after transaction succeeds
        consumer.commit()

# If crash happens:
# 1. Transaction rolls back (no partial data)
# 2. Kafka offset not committed
# 3. Restart reprocesses from last successful transaction
```

## Checkpoint Granularity

### Too Fine-Grained (Slow)

```python
def process_batch(records):
    for record in records:
        transform_and_load(record)
        
        # ❌ Checkpoint every single record (expensive!)
        save_checkpoint(record.id)
        
# 1 million records = 1 million checkpoint writes
# Checkpoint overhead dominates processing time
```

### Too Coarse-Grained (Risky)

```python
def process_batch(records):
    for record in records:
        transform_and_load(record)
    
    # ❌ Checkpoint only at end (lose all progress on failure)
    save_checkpoint(records[-1].id)

# Process 1 million records
# Crash at record 999,999
# Restart: reprocess all 1 million
```

### Balanced Approach

```python
def process_batch(records):
    batch = []
    checkpoint_interval = 1000  # ✅ Tune based on workload
    
    for i, record in enumerate(records):
        batch.append(record)
        
        if len(batch) >= 100:
            # Process batch
            transform_and_load_batch(batch)
            batch = []
            
            # ✅ Checkpoint every N records
            if (i + 1) % checkpoint_interval == 0:
                save_checkpoint(record.id)

# 1 million records = 1,000 checkpoints
# Crash: lose at most 1,000 records of progress
```

## Checkpoint Storage

### Database Checkpoints

```python
CREATE TABLE checkpoints (
    job_id VARCHAR(255) NOT NULL,
    partition_id VARCHAR(255) NOT NULL,
    checkpoint_value VARCHAR(255) NOT NULL,
    metadata JSONB,
    updated_at TIMESTAMP NOT NULL,
    
    PRIMARY KEY (job_id, partition_id)
);

class CheckpointStore:
    def save(self, job_id, partition, value, metadata=None):
        self.db.execute("""
            INSERT INTO checkpoints 
            (job_id, partition_id, checkpoint_value, metadata, updated_at)
            VALUES (?, ?, ?, ?, ?)
            ON CONFLICT (job_id, partition_id) DO UPDATE SET
                checkpoint_value = EXCLUDED.checkpoint_value,
                metadata = EXCLUDED.metadata,
                updated_at = EXCLUDED.updated_at
        """, job_id, partition, value, 
             json.dumps(metadata), datetime.now())
    
    def load(self, job_id, partition):
        row = self.db.query_one("""
            SELECT checkpoint_value, metadata 
            FROM checkpoints 
            WHERE job_id = ? AND partition_id = ?
        """, job_id, partition)
        
        if row:
            return {
                'value': row.checkpoint_value,
                'metadata': json.loads(row.metadata) if row.metadata else {}
            }
        return None
```

### File-Based Checkpoints

```python
import json
import os

class FileCheckpointStore:
    def __init__(self, checkpoint_dir):
        self.checkpoint_dir = checkpoint_dir
        os.makedirs(checkpoint_dir, exist_ok=True)
    
    def save(self, job_id, partition, value, metadata=None):
        checkpoint_file = os.path.join(
            self.checkpoint_dir, 
            f"{job_id}_{partition}.json"
        )
        
        data = {
            'value': value,
            'metadata': metadata or {},
            'timestamp': datetime.now().isoformat()
        }
        
        # ✅ Atomic write (write to temp, then rename)
        temp_file = f"{checkpoint_file}.tmp"
        with open(temp_file, 'w') as f:
            json.dump(data, f)
        
        os.rename(temp_file, checkpoint_file)
    
    def load(self, job_id, partition):
        checkpoint_file = os.path.join(
            self.checkpoint_dir,
            f"{job_id}_{partition}.json"
        )
        
        if os.path.exists(checkpoint_file):
            with open(checkpoint_file, 'r') as f:
                return json.load(f)
        return None
```

## Watermarks for Event Time

When processing out-of-order events:

```python
class EventTimeCheckpoint:
    def __init__(self):
        self.processing_time = None
        self.event_time_watermark = None
        self.records_processed = 0
    
    def update(self, event_time):
        """Update watermark based on event times seen."""
        self.processing_time = datetime.now()
        self.records_processed += 1
        
        # Watermark: min event time seen (conservative)
        if self.event_time_watermark is None:
            self.event_time_watermark = event_time
        else:
            self.event_time_watermark = min(
                self.event_time_watermark, 
                event_time
            )

def process_with_watermarks():
    checkpoint = load_checkpoint('job_id')
    
    # Resume from watermark
    min_event_time = (checkpoint.event_time_watermark 
                     if checkpoint else datetime.min)
    
    events = get_events_after(min_event_time)
    
    current_checkpoint = EventTimeCheckpoint()
    
    for event in events:
        process_event(event)
        current_checkpoint.update(event.timestamp)
        
        # Save checkpoint periodically
        if current_checkpoint.records_processed % 1000 == 0:
            save_checkpoint('job_id', current_checkpoint)
```

## Recovery Strategies

### Automatic Recovery

```python
import time

def process_with_retry(max_retries=3):
    attempt = 0
    
    while attempt < max_retries:
        try:
            # Load checkpoint
            checkpoint = load_checkpoint('job_id')
            
            # Resume from checkpoint
            process_from_checkpoint(checkpoint)
            
            # Success
            break
            
        except Exception as e:
            attempt += 1
            logger.error(f"Attempt {attempt} failed: {e}")
            
            if attempt < max_retries:
                # Exponential backoff
                wait_time = 2 ** attempt
                logger.info(f"Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                logger.error("Max retries exceeded")
                raise
```

### Manual Recovery

```python
# Tool to inspect and manipulate checkpoints
class CheckpointManager:
    def list_checkpoints(self, job_id):
        """List all checkpoints for a job."""
        return self.db.query("""
            SELECT partition_id, checkpoint_value, updated_at
            FROM checkpoints
            WHERE job_id = ?
            ORDER BY partition_id
        """, job_id)
    
    def reset_checkpoint(self, job_id, partition, value):
        """Manually set checkpoint (for reprocessing)."""
        self.save(job_id, partition, value, {
            'reset_by': 'admin',
            'reset_at': datetime.now().isoformat()
        })
    
    def delete_checkpoint(self, job_id, partition):
        """Delete checkpoint (restart from beginning)."""
        self.db.execute("""
            DELETE FROM checkpoints
            WHERE job_id = ? AND partition_id = ?
        """, job_id, partition)

# CLI tool
manager = CheckpointManager()

# View current state
manager.list_checkpoints('daily-orders')

# Reprocess last 7 days
seven_days_ago = datetime.now() - timedelta(days=7)
manager.reset_checkpoint('daily-orders', 'partition-0', seven_days_ago)
```

## Why It's a Problem

1. **Wasted work**: Reprocess successfully handled data after failures
2. **Long recovery**: Pipelines take hours to catch up after crashes
3. **Resource waste**: Repeatedly process the same data
4. **Never completing**: Frequent failures prevent job completion

## Symptoms

- Pipelines restart from beginning after every failure
- Long-running jobs never finish due to intermittent errors
- Same data appears multiple times in logs
- High resource usage reprocessing old data

## Benefits

- **Fast recovery**: Resume from last checkpoint, not beginning
- **Resilience**: Tolerate frequent failures without data loss
- **Efficiency**: Process each record once (or few times)
- **Progress guarantees**: Large jobs can complete despite failures

## Best Practices

1. **Checkpoint frequently**: Balance overhead vs. progress saved
2. **Atomic commits**: Commit checkpoint and output together
3. **Idempotent processing**: Combine with idempotency for safety
4. **Monitor checkpoints**: Track checkpoint lag and recovery time
5. **Test recovery**: Regularly test failure and recovery scenarios

## See Also

- [Idempotent Processing](./idempotent-processing.md)
- [Exactly-Once Semantics](./exactly-once-semantics.md)
- [State Management](./state-management.md)
- [Watermarks and Windowing](./watermarks-windowing.md)
