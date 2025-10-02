# 🔍 Vector Search API

## Overview

The Vector Search API provides semantic search and deduplication capabilities using vector embeddings. Powered by Qdrant and Weaviate vector databases.

## Base Endpoints

All vector search endpoints are under `/api/v1/vector-search`

## Authentication

All requests require JWT authentication:
```http
Authorization: Bearer {your_access_token}
```

---

## Collections Management

### Create Collection

Create a new vector collection for storing embeddings.

```http
POST /api/v1/vector-search/collections
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "customer_records",
  "dimension": 768,
  "distance_metric": "cosine",
  "provider": "qdrant",
  "metadata_schema": {
    "customer_id": "string",
    "segment": "string",
    "created_at": "datetime"
  }
}
```

**Parameters:**
- `name` (string, required): Collection name
- `dimension` (integer, required): Vector dimension (768 for sentence-transformers)
- `distance_metric` (string): "cosine", "euclidean", or "dot" (default: "cosine")
- `provider` (string): "qdrant" or "weaviate" (default: "qdrant")
- `metadata_schema` (object): Schema for metadata fields

**Response:**
```json
{
  "collection_name": "customer_records",
  "dimension": 768,
  "distance_metric": "cosine",
  "provider": "qdrant",
  "vectors_count": 0,
  "status": "ready"
}
```

### Get Collection Statistics

```http
GET /api/v1/vector-search/collections/{name}/stats
Authorization: Bearer {token}
```

**Response:**
```json
{
  "collection_name": "customer_records",
  "vectors_count": 15234,
  "points_count": 15234,
  "indexed_vectors_count": 15234,
  "segments_count": 4,
  "status": "green",
  "disk_size_bytes": 45678901
}
```

### Delete Collection

```http
DELETE /api/v1/vector-search/collections/{name}
Authorization: Bearer {token}
```

---

## Indexing Operations

### Index Single Record

Add a single record to the vector collection.

```http
POST /api/v1/vector-search/index
Content-Type: application/json
Authorization: Bearer {token}

{
  "collection_name": "customer_records",
  "record_id": "cust_12345",
  "text": "John Doe, john.doe@example.com, Premium customer from New York",
  "metadata": {
    "customer_id": "cust_12345",
    "email": "john.doe@example.com",
    "segment": "premium",
    "location": "New York"
  }
}
```

**Response:**
```json
{
  "record_id": "cust_12345",
  "indexed": true,
  "vector_id": "vec_789abc",
  "embedding_dimension": 768
}
```

### Batch Index

Index multiple records in a single request for better performance.

```http
POST /api/v1/vector-search/batch-index
Content-Type: application/json
Authorization: Bearer {token}

{
  "collection_name": "customer_records",
  "records": [
    {
      "record_id": "cust_001",
      "text": "Alice Smith, alice@example.com, VIP customer",
      "metadata": {"segment": "vip"}
    },
    {
      "record_id": "cust_002",
      "text": "Bob Johnson, bob@example.com, Standard customer",
      "metadata": {"segment": "standard"}
    }
  ],
  "batch_size": 100
}
```

**Response:**
```json
{
  "total_records": 2,
  "indexed_count": 2,
  "failed_count": 0,
  "processing_time_ms": 145,
  "failed_records": []
}
```

---

## Search Operations

### Semantic Search

Find similar records based on semantic meaning.

```http
POST /api/v1/vector-search/search
Content-Type: application/json
Authorization: Bearer {token}

{
  "collection_name": "customer_records",
  "query_text": "premium customer from NYC",
  "limit": 10,
  "score_threshold": 0.7,
  "filters": {
    "segment": "premium"
  }
}
```

**Response:**
```json
{
  "query": "premium customer from NYC",
  "results": [
    {
      "record_id": "cust_12345",
      "score": 0.94,
      "text": "John Doe, john.doe@example.com, Premium customer from New York",
      "metadata": {
        "customer_id": "cust_12345",
        "segment": "premium",
        "location": "New York"
      }
    },
    {
      "record_id": "cust_67890",
      "score": 0.87,
      "text": "Jane Smith, jane.smith@example.com, Premium member NYC",
      "metadata": {
        "customer_id": "cust_67890",
        "segment": "premium",
        "location": "New York"
      }
    }
  ],
  "total_results": 2,
  "search_time_ms": 23
}
```

### Find Duplicates

Detect semantically similar records for deduplication.

```http
POST /api/v1/vector-search/duplicates
Content-Type: application/json
Authorization: Bearer {token}

{
  "collection_name": "customer_records",
  "similarity_threshold": 0.90,
  "fields": ["name", "email", "phone"],
  "batch_size": 1000
}
```

