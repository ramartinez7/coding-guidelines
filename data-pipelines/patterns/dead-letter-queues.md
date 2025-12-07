# Dead Letter Queues

> Move unparseable or invalid records to a separate queue for inspection—don't block valid data with bad data.

## Problem

Pipelines encounter invalid, corrupted, or unparseable data. If the pipeline crashes or skips these records silently, you lose visibility into data quality issues. If it retries forever, bad data blocks all valid data behind it, bringing the entire pipeline to a halt.

## Example

### ❌ Before (Crash or Block)

```python
def process_orders(messages):
    for message in messages:
        # Parse JSON
        try:
            data = json.loads(message)
        except json.JSONDecodeError:
            # ❌ Option 1: Crash (stops entire pipeline)
            raise
            
            # ❌ Option 2: Skip silently (lose data, no visibility)
            continue
        
        # Validate
        if not data.get("order_id"):
            # ❌ Option 3: Retry forever (blocks valid data)
            raise ValueError("Missing order_id")
        
        # Process
        transform_and_load(data)

# Bad message blocks all messages behind it
# OR bad messages disappear without trace
```

### ✅ After (Dead Letter Queue)

```python
def process_orders(messages):
    for message in messages:
        try:
            # Parse
            data = json.loads(message)
            
            # Validate
            order = validate_order(data)
            
            # Process
            transform_and_load(order)
            
        except Exception as e:
            # ✅ Send to DLQ instead of crashing or skipping
            send_to_dlq(message, error=str(e), exception_type=type(e).__name__)
            
            # Continue processing other messages
            continue

def send_to_dlq(original_message, error, exception_type):
    """Send failed message to dead letter queue with metadata."""
    dlq_message = {
        "original_message": original_message,
        "error": error,
        "exception_type": exception_type,
        "timestamp": datetime.now().isoformat(),
        "pipeline_stage": "order-processing",
        "retry_count": 0
    }
    
    # Send to dedicated DLQ topic/queue
    kafka_producer.send("orders-dlq", json.dumps(dlq_message))
    
    # Log for alerting
    logger.error(f"Sent to DLQ: {error}", extra={
        "dlq_message": dlq_message
    })
```

## Retry with DLQ

Combine retries with eventual DLQ routing:

```python
def process_with_retry(message, max_retries=3):
    """Try processing multiple times before sending to DLQ."""
    retry_count = message.get("retry_count", 0)
    
    try:
        # Parse and process
        data = json.loads(message["data"])
        order = validate_order(data)
        transform_and_load(order)
        
    except TransientError as e:
        # Transient error: retry with backoff
        if retry_count < max_retries:
            retry_message = {
                **message,
                "retry_count": retry_count + 1,
                "last_error": str(e),
                "retry_at": (datetime.now() + timedelta(seconds=2**retry_count)).isoformat()
            }
            
            # Send to retry queue
            retry_queue.send(retry_message)
            logger.warning(f"Retrying ({retry_count + 1}/{max_retries}): {e}")
        else:
            # Exhausted retries: send to DLQ
            send_to_dlq(message, error=f"Max retries exceeded: {e}",
                       exception_type="TransientError")
    
    except PermanentError as e:
        # Permanent error: go straight to DLQ
        send_to_dlq(message, error=str(e), exception_type="PermanentError")

# Classify errors
class TransientError(Exception):
    """Temporary failure that might succeed on retry."""
    pass

class PermanentError(Exception):
    """Permanent failure that won't succeed on retry."""
    pass

def validate_order(data):
    # Check for required fields (permanent error)
    if not data.get("order_id"):
        raise PermanentError("Missing required field: order_id")
    
    # Check external service (transient error)
    if not customer_service.exists(data["customer_id"]):
        if customer_service.is_healthy():
            raise PermanentError(f"Customer {data['customer_id']} not found")
        else:
            raise TransientError("Customer service unavailable")
    
    return Order(**data)
```

