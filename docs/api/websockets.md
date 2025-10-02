# WebSocket & Real-Time API Reference

Complete reference for WebSocket connections and real-time pipeline updates.

## Overview

The AI-ETL platform provides real-time updates for pipeline execution, connector status, and system events through WebSocket connections and Server-Sent Events (SSE). This enables monitoring pipeline runs in real-time without polling.

## Base URL

```
Production: wss://api.ai-etl.com/api/v1/ws
Development: ws://localhost:8000/api/v1/ws
```

---

## Connection Methods

### 1. WebSocket (Recommended)

Full bidirectional communication for real-time updates.

### 2. Server-Sent Events (SSE)

Unidirectional server-to-client streaming for simple monitoring scenarios.

### 3. Long Polling (Fallback)

HTTP-based fallback for environments that don't support WebSocket/SSE.

---

## WebSocket Connection

### Authentication

WebSocket connections require JWT authentication via query parameter or initial message:

#### Option 1: Query Parameter (Recommended)

```javascript
const ws = new WebSocket('ws://localhost:8000/api/v1/ws?token=YOUR_JWT_TOKEN');
```

#### Option 2: Initial Message

```javascript
const ws = new WebSocket('ws://localhost:8000/api/v1/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'auth',
    token: 'YOUR_JWT_TOKEN'
  }));
};
```

### Connection Lifecycle

```javascript
const ws = new WebSocket('ws://localhost:8000/api/v1/ws?token=YOUR_JWT_TOKEN');

ws.onopen = (event) => {
  console.log('WebSocket connected');

  // Subscribe to pipeline events
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'pipeline',
    pipeline_id: '850e8400-e29b-41d4-a716-446655440003'
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);

  // Handle different event types
  switch (data.type) {
    case 'pipeline.started':
      console.log('Pipeline started:', data.payload);
      break;
    case 'pipeline.progress':
      console.log('Progress:', data.payload.progress);
      break;
    case 'pipeline.completed':
      console.log('Pipeline completed:', data.payload);
      break;
    case 'pipeline.failed':
      console.error('Pipeline failed:', data.payload);
      break;
  }
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = (event) => {
  console.log('WebSocket closed:', event.code, event.reason);

  // Implement reconnection logic
  if (event.code !== 1000) {
    setTimeout(() => reconnect(), 5000);
  }
};
```

---

## Event Types

### Pipeline Events

#### pipeline.started

Sent when a pipeline run starts execution.

```json
{
  "type": "pipeline.started",
  "timestamp": "2024-01-26T10:00:00Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "pipeline_name": "Daily Sales ETL",
    "triggered_by": "user@example.com",
    "configuration": {
      "date": "2024-01-26"
    },
    "estimated_duration_seconds": 180
  }
}
```

#### pipeline.progress

Sent periodically during pipeline execution with progress updates.

```json
{
  "type": "pipeline.progress",
  "timestamp": "2024-01-26T10:01:30Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "progress": 45,
    "current_task": "transform_data",
    "tasks_completed": 2,
    "tasks_total": 5,
    "elapsed_seconds": 90,
    "estimated_remaining_seconds": 90,
    "stats": {
      "rows_processed": 450000,
      "rows_total": 1000000,
      "throughput_rows_per_sec": 5000
    }
  }
}
```

#### pipeline.task_started

Sent when an individual task within the pipeline starts.

```json
{
  "type": "pipeline.task_started",
  "timestamp": "2024-01-26T10:00:30Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "task_id": "extract_source_data",
    "task_name": "Extract Source Data",
    "task_type": "extract"
  }
}
```

#### pipeline.task_completed

Sent when an individual task completes successfully.

```json
{
  "type": "pipeline.task_completed",
  "timestamp": "2024-01-26T10:01:00Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "task_id": "extract_source_data",
    "task_name": "Extract Source Data",
    "duration_seconds": 30,
    "stats": {
      "rows_extracted": 500000,
      "bytes_processed": 52428800
    }
  }
}
```

#### pipeline.task_failed

Sent when a task fails.

```json
{
  "type": "pipeline.task_failed",
  "timestamp": "2024-01-26T10:01:15Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "task_id": "validate_data",
    "task_name": "Validate Data",
    "error_type": "ValidationError",
    "error_message": "Found 15 records with invalid date format",
    "error_details": {
      "invalid_records": 15,
      "sample_errors": [
        {"row": 1234, "field": "sale_date", "value": "invalid-date"}
      ]
    },
    "retry_attempt": 1,
    "max_retries": 3
  }
}
```

#### pipeline.completed

Sent when pipeline execution completes successfully.

