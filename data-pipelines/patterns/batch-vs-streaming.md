# Batch vs Streaming

> Choose between batch (bounded, scheduled) and streaming (unbounded, continuous) based on latency and throughput requirements.

## Problem

Data pipelines must balance latency, throughput, complexity, and cost. Choosing the wrong processing model leads to over-engineered solutions (streaming when batch suffices) or unmet requirements (batch when real-time is needed).

## When to Use Each

### Use Batch When:

- **Latency tolerance**: Results needed in minutes/hours, not seconds
- **Bounded datasets**: Processing complete snapshots (daily reports, monthly aggregates)
- **Complex transformations**: Multi-pass algorithms, sorts, joins across large datasets
- **Cost sensitive**: Batch is simpler and often cheaper
- **Scheduled reports**: Daily/weekly/monthly business reports

### Use Streaming When:

- **Low latency**: Results needed in seconds or sub-seconds
- **Continuous data**: Unbounded streams (logs, events, sensor data)
- **Real-time reactions**: Fraud detection, anomaly detection, alerting
- **Incremental updates**: Keep dashboards/metrics current
- **Event-driven**: React to events as they occur

## Examples

### ❌ Anti-Pattern: Batch for Real-Time

```python
# Fraud detection that runs hourly
def detect_fraud_batch():
    # Query last hour of transactions
    transactions = db.query("""
        SELECT * FROM transactions
        WHERE created_at > NOW() - INTERVAL '1 hour'
    """)
    
    # Analyze for fraud
    for txn in transactions:
        if is_fraudulent(txn):
            alert_fraud_team(txn)
    
    # ❌ Problem: Fraud detected up to 1 hour late!
    # Fraudster could make many transactions before detection

# Run every hour
schedule.every().hour.do(detect_fraud_batch)
```

### ✅ Streaming for Real-Time

```python
# Fraud detection on live stream
def detect_fraud_stream():
    consumer = KafkaConsumer('transactions')
    
    for message in consumer:
        txn = json.loads(message.value)
        
        # ✅ Analyze immediately
        if is_fraudulent(txn):
            alert_fraud_team(txn)
            block_card(txn.card_id)
        
        # Fraud detected within seconds
```

### ❌ Anti-Pattern: Streaming for Batch Reports

```python
# Daily sales report using streaming
def generate_daily_report_stream():
    consumer = KafkaConsumer('orders')
    
    daily_totals = {}
    
    # ❌ Keep state in memory for entire day
    for message in consumer:
        order = json.loads(message.value)
        date = order['created_at'].date()
        
        if date not in daily_totals:
            daily_totals[date] = 0
        
        daily_totals[date] += order['total']
        
        # When do we emit the report? How do we handle restarts?
    
    # ❌ Complex, stateful, hard to recover from failures
```

### ✅ Batch for Reports

```python
# Daily sales report using batch
def generate_daily_report_batch(date):
    # ✅ Simple SQL query
    result = db.query("""
        SELECT 
            DATE(created_at) as date,
            SUM(total) as daily_total,
            COUNT(*) as order_count
        FROM orders
        WHERE DATE(created_at) = ?
        GROUP BY DATE(created_at)
    """, date)
    
    return result

# Run daily at 1 AM
schedule.every().day.at("01:00").do(
    lambda: generate_daily_report_batch(date.today() - timedelta(days=1))
)
```

## Hybrid: Lambda Architecture

Combine batch and streaming for both completeness and low latency:

```python
# Speed layer: Real-time approximate results
class SpeedLayer:
    def __init__(self):
        self.consumer = KafkaConsumer('orders')
        self.cache = Redis()
    
    def process_stream(self):
        for message in self.consumer:
            order = json.loads(message.value)
            date = order['created_at'].date()
            
            # Increment real-time counter
            key = f"daily_total:{date}"
            self.cache.incrbyfloat(key, order['total'])
            self.cache.expire(key, 86400 * 2)  # Keep 2 days

# Batch layer: Complete accurate results
class BatchLayer:
    def compute_daily_totals(self, date):
        return db.query("""
            SELECT DATE(created_at) as date, SUM(total) as total
            FROM orders
            WHERE DATE(created_at) = ?
            GROUP BY DATE(created_at)
        """, date)
    
    def run_daily(self, date):
        # Compute accurate totals
        totals = self.compute_daily_totals(date)
        
        # Store in serving layer
        serving_db.upsert("daily_totals", totals)

# Serving layer: Merge results
class ServingLayer:
    def get_daily_total(self, date):
        # Check if batch completed for this date
        batch_result = serving_db.query_one(
            "SELECT total FROM daily_totals WHERE date = ?", date
        )
        
        if batch_result:
            # ✅ Return accurate batch result
            return batch_result.total
        else:
            # ⚡ Return real-time approximation
            key = f"daily_total:{date}"
            return float(cache.get(key) or 0)

# Usage: Get current total (might be from batch or stream)
total = serving_layer.get_daily_total(date.today())
```

## Micro-Batch: Best of Both Worlds

Process small batches continuously:

```python
# Apache Spark Structured Streaming (micro-batch)
def process_micro_batches():
    spark = SparkSession.builder.appName("micro-batch").getOrCreate()
    
    # Read stream in micro-batches
    orders_stream = spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "localhost:9092") \
        .option("subscribe", "orders") \
        .load()
    
    # Process each micro-batch
    aggregates = orders_stream \
        .groupBy(window("timestamp", "5 minutes"), "product_id") \
        .agg(sum("amount").alias("total_sales"))
    
    # Write results
    query = aggregates.writeStream \
        .outputMode("update") \
        .format("console") \
        .trigger(processingTime="10 seconds") \  # ✅ Process every 10 seconds
        .start()
    
    query.awaitTermination()

# ✅ Benefits:
# - Low latency (10 second windows)
# - Batch efficiency (process 10s of data at once)
# - Simpler than pure streaming (no per-event overhead)
```