## DLQ Processing and Monitoring

### Monitor DLQ

```python
from prometheus_client import Counter, Gauge

dlq_messages = Counter(
    'dlq_messages_total',
    'Total messages sent to DLQ',
    ['error_type', 'pipeline_stage']
)

dlq_size = Gauge(
    'dlq_queue_size',
    'Current number of messages in DLQ'
)

def send_to_dlq(message, error, exception_type):
    dlq_message = {
        "original_message": message,
        "error": error,
        "exception_type": exception_type,
        "timestamp": datetime.now().isoformat(),
        "pipeline_stage": "order-processing"
    }
    
    kafka_producer.send("orders-dlq", json.dumps(dlq_message))
    
    # Track metrics
    dlq_messages.labels(
        error_type=exception_type,
        pipeline_stage="order-processing"
    ).inc()
    
    # Alert if DLQ grows too large
    current_size = get_dlq_size()
    dlq_size.set(current_size)
    
    if current_size > 1000:
        alert_service.send(f"DLQ size exceeded threshold: {current_size}")
```

### Inspect DLQ

```python
class DLQInspector:
    def list_errors(self, limit=100):
        """List recent DLQ messages."""
        messages = kafka_consumer.consume("orders-dlq", limit=limit)
        return [json.loads(msg) for msg in messages]
    
    def group_by_error_type(self):
        """Aggregate errors by type."""
        messages = self.list_errors(limit=1000)
        
        by_type = {}
        for msg in messages:
            error_type = msg["exception_type"]
            by_type[error_type] = by_type.get(error_type, 0) + 1
        
        return sorted(by_type.items(), key=lambda x: x[1], reverse=True)
    
    def sample_errors(self, error_type, limit=10):
        """Get sample messages for specific error type."""
        messages = self.list_errors(limit=1000)
        samples = [
            msg for msg in messages 
            if msg["exception_type"] == error_type
        ]
        return samples[:limit]

# Usage
inspector = DLQInspector()

# What are the most common errors?
print(inspector.group_by_error_type())
# [('PermanentError', 450), ('ValidationError', 200), ('TransientError', 50)]

# Get examples of validation errors
samples = inspector.sample_errors('ValidationError', limit=5)
for sample in samples:
    print(f"Error: {sample['error']}")
    print(f"Data: {sample['original_message']}")
```

### Reprocess from DLQ

```python
class DLQReprocessor:
    def reprocess_all(self, dlq_topic, target_topic):
        """Reprocess all messages from DLQ."""
        consumer = KafkaConsumer(dlq_topic)
        producer = KafkaProducer()
        
        reprocessed = 0
        failed = 0
        
        for message in consumer:
            dlq_data = json.loads(message.value)
            original = dlq_data["original_message"]
            
            try:
                # Try processing again (maybe bug was fixed)
                data = json.loads(original)
                order = validate_order(data)
                
                # Success: send back to main topic
                producer.send(target_topic, json.dumps(order))
                reprocessed += 1
                
            except Exception as e:
                # Still failing: keep in DLQ
                logger.error(f"Reprocessing failed: {e}")
                failed += 1
        
        logger.info(f"Reprocessed {reprocessed}, failed {failed}")
        return reprocessed, failed
    
    def reprocess_by_error_type(self, dlq_topic, target_topic, error_type):
        """Reprocess only specific error types."""
        consumer = KafkaConsumer(dlq_topic)
        producer = KafkaProducer()
        
        for message in consumer:
            dlq_data = json.loads(message.value)
            
            if dlq_data["exception_type"] == error_type:
                original = dlq_data["original_message"]
                
                # Try processing again
                try:
                    data = json.loads(original)
                    order = validate_order(data)
                    producer.send(target_topic, json.dumps(order))
                except Exception:
                    # Still failing: keep in DLQ
                    pass

# Reprocess after fixing a bug
reprocessor = DLQReprocessor()
reprocessor.reprocess_by_error_type("orders-dlq", "orders", "ValidationError")
```

