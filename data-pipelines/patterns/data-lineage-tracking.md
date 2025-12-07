# Data Lineage Tracking

> Track data provenance from source through all transformations to destination—enable debugging and compliance auditing.

## Problem

When data issues arise, you need to trace where data came from, what transformations were applied, and where it went. Without lineage tracking, debugging requires manual log diving across multiple systems. Compliance requirements (GDPR, HIPAA) often mandate tracking data flow for auditing purposes.

## Example

### ❌ Before (No Lineage)

```python
def process_order(order_data):
    # Transform data
    enriched = enrich_with_customer(order_data)
    normalized = normalize_format(enriched)
    
    # Load to destination
    database.insert("orders", normalized)
    
    # ❌ No trace of:
    # - Where did order_data come from?
    # - When was it processed?
    # - What transformations were applied?
    # - Who/what triggered this?

# When data is wrong:
# - Which source file/message contained this?
# - What was the original value before transformation?
# - When did this record get processed?
# - Was it reprocessed? How many times?
```

### ✅ After (With Lineage)

```python
from dataclasses import dataclass
from typing import List
import uuid

@dataclass
class LineageInfo:
    record_id: str
    source_system: str
    source_identifier: str
    ingestion_timestamp: datetime
    transformations: List[str]
    destination_system: str
    processed_by: str
    correlation_id: str

def process_order(order_data, source_metadata):
    # Generate correlation ID
    correlation_id = str(uuid.uuid4())
    
    lineage = LineageInfo(
        record_id=order_data["order_id"],
        source_system=source_metadata["system"],
        source_identifier=source_metadata["message_id"],
        ingestion_timestamp=datetime.now(),
        transformations=[],
        destination_system="analytics_db",
        processed_by="order-processor-v2.1",
        correlation_id=correlation_id
    )
    
    # Track each transformation
    enriched = enrich_with_customer(order_data)
    lineage.transformations.append("enrich_with_customer")
    
    normalized = normalize_format(enriched)
    lineage.transformations.append("normalize_format")
    
    # Store with lineage
    database.insert("orders", {
        **normalized,
        "_lineage": lineage.__dict__
    })
    
    # Also store in lineage table for queries
    store_lineage(lineage)

def store_lineage(lineage: LineageInfo):
    database.insert("data_lineage", {
        "record_id": lineage.record_id,
        "source_system": lineage.source_system,
        "source_identifier": lineage.source_identifier,
        "ingestion_timestamp": lineage.ingestion_timestamp,
        "transformations": json.dumps(lineage.transformations),
        "destination_system": lineage.destination_system,
        "processed_by": lineage.processed_by,
        "correlation_id": lineage.correlation_id
    })
```

## Lineage Schema

### Database Design

```sql
-- Core lineage table
CREATE TABLE data_lineage (
    id SERIAL PRIMARY KEY,
    record_id VARCHAR(255) NOT NULL,
    correlation_id UUID NOT NULL,
    
    -- Source information
    source_system VARCHAR(255) NOT NULL,
    source_identifier VARCHAR(255) NOT NULL,
    source_timestamp TIMESTAMP,
    
    -- Processing information
    ingestion_timestamp TIMESTAMP NOT NULL,
    processing_timestamp TIMESTAMP NOT NULL,
    processed_by VARCHAR(255) NOT NULL,
    pipeline_version VARCHAR(50),
    
    -- Transformation tracking
    transformations JSONB,
    
    -- Destination information
    destination_system VARCHAR(255) NOT NULL,
    destination_identifier VARCHAR(255),
    
    -- Metadata
    metadata JSONB,
    
    INDEX idx_record_id (record_id),
    INDEX idx_correlation_id (correlation_id),
    INDEX idx_source_system (source_system),
    INDEX idx_ingestion_timestamp (ingestion_timestamp)
);

-- Transformation details
CREATE TABLE transformation_log (
    id SERIAL PRIMARY KEY,
    lineage_id INT NOT NULL,
    step_order INT NOT NULL,
    transformation_name VARCHAR(255) NOT NULL,
    input_schema VARCHAR(50),
    output_schema VARCHAR(50),
    duration_ms INT,
    records_in INT,
    records_out INT,
    
    FOREIGN KEY (lineage_id) REFERENCES data_lineage(id)
);

-- Field-level lineage (for sensitive data)
CREATE TABLE field_lineage (
    id SERIAL PRIMARY KEY,
    lineage_id INT NOT NULL,
    field_name VARCHAR(255) NOT NULL,
    source_field VARCHAR(255),
    transformation_applied VARCHAR(255),
    was_encrypted BOOLEAN DEFAULT FALSE,
    was_masked BOOLEAN DEFAULT FALSE,
    
    FOREIGN KEY (lineage_id) REFERENCES data_lineage(id)
);
```

