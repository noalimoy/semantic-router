# Redis Storage for Response API

The Redis storage backend provides persistent, distributed storage for Response API conversations. This solution offers excellent performance with automatic TTL expiration and support for both standalone and cluster deployments.

## Overview

Redis storage is ideal for:

- **Production environments** requiring persistence across restarts
- **Distributed deployments** sharing state across multiple router instances
- **Stateful conversations** with automatic chain management
- **TTL-based expiration** for automatic cleanup
- **High availability** with cluster support

## Configuration

### Where to Configure

Response API storage is configured in **`config/config.yaml`**:

```yaml
# config/config.yaml
response_api:
  enabled: true
  store_backend: "memory"  # Default: "memory", Options: "redis", "milvus"
  ttl_seconds: 86400       # 24 hours (default)
  max_responses: 1000      # Memory store limit
```

**To enable Redis storage**, change `store_backend` to `"redis"` and add Redis configuration:

```yaml
# config/config.yaml
response_api:
  enabled: true
  store_backend: "redis"   # Switch to Redis
  ttl_seconds: 2592000     # 30 days (recommended for Redis)
  redis:
    address: "localhost:6379"
    password: ""
    db: 0
    key_prefix: "sr:"
```

**Configuration examples:**

- See `config/response-api/redis-simple.yaml` for basic standalone setup
- See `config/response-api/redis-cluster.yaml` for cluster configuration
- See `config/response-api/redis-production.yaml` for production with TLS

### Standalone Redis Configuration

For simple deployments:

```yaml
# config/config.yaml
response_api:
  redis:
    address: "localhost:6379"
    db: 0
    key_prefix: "sr:"
    pool_size: 10
    min_idle_conns: 2
    max_retries: 3
    dial_timeout: 5
    read_timeout: 3
    write_timeout: 3
```

### Redis Cluster Configuration

For distributed deployments:

```yaml
# config/config.yaml
response_api:
  redis:
    cluster_mode: true
    cluster_addresses:
      - "redis-node-1:6379"
      - "redis-node-2:6379"
      - "redis-node-3:6379"
    db: 0  # Must be 0 for cluster
    key_prefix: "sr:"
    pool_size: 20
```

### External Configuration File

For complex configurations, use `config/response-api/redis.yaml`:

```yaml
# config/config.yaml
response_api:
  redis:
    config_path: "config/response-api/redis-production.yaml"
```

Example production config:

```yaml
# config/response-api/redis-production.yaml
address: "redis.production.svc.cluster.local:6379"
password: "${REDIS_PASSWORD}"
db: 0
key_prefix: "sr:"

# Connection pooling
pool_size: 20
min_idle_conns: 5
max_retries: 3

# TLS/SSL
tls_enabled: true
tls_cert_path: "/etc/ssl/certs/redis-client.crt"
tls_key_path: "/etc/ssl/certs/redis-client.key"
```

## Setup and Deployment

### 1. Start Redis

**Using Docker:**

> **Note:** Response API uses **standard Redis** (image: `redis:7-alpine`), not Redis Stack. 
> 
> **Why?** Response API only needs basic key-value storage (`GET`, `SET`, `DEL`, `EXPIRE`). 
> 

```bash
# Simple Redis (recommended for Response API)
docker run -d \
  --name redis-response-api \
  -p 6379:6379 \
  redis:7-alpine

# With persistence (production)
docker run -d \
  --name redis-response-api \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7-alpine redis-server --save 60 1 --appendonly yes
```

**Verify Redis is running:**

```bash
docker exec redis-response-api redis-cli PING
# Expected: PONG
```

### 2. Configure Semantic Router

Update `config/config.yaml`:

```yaml
response_api:
  enabled: true
  store_backend: "redis"
  redis:
    address: "localhost:6379"
    db: 0
    key_prefix: "sr:"
```

### 3. Run Semantic Router

```bash
# Build and run
make build-router
make run-router
```

### 4. Run EnvoyProxy

```bash
# Start Envoy proxy
make run-envoy
```

### 5. Test Response API

**Create a response:**

```bash
curl -X POST http://localhost:8080/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MoM",
    "input": [{"type": "text", "text": "What is machine learning?"}],
    "instructions": "You are a helpful assistant.",
    "store": true
  }'
```

**Response:**

```json
{
  "id": "resp_abc123...",
  "object": "response",
  "status": "completed",
  "conversation_id": "conv_xyz789",
  "output": [...],
  "created_at": 1234567890
}
```

**Create a follow-up response:**

```bash
curl -X POST http://localhost:8080/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MoM",
    "input": [{"type": "text", "text": "Give me an example"}],
    "previous_response_id": "resp_abc123...",
    "store": true
  }'
```

**How conversation history works:**

When you include `previous_response_id` in a request, the system automatically:

1. Calls `GetConversationChain()` internally
2. Retrieves the full conversation history
3. Passes it to the LLM as context

There is no separate endpoint to retrieve the chain - it's handled automatically during request processing.

## Key Schema

Redis stores data with the following key patterns:

```
sr:response:{response_id}       → StoredResponse (JSON)
sr:conversation:{conv_id}        → StoredConversation (JSON)
```

**How conversation chains work:**

Unlike traditional databases with join tables, Response API uses **embedded links**:

- Each response has `previous_response_id` field
- Chains are built by following these links backwards

**Example:**

```bash
# Response 3 (latest)
GET sr:response:resp_3
→ {"id": "resp_3", "previous_response_id": "resp_2", ...}

# Response 2 (middle)
GET sr:response:resp_2
→ {"id": "resp_2", "previous_response_id": "resp_1", ...}

# Response 1 (first)
GET sr:response:resp_1
→ {"id": "resp_1", "previous_response_id": "", ...}
```

**How chains are retrieved:**

Chains are built automatically when you provide `previous_response_id` in a request. The system:

1. Uses `GetConversationChain(responseID)` internally
2. Follows `previous_response_id` links backwards using Redis pipelining
3. Returns responses in chronological order (oldest first)
4. Passes the full history as context to the LLM

**Example flow:**

```
Request with previous_response_id="resp_3"
  ↓
System calls GetConversationChain("resp_3")
  ↓
Redis pipelined GET: resp_3 → resp_2 → resp_1
  ↓
Returns: [resp_1, resp_2, resp_3] (chronological)
```

## Monitoring

### Redis CLI Commands

```bash
# Connect to Redis CLI
docker exec -it redis-response-api redis-cli

# View all Response API keys
KEYS sr:*

# List response keys only
KEYS sr:response:*

# Get a specific response
GET sr:response:resp_abc123

# Check TTL (time to live)
TTL sr:response:resp_abc123

# Count total keys
DBSIZE

# Monitor live commands (useful for debugging)
MONITOR

# Get Redis server info
INFO
```

## Performance Optimization

### Connection Pooling

Adjust pool size for high-traffic deployments:

```yaml
response_api:
  redis:
    pool_size: 20          # Default: 10
    min_idle_conns: 5      # Default: 2
```

### Pipelining

The Redis store automatically uses pipelining for `GetConversationChain()` to minimize network round-trips.

**Performance improvement:**

- Without pipelining: 10 responses = 10 network calls
- With pipelining: 10 responses = ~2 network calls

### TTL Management

Adjust TTL based on your needs:

```yaml
response_api:
  ttl_seconds: 86400  # 24 hours (shorter for frequent updates)
  # ttl_seconds: 2592000  # 30 days (default, longer retention)
```

## Reference

- [Response API Overview](../intelligent-route/router-memory.md) - General Response API documentation