**Response:**
```json
{
  "duplicate_groups": [
    {
      "group_id": 1,
      "similarity_score": 0.95,
      "records": [
        {
          "record_id": "cust_001",
          "text": "John Doe, john.doe@example.com",
          "metadata": {"customer_id": "cust_001"}
        },
        {
          "record_id": "cust_002",
          "text": "John Doe, johndoe@example.com",
          "metadata": {"customer_id": "cust_002"}
        }
      ]
    }
  ],
  "total_groups": 1,
  "total_duplicates": 2,
  "deduplication_rate": 0.013
}
```

---

## Use Cases

### 1. Customer Deduplication

```python
import httpx

async def deduplicate_customers():
    # Create collection
    await client.post("/vector-search/collections", json={
        "name": "customers",
        "dimension": 768,
        "distance_metric": "cosine"
    })

    # Index customer data
    customers = [
        {"record_id": f"cust_{i}", "text": f"Customer {i}..."}
        for i in range(10000)
    ]

    await client.post("/vector-search/batch-index", json={
        "collection_name": "customers",
        "records": customers,
        "batch_size": 500
    })

    # Find duplicates
    response = await client.post("/vector-search/duplicates", json={
        "collection_name": "customers",
        "similarity_threshold": 0.92
    })

    return response.json()
```

### 2. Semantic Product Search

```python
async def search_similar_products(query: str):
    response = await client.post("/vector-search/search", json={
        "collection_name": "product_catalog",
        "query_text": query,
        "limit": 20,
        "score_threshold": 0.75
    })

    return response.json()["results"]

# Usage
results = await search_similar_products("wireless bluetooth headphones")
```

### 3. Document Similarity

```python
async def find_similar_documents(document_text: str):
    response = await client.post("/vector-search/search", json={
        "collection_name": "knowledge_base",
        "query_text": document_text,
        "limit": 5,
        "filters": {
            "document_type": "technical"
        }
    })

    return response.json()
```

---

## Performance Considerations

### Indexing Performance

| Records | Batch Size | Time (avg) | Throughput |
|---------|------------|------------|------------|
| 10,000  | 100        | 12s        | ~833/s     |
| 10,000  | 500        | 8s         | ~1,250/s   |
| 10,000  | 1000       | 6s         | ~1,666/s   |
| 100,000 | 1000       | 58s        | ~1,724/s   |

### Search Performance

- **Single query**: 10-50ms
- **Batch search (10 queries)**: 80-200ms
- **Duplicate detection (10K records)**: 2-5 seconds

### Best Practices

1. **Batch Indexing**: Use batch sizes of 500-1000 for optimal throughput
2. **Filters**: Apply metadata filters to reduce search space
3. **Threshold Tuning**: Start with 0.85-0.90 for deduplication
4. **Dimension Selection**:
   - 384: Fast, lower accuracy (all-MiniLM-L6-v2)
   - 768: Balanced (all-mpnet-base-v2) ⭐ Recommended
   - 1024: High accuracy, slower

---

## Error Handling

### Common Error Codes

| Code | Description | Solution |
|------|-------------|----------|
| `COLLECTION_NOT_FOUND` | Collection doesn't exist | Create collection first |
| `INVALID_DIMENSION` | Vector dimension mismatch | Check embedding model dimension |
| `QUOTA_EXCEEDED` | Storage or rate limit exceeded | Upgrade plan or wait |
| `VECTOR_DB_UNAVAILABLE` | Qdrant/Weaviate connection error | Check service status |

### Example Error Response

```json
{
  "error": {
    "code": "COLLECTION_NOT_FOUND",
    "message": "Collection 'customer_records' does not exist",
    "details": {
      "collection_name": "customer_records",
      "available_collections": ["products", "documents"]
    },
    "request_id": "req_abc123"
  }
}
```

---

## Rate Limits

| Plan | Requests/Hour | Max Collection Size | Max Collections |
|------|---------------|---------------------|-----------------|
| Free | 100 | 10,000 vectors | 3 |
| Pro | 10,000 | 1M vectors | 50 |
| Enterprise | Unlimited | Unlimited | Unlimited |

---

## Configuration

### Environment Variables

```bash
# Vector Database Selection
VECTOR_DB_PROVIDER=qdrant  # or weaviate

# Qdrant Configuration
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=your_api_key
QDRANT_COLLECTION_PREFIX=ai_etl_

# Weaviate Configuration
WEAVIATE_URL=http://localhost:8085
WEAVIATE_API_KEY=your_api_key

# Embedding Model
EMBEDDING_MODEL=sentence-transformers/all-mpnet-base-v2
EMBEDDING_DIMENSION=768
```

---

## Related Documentation

- [Semantic Deduplication Service](../services/vector-search-service.md)
- [AI Enhancements Setup](../AI_ENHANCEMENTS_SETUP.md)
- [Performance Optimization](../troubleshooting/performance.md)

---

[← Back to API Reference](./rest-api.md)