## Propagating Lineage Through Pipeline

### Multi-Stage Pipeline

```python
class LineageContext:
    def __init__(self, source_system, source_id):
        self.record_id = None
        self.correlation_id = str(uuid.uuid4())
        self.source_system = source_system
        self.source_id = source_id
        self.ingestion_time = datetime.now()
        self.transformations = []
        self.current_stage = None
    
    def start_transformation(self, name):
        self.current_stage = {
            "name": name,
            "start_time": datetime.now()
        }
    
    def end_transformation(self):
        if self.current_stage:
            self.current_stage["end_time"] = datetime.now()
            self.current_stage["duration_ms"] = (
                (self.current_stage["end_time"] - 
                 self.current_stage["start_time"]).total_seconds() * 1000
            )
            self.transformations.append(self.current_stage)
            self.current_stage = None

# Stage 1: Ingestion
def ingest_from_kafka(message):
    lineage = LineageContext(
        source_system="kafka",
        source_id=f"{message.topic}:{message.partition}:{message.offset}"
    )
    
    data = json.loads(message.value)
    lineage.record_id = data["order_id"]
    
    # Pass lineage with data
    return data, lineage

# Stage 2: Enrichment
def enrich_order(order, lineage):
    lineage.start_transformation("customer_enrichment")
    
    customer = customer_service.get(order["customer_id"])
    enriched = {**order, "customer_name": customer.name}
    
    lineage.end_transformation()
    return enriched, lineage

# Stage 3: Normalization
def normalize_order(order, lineage):
    lineage.start_transformation("format_normalization")
    
    normalized = {
        "id": order["order_id"],
        "customer": order["customer_id"],
        "amount": float(order["total"])
    }
    
    lineage.end_transformation()
    return normalized, lineage

# Stage 4: Storage
def store_order(order, lineage):
    lineage.start_transformation("database_write")
    
    database.insert("orders", order)
    store_lineage(lineage)
    
    lineage.end_transformation()

# Full pipeline
def process_message(message):
    data, lineage = ingest_from_kafka(message)
    data, lineage = enrich_order(data, lineage)
    data, lineage = normalize_order(data, lineage)
    store_order(data, lineage)
```

## Lineage Queries

### Find Source of Data

```python
def trace_to_source(record_id):
    """Find original source of a record."""
    lineage = database.query_one("""
        SELECT source_system, source_identifier, source_timestamp
        FROM data_lineage
        WHERE record_id = ?
        ORDER BY ingestion_timestamp DESC
        LIMIT 1
    """, record_id)
    
    return lineage

# Usage
source = trace_to_source("ORD-12345")
print(f"Source: {source.source_system}")
print(f"Original ID: {source.source_identifier}")
print(f"Created: {source.source_timestamp}")
```

### Find All Transformations

```python
def get_transformation_history(record_id):
    """Get all transformations applied to a record."""
    lineage = database.query_one("""
        SELECT transformations, correlation_id
        FROM data_lineage
        WHERE record_id = ?
        ORDER BY ingestion_timestamp DESC
        LIMIT 1
    """, record_id)
    
    return json.loads(lineage.transformations)

# Usage
transforms = get_transformation_history("ORD-12345")
for t in transforms:
    print(f"{t['name']}: {t['duration_ms']}ms")
```

### Find Downstream Dependencies

```python
def find_downstream(source_record_id):
    """Find all records derived from a source record."""
    # Using correlation_id to track related records
    lineage = database.query_one("""
        SELECT correlation_id FROM data_lineage
        WHERE record_id = ?
    """, source_record_id)
    
    derived = database.query("""
        SELECT record_id, destination_system, processing_timestamp
        FROM data_lineage
        WHERE correlation_id = ?
        AND record_id != ?
        ORDER BY processing_timestamp
    """, lineage.correlation_id, source_record_id)
    
    return derived

# Usage: Find all records derived from original order
derived = find_downstream("ORD-12345")
for record in derived:
    print(f"{record.destination_system}: {record.record_id}")
```

