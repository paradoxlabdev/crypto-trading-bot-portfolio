# Technical Highlights - Deep Dive

This document provides detailed explanations of key technical solutions and optimizations implemented in the bot.

## Table of Contents

- [Smart Token Re-check Logic](#smart-token-re-check-logic)
- [Multi-Layer Caching Architecture](#multi-layer-caching-architecture)
- [WebSocket Flood Protection](#websocket-flood-protection)
- [Optimized Multiplier Tracking](#optimized-multiplier-tracking)
- [Redis Connection Resilience](#redis-connection-resilience)
- [Telegram Rate Limiting](#telegram-rate-limiting)
- [Performance Optimizations](#performance-optimizations)

---

## Smart Token Re-check Logic

### Problem Statement

When a token is rejected (no channels meet user filters), new channels may call the same token later. We need to:
1. Re-check the token when new channels appear
2. Only process **new channels** (not all channels again)
3. Avoid duplicate API calls for channels already checked

### Solution

**Data Structure:**
```json
{
  "timestamp": "2025-01-14T10:00:00Z",
  "channels_checked": ["@channel1", "@channel2"],
  "status": "rejected",
  "channels_called_at": {
    "channel1": "2025-01-14T10:00:00Z",
    "channel2": "2025-01-14T10:05:00Z"
  }
}
```

**Algorithm:**
1. Store `channels_checked` list when token is rejected
2. Store `channels_called_at` map (channel_id → timestamp)
3. On re-check:
   - Compare current token's `channel_calls` with `channels_checked`
   - Find channels that:
     - Are NOT in `channels_checked` (new channel), OR
     - Have `called_at` > previous `called_at` (same channel, new call)
   - Only process these **new channels**
   - If any new channel passes filters → upgrade to "accepted"

**Effect:** on re-check, we pay the per-channel API cost only for the delta of new channels, not the full list. In practice that's typically 1–2 calls instead of the original 10–20 per re-check.

**Code Logic (Pseudocode):**
```python
async def get_new_channels_since_last_check(token_data, user_id):
    processed_info = await get_processed_token_info(token_address, user_id)
    if not processed_info:
        return all_channels  # First time
    
    previous_channels = set(processed_info['channels_checked'])
    prev_called_map = processed_info['channels_called_at']
    
    new_channels = []
    for channel in token_data['channel_calls']:
        channel_id = channel['channel_id']
        called_at = channel['called_at']
        
        # Check if new channel OR new call from same channel
        is_new = (
            channel_id not in previous_channels or
            called_at > prev_called_map.get(channel_id, '')
        )
        
        if is_new and within_lookback_window(called_at):
            new_channels.append(channel_id)
    
    return new_channels
```

---

## Multi-Layer Caching Architecture

### Architecture Overview

```
┌─────────────────────────────────────────┐
│   L1: In-Memory Cache (SmartCache)      │
│   - Fastest access (~0.1ms)             │
│   - TTL: 30s - 5min                     │
│   - LRU eviction                         │
└──────────────┬──────────────────────────┘
               │ Cache Miss
               ▼
┌─────────────────────────────────────────┐
│   L2: Redis Cache (Processed Tokens)    │
│   - Fast access (~1-5ms)                 │
│   - TTL: 1h - 14 days                   │
│   - Persistent across restarts           │
└──────────────┬──────────────────────────┘
               │ Cache Miss
               ▼
┌─────────────────────────────────────────┐
│   L3: External API                      │
│   - Slow access (~200-500ms)             │
│   - Rate limited                         │
│   - Network dependent                    │
└─────────────────────────────────────────┘
```

### SmartCache Implementation

**Features:**
- TTL-based expiration
- LRU eviction when full
- Thread-safe operations
- Periodic cleanup

**Usage:**
```python
# Channel data cache (30 min TTL)
channels_cache = SmartCache(
    default_ttl=1800,      # 30 minutes
    max_size=1000,         # Max 1000 entries
    cleanup_interval=300   # Cleanup every 5 min
)

# User settings cache (5 min TTL)
user_settings_cache = SmartCache(
    default_ttl=300,       # 5 minutes
    max_size=5000,         # Max 5000 entries
    cleanup_interval=60   # Cleanup every 1 min
)
```

**Relative cost per lookup** (local measurements, vary with load):
- Cache hit (in-memory): sub-millisecond
- Cache miss, L2 Redis hit: single-digit milliseconds
- Cache miss, L3 external API: hundreds of milliseconds, rate-limited

Hit rate on channel metadata is high enough in normal operation that the external API is not the bottleneck.

### Redis Cache Strategy

**Processed Tokens:**
- **Accepted:** 14 days TTL (prevents duplicate notifications)
- **Rejected:** 1 hour TTL (allows re-checking)

**Channel Metadata:**
- 30 minutes TTL — long enough that repeat lookups within a processing burst hit the cache, short enough that channel stats stay fresh.

**Atomic Operations:**
```python
# Atomic upgrade from rejected → accepted
async def mark_token_processed(token_address, user_id, status):
    key = f"processed_token:{token_address}:{user_id}"
    
    # Only upgrade if status improves (rejected → accepted)
    if status == "accepted":
        # Use SETEX with NX (only if not exists)
        # Or upgrade if existing status is "rejected"
        await redis.setex(key, ttl, value)
```

---

## WebSocket Flood Protection

### Problem Statement

WebSocket can send 1000+ token events per minute, overwhelming the system if not handled properly.

### Solution Components

#### 1. Queue-Based Processing

```python
token_queue = asyncio.Queue(maxsize=10000)

async def process_tokens():
    while True:
        token = await token_queue.get()
        await process_token_for_all_users(token)
        token_queue.task_done()
```

**Benefits:**
- Buffers incoming tokens
- Prevents memory overflow
- Allows backpressure handling

#### 2. Rate Limiting

```python
class RateLimiter:
    def __init__(self, max_calls_per_minute=100):
        self.max_calls = max_calls_per_minute
        self.calls = 0
        self.last_reset = time.time()
    
    async def check_rate_limit(self):
        current_time = time.time()
        if current_time - self.last_reset >= 60:
            self.calls = 0
            self.last_reset = current_time
        
        if self.calls >= self.max_calls:
            wait_time = 60 - (current_time - self.last_reset)
            await asyncio.sleep(wait_time)
            return False
        
        self.calls += 1
        return True
```

**Result:** Limits API calls to 100/minute, preventing overload.

#### 3. Semaphore-Based Concurrency Control

```python
# Limit concurrent user processing
user_processing_semaphore = asyncio.Semaphore(max_concurrent_users=5)

async def process_token_for_user(token, user_id):
    async with user_processing_semaphore:
        # Process token for user
        # Only 5 users processed concurrently
        pass
```

**Result:** Prevents Redis/Telegram overload from too many concurrent operations.

#### 4. Smart Deduplication

```python
# Check if token already processed
is_processed = await is_token_processed_for_user(token_address, user_id)

if is_processed:
    # Skip or re-check new channels only
    pass
else:
    # Full processing
    pass
```

**Effect:** the large majority of repeat events hit the dedup check and exit immediately, without re-running filter evaluation.

---

## Optimized Multiplier Tracking

### Problem Statement

Track price multipliers for millions of token-call entries across many users in real time. The original implementation used Redis `SCAN` to find which users had a given token tracked, which walked the entire matching key space on every price update.

### Solution: In-Memory Cache

**Before (Slow):**
```python
# Redis SCAN - O(N) operation
async def get_tracked_calls_for_token(token_address):
    pattern = f"multiplier:{token_address}:*"
    keys = await redis.scan(match=pattern)  # ~500ms for 100 users
    return keys
```

**After (Fast):**
```python
# In-memory dictionary - O(1) operation
class MultiplierTracker:
    def __init__(self):
        # Cache: token_address → [user_ids]
        self.tracked_tokens_by_token = {}
    
    def get_users_for_token(self, token_address):
        return self.tracked_tokens_by_token.get(token_address, [])
        # ~0.0001ms lookup
```

**Mechanical change:** O(N) over the Redis keyspace → O(1) dict lookup. Wall-clock gain is proportional to keyspace size; at this scale the operation moved from "visible on the event-loop profile" to "unmeasurable noise".

**Cache Maintenance:**
```python
# Rebuild cache on startup
async def rebuild_token_cache(self):
    # SCAN all multiplier keys (one-time cost)
    # Build in-memory dictionary
    # ~5-10 seconds for 2.5M keys
    
# Update cache incrementally
async def track_call(self, token_address, user_id):
    # Add to Redis
    await redis.set(f"multiplier:{token}:{user}", data)
    
    # Update in-memory cache
    if token_address not in self.tracked_tokens_by_token:
        self.tracked_tokens_by_token[token_address] = []
    self.tracked_tokens_by_token[token_address].append(user_id)
```

**Memory Usage:**
- 2.5M entries × ~20 bytes = ~50MB
- Acceptable for 100 users × 25K tokens

---

## Redis Connection Resilience

### Problem Statement

Redis connection failures can crash the bot or cause data loss. Need automatic recovery.

### Solution: RedisManager

**Features:**
1. **Health Monitoring**
   ```python
   async def check_health(self):
       try:
           await redis.ping()
           self.redis_connection_healthy = True
       except Exception:
           self.redis_connection_healthy = False
   ```

2. **Automatic Reconnection**
   ```python
   async def reconnect(self):
       attempts = 0
       while attempts < self.max_reconnect_attempts:
           try:
               await self.connect()
               self.redis_connection_healthy = True
               return True
           except Exception:
               attempts += 1
               await asyncio.sleep(2 ** attempts)  # Exponential backoff
       return False
   ```

3. **Circuit Breaker**
   ```python
   if unhealthy_time > max_unhealthy_time:
       # Stop retrying, use fallback
       self.circuit_breaker_open = True
   ```

4. **Graceful Degradation**
   ```python
   # Runtime fallback cache
   _processed_runtime = {}  # In-memory cache
   
   async def is_token_processed(self, token_address, user_id):
       # Try Redis first
       result = await redis.get(key)
       if result:
           return result
       
       # Fallback to runtime cache
       return self._processed_runtime.get(key)
   ```

**Effect:** a Redis outage that doesn't exceed the circuit-breaker window degrades gracefully — reads serve from the runtime cache, writes buffer or drop non-critical data, reconnect is transparent once Redis is back.

---

## Telegram Rate Limiting

### Problem Statement

Telegram API limits:
- 30 messages/second globally
- Risk of bans if exceeded

### Solution: Multi-Level Rate Limiting

**Implementation:**
```python
class TelegramRateLimiter:
    def __init__(self):
        self.per_user_limits = {}  # user_id → last_message_time
        self.global_limit = RateLimiter(max_per_second=30)
        self.per_user_lock = {}  # user_id → asyncio.Lock()
    
    async def wait_if_needed(self, user_id, priority=False):
        # Per-user limit (1 msg/sec)
        if not priority:
            async with self._get_user_lock(user_id):
                last_time = self.per_user_limits.get(user_id, 0)
                elapsed = time.time() - last_time
                if elapsed < 1.0:
                    await asyncio.sleep(1.0 - elapsed)
                self.per_user_limits[user_id] = time.time()
        
        # Global limit (30 msg/sec)
        await self.global_limit.wait_if_needed()
```

**Priority System:**
- **Normal messages:** Both per-user and global limits
- **Menu updates:** Only global limit (bypasses per-user)
- **Admin messages:** Higher priority

**Effect:** the client-side ceiling is below Telegram's enforcement threshold, so rate-limit bans stop being a concern.

---

## Performance Optimizations

### 1. Batch Redis Operations

**Before:**
```python
# Individual operations
for key in keys:
    value = await redis.get(key)  # 1000 operations
```

**After:**
```python
# Batch operations
values = await redis.mget(keys)  # 1 operation
```

**Effect:** one round-trip instead of N. The saving scales with batch size.

### 2. Parallel Processing

**Before:**
```python
# Sequential processing
for user in users:
    await process_user(user)
```

**After:**
```python
# Bounded-concurrent processing
async with semaphore:
    tasks = [process_user(user) for user in users]
    await asyncio.gather(*tasks)
```

**Effect:** wall time for a fan-out drops from `N × per_user` to roughly `ceil(N / concurrency) × per_user`, at the cost of bounded peak load downstream (controlled by the semaphore).

### 3. Connection Pooling

```python
# Reuse connections
connector = aiohttp.TCPConnector(
    limit=100,              # Max 100 connections
    limit_per_host=10,      # Max 10 per host
    ttl_dns_cache=300,      # DNS cache 5 min
    keepalive_timeout=30    # Keep-alive 30s
)
```

**Effect:** connection setup cost is amortized across many requests; DNS lookups are cached; idle connections are reused rather than re-negotiated.

### 4. Async I/O

All network operations use `async/await`:
- Non-blocking I/O
- Concurrent operations
- Better resource utilization

Every outbound call (Redis, PostgreSQL, HTTP, Telegram) is async, so the event loop isn't blocked on any single operation and the process can hold many concurrent in-flight operations at low memory cost.

---

## Dual-Write Architecture (Redis + PostgreSQL)

### Problem Statement

Need both fast real-time operations (Redis) and permanent storage for analytics, audit trail, and compliance (PostgreSQL).

### Solution: Dual-Write Pattern

**Architecture:**
```
User Action
    ├── Write to Redis (fast, critical path)
    │   └── System continues immediately
    │
    └── Write to PostgreSQL (async, non-blocking)
        └── If fails, logged but doesn't affect user
```

**Implementation:**
```python
# 1. Write to Redis (primary)
await redis_client.setex(
    f"subscription:{chat_id}",
    ttl,
    json.dumps(subscription_data)
)

# 2. Write to PostgreSQL (async, non-blocking)
if hasattr(self, 'db_repo') and self.db_repo:
    try:
        await self.db_repo.save_subscription(
            chat_id=chat_id,
            plan=plan,
            expires_at=expires_at,
            ...
        )
        logger.info("✅ Saved to PostgreSQL")
    except Exception as e:
        # Non-fatal - Redis write already succeeded
        logger.error(f"❌ PostgreSQL write failed (non-fatal): {e}")
```

**Data Stored:**
- **Redis:** Active subscriptions, processed tokens, cache, real-time state
- **PostgreSQL:** Subscription history, payments, user activity, token performance, notifications

**Benefits:**
- Fast operations (Redis)
- Permanent storage (PostgreSQL)
- Analytics capabilities
- Audit trail for compliance
- Graceful degradation (works if PostgreSQL fails)

**Latency characteristics:** Redis writes are on the critical path and are fast enough to not be felt by the user. PostgreSQL writes are off the critical path — their latency only affects the time until analytics tables are consistent, which is acceptable for reporting use cases.

---

## Summary

Taken together, these patterns let the bot keep a sub-millisecond hot path under bursty load while persisting enough data for analytics and audit. The biggest single win was the `SCAN → dict` swap on the multiplier hot path; the rest is conventional work done carefully.

---

*Last updated: April 2026*
