# Streaming Pipeline Examples

This guide provides comprehensive examples of real-time streaming data pipelines using Kafka and the AI-ETL platform. Each example includes complete working code, configuration, and best practices.

## Table of Contents

- [Overview](#overview)
- [Example 1: Real-time Event Processing (Kafka)](#example-1-real-time-event-processing-kafka)
- [Example 2: CDC Streaming with Debezium](#example-2-cdc-streaming-with-debezium)
- [Example 3: Stream Aggregations](#example-3-stream-aggregations)
- [Example 4: Stream Joins and Enrichment](#example-4-stream-joins-and-enrichment)
- [Kafka Configuration and Setup](#kafka-configuration-and-setup)
- [State Management](#state-management)
- [Exactly-Once Semantics](#exactly-once-semantics)
- [Performance Tuning](#performance-tuning)

## Overview

Streaming patterns supported by AI-ETL:

| Pattern | Use Case | Latency | Complexity |
|---------|----------|---------|------------|
| **Event Streaming** | Click tracking, IoT sensors | < 1 sec | Low |
| **CDC Streaming** | Database replication | < 5 sec | Medium |
| **Stream Aggregations** | Real-time analytics | 1-60 sec | Medium |
| **Stream Joins** | Event enrichment | 1-10 sec | High |
| **Stream-to-Batch** | Lambda architecture | Minutes | Medium |

## Example 1: Real-time Event Processing (Kafka)

### Use Case
Process user clickstream events in real-time, enrich with user data, and store in ClickHouse for analytics.

### Natural Language Request
```
"Process clickstream events from Kafka topic 'user-events',
enrich with user profiles from Redis cache,
calculate session metrics, and store in ClickHouse in real-time."
```

### Configuration

```python
# Configure streaming pipeline via API
import requests

response = requests.post(
    'http://localhost:8000/api/v1/streaming/create',
    headers={'Authorization': 'Bearer YOUR_TOKEN'},
    json={
        'pipeline_name': 'clickstream_processing',
        'type': 'kafka_consumer',
        'config': {
            'bootstrap_servers': 'localhost:9092',
            'topics': ['user-events'],
            'group_id': 'clickstream-processor',
            'auto_offset_reset': 'latest',
            'enable_auto_commit': False,
            'max_poll_records': 500
        },
        'processing': {
            'enrichment': {
                'type': 'redis_lookup',
                'key_field': 'user_id',
                'cache_prefix': 'user:profile:'
            },
            'aggregation': {
                'window_size_seconds': 60,
                'metrics': ['event_count', 'unique_users', 'session_duration']
            }
        },
        'output': {
            'type': 'clickhouse',
            'table': 'clickstream_events',
            'batch_size': 1000,
            'flush_interval_ms': 5000
        }
    }
)
```

### Generated Streaming Application

```python
# backend/streaming/clickstream_processor.py

import asyncio
import json
from typing import Dict, Any, List
from datetime import datetime, timedelta
from aiokafka import AIOKafkaConsumer, AIOKafkaProducer
from aiokafka.errors import KafkaError
import clickhouse_connect
import redis.asyncio as aioredis
from collections import defaultdict
import logging

logger = logging.getLogger(__name__)


class ClickstreamProcessor:
    """Real-time clickstream event processor."""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.consumer = None
        self.producer = None
        self.redis_client = None
        self.clickhouse_client = None
        self.buffer = []
        self.buffer_lock = asyncio.Lock()
        self.running = False

        # Session tracking
        self.sessions = defaultdict(lambda: {
            'start_time': None,
            'last_event_time': None,
            'event_count': 0,
            'pages_viewed': set()
        })

    async def initialize(self):
        """Initialize connections."""

        # Kafka consumer
        self.consumer = AIOKafkaConsumer(
            *self.config['topics'],
            bootstrap_servers=self.config['bootstrap_servers'],
            group_id=self.config['group_id'],
            auto_offset_reset=self.config.get('auto_offset_reset', 'latest'),
            enable_auto_commit=False,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            max_poll_records=self.config.get('max_poll_records', 500),
            session_timeout_ms=30000,
            heartbeat_interval_ms=10000
        )

        await self.consumer.start()
        logger.info(f"Kafka consumer started, subscribed to: {self.config['topics']}")

        # Redis for enrichment
        self.redis_client = await aioredis.from_url(
            'redis://localhost:6379',
            encoding='utf-8',
            decode_responses=True
        )

        # ClickHouse for output
        self.clickhouse_client = clickhouse_connect.get_client(
            host='localhost',
            port=8123,
            database='analytics'
        )

        # Create output table
        await self._create_output_table()

        logger.info("Clickstream processor initialized")

    async def _create_output_table(self):
        """Create ClickHouse table for events."""

        create_table_sql = """
        CREATE TABLE IF NOT EXISTS clickstream_events (
            event_id String,
            event_type String,
            user_id UInt64,
            session_id String,
            timestamp DateTime64(3),
            page_url String,
            referrer String,
            user_agent String,
            ip_address String,
            country String,
            city String,
            -- Enriched user data
            user_email String,
            user_segment String,
            user_signup_date Date,
            -- Session metrics
            session_duration_seconds UInt32,
            session_event_count UInt32,
            session_pages_viewed UInt32,
            -- Event properties
            properties String,  -- JSON
            processed_at DateTime DEFAULT now()
        ) ENGINE = MergeTree()
        PARTITION BY toYYYYMM(timestamp)
        ORDER BY (timestamp, user_id, event_id)
        SETTINGS index_granularity = 8192
        """

        self.clickhouse_client.command(create_table_sql)

    async def start(self):
        """Start processing events."""

        self.running = True

        # Start background tasks
        flush_task = asyncio.create_task(self._periodic_flush())

        try:
            async for message in self.consumer:
                if not self.running:
                    break

                try:
                    # Process event
                    event = message.value
                    processed_event = await self._process_event(event)

                    # Add to buffer
                    async with self.buffer_lock:
                        self.buffer.append(processed_event)

                        # Flush if buffer is full
                        if len(self.buffer) >= self.config.get('batch_size', 1000):
                            await self._flush_buffer()

                    # Commit offset
                    await self.consumer.commit()

                except Exception as e:
                    logger.error(f"Error processing event: {str(e)}", exc_info=True)

        finally:
            flush_task.cancel()
            await self.stop()

    async def _process_event(self, event: Dict[str, Any]) -> Dict[str, Any]:
        """Process individual event with enrichment."""

        # Extract fields
        user_id = event.get('user_id')
        session_id = event.get('session_id')
        event_timestamp = datetime.fromisoformat(event.get('timestamp'))

        # Enrich with user profile from Redis
        user_profile = await self._get_user_profile(user_id)

        # Update session tracking
        session_metrics = self._update_session(session_id, event_timestamp, event)

        # Build processed event
        processed_event = {
            'event_id': event.get('event_id'),
            'event_type': event.get('event_type'),
            'user_id': user_id,
            'session_id': session_id,
            'timestamp': event_timestamp,
            'page_url': event.get('page_url', ''),
            'referrer': event.get('referrer', ''),
            'user_agent': event.get('user_agent', ''),
            'ip_address': event.get('ip_address', ''),
            'country': event.get('country', ''),
            'city': event.get('city', ''),
            # Enriched data
            'user_email': user_profile.get('email', ''),
            'user_segment': user_profile.get('segment', 'unknown'),
            'user_signup_date': user_profile.get('signup_date'),
            # Session metrics
            'session_duration_seconds': session_metrics['duration_seconds'],
            'session_event_count': session_metrics['event_count'],
            'session_pages_viewed': session_metrics['pages_viewed'],
            # Properties as JSON
            'properties': json.dumps(event.get('properties', {}))
        }

        return processed_event

    async def _get_user_profile(self, user_id: int) -> Dict[str, Any]:
        """Get user profile from Redis cache."""

        cache_key = f"user:profile:{user_id}"

        try:
            cached = await self.redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
        except Exception as e:
            logger.warning(f"Redis lookup failed for user {user_id}: {str(e)}")

        return {}

    def _update_session(
        self,
        session_id: str,
        event_time: datetime,
        event: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Update session tracking and return metrics."""

        session = self.sessions[session_id]

        if session['start_time'] is None:
            session['start_time'] = event_time

        session['last_event_time'] = event_time
        session['event_count'] += 1

        # Track unique pages
        if event.get('event_type') == 'page_view':
            session['pages_viewed'].add(event.get('page_url', ''))

        # Calculate duration
        duration = (event_time - session['start_time']).total_seconds()

        return {
            'duration_seconds': int(duration),
            'event_count': session['event_count'],
            'pages_viewed': len(session['pages_viewed'])
        }

    async def _periodic_flush(self):
        """Periodically flush buffer to ClickHouse."""

        flush_interval = self.config.get('flush_interval_ms', 5000) / 1000

        while self.running:
            await asyncio.sleep(flush_interval)

            async with self.buffer_lock:
                if self.buffer:
                    await self._flush_buffer()

    async def _flush_buffer(self):
        """Flush events to ClickHouse."""

        if not self.buffer:
            return

        try:
            # Convert to DataFrame for insertion
            import pandas as pd
            df = pd.DataFrame(self.buffer)

            # Insert to ClickHouse
            self.clickhouse_client.insert_df('clickstream_events', df)

            logger.info(f"Flushed {len(self.buffer)} events to ClickHouse")

            self.buffer = []

        except Exception as e:
            logger.error(f"Error flushing buffer: {str(e)}", exc_info=True)
            raise

    async def stop(self):
        """Stop processing and cleanup."""

        self.running = False

        # Flush remaining events
        async with self.buffer_lock:
            if self.buffer:
                await self._flush_buffer()

        # Close connections
        if self.consumer:
            await self.consumer.stop()

        if self.redis_client:
            await self.redis_client.close()

        logger.info("Clickstream processor stopped")


# FastAPI endpoint to start/stop processor
from fastapi import APIRouter, HTTPException
from backend.services.streaming_service import StreamingService

router = APIRouter(prefix="/api/v1/streaming")

@router.post("/clickstream/start")
async def start_clickstream_processor(config: Dict[str, Any]):
    """Start clickstream processor."""

    try:
        processor = ClickstreamProcessor(config)
        await processor.initialize()

        # Run in background
        asyncio.create_task(processor.start())

        return {"status": "started", "processor": "clickstream"}

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Example 2: CDC Streaming with Debezium

### Use Case
Capture database changes from PostgreSQL in real-time and replicate to ClickHouse using Debezium CDC.

### Debezium Configuration

```json
{
  "name": "postgres-orders-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-server",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "secure_password",
    "database.dbname": "orders_db",
    "database.server.name": "orders",
    "table.include.list": "public.orders,public.order_items",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_orders_slot",
    "publication.name": "debezium_orders_pub",
    "publication.autocreate.mode": "filtered",
    "tombstones.on.delete": "false",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false"
  }
}
```

### CDC Stream Consumer

```python
# backend/streaming/debezium_cdc_consumer.py

import asyncio
import json
from typing import Dict, Any
from aiokafka import AIOKafkaConsumer
import clickhouse_connect
from datetime import datetime
import logging

logger = logging.getLogger(__name__)


class DebeziumCDCConsumer:
    """Consumer for Debezium CDC events."""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.consumer = None
        self.clickhouse_client = None
        self.running = False

        # CDC operation types
        self.op_types = {
            'c': 'INSERT',  # Create
            'u': 'UPDATE',  # Update
            'd': 'DELETE',  # Delete
            'r': 'READ'     # Snapshot read
        }

    async def initialize(self):
        """Initialize Kafka consumer and ClickHouse client."""

        # Kafka consumer for CDC topics
        self.consumer = AIOKafkaConsumer(
            'orders.public.orders',
            'orders.public.order_items',
            bootstrap_servers=self.config['bootstrap_servers'],
            group_id='cdc-clickhouse-replicator',
            auto_offset_reset='earliest',
            enable_auto_commit=False,
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )

        await self.consumer.start()

        # ClickHouse client
        self.clickhouse_client = clickhouse_connect.get_client(
            host='localhost',
            port=8123,
            database='replica'
        )

        # Create replica tables
        await self._create_replica_tables()

        logger.info("Debezium CDC consumer initialized")

    async def _create_replica_tables(self):
        """Create replica tables in ClickHouse."""

        # Orders table with ReplacingMergeTree for updates
        self.clickhouse_client.command("""
            CREATE TABLE IF NOT EXISTS orders (
                order_id UInt64,
                customer_id UInt64,
                order_date Date,
                status String,
                total_amount Decimal(18, 2),
                created_at DateTime,
                updated_at DateTime,
                _version UInt64,
                _deleted UInt8 DEFAULT 0
            ) ENGINE = ReplacingMergeTree(_version)
            PARTITION BY toYYYYMM(order_date)
            ORDER BY (order_id)
        """)

        # Order items table
        self.clickhouse_client.command("""
            CREATE TABLE IF NOT EXISTS order_items (
                item_id UInt64,
                order_id UInt64,
                product_id UInt64,
                quantity UInt32,
                price Decimal(18, 2),
                created_at DateTime,
                updated_at DateTime,
                _version UInt64,
                _deleted UInt8 DEFAULT 0
            ) ENGINE = ReplacingMergeTree(_version)
            PARTITION BY order_id % 100
            ORDER BY (item_id)
        """)

    async def start(self):
        """Start consuming CDC events."""

        self.running = True

        try:
            async for message in self.consumer:
                if not self.running:
                    break

                try:
                    # Process CDC event
                    await self._process_cdc_event(message)

                    # Commit offset
                    await self.consumer.commit()

                except Exception as e:
                    logger.error(f"Error processing CDC event: {str(e)}", exc_info=True)

        finally:
            await self.stop()

    async def _process_cdc_event(self, message):
        """Process individual CDC event."""

        topic = message.topic
        event = message.value

        # Debezium event structure
        payload = event.get('payload', {})
        before = payload.get('before')
        after = payload.get('after')
        op = payload.get('op')  # c, u, d, r

        operation = self.op_types.get(op, 'UNKNOWN')

        # Determine table from topic
        table_name = topic.split('.')[-1]  # orders.public.orders -> orders

        logger.info(f"CDC Event: {operation} on {table_name}")

        if operation == 'INSERT' or operation == 'READ':
            await self._handle_insert(table_name, after)

        elif operation == 'UPDATE':
            await self._handle_update(table_name, before, after)

        elif operation == 'DELETE':
            await self._handle_delete(table_name, before)

    async def _handle_insert(self, table: str, record: Dict[str, Any]):
        """Handle INSERT operation."""

        if not record:
            return

        # Add version for ReplacingMergeTree
        record['_version'] = int(datetime.utcnow().timestamp() * 1000000)
        record['_deleted'] = 0

        # Insert to ClickHouse
        import pandas as pd
        df = pd.DataFrame([record])
        self.clickhouse_client.insert_df(table, df)

        logger.debug(f"Inserted record to {table}")

    async def _handle_update(self, table: str, before: Dict[str, Any], after: Dict[str, Any]):
        """Handle UPDATE operation."""

        if not after:
            return

        # Add new version
        after['_version'] = int(datetime.utcnow().timestamp() * 1000000)
        after['_deleted'] = 0

        # Insert new version (ReplacingMergeTree will handle deduplication)
        import pandas as pd
        df = pd.DataFrame([after])
        self.clickhouse_client.insert_df(table, df)

        logger.debug(f"Updated record in {table}")

    async def _handle_delete(self, table: str, record: Dict[str, Any]):
        """Handle DELETE operation."""

        if not record:
            return

        # Soft delete: insert tombstone record
        record['_version'] = int(datetime.utcnow().timestamp() * 1000000)
        record['_deleted'] = 1

        import pandas as pd
        df = pd.DataFrame([record])
        self.clickhouse_client.insert_df(table, df)

        logger.debug(f"Deleted record from {table}")

    async def stop(self):
        """Stop consumer."""

        self.running = False

        if self.consumer:
            await self.consumer.stop()

        logger.info("Debezium CDC consumer stopped")
```

## Example 3: Stream Aggregations

### Use Case
Real-time aggregation of sales metrics with tumbling windows (1-minute windows).

### Stream Aggregation Processor

```python
# backend/streaming/sales_aggregator.py

import asyncio
import json
from typing import Dict, Any, List
from datetime import datetime, timedelta
from aiokafka import AIOKafkaConsumer, AIOKafkaProducer
from collections import defaultdict
import clickhouse_connect
import logging

logger = logging.getLogger(__name__)


class SalesAggregator:
    """Real-time sales metrics aggregator."""

    def __init__(self, window_size_seconds: int = 60):
        self.window_size = window_size_seconds
        self.consumer = None
        self.producer = None
        self.clickhouse_client = None
        self.running = False

        # Window state
        self.current_window_start = None
        self.window_data = defaultdict(lambda: {
            'total_sales': 0,
            'total_revenue': 0,
            'unique_customers': set(),
            'unique_products': set(),
            'transactions': []
        })

    async def initialize(self):
        """Initialize connections."""

        # Consumer for sales events
        self.consumer = AIOKafkaConsumer(
            'sales-events',
            bootstrap_servers='localhost:9092',
            group_id='sales-aggregator',
            auto_offset_reset='latest',
            enable_auto_commit=False,
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )

        await self.consumer.start()

        # Producer for aggregated metrics
        self.producer = AIOKafkaProducer(
            bootstrap_servers='localhost:9092',
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

        await self.producer.start()

        # ClickHouse for persistent storage
        self.clickhouse_client = clickhouse_connect.get_client(
            host='localhost',
            database='analytics'
        )

        # Create aggregation table
        self.clickhouse_client.command("""
            CREATE TABLE IF NOT EXISTS sales_aggregates_1min (
                window_start DateTime,
                window_end DateTime,
                category String,
                total_sales UInt64,
                total_revenue Decimal(18, 2),
                unique_customers UInt64,
                unique_products UInt64,
                avg_transaction_value Decimal(18, 2),
                processed_at DateTime DEFAULT now()
            ) ENGINE = MergeTree()
            PARTITION BY toYYYYMM(window_start)
            ORDER BY (window_start, category)
        """)

        logger.info("Sales aggregator initialized")

    async def start(self):
        """Start aggregating sales events."""

        self.running = True
        self.current_window_start = self._get_window_start(datetime.utcnow())

        # Start window timer task
        window_task = asyncio.create_task(self._window_timer())

        try:
            async for message in self.consumer:
                if not self.running:
                    break

                try:
                    event = message.value
                    await self._process_event(event)

                    await self.consumer.commit()

                except Exception as e:
                    logger.error(f"Error processing event: {str(e)}", exc_info=True)

        finally:
            window_task.cancel()
            await self.stop()

    def _get_window_start(self, timestamp: datetime) -> datetime:
        """Get window start time for a timestamp."""

        seconds_since_epoch = int(timestamp.timestamp())
        window_start_seconds = (seconds_since_epoch // self.window_size) * self.window_size

        return datetime.fromtimestamp(window_start_seconds)

    async def _process_event(self, event: Dict[str, Any]):
        """Process sales event into current window."""

        event_time = datetime.fromisoformat(event['timestamp'])
        window_start = self._get_window_start(event_time)

        # Check if event belongs to current window
        if window_start != self.current_window_start:
            # Event from different window - trigger early flush if needed
            if window_start < self.current_window_start:
                logger.warning(f"Late event: {event_time} in window {window_start}")
                return  # Discard late events

        # Extract event data
        category = event.get('category', 'unknown')
        customer_id = event.get('customer_id')
        product_id = event.get('product_id')
        amount = event.get('amount', 0)

        # Update window aggregates
        window = self.window_data[category]
        window['total_sales'] += 1
        window['total_revenue'] += amount
        window['unique_customers'].add(customer_id)
        window['unique_products'].add(product_id)
        window['transactions'].append(amount)

    async def _window_timer(self):
        """Timer to trigger window completion."""

        while self.running:
            # Wait until window ends
            window_end = self.current_window_start + timedelta(seconds=self.window_size)
            now = datetime.utcnow()

            if now >= window_end:
                # Window completed
                await self._flush_window()

                # Move to next window
                self.current_window_start = self._get_window_start(now)
                self.window_data.clear()

            # Sleep until next check
            await asyncio.sleep(1)

    async def _flush_window(self):
        """Flush completed window aggregates."""

        if not self.window_data:
            return

        window_end = self.current_window_start + timedelta(seconds=self.window_size)

        aggregates = []

        for category, data in self.window_data.items():
            avg_value = (
                data['total_revenue'] / data['total_sales']
                if data['total_sales'] > 0 else 0
            )

            aggregate = {
                'window_start': self.current_window_start,
                'window_end': window_end,
                'category': category,
                'total_sales': data['total_sales'],
                'total_revenue': float(data['total_revenue']),
                'unique_customers': len(data['unique_customers']),
                'unique_products': len(data['unique_products']),
                'avg_transaction_value': float(avg_value)
            }

            aggregates.append(aggregate)

            # Publish to Kafka topic
            await self.producer.send(
                'sales-aggregates-1min',
                value=aggregate
            )

        # Store in ClickHouse
        import pandas as pd
        df = pd.DataFrame(aggregates)
        self.clickhouse_client.insert_df('sales_aggregates_1min', df)

        logger.info(f"Flushed window {self.current_window_start}: {len(aggregates)} aggregates")

    async def stop(self):
        """Stop aggregator."""

        self.running = False

        # Flush remaining data
        await self._flush_window()

        if self.consumer:
            await self.consumer.stop()

        if self.producer:
            await self.producer.stop()

        logger.info("Sales aggregator stopped")
```

## Example 4: Stream Joins and Enrichment

### Use Case
Join clickstream events with user profile updates in real-time using stream-stream join.

### Stream Join Processor

```python
# backend/streaming/stream_joiner.py

import asyncio
import json
from typing import Dict, Any, Optional
from datetime import datetime, timedelta
from aiokafka import AIOKafkaConsumer, AIOKafkaProducer
from collections import deque
import logging

logger = logging.getLogger(__name__)


class StreamJoiner:
    """Join two Kafka streams with time window."""

    def __init__(self, join_window_seconds: int = 300):
        self.join_window = join_window_seconds
        self.consumer_events = None
        self.consumer_profiles = None
        self.producer = None
        self.running = False

        # Join state - stores recent events/profiles
        self.events_buffer = deque(maxlen=10000)
        self.profiles_buffer = {}  # user_id -> profile

    async def initialize(self):
        """Initialize Kafka consumers and producer."""

        # Consumer for clickstream events
        self.consumer_events = AIOKafkaConsumer(
            'clickstream-events',
            bootstrap_servers='localhost:9092',
            group_id='stream-joiner-events',
            auto_offset_reset='latest',
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )

        # Consumer for profile updates
        self.consumer_profiles = AIOKafkaConsumer(
            'user-profile-updates',
            bootstrap_servers='localhost:9092',
            group_id='stream-joiner-profiles',
            auto_offset_reset='latest',
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )

        # Producer for joined stream
        self.producer = AIOKafkaProducer(
            bootstrap_servers='localhost:9092',
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

        await self.consumer_events.start()
        await self.consumer_profiles.start()
        await self.producer.start()

        logger.info("Stream joiner initialized")

    async def start(self):
        """Start joining streams."""

        self.running = True

        # Run consumers in parallel
        events_task = asyncio.create_task(self._consume_events())
        profiles_task = asyncio.create_task(self._consume_profiles())
        cleanup_task = asyncio.create_task(self._cleanup_old_data())

        try:
            await asyncio.gather(events_task, profiles_task, cleanup_task)
        finally:
            await self.stop()

    async def _consume_events(self):
        """Consume clickstream events."""

        async for message in self.consumer_events:
            if not self.running:
                break

            try:
                event = message.value
                event['event_timestamp'] = datetime.fromisoformat(event['timestamp'])

                # Try to join with profile
                user_id = event.get('user_id')
                joined_event = await self._join_event_with_profile(event, user_id)

                if joined_event:
                    # Publish joined event
                    await self.producer.send(
                        'enriched-clickstream',
                        value=joined_event
                    )
                else:
                    # Store event for later join
                    self.events_buffer.append(event)

            except Exception as e:
                logger.error(f"Error processing event: {str(e)}", exc_info=True)

    async def _consume_profiles(self):
        """Consume profile updates."""

        async for message in self.consumer_profiles:
            if not self.running:
                break

            try:
                profile = message.value
                user_id = profile.get('user_id')
                profile['profile_timestamp'] = datetime.fromisoformat(profile['timestamp'])

                # Update profile buffer
                self.profiles_buffer[user_id] = profile

                # Try to join with buffered events
                await self._join_profile_with_events(profile, user_id)

            except Exception as e:
                logger.error(f"Error processing profile: {str(e)}", exc_info=True)

    async def _join_event_with_profile(
        self,
        event: Dict[str, Any],
        user_id: int
    ) -> Optional[Dict[str, Any]]:
        """Join event with most recent profile."""

        profile = self.profiles_buffer.get(user_id)

        if not profile:
            return None

        # Check if profile is within join window
        time_diff = abs((event['event_timestamp'] - profile['profile_timestamp']).total_seconds())

        if time_diff > self.join_window:
            return None

        # Create joined record
        joined = {
            **event,
            'user_email': profile.get('email'),
            'user_name': profile.get('name'),
            'user_segment': profile.get('segment'),
            'user_country': profile.get('country'),
            'profile_updated_at': profile['timestamp'],
            'join_timestamp': datetime.utcnow().isoformat()
        }

        return joined

    async def _join_profile_with_events(self, profile: Dict[str, Any], user_id: int):
        """Join profile with buffered events."""

        # Find matching events in buffer
        matched_indices = []

        for i, event in enumerate(self.events_buffer):
            if event.get('user_id') == user_id:
                # Check time window
                time_diff = abs((event['event_timestamp'] - profile['profile_timestamp']).total_seconds())

                if time_diff <= self.join_window:
                    # Create joined record
                    joined = {
                        **event,
                        'user_email': profile.get('email'),
                        'user_name': profile.get('name'),
                        'user_segment': profile.get('segment'),
                        'user_country': profile.get('country'),
                        'profile_updated_at': profile['timestamp'],
                        'join_timestamp': datetime.utcnow().isoformat()
                    }

                    # Publish joined event
                    await self.producer.send(
                        'enriched-clickstream',
                        value=joined
                    )

                    matched_indices.append(i)

        # Remove matched events from buffer
        for i in reversed(matched_indices):
            del self.events_buffer[i]

    async def _cleanup_old_data(self):
        """Periodically clean up old data from buffers."""

        while self.running:
            await asyncio.sleep(60)  # Cleanup every minute

            now = datetime.utcnow()

            # Clean old events from buffer
            while self.events_buffer:
                event = self.events_buffer[0]
                age = (now - event['event_timestamp']).total_seconds()

                if age > self.join_window:
                    self.events_buffer.popleft()
                else:
                    break

            # Clean old profiles
            expired_users = []
            for user_id, profile in self.profiles_buffer.items():
                age = (now - profile['profile_timestamp']).total_seconds()
                if age > self.join_window * 2:  # Keep profiles longer
                    expired_users.append(user_id)

            for user_id in expired_users:
                del self.profiles_buffer[user_id]

            logger.debug(f"Cleanup: {len(self.events_buffer)} events, {len(self.profiles_buffer)} profiles buffered")

    async def stop(self):
        """Stop joiner."""

        self.running = False

        if self.consumer_events:
            await self.consumer_events.stop()

        if self.consumer_profiles:
            await self.consumer_profiles.stop()

        if self.producer:
            await self.producer.stop()

        logger.info("Stream joiner stopped")
```

## Kafka Configuration and Setup

### Topic Configuration

```bash
# Create topics with proper configuration
kafka-topics --create \
  --topic clickstream-events \
  --bootstrap-server localhost:9092 \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=86400000 \
  --config compression.type=lz4 \
  --config min.insync.replicas=2

kafka-topics --create \
  --topic sales-aggregates-1min \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config cleanup.policy=compact
```

### Consumer Configuration

```python
# Optimal consumer configuration
consumer_config = {
    'bootstrap_servers': 'localhost:9092',
    'group_id': 'my-consumer-group',
    'auto_offset_reset': 'earliest',
    'enable_auto_commit': False,  # Manual commit for exactly-once
    'max_poll_records': 500,
    'max_poll_interval_ms': 300000,  # 5 minutes
    'session_timeout_ms': 30000,     # 30 seconds
    'heartbeat_interval_ms': 10000,  # 10 seconds
    'fetch_min_bytes': 1024,
    'fetch_max_wait_ms': 500,
    'compression_type': 'lz4'
}
```

## State Management

### RocksDB State Store

```python
import rocksdict

class StatefulProcessor:
    """Processor with persistent state."""

    def __init__(self, state_dir: str):
        # RocksDB for state storage
        self.state_db = rocksdict.Rdict(state_dir)

    def update_state(self, key: str, value: Any):
        """Update state."""
        self.state_db[key] = json.dumps(value)

    def get_state(self, key: str) -> Optional[Any]:
        """Get state."""
        value = self.state_db.get(key)
        return json.loads(value) if value else None

    def checkpoint(self):
        """Create checkpoint."""
        # RocksDB handles this automatically
        pass
```

## Exactly-Once Semantics

### Transactional Producer

```python
from aiokafka import AIOKafkaProducer

async def exactly_once_processing():
    """Process with exactly-once semantics."""

    producer = AIOKafkaProducer(
        bootstrap_servers='localhost:9092',
        transactional_id='my-processor-1',
        enable_idempotence=True,
        acks='all',
        max_in_flight_requests_per_connection=5
    )

    await producer.start()

    try:
        # Begin transaction
        async with producer.transaction():
            # Process and produce
            result = process_data(data)

            await producer.send(
                'output-topic',
                value=result
            )

            # Commit consumer offset within transaction
            await producer.send_offsets_to_transaction(
                offsets,
                group_id='my-consumer-group'
            )

        # Transaction committed automatically

    finally:
        await producer.stop()
```

## Performance Tuning

### 1. Partitioning Strategy

```python
# Custom partitioner for better distribution
from kafka.partitioner import Partitioner

class UserPartitioner(Partitioner):
    """Partition by user_id for co-located processing."""

    def partition(self, key, all_partitions, available_partitions):
        if key is None:
            return random.choice(available_partitions)

        user_id = int(key)
        return user_id % len(all_partitions)
```

### 2. Batching

```python
# Batch events for better throughput
batch_size = 1000
batch_timeout = 5.0  # seconds

async def batch_processor():
    batch = []
    last_flush = time.time()

    async for event in event_stream:
        batch.append(event)

        if len(batch) >= batch_size or (time.time() - last_flush) >= batch_timeout:
            await process_batch(batch)
            batch = []
            last_flush = time.time()
```

### 3. Parallelization

```python
# Process partitions in parallel
import asyncio

async def parallel_consumer():
    tasks = []

    for partition in range(num_partitions):
        task = asyncio.create_task(
            process_partition(partition)
        )
        tasks.append(task)

    await asyncio.gather(*tasks)
```

## Summary

These examples demonstrate:

1. **Real-time Event Processing** - Clickstream with enrichment
2. **CDC Streaming** - Database replication with Debezium
3. **Stream Aggregations** - Windowed aggregations
4. **Stream Joins** - Time-based stream joining

Key takeaways:
- Use proper Kafka configuration for reliability
- Implement state management for stateful processing
- Enable exactly-once semantics for critical data
- Optimize with batching and parallelization
- Monitor lag and throughput

For more examples, see:
- [ETL Examples](./etl.md)
- [Analytics Examples](./analytics.md)