### Compliance: Data Deletion

```python
def delete_customer_data(customer_id):
    """Delete all data for a customer (GDPR right to erasure)."""
    
    # Find all records for customer
    records = database.query("""
        SELECT DISTINCT record_id, destination_system
        FROM data_lineage
        WHERE metadata->>'customer_id' = ?
    """, customer_id)
    
    deleted = []
    for record in records:
        # Delete from destination
        if record.destination_system == "analytics_db":
            database.execute(
                "DELETE FROM orders WHERE order_id = ?",
                record.record_id
            )
        elif record.destination_system == "data_warehouse":
            warehouse.execute(
                "DELETE FROM fact_orders WHERE order_id = ?",
                record.record_id
            )
        
        # Record deletion in lineage
        database.execute("""
            INSERT INTO deletion_log 
            (record_id, deleted_at, reason)
            VALUES (?, ?, ?)
        """, record.record_id, datetime.now(), "GDPR deletion request")
        
        deleted.append(record.record_id)
    
    return deleted
```

## Column-Level Lineage

Track lineage at field level for sensitive data:

```python
class FieldLineage:
    def __init__(self):
        self.field_mappings = []
    
    def map_field(self, source_field, dest_field, transformation=None):
        self.field_mappings.append({
            "source": source_field,
            "destination": dest_field,
            "transformation": transformation
        })

def transform_with_field_lineage(order):
    lineage = FieldLineage()
    
    result = {}
    
    # Direct mapping
    result["id"] = order["order_id"]
    lineage.map_field("order_id", "id", "direct")
    
    # Transformation
    result["amount_cents"] = int(float(order["total"]) * 100)
    lineage.map_field("total", "amount_cents", "dollars_to_cents")
    
    # Derivation
    result["tax"] = float(order["total"]) * 0.08
    lineage.map_field("total", "tax", "calculate_tax_8pct")
    
    # PII handling
    result["email_hash"] = hashlib.sha256(order["email"].encode()).hexdigest()
    lineage.map_field("email", "email_hash", "sha256_hash")
    
    return result, lineage
```

## Visualization and Tools

### Lineage Diagram Generation

```python
def generate_lineage_diagram(record_id):
    """Generate visual lineage diagram."""
    lineage = database.query_one("""
        SELECT * FROM data_lineage WHERE record_id = ?
    """, record_id)
    
    transformations = json.loads(lineage.transformations)
    
    # Generate Mermaid diagram
    diagram = ["graph LR"]
    diagram.append(f"    Source[{lineage.source_system}]")
    
    prev = "Source"
    for i, t in enumerate(transformations):
        node = f"T{i}[{t['name']}]"
        diagram.append(f"    {prev} --> {node}")
        prev = node
    
    diagram.append(f"    {prev} --> Dest[{lineage.destination_system}]")
    
    return "\n".join(diagram)

# Output:
# graph LR
#     Source[kafka] --> T0[customer_enrichment]
#     T0 --> T1[format_normalization]
#     T1 --> Dest[analytics_db]
```

## Why It's a Problem

1. **Debugging difficulty**: Can't trace data issues to source
2. **Compliance risk**: Can't prove data handling for audits
3. **Impact analysis**: Can't identify downstream effects of changes
4. **Data quality**: Can't analyze where corruption occurs

## Symptoms

- Hours spent manually tracing data through logs
- Unable to answer "where did this data come from?"
- Compliance audit failures
- Can't identify root cause of data quality issues

## Benefits

- **Fast debugging**: Trace any record to its source instantly
- **Compliance**: Prove data handling for regulations
- **Impact analysis**: Know what will break before making changes
- **Quality monitoring**: Identify transformation stages that introduce errors

## Best Practices

1. **Capture at ingestion**: Start tracking as soon as data enters
2. **Correlation IDs**: Use UUIDs to link related records
3. **Transformation logs**: Record every transformation applied
4. **Metadata**: Include version, timestamp, processing host
5. **Query performance**: Index lineage tables for fast lookups

## See Also

- [Correlation IDs](./correlation-ids.md)
- [Observability Patterns](./observability-patterns.md)
- [Audit Logging](./audit-logging.md)
- [Data Quality Monitoring](./data-quality-monitoring.md)
