# Code Examples - Sanitized Snippets

This document contains sanitized code examples demonstrating key implementations. All sensitive data, API keys, and business logic have been removed or anonymized.

## Table of Contents

- [Smart Token Re-check Logic](#smart-token-re-check-logic)
- [Multi-Layer Caching](#multi-layer-caching)
- [Redis Connection Resilience](#redis-connection-resilience)
- [Rate Limiting](#rate-limiting)
- [Optimized Multiplier Tracking](#optimized-multiplier-tracking)

---

## Smart Token Re-check Logic

### Core Algorithm

```python
async def get_new_channels_since_last_check(self, token_data, user_id):
    """
    Returns list of new channels that weren't checked before.
    Only processes new channels, not all channels again.
    """
    token_address = token_data.get('address')
    if not token_address:
        return []
    
    processed_info = await self.get_processed_token_info(token_address, user_id)
    if not processed_info:
        # First time - all channels are "new"
        return self._get_channels_within_lookback(token_data)
    
    last_check_time = self.parse_datetime_with_timezone(
        processed_info.get('timestamp', '1970-01-01T00:00:00')
    )
    previous_channels = set(processed_info.get('channels_checked', []))
    prev_called_map = processed_info.get('channels_called_at', {}) or {}
    
    new_channels = []
    for channel in token_data.get('channel_calls', []):
        channel_id = channel.get('channel_id')
        called_at = channel.get('called_at')
        
        if not channel_id or not called_at:
            continue
        
        call_time = self.parse_datetime_with_timezone(called_at)
        if not call_time:
            continue
        
        # Check if channel is new OR has new call time
        prev_called_at = prev_called_map.get(channel_id)
        prev_called_dt = self.parse_datetime_with_timezone(prev_called_at) if prev_called_at else None
        
        is_new_time = prev_called_dt is None or (call_time and call_time > prev_called_dt)
        is_new_channel = channel_id not in previous_channels
        
        if (call_time >= self.get_current_lookback_time() and
            call_time > last_check_time and
            (is_new_channel or is_new_time)):
            new_channels.append(channel_id)
    
    return new_channels
```

### Atomic Status Upgrade

```python
async def _write_processed_token_atomic(self, key, ttl, value_dict):
    """
    Atomically write processed_token with upgrade semantics.
    Only upgrades from rejected → accepted, never downgrades.
    """
    value = json.dumps(value_dict)
    
    # Try to set only if key doesn't exist
    set_res = await self._safe_redis_operation(
        self.redis_client.set,
        key,
        value,
        ex=ttl,
        nx=True,  # Only if not exists
    )
    
    if set_res:
        return True
    
    # Key exists - check if we should upgrade
    existing_raw = await self._safe_redis_operation(
        self.redis_client.get, key
    )
    if existing_raw is None:
        # Key vanished - write fresh
        await self._safe_redis_operation(
            self.redis_client.setex, key, ttl, value
        )
        return True
    
    existing_data = json.loads(self._decode_json_value(existing_raw))
    existing_status = existing_data.get("status")
    incoming_status = value_dict.get("status")
    
    # Only upgrade from rejected → accepted
    if incoming_status == "accepted" and existing_status == "rejected":
        await self._safe_redis_operation(
            self.redis_client.setex, key, ttl, value
        )
        return True
    
    # Keep existing value
    return False
```

---

## Multi-Layer Caching

### SmartCache Implementation

```python
class SmartCache:
    """
    Thread-safe cache with TTL and LRU eviction.
    """
    def __init__(self, default_ttl=300, max_size=1000, cleanup_interval=60):
        self.default_ttl = default_ttl
        self.max_size = max_size
        self.cleanup_interval = cleanup_interval
        self._cache = {}
        self._access_times = {}
        self._lock = asyncio.Lock()
        self._last_cleanup = time.time()
    
    async def get(self, key):
        """Get value from cache, return None if not found or expired."""
        async with self._lock:
            if key not in self._cache:
                return None
            
            entry = self._cache[key]
            if time.time() > entry['expires_at']:
                # Expired
                del self._cache[key]
                self._access_times.pop(key, None)
                return None
            
            # Update access time for LRU
            self._access_times[key] = time.time()
            return entry['value']
    
    async def set(self, key, value, ttl=None):
        """Set value in cache with optional TTL."""
        async with self._lock:
            ttl = ttl or self.default_ttl
            expires_at = time.time() + ttl
            
            # Evict if full (LRU)
            if len(self._cache) >= self.max_size and key not in self._cache:
                self._evict_lru()
            
            self._cache[key] = {
                'value': value,
                'expires_at': expires_at
            }
            self._access_times[key] = time.time()
            
            # Periodic cleanup
            if time.time() - self._last_cleanup > self.cleanup_interval:
                await self._cleanup_expired()
    
    def _evict_lru(self):
        """Evict least recently used entry."""
        if not self._access_times:
            # Fallback: remove first entry
            if self._cache:
                key = next(iter(self._cache))
                del self._cache[key]
            return
        
        lru_key = min(self._access_times, key=self._access_times.get)
        del self._cache[lru_key]
        del self._access_times[lru_key]
    
    async def _cleanup_expired(self):
        """Remove expired entries."""
        now = time.time()
        expired_keys = [
            key for key, entry in self._cache.items()
            if now > entry['expires_at']
        ]
        for key in expired_keys:
            del self._cache[key]
            self._access_times.pop(key, None)
        self._last_cleanup = now
```

### Cache Usage Pattern

```python
# Channel data cache
async def get_channel_info(self, channel_id):
    # Try L1 cache first
    cached = await self.channels_cache.get(channel_id)
    if cached:
        return cached
    
    # Try L2 cache (Redis)
    redis_key = f"channel:{channel_id}:info"
    redis_data = await self._safe_redis_operation(
        self.redis_client.get, redis_key
    )
    if redis_data:
        data = json.loads(redis_data)
        # Populate L1 cache
        await self.channels_cache.set(channel_id, data, ttl=1800)
        return data
    
    # L3: API call
    data = await self._fetch_channel_from_api(channel_id)
    
    # Populate both caches
    await self.channels_cache.set(channel_id, data, ttl=1800)
    await self._safe_redis_operation(
        self.redis_client.setex,
        redis_key,
        1800,  # 30 min
        json.dumps(data)
    )
    
    return data
```

---

## Redis Connection Resilience

### RedisManager Implementation

```python
class RedisManager:
    """
    Manages Redis connection with health monitoring and auto-reconnect.
    """
    def __init__(self, host, port, db, password, logger, max_reconnect_attempts=5):
        self.host = host
        self.port = port
        self.db = db
        self.password = password
        self.logger = logger
        self.max_reconnect_attempts = max_reconnect_attempts
        self.redis_client = None
        self.redis_connection_healthy = False
        self.redis_unhealthy_start_time = None
        self.redis_max_unhealthy_time = 300  # 5 minutes
    
    def connect(self):
        """Establish Redis connection."""
        try:
            self.redis_client = redis.Redis(
                host=self.host,
                port=self.port,
                db=self.db,
                password=self.password,
                decode_responses=False,
                socket_connect_timeout=5,
                socket_timeout=5,
                retry_on_timeout=True,
                health_check_interval=30
            )
            # Test connection
            self.redis_client.ping()
            self.redis_connection_healthy = True
            self.logger.info("✅ Redis connected successfully")
        except Exception as e:
            self.redis_connection_healthy = False
            self.logger.error(f"❌ Redis connection failed: {e}")
            raise
    
    async def check_health(self):
        """Check Redis health and update status."""
        try:
            start_time = time.time()
            await asyncio.to_thread(self.redis_client.ping)
            latency = (time.time() - start_time) * 1000
            
            if latency > 1000:  # > 1 second
                self.logger.warning(f"⚠️ Redis slow: {latency:.0f}ms")
            
            self.redis_connection_healthy = True
            self.redis_unhealthy_start_time = None
            return True
        except Exception as e:
            self.redis_connection_healthy = False
            if self.redis_unhealthy_start_time is None:
                self.redis_unhealthy_start_time = time.time()
            
            unhealthy_duration = time.time() - self.redis_unhealthy_start_time
            if unhealthy_duration > self.redis_max_unhealthy_time:
                self.logger.error(
                    f"❌ Redis unhealthy for {unhealthy_duration:.0f}s - "
                    "circuit breaker may activate"
                )
            
            return False
    
    async def reconnect(self):
        """Attempt to reconnect with exponential backoff."""
        attempts = 0
        while attempts < self.max_reconnect_attempts:
            try:
                wait_time = min(2 ** attempts, 30)  # Max 30 seconds
                await asyncio.sleep(wait_time)
                
                self.connect()
                self.logger.info(f"✅ Redis reconnected after {attempts + 1} attempts")
                return True
            except Exception as e:
                attempts += 1
                self.logger.warning(
                    f"⚠️ Redis reconnect attempt {attempts}/{self.max_reconnect_attempts} failed: {e}"
                )
        
        self.logger.error("❌ Redis reconnection failed after all attempts")
        return False
    
    async def safe_operation(self, operation, *args, **kwargs):
        """Safely execute Redis operation with error handling."""
        if not self.redis_connection_healthy:
            # Try reconnect once
            await self.reconnect()
        
        try:
            if asyncio.iscoroutinefunction(operation):
                return await operation(*args, **kwargs)
            else:
                return await asyncio.to_thread(operation, *args, **kwargs)
        except redis.ConnectionError:
            self.redis_connection_healthy = False
            self.logger.error("❌ Redis connection error")
            return None
        except Exception as e:
            self.logger.error(f"❌ Redis operation error: {e}")
            return None
```

---

## Rate Limiting

### Telegram Rate Limiter

```python
class TelegramRateLimiter:
    """
    Multi-level rate limiter for Telegram API.
    - Per-user: 1 message/second
    - Global: 30 messages/second
    """
    def __init__(self):
        self.per_user_limits = {}  # user_id → last_message_time
        self.global_limiter = SimpleRateLimiter(max_per_second=30)
        self.per_user_locks = {}  # user_id → asyncio.Lock()
        self._lock = asyncio.Lock()
    
    def _get_user_lock(self, user_id):
        """Get or create lock for user."""
        async with self._lock:
            if user_id not in self.per_user_locks:
                self.per_user_locks[user_id] = asyncio.Lock()
            return self.per_user_locks[user_id]
    
    async def wait_if_needed(self, user_id, priority=False):
        """
        Wait if rate limit would be exceeded.
        Priority operations skip per-user limit.
        """
        # Per-user limit (1 msg/sec) - skip for priority
        if not priority:
            async with self._get_user_lock(user_id):
                last_time = self.per_user_limits.get(user_id, 0)
                elapsed = time.time() - last_time
                if elapsed < 1.0:
                    wait_time = 1.0 - elapsed
                    await asyncio.sleep(wait_time)
                self.per_user_limits[user_id] = time.time()
        
        # Global limit (30 msg/sec)
        await self.global_limiter.wait_if_needed()

class SimpleRateLimiter:
    """Simple rate limiter with token bucket algorithm."""
    def __init__(self, max_per_second=30):
        self.max_per_second = max_per_second
        self.tokens = max_per_second
        self.last_update = time.time()
        self._lock = asyncio.Lock()
    
    async def wait_if_needed(self):
        """Wait if tokens are exhausted."""
        async with self._lock:
            now = time.time()
            elapsed = now - self.last_update
            
            # Refill tokens
            self.tokens = min(
                self.max_per_second,
                self.tokens + elapsed * self.max_per_second
            )
            self.last_update = now
            
            if self.tokens >= 1.0:
                self.tokens -= 1.0
                return
            
            # Wait for token
            wait_time = (1.0 - self.tokens) / self.max_per_second
            await asyncio.sleep(wait_time)
            self.tokens = 0.0
```

---

## Optimized Multiplier Tracking

### In-Memory Cache Implementation

```python
class MultiplierTracker:
    """
    Tracks token price multipliers with in-memory cache for O(1) lookups.
    """
    def __init__(self, redis_client, bot_instance):
        self.redis_client = redis_client
        self.bot_instance = bot_instance
        
        # In-memory cache: token_address → [user_ids]
        # O(1) lookup instead of O(N) Redis SCAN
        self.tracked_tokens_by_token = {}
        
        # Price cache: token_address → marketcap_usd
        self.price_cache = {}
        
        # Rate limiting: token_address → last_update_time
        self.last_price_update_time = {}
    
    async def rebuild_token_cache(self):
        """
        Rebuild in-memory cache from Redis on startup.
        One-time cost: ~5-10 seconds for 2.5M keys.
        """
        self.logger.info("🔄 Rebuilding token cache from Redis...")
        pattern = "multiplier:*"
        cursor = 0
        count = 0
        
        while True:
            cursor, keys = await asyncio.to_thread(
                self.redis_client.scan,
                cursor,
                match=pattern,
                count=1000
            )
            
            for key in keys:
                # Parse key: multiplier:{token}:{user}
                parts = key.decode().split(':')
                if len(parts) == 3:
                    token_address = parts[1]
                    user_id = int(parts[2])
                    
                    if token_address not in self.tracked_tokens_by_token:
                        self.tracked_tokens_by_token[token_address] = []
                    self.tracked_tokens_by_token[token_address].append(user_id)
            
            count += len(keys)
            if cursor == 0:
                break
        
        self.logger.info(
            f"✅ Rebuilt cache: {len(self.tracked_tokens_by_token)} tokens, "
            f"{count} total entries"
        )
    
    async def track_call(self, token_address, user_id, channel_id, fdv_at_call):
        """
        Track a token call for multiplier notifications.
        Updates both Redis and in-memory cache.
        """
        # Save to Redis
        key = f"multiplier:{token_address}:{user_id}"
        data = {
            "channel_id": channel_id,
            "fdv_at_call": fdv_at_call,
            "timestamp": datetime.now(timezone.utc).isoformat()
        }
        await asyncio.to_thread(
            self.redis_client.setex,
            key,
            7 * 24 * 60 * 60,  # 7 days TTL
            json.dumps(data)
        )
        
        # Update in-memory cache
        if token_address not in self.tracked_tokens_by_token:
            self.tracked_tokens_by_token[token_address] = []
        if user_id not in self.tracked_tokens_by_token[token_address]:
            self.tracked_tokens_by_token[token_address].append(user_id)
    
    async def process_price_update(self, token_address, current_fdv):
        """
        Process price update and send multiplier notifications.
        Uses in-memory cache for O(1) lookup.
        """
        # Rate limiting: 5 second minimum interval, 2% change threshold
        last_update = self.last_price_update_time.get(token_address, 0)
        if time.time() - last_update < 5:
            return  # Too soon
        
        prev_price = self.price_cache.get(token_address)
        if prev_price:
            change_pct = abs((current_fdv - prev_price) / prev_price)
            if change_pct < 0.02:  # Less than 2% change
                return  # Not significant enough
        
        # Get users for this token (O(1) lookup!)
        user_ids = self.tracked_tokens_by_token.get(token_address, [])
        if not user_ids:
            return
        
        # Process for each user
        for user_id in user_ids:
            await self._check_and_notify_multiplier(
                token_address, user_id, current_fdv
            )
        
        # Update cache
        self.price_cache[token_address] = current_fdv
        self.last_price_update_time[token_address] = time.time()
    
    async def _check_and_notify_multiplier(self, token_address, user_id, current_fdv):
        """Check if multiplier threshold reached and send notification."""
        # Get tracked call data from Redis
        key = f"multiplier:{token_address}:{user_id}"
        data_raw = await asyncio.to_thread(self.redis_client.get, key)
        if not data_raw:
            return
        
        data = json.loads(data_raw)
        fdv_at_call = data.get('fdv_at_call', 0)
        
        if fdv_at_call == 0:
            return
        
        multiplier = current_fdv / fdv_at_call
        
        # Check thresholds (2x, 5x, 10x, etc.)
        thresholds = [2, 5, 10, 20, 50, 100]
        for threshold in thresholds:
            if multiplier >= threshold:
                # Send notification (with deduplication)
                await self._send_multiplier_notification(
                    token_address, user_id, multiplier, threshold
                )
                break
```

---

## Dual-Write Pattern (Redis + PostgreSQL)

### PostgreSQL Manager

```python
class PostgresManager:
    """
    Manages PostgreSQL connection pool with async operations.
    """
    def __init__(self, host, port, database, user, password):
        self.host = host
        self.port = port
        self.database = database
        self.user = user
        self.password = password
        self.pool = None
    
    async def connect(self):
        """Initialize connection pool."""
        self.pool = await asyncpg.create_pool(
            host=self.host,
            port=self.port,
            database=self.database,
            user=self.user,
            password=self.password,
            min_size=5,
            max_size=20,
            command_timeout=30,
            timeout=10,
        )
        logger.info("✅ PostgreSQL connection pool created")
    
    async def fetchrow(self, query, *args):
        """Execute query and return single row."""
        async with self.pool.acquire() as conn:
            return await conn.fetchrow(query, *args)
    
    async def execute(self, query, *args):
        """Execute query (INSERT/UPDATE/DELETE)."""
        async with self.pool.acquire() as conn:
            return await conn.execute(query, *args)
```

### Database Repository

```python
class DatabaseRepository:
    """
    High-level database operations with dual-write pattern.
    """
    def __init__(self, postgres_manager):
        self.pg = postgres_manager
    
    async def save_subscription(
        self,
        chat_id: int,
        plan: str,
        expires_at: datetime,
        payment_method: str,
        amount_sol: float = None,
        amount_usd: float = None,
    ):
        """
        Save subscription to PostgreSQL.
        Called asynchronously after Redis write succeeds.
        """
        query = """
            INSERT INTO subscriptions (
                chat_id, plan, status, expires_at,
                payment_method, amount_sol, amount_usd
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            RETURNING id
        """
        try:
            row = await self.pg.fetchrow(
                query,
                chat_id,
                plan,
                'active',
                expires_at,
                payment_method,
                amount_sol,
                amount_usd,
            )
            logger.info(f"✅ Subscription saved to PostgreSQL: id={row['id']}")
            return row
        except Exception as e:
            logger.error(f"❌ Failed to save subscription to PostgreSQL: {e}")
            raise
```

### Dual-Write Implementation

```python
async def save_subscription_dual_write(
    self,
    chat_id: int,
    plan: str,
    expires_at: datetime,
):
    """
    Save subscription using dual-write pattern.
    1. Write to Redis (fast, critical path)
    2. Write to PostgreSQL (async, non-blocking)
    """
    subscription_data = {
        "plan": plan,
        "expires_at": expires_at.isoformat(),
        "status": "active",
    }
    
    # 1. Write to Redis (primary - fast)
    redis_key = f"subscription:{chat_id}"
    await self._safe_redis_operation(
        self.redis_client.setex,
        redis_key,
        86400 * 365,  # 1 year TTL
        json.dumps(subscription_data)
    )
    logger.info(f"✅ Subscription saved to Redis: {chat_id}")
    
    # 2. Write to PostgreSQL (async, non-blocking)
    if hasattr(self, 'db_repo') and self.db_repo:
        try:
            await self.db_repo.save_subscription(
                chat_id=chat_id,
                plan=plan,
                expires_at=expires_at,
                payment_method="solana",
            )
            logger.info(f"✅ Subscription saved to PostgreSQL: {chat_id}")
        except Exception as e:
            # Non-fatal - Redis write already succeeded
            logger.error(
                f"❌ Failed to save subscription to PostgreSQL (non-fatal): {e}"
            )
            # System continues normally
```

### Async Activity Logging

```python
async def log_user_activity_async(
    self,
    chat_id: int,
    activity_type: str,
    metadata: dict = None,
):
    """
    Log user activity to PostgreSQL asynchronously.
    Non-blocking, fire-and-forget pattern.
    """
    if not hasattr(self, 'db_repo') or not self.db_repo:
        return  # Skip if PostgreSQL not available
    
    try:
        query = """
            INSERT INTO user_activity_log (
                chat_id, activity_type, metadata, created_at
            )
            VALUES ($1, $2, $3, NOW())
        """
        metadata_json = json.dumps(metadata) if metadata else None
        
        # Fire-and-forget (don't await, don't block)
        asyncio.create_task(
            self.db_repo.pg.execute(query, chat_id, activity_type, metadata_json)
        )
    except Exception as e:
        # Silent failure - logging shouldn't break the app
        logger.debug(f"Failed to log activity (non-fatal): {e}")
```

---

## Summary

These code examples demonstrate:
- **Smart algorithms** for efficient token processing
- **Multi-layer caching** for performance
- **Resilient error handling** for reliability
- **Rate limiting** for API protection
- **Optimized data structures** for scalability
- **Dual-write architecture** for analytics and audit trail

All code has been sanitized to remove sensitive information while preserving the core logic and patterns.

---

**Last Updated:** 2025-12-25  
**Version:** v2.0 (PostgreSQL Dual-Write Architecture)