## Decision Matrix

| Requirement | Batch | Streaming | Hybrid |
|------------|-------|-----------|--------|
| Latency < 1 minute | ❌ | ✅ | ✅ |
| Historical reprocessing | ✅ | ❌ | ✅ |
| Complex joins/sorts | ✅ | ⚠️ | ✅ |
| Cost efficiency | ✅ | ❌ | ⚠️ |
| Operational complexity | ✅ | ❌ | ❌ |
| Exactly-once guarantee | ✅ | ⚠️ | ⚠️ |
| Out-of-order handling | ✅ | ⚠️ | ✅ |

## Batch Processing Pattern

```python
def process_daily_batch(date):
    """
    Standard batch processing pattern:
    1. Extract data for time range
    2. Transform in memory or distributed system
    3. Load to destination
    4. Record completion
    """
    
    # Extract
    records = extract_orders(date)
    
    # Transform
    transformed = []
    for batch in chunk(records, 1000):
        result = transform_batch(batch)
        transformed.extend(result)
    
    # Load
    load_to_warehouse(transformed)
    
    # Checkpoint
    mark_batch_complete(date)

# Schedule
def run_batch_pipeline():
    yesterday = date.today() - timedelta(days=1)
    
    # Check if already processed
    if is_batch_complete(yesterday):
        logger.info(f"Batch for {yesterday} already complete")
        return
    
    # Process
    process_daily_batch(yesterday)
```

## Streaming Processing Pattern

```python
def process_order_stream():
    """
    Standard streaming pattern:
    1. Consume from stream
    2. Transform per-record or micro-batch
    3. Write to sink
    4. Commit offset
    """
    
    consumer = KafkaConsumer('orders', enable_auto_commit=False)
    
    for message in consumer:
        try:
            # Transform
            order = transform_order(json.loads(message.value))
            
            # Sink
            write_to_database(order)
            
            # Commit
            consumer.commit()
            
        except Exception as e:
            # Dead letter queue
            send_to_dlq(message, error=str(e))
            consumer.commit()  # Skip bad message

# Run continuously
while True:
    try:
        process_order_stream()
    except Exception as e:
        logger.error(f"Stream processor crashed: {e}")
        time.sleep(10)  # Backoff before restart
```

## Converting Batch to Stream

Incrementalize batch jobs:

```python
# Batch: Full recomputation
def compute_daily_metrics_batch(date):
    orders = db.query("""
        SELECT * FROM orders WHERE DATE(created_at) = ?
    """, date)
    
    metrics = {
        "total_orders": len(orders),
        "total_revenue": sum(o.amount for o in orders),
        "avg_order_size": sum(o.amount for o in orders) / len(orders)
    }
    
    db.upsert("daily_metrics", date=date, **metrics)

# Stream: Incremental updates
def compute_daily_metrics_stream():
    consumer = KafkaConsumer('orders')
    
    for message in consumer:
        order = json.loads(message.value)
        date = order['created_at'].date()
        
        # Increment counters
        db.execute("""
            INSERT INTO daily_metrics (date, total_orders, total_revenue)
            VALUES (?, 1, ?)
            ON CONFLICT (date) DO UPDATE SET
                total_orders = daily_metrics.total_orders + 1,
                total_revenue = daily_metrics.total_revenue + EXCLUDED.total_revenue
        """, date, order['amount'])
        
        # Recompute average
        metrics = db.query_one("""
            SELECT total_revenue, total_orders FROM daily_metrics
            WHERE date = ?
        """, date)
        
        avg = metrics.total_revenue / metrics.total_orders
        db.execute("""
            UPDATE daily_metrics SET avg_order_size = ? WHERE date = ?
        """, avg, date)
```

## Why It's a Problem

1. **Over-engineering**: Using streaming when batch suffices increases complexity
2. **Unmet requirements**: Using batch when streaming needed misses SLAs
3. **Cost overruns**: Streaming is more expensive than batch
4. **Maintenance burden**: Wrong choice makes operations harder

## Symptoms

- Batch jobs can't meet latency requirements
- Streaming pipelines that could be simple batch jobs
- High infrastructure costs for low-value real-time processing
- Complex stateful streaming for use cases that don't require it

## Benefits

- **Right-sized solution**: Match complexity to requirements
- **Cost optimization**: Use batch where possible, stream where necessary
- **Operational simplicity**: Simpler systems are easier to maintain
- **Meet SLAs**: Choose model that satisfies latency requirements

## Decision Framework

Ask these questions:

1. **What's the maximum acceptable latency?**
   - Seconds → Streaming
   - Minutes/Hours → Batch or Micro-batch
   - Days → Batch

2. **Is the data bounded or unbounded?**
   - Bounded → Batch
   - Unbounded → Streaming or Micro-batch

3. **Do you need to reprocess historical data?**
   - Yes → Batch (or Lambda)
   - No → Streaming

4. **What's the complexity budget?**
   - Low → Batch
   - High → Streaming acceptable

5. **What's the cost constraint?**
   - Tight → Batch
   - Flexible → Streaming acceptable

## See Also

- [Lambda Architecture](./lambda-architecture.md)
- [Kappa Architecture](./kappa-architecture.md)
- [Checkpointing and Recovery](./checkpointing-recovery.md)
- [State Management](./state-management.md)
