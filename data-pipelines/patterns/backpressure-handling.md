# Backpressure Handling

> Handle mismatched throughput between producers and consumers—use bounded buffers and flow control to prevent overload.

## Problem

Producers and consumers operate at different speeds. Fast producers overwhelm slow consumers, causing memory exhaustion, dropped messages, or cascading failures. Without backpressure mechanisms, the entire system becomes unstable.

## Example

### ❌ Before (No Backpressure)

```python
from queue import Queue
import threading

# Unbounded queue
buffer = Queue()  # ❌ No size limit

def producer():
    """Produces 1000 messages/sec."""
    while True:
        message = generate_message()
        buffer.put(message)  # Never blocks
        # ❌ Keeps adding even if consumer is slow

def consumer():
    """Processes 100 messages/sec."""
    while True:
        message = buffer.get()
        process_message(message)  # Takes 10ms per message
        # ❌ Can't keep up with producer

# Start threads
threading.Thread(target=producer, daemon=True).start()
threading.Thread(target=consumer, daemon=True).start()

# Result: Queue grows unbounded -> OutOfMemoryError
```

### ✅ After (With Backpressure)

```python
from queue import Queue
import threading

# Bounded queue
buffer = Queue(maxsize=1000)  # ✅ Size limit

def producer():
    """Produces 1000 messages/sec, but blocks when full."""
    while True:
        message = generate_message()
        
        # ✅ Blocks when queue is full
        buffer.put(message, block=True)
        
        # Producer slows down to consumer's pace

def consumer():
    """Processes 100 messages/sec."""
    while True:
        message = buffer.get()
        process_message(message)

# Buffer never exceeds 1000 items
# Producer automatically throttled by consumer
```

## Backpressure Strategies

### 1. Bounded Buffers

Block producers when buffer is full:

```python
class BoundedBuffer:
    def __init__(self, max_size):
        self.max_size = max_size
        self.buffer = []
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)
        self.not_empty = threading.Condition(self.lock)
    
    def put(self, item, timeout=None):
        """Add item, block if full."""
        with self.not_full:
            # Wait until buffer has space
            while len(self.buffer) >= self.max_size:
                if not self.not_full.wait(timeout):
                    raise TimeoutError("Buffer full")
            
            # Add item
            self.buffer.append(item)
            self.not_empty.notify()
    
    def get(self, timeout=None):
        """Remove item, block if empty."""
        with self.not_empty:
            # Wait until buffer has items
            while len(self.buffer) == 0:
                if not self.not_empty.wait(timeout):
                    raise TimeoutError("Buffer empty")
            
            # Remove item
            item = self.buffer.pop(0)
            self.not_full.notify()
            return item
```

### 2. Rate Limiting

Limit producer rate explicitly:

```python
import time
from collections import deque

class RateLimiter:
    def __init__(self, max_rate_per_second):
        self.max_rate = max_rate_per_second
        self.interval = 1.0 / max_rate_per_second
        self.last_request = 0
    
    def acquire(self):
        """Wait if necessary to maintain rate limit."""
        now = time.time()
        elapsed = now - self.last_request
        
        if elapsed < self.interval:
            # Need to wait
            time.sleep(self.interval - elapsed)
        
        self.last_request = time.time()

# Usage
rate_limiter = RateLimiter(max_rate_per_second=100)

def producer_with_rate_limit():
    while True:
        rate_limiter.acquire()  # ✅ Wait if too fast
        message = generate_message()
        buffer.put(message)
```

### 3. Token Bucket

Allow bursts while maintaining average rate:

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()
        self.lock = threading.Lock()
    
    def _refill(self):
        """Refill tokens based on elapsed time."""
        now = time.time()
        elapsed = now - self.last_refill
        
        # Add tokens based on time passed
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
    
    def acquire(self, tokens=1, blocking=True):
        """Acquire tokens, optionally blocking."""
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens:
                # Tokens available
                self.tokens -= tokens
                return True
            
            if not blocking:
                return False
            
            # Wait until enough tokens available
            wait_time = (tokens - self.tokens) / self.refill_rate
            time.sleep(wait_time)
            
            self._refill()
            self.tokens -= tokens
            return True

# Usage: Allow bursts up to 1000, but average 100/sec
bucket = TokenBucket(capacity=1000, refill_rate=100)

def producer_with_bursts():
    while True:
        bucket.acquire()  # ✅ Allow bursts, limit average
        message = generate_message()
        buffer.put(message)
```

### 4. Drop/Shed

Drop messages when overloaded (lossy backpressure):

```python
class DroppingBuffer:
    def __init__(self, max_size, drop_strategy='tail'):
        self.max_size = max_size
        self.buffer = deque()
        self.drop_strategy = drop_strategy
        self.dropped_count = 0
    
    def put(self, item):
        """Add item, drop if full."""
        if len(self.buffer) >= self.max_size:
            if self.drop_strategy == 'tail':
                # Drop newest (just received)
                self.dropped_count += 1
                return False
            elif self.drop_strategy == 'head':
                # Drop oldest
                self.buffer.popleft()
                self.dropped_count += 1
        
        self.buffer.append(item)
        return True

# Usage
buffer = DroppingBuffer(max_size=1000, drop_strategy='tail')

def producer_with_dropping():
    while True:
        message = generate_message()
        
        if not buffer.put(message):
            logger.warning("Buffer full, dropped message")
            metrics.inc('messages_dropped')