```json
{
  "type": "pipeline.completed",
  "timestamp": "2024-01-26T10:03:00Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "status": "success",
    "duration_seconds": 180,
    "stats": {
      "rows_processed": 1000000,
      "rows_inserted": 995000,
      "rows_updated": 5000,
      "rows_failed": 0,
      "bytes_processed": 104857600,
      "throughput_rows_per_sec": 5555
    },
    "artifacts": {
      "log_url": "s3://logs/run_950e8400.log",
      "metrics_url": "clickhouse://metrics/run_950e8400"
    }
  }
}
```

#### pipeline.failed

Sent when pipeline execution fails.

```json
{
  "type": "pipeline.failed",
  "timestamp": "2024-01-26T10:02:30Z",
  "channel": "pipeline",
  "pipeline_id": "850e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "run_id": "950e8400-e29b-41d4-a716-446655440004",
    "status": "failed",
    "duration_seconds": 150,
    "error_type": "ConnectionError",
    "error_message": "Failed to connect to target database",
    "error_details": {
      "connection_id": "target-db-connection",
      "error_code": "CONNECTION_TIMEOUT",
      "retry_count": 3
    },
    "failed_task": "load_to_target",
    "can_retry": true,
    "stats": {
      "rows_processed": 750000,
      "rows_inserted": 0
    }
  }
}
```

### Connector Events

#### connector.status_changed

Sent when a connector's health status changes.

```json
{
  "type": "connector.status_changed",
  "timestamp": "2024-01-26T10:15:00Z",
  "channel": "connector",
  "connector_id": "c50e8400-e29b-41d4-a716-446655440003",
  "payload": {
    "previous_status": "healthy",
    "current_status": "degraded",
    "reason": "Response time increased above threshold",
    "metrics": {
      "response_time_ms": 850,
      "threshold_ms": 500,
      "availability_percent": 98.5
    }
  }
}
```

#### connector.discovered

Sent when AI auto-discovery finds a new connector.

```json
{
  "type": "connector.discovered",
  "timestamp": "2024-01-26T10:20:00Z",
  "channel": "discovery",
  "payload": {
    "type": "postgresql",
    "host": "192.168.1.15",
    "port": 5432,
    "name": "dev-postgres-02",
    "confidence_score": 0.92,
    "discovery_method": "port_scan"
  }
}
```

### System Events

#### system.alert

System-level alerts and warnings.

```json
{
  "type": "system.alert",
  "timestamp": "2024-01-26T10:25:00Z",
  "channel": "system",
  "payload": {
    "severity": "warning",
    "category": "resource",
    "message": "Kafka consumer lag exceeding threshold",
    "details": {
      "topic": "events",
      "consumer_group": "analytics",
      "lag": 50000,
      "threshold": 10000
    },
    "recommended_action": "Scale up consumer instances"
  }
}
```

---

## Subscription Management

### Subscribe to Channel

```javascript
// Subscribe to pipeline events
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'pipeline',
  pipeline_id: '850e8400-e29b-41d4-a716-446655440003'
}));

// Subscribe to connector events
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'connector',
  connector_id: 'c50e8400-e29b-41d4-a716-446655440003'
}));

// Subscribe to system alerts
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'system'
}));

// Subscribe to project-wide events
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'project',
  project_id: '550e8400-e29b-41d4-a716-446655440000'
}));
```

### Unsubscribe from Channel

```javascript
ws.send(JSON.stringify({
  type: 'unsubscribe',
  channel: 'pipeline',
  pipeline_id: '850e8400-e29b-41d4-a716-446655440003'
}));
```

### List Active Subscriptions

```javascript
ws.send(JSON.stringify({
  type: 'list_subscriptions'
}));

// Response
{
  "type": "subscriptions",
  "payload": {
    "subscriptions": [
      {
        "channel": "pipeline",
        "pipeline_id": "850e8400-e29b-41d4-a716-446655440003"
      },
      {
        "channel": "system"
      }
    ]
  }
}
```

---

## Heartbeat & Keepalive

### Client-Side Ping

Send periodic pings to keep connection alive:

```javascript
// Send ping every 30 seconds
setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'ping' }));
  }
}, 30000);

// Handle pong response
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'pong') {
    console.log('Keepalive confirmed');
  }
};
```

### Server-Side Heartbeat

Server sends periodic heartbeat messages:

```json
{
  "type": "heartbeat",
  "timestamp": "2024-01-26T10:30:00Z",
  "server_time": "2024-01-26T10:30:00.123Z",
  "active_connections": 142
}
```

---

## Reconnection Strategy

Implement exponential backoff for reconnection:

```javascript
class WebSocketClient {
  constructor(url, token) {
    this.url = url;
    this.token = token;
    this.reconnectDelay = 1000;
    this.maxReconnectDelay = 30000;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 10;
    this.ws = null;
    this.subscriptions = [];
  }

  connect() {
    this.ws = new WebSocket(`${this.url}?token=${this.token}`);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
      this.reconnectDelay = 1000;

      // Resubscribe to channels
      this.subscriptions.forEach(sub => {
        this.subscribe(sub.channel, sub.id);
      });
    };

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleMessage(data);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onclose = (event) => {
      console.log('WebSocket closed:', event.code, event.reason);

      if (event.code !== 1000 && this.reconnectAttempts < this.maxReconnectAttempts) {
        this.reconnect();
      }
    };
  }

  reconnect() {
    this.reconnectAttempts++;
    console.log(`Reconnecting... Attempt ${this.reconnectAttempts}`);

    setTimeout(() => {
      this.connect();
    }, this.reconnectDelay);

    // Exponential backoff
    this.reconnectDelay = Math.min(
      this.reconnectDelay * 2,
      this.maxReconnectDelay
    );
  }

  subscribe(channel, id = null) {
    const sub = { channel, id };
    this.subscriptions.push(sub);

    this.ws.send(JSON.stringify({
      type: 'subscribe',
      channel: channel,
      [`${channel}_id`]: id
    }));
  }

  handleMessage(data) {
    // Implement message handling
    console.log('Received:', data);
  }
}

// Usage
const client = new WebSocketClient(
  'ws://localhost:8000/api/v1/ws',
  'YOUR_JWT_TOKEN'
);
client.connect();
client.subscribe('pipeline', '850e8400-e29b-41d4-a716-446655440003');
```

---

## Server-Sent Events (SSE)

For simpler one-way streaming from server to client:

### JavaScript Client

```javascript
const eventSource = new EventSource(
  'http://localhost:8000/api/v1/sse/pipeline/850e8400-e29b-41d4-a716-446655440003?token=YOUR_JWT_TOKEN'
);

eventSource.addEventListener('pipeline.started', (event) => {
  const data = JSON.parse(event.data);
  console.log('Pipeline started:', data);
});

eventSource.addEventListener('pipeline.progress', (event) => {
  const data = JSON.parse(event.data);
  console.log('Progress:', data.progress);
});

eventSource.addEventListener('pipeline.completed', (event) => {
  const data = JSON.parse(event.data);
  console.log('Pipeline completed:', data);
  eventSource.close();
});

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  eventSource.close();
};
```

### Python Client

```python
import sseclient
import requests
import json

url = 'http://localhost:8000/api/v1/sse/pipeline/850e8400-e29b-41d4-a716-446655440003'
headers = {'Authorization': 'Bearer YOUR_JWT_TOKEN'}

response = requests.get(url, headers=headers, stream=True)
client = sseclient.SSEClient(response)

for event in client.events():
    data = json.loads(event.data)
    print(f'{event.event}: {data}')

    if event.event == 'pipeline.completed':
        break
```

---

## Long Polling (Fallback)

For environments without WebSocket/SSE support:

### JavaScript Client

```javascript
async function pollPipelineStatus(runId, token) {
  while (true) {
    try {
      const response = await fetch(
        `http://localhost:8000/api/v1/runs/${runId}/events?wait=30`,
        {
          headers: {
            'Authorization': `Bearer ${token}`
          }
        }
      );

      const events = await response.json();

      for (const event of events) {
        console.log('Event:', event);

        if (event.event_type === 'completed' || event.event_type === 'failed') {
          return event;
        }
      }
    } catch (error) {
      console.error('Polling error:', error);
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
}

// Usage
pollPipelineStatus('950e8400-e29b-41d4-a716-446655440004', 'YOUR_JWT_TOKEN');
```

---

## React Integration Example

```typescript
import { useEffect, useState } from 'react';

interface PipelineEvent {
  type: string;
  timestamp: string;
  payload: any;
}

export function usePipelineWebSocket(pipelineId: string, token: string) {
  const [events, setEvents] = useState<PipelineEvent[]>([]);
  const [status, setStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    const ws = new WebSocket(`ws://localhost:8000/api/v1/ws?token=${token}`);

    ws.onopen = () => {
      setStatus('connected');
      ws.send(JSON.stringify({
        type: 'subscribe',
        channel: 'pipeline',
        pipeline_id: pipelineId
      }));
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setEvents(prev => [...prev, data]);

      if (data.type === 'pipeline.progress') {
        setProgress(data.payload.progress);
      }
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    ws.onclose = () => {
      setStatus('disconnected');
    };

    return () => {
      ws.close();
    };
  }, [pipelineId, token]);

  return { events, status, progress };
}