## DLQ Storage Options

### Kafka Topic

```python
# Separate topic for DLQ
MAIN_TOPIC = "orders"
DLQ_TOPIC = "orders-dlq"

# Configure with longer retention
kafka_admin.create_topic(
    DLQ_TOPIC,
    config={
        "retention.ms": 30 * 24 * 60 * 60 * 1000  # 30 days
    }
)
```

### Database Table

```python
CREATE TABLE dead_letter_queue (
    id SERIAL PRIMARY KEY,
    topic VARCHAR(255) NOT NULL,
    partition_id INT,
    offset BIGINT,
    original_message TEXT NOT NULL,
    error_message TEXT NOT NULL,
    exception_type VARCHAR(255) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    
    INDEX idx_exception_type (exception_type),
    INDEX idx_created_at (created_at)
);

def send_to_dlq_db(message, error, exception_type):
    db.execute("""
        INSERT INTO dead_letter_queue 
        (topic, original_message, error_message, exception_type, created_at)
        VALUES (?, ?, ?, ?, ?)
    """, "orders", message, error, exception_type, datetime.now())
```

### Object Storage

```python
import boto3
import hashlib

s3_client = boto3.client('s3')

def send_to_dlq_s3(message, error, exception_type):
    # Generate unique key
    message_hash = hashlib.sha256(message.encode()).hexdigest()
    timestamp = datetime.now().strftime("%Y/%m/%d/%H")
    key = f"dlq/{timestamp}/{exception_type}/{message_hash}.json"
    
    dlq_message = {
        "original_message": message,
        "error": error,
        "exception_type": exception_type,
        "timestamp": datetime.now().isoformat()
    }
    
    # Store in S3
    s3_client.put_object(
        Bucket='data-pipeline-dlq',
        Key=key,
        Body=json.dumps(dlq_message),
        ContentType='application/json'
    )
```

## Partial DLQ (Filtered)

Only send specific errors to DLQ:

```python
def process_order(message):
    try:
        data = json.loads(message)
        order = validate_order(data)
        transform_and_load(order)
        
    except json.JSONDecodeError as e:
        # Unparseable: send to DLQ
        send_to_dlq(message, str(e), "JSONDecodeError")
        
    except ValidationError as e:
        # Invalid schema: send to DLQ
        send_to_dlq(message, str(e), "ValidationError")
        
    except TransientError as e:
        # Temporary issue: retry, don't DLQ
        raise
        
    except Exception as e:
        # Unexpected error: alert and DLQ
        alert_service.send(f"Unexpected error: {e}")
        send_to_dlq(message, str(e), type(e).__name__)
```

## Why It's a Problem

1. **Pipeline blockage**: Bad data stops all processing
2. **Silent failures**: Invalid data disappears without trace
3. **No visibility**: Can't diagnose data quality issues
4. **Lost data**: Unrecoverable errors result in data loss

## Symptoms

- Pipelines stuck on single bad message
- Missing data in destination with no error logs
- Repeated crashes on same malformed input
- No way to inspect or recover failed messages

## Benefits

- **Unblock processing**: Valid data flows even when bad data arrives
- **Visibility**: All failures recorded with context
- **Recoverability**: Failed messages can be reprocessed after fixes
- **Diagnosis**: Patterns in DLQ reveal data quality issues

## Best Practices

1. **Always use DLQ**: Don't skip or crash on bad data
2. **Include context**: Store original message, error, timestamp, stage
3. **Monitor DLQ size**: Alert when it grows unexpectedly
4. **Classify errors**: Permanent vs. transient, retryable vs. not
5. **Retention policy**: Keep DLQ data long enough for investigation

## See Also

- [Data Validation at Boundaries](./data-validation-boundaries.md)
- [Idempotent Processing](./idempotent-processing.md)
- [Data Quality Monitoring](./data-quality-monitoring.md)