```

### 5. Dynamic Batching

Adjust batch size based on consumer capacity:

```python
class AdaptiveBatcher:
    def __init__(self, initial_batch_size=100):
        self.batch_size = initial_batch_size
        self.min_batch_size = 10
        self.max_batch_size = 1000
    
    def adjust_batch_size(self, processing_time, target_time=1.0):
        """Adjust batch size to hit target processing time."""
        
        if processing_time > target_time * 1.5:
            # Too slow, reduce batch size
            self.batch_size = max(
                self.min_batch_size,
                int(self.batch_size * 0.8)
            )
        elif processing_time < target_time * 0.5:
            # Too fast, increase batch size
            self.batch_size = min(
                self.max_batch_size,
                int(self.batch_size * 1.2)
            )
        
        return self.batch_size

# Usage
batcher = AdaptiveBatcher()

def consumer_with_adaptive_batching():
    while True:
        # Get batch
        batch = []
        for _ in range(batcher.batch_size):
            if not buffer.empty():
                batch.append(buffer.get())
        
        # Process batch
        start = time.time()
        process_batch(batch)
        processing_time = time.time() - start
        
        # Adjust for next iteration
        batcher.adjust_batch_size(processing_time)
```

## Reactive Streams

Use reactive programming for backpressure:

```python
# Using RxPY (reactive extensions)
import rx
from rx import operators as ops

def create_backpressure_stream():
    # Source produces items
    source = rx.interval(0.01)  # 100 items/sec
    
    # ✅ Buffer with backpressure
    stream = source.pipe(
        # Buffer up to 1000 items
        ops.buffer_with_count(100),
        
        # Process batch
        ops.flat_map(lambda batch: rx.from_iterable(
            process_batch(batch)
        )),
        
        # Apply backpressure if consumer slow
        ops.throttle_first(0.1)  # Max 10 results/sec
    )
    
    return stream

# Consumer controls pace
stream = create_backpressure_stream()
stream.subscribe(
    on_next=lambda x: handle_result(x),
    on_error=lambda e: logger.error(e)
)
```

## Kafka Backpressure

Kafka provides built-in backpressure through consumer lag:

```python
from kafka import KafkaConsumer

def consume_with_backpressure():
    consumer = KafkaConsumer(
        'orders',
        max_poll_records=100,  # ✅ Limit batch size
        max_poll_interval_ms=300000,  # 5 minutes max between polls
    )
    
    for batch in consumer:
        # Process batch
        messages = consumer.poll(timeout_ms=1000)
        
        if messages:
            process_batch(messages)
            
            # ✅ Only commit after successful processing
            consumer.commit()
            
            # If processing slow, consumer lag increases
            # Monitoring alerts on high lag

# Monitor consumer lag
def check_consumer_lag():
    from kafka.admin import KafkaAdminClient
    
    admin = KafkaAdminClient(bootstrap_servers='localhost:9092')
    
    # Get consumer group offsets
    offsets = admin.list_consumer_group_offsets('my-group')
    
    # Compare with latest offsets
    for partition, offset in offsets.items():
        latest = get_latest_offset(partition)
        lag = latest - offset.offset
        
        if lag > 10000:
            alert(f"High consumer lag: {lag}")
```

## Circuit Breaker

Stop sending when downstream is failing:

```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.state = 'closed'  # closed, open, half_open
        self.last_failure_time = None
    
    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker."""
        
        if self.state == 'open':
            # Check if timeout passed
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'half_open'
            else:
                raise Exception("Circuit breaker is open")
        
        try:
            result = func(*args, **kwargs)
            
            # Success
            if self.state == 'half_open':
                self.state = 'closed'
            self.failures = 0
            
            return result
            
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            
            if self.failures >= self.failure_threshold:
                self.state = 'open'
                logger.error("Circuit breaker opened")
            
            raise

# Usage
breaker = CircuitBreaker()

def producer_with_circuit_breaker():
    while True:
        message = generate_message()
        
        try:
            # ✅ Stop sending if downstream failing
            breaker.call(send_to_downstream, message)
        except Exception as e:
            logger.error(f"Circuit open, buffering: {e}")
            local_buffer.append(message)
```

## Why It's a Problem

1. **Memory exhaustion**: Unbounded buffers cause OOM errors
2. **Cascading failures**: Overloaded components fail, affecting others
3. **Message loss**: Dropped messages when buffers overflow
4. **Latency spikes**: Queuing delays increase response times

## Symptoms

- Out of memory errors
- Increasing queue depths
- Consumer lag growing unbounded
- Messages dropped or lost

## Benefits

- **Stability**: System remains stable under load
- **Graceful degradation**: Slow down instead of crashing
- **Resource protection**: Prevent memory exhaustion
- **Feedback mechanism**: Producers aware of consumer capacity

## Best Practices

1. **Bounded buffers**: Always set size limits
2. **Monitor lag**: Alert on growing queue depths
3. **Rate limiting**: Limit producer rate explicitly
4. **Circuit breakers**: Stop sending to failing systems
5. **Load shedding**: Drop when necessary, prefer tail drop

## See Also

- [Rate Limiting](./rate-limiting.md)
- [Circuit Breakers](./circuit-breakers.md)
- [Batch vs Streaming](./batch-vs-streaming.md)