// Component usage
function PipelineMonitor({ pipelineId }: { pipelineId: string }) {
  const { events, status, progress } = usePipelineWebSocket(pipelineId, localStorage.getItem('token'));

  return (
    <div>
      <div>Status: {status}</div>
      <div>Progress: {progress}%</div>
      <div>
        {events.map((event, index) => (
          <div key={index}>
            {event.type}: {JSON.stringify(event.payload)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Python Integration Example

```python
import asyncio
import websockets
import json

async def monitor_pipeline(pipeline_id: str, token: str):
    uri = f"ws://localhost:8000/api/v1/ws?token={token}"

    async with websockets.connect(uri) as websocket:
        # Subscribe to pipeline events
        await websocket.send(json.dumps({
            "type": "subscribe",
            "channel": "pipeline",
            "pipeline_id": pipeline_id
        }))

        # Listen for events
        async for message in websocket:
            event = json.loads(message)

            if event["type"] == "pipeline.started":
                print(f"Pipeline started: {event['payload']['run_id']}")

            elif event["type"] == "pipeline.progress":
                progress = event["payload"]["progress"]
                print(f"Progress: {progress}%")

            elif event["type"] == "pipeline.completed":
                print(f"Pipeline completed successfully")
                stats = event["payload"]["stats"]
                print(f"Rows processed: {stats['rows_processed']}")
                break

            elif event["type"] == "pipeline.failed":
                print(f"Pipeline failed: {event['payload']['error_message']}")
                break

# Usage
asyncio.run(monitor_pipeline(
    "850e8400-e29b-41d4-a716-446655440003",
    "YOUR_JWT_TOKEN"
))
```

---

## Error Handling

### Connection Errors

```json
{
  "type": "error",
  "error_code": "CONNECTION_FAILED",
  "message": "WebSocket connection failed",
  "timestamp": "2024-01-26T10:35:00Z"
}
```

### Authentication Errors

```json
{
  "type": "error",
  "error_code": "UNAUTHORIZED",
  "message": "Invalid or expired token",
  "timestamp": "2024-01-26T10:35:00Z"
}
```

### Subscription Errors

```json
{
  "type": "error",
  "error_code": "SUBSCRIPTION_FAILED",
  "message": "Pipeline not found",
  "details": {
    "pipeline_id": "invalid-id"
  },
  "timestamp": "2024-01-26T10:35:00Z"
}
```

---

## Best Practices

### 1. Implement Reconnection Logic

Always implement exponential backoff for reconnection:

```javascript
const maxRetries = 10;
let retryCount = 0;
let retryDelay = 1000;

function reconnect() {
  if (retryCount >= maxRetries) {
    console.error('Max reconnection attempts reached');
    return;
  }

  setTimeout(() => {
    retryCount++;
    retryDelay *= 2;
    connect();
  }, retryDelay);
}
```

### 2. Handle Heartbeat

Respond to server heartbeat messages:

```javascript
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  if (data.type === 'heartbeat') {
    // Optional: Send pong
    ws.send(JSON.stringify({ type: 'pong' }));
  }
};
```

### 3. Clean Up Subscriptions

Unsubscribe when component unmounts:

```javascript
useEffect(() => {
  // Subscribe
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'pipeline',
    pipeline_id: pipelineId
  }));

  return () => {
    // Unsubscribe on cleanup
    ws.send(JSON.stringify({
      type: 'unsubscribe',
      channel: 'pipeline',
      pipeline_id: pipelineId
    }));
  };
}, [pipelineId]);
```

### 4. Buffer Events

Buffer events during reconnection:

```javascript
const eventBuffer = [];
let isConnected = false;

function sendEvent(event) {
  if (isConnected) {
    ws.send(JSON.stringify(event));
  } else {
    eventBuffer.push(event);
  }
}

ws.onopen = () => {
  isConnected = true;

  // Flush buffer
  while (eventBuffer.length > 0) {
    ws.send(JSON.stringify(eventBuffer.shift()));
  }
};
```

### 5. Monitor Connection Quality

Track connection metrics:

```javascript
const metrics = {
  reconnectCount: 0,
  messageCount: 0,
  lastMessageTime: null,
  avgLatency: 0
};

ws.onmessage = (event) => {
  metrics.messageCount++;
  metrics.lastMessageTime = Date.now();

  const data = JSON.parse(event.data);
  if (data.server_time) {
    const latency = Date.now() - new Date(data.server_time).getTime();
    metrics.avgLatency = (metrics.avgLatency + latency) / 2;
  }
};
```

---

## Rate Limits

WebSocket connections are subject to rate limits:

| Plan | Max Concurrent Connections | Messages/Second |
|------|---------------------------|-----------------|
| Free | 5 | 10 |
| Pro | 50 | 100 |
| Enterprise | 500 | 1000 |

---

## Related Documentation

- [Pipeline API](./pipelines.md) - Pipeline management
- [Connector API](./connectors.md) - Connector management
- [Authentication](./authentication.md) - Authentication guide
- [REST API Overview](./rest-api.md) - General API documentation

---

[← Back to API Documentation](./rest-api.md) | [Pipeline API →](./pipelines.md)
