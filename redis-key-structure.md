# Redis Key Structure & Naming Conventions

## Overview

Redis is used as the primary storage for real-time operations, caching, and session management. All keys follow a consistent naming pattern: `{prefix}:{identifier}:{optional_subkey}`

**Database:** Redis DB 14 (isolated from other applications)  
**Total Keys:** ~50-100 different key patterns, 1000s-100,000s of active keys  
**Data Size:** ~50-500 MB (depends on user count)

---

## Key Naming Patterns

### Format
```
{prefix}:{identifier}:{optional_subkey}
```

### Examples
```
user_settings:123456789
subscription:123456789
processed_token:ABC123...XYZ:123456789
multiplier:ABC123...XYZ:123456789
```

---

## Key Categories

### 1. User Data

```
┌─────────────────────────────────────────────────────────────┐
│                    USER DATA KEYS                           │
└─────────────────────────────────────────────────────────────┘

user_settings:{chat_id}
├─ Type: String (JSON)
├─ TTL: None (persistent)
├─ Content: User preferences (filters, blockchain, etc.)
└─ Example: user_settings:123456789

subscription:{chat_id}
├─ Type: String (JSON)
├─ TTL: Based on subscription expiry
├─ Content: Subscription plan, expiry date, status
└─ Example: subscription:123456789

user_referred_by:{chat_id}
├─ Type: String
├─ TTL: None (persistent)
├─ Content: Parent referrer's chat_id
└─ Example: user_referred_by:123456789 → "987654321"

user_referral_code:{chat_id}
├─ Type: String
├─ TTL: None (persistent)
├─ Content: User's unique referral code
└─ Example: user_referral_code:123456789 → "ABC123"

referral_code:{code}
├─ Type: String
├─ TTL: None (persistent)
├─ Content: chat_id of code owner
└─ Example: referral_code:ABC123 → "123456789"

conversation_state:{chat_id}
├─ Type: String
├─ TTL: 5 minutes
├─ Content: Current input state (waiting for filter value, etc.)
└─ Example: conversation_state:123456789 → "waiting_min_win_rate"

blocked_user:{chat_id}
├─ Type: String (boolean flag)
├─ TTL: None (persistent)
├─ Content: "1" if user blocked bot
└─ Example: blocked_user:123456789 → "1"
```

---

### 2. Token Processing

```
┌─────────────────────────────────────────────────────────────┐
│                TOKEN PROCESSING KEYS                         │
└─────────────────────────────────────────────────────────────┘

processed_token:{token_address}:{user_id}
├─ Type: String (JSON)
├─ TTL: 14 days (accepted) / 1 hour (rejected)
├─ Content: Processing status, channels checked, timestamp
└─ Example: processed_token:ABC123...XYZ:123456789
└─ Value: {"status": "accepted", "channels_checked": [...], "timestamp": "..."}

token_price:{token_address}
├─ Type: String (JSON)
├─ TTL: 5 minutes
├─ Content: Cached token price data (FDV, marketcap, etc.)
└─ Example: token_price:ABC123...XYZ

channel:{channel_id}:info
├─ Type: String (JSON)
├─ TTL: 30 minutes
├─ Content: Channel statistics (win_rate, avg_gain, etc.)
└─ Example: channel:@crypto_channel:info
```

---

### 3. Multiplier Tracking

```
┌─────────────────────────────────────────────────────────────┐
│              MULTIPLIER TRACKING KEYS                        │
└─────────────────────────────────────────────────────────────┘

multiplier:{token_address}:{user_id}
├─ Type: String (JSON)
├─ TTL: 7 days
├─ Content: Tracking data, highest multiplier, notification flags
└─ Example: multiplier:ABC123...XYZ:123456789
└─ Value: {
│     "channel_id": "@crypto_channel",
│     "fdv_at_call": 50000.0,
│     "highest_multiplier_sent": 2.5,
│     "timestamp": "2025-12-25T10:00:00Z"
│   }

price:{token_address}
├─ Type: String
├─ TTL: 5 minutes
├─ Content: Current marketcap/FDV value
└─ Example: price:ABC123...XYZ → "125000.50"
```

---

### 4. Payments & Subscriptions

```
┌─────────────────────────────────────────────────────────────┐
│            PAYMENTS & SUBSCRIPTIONS KEYS                     │
└─────────────────────────────────────────────────────────────┘

used_transaction:{tx_hash}
├─ Type: String (boolean flag)
├─ TTL: 1 year
├─ Content: "1" if transaction already processed
└─ Example: used_transaction:ABC123...XYZ → "1"

payment_verification:{tx_hash}
├─ Type: String (JSON)
├─ TTL: 24 hours
├─ Content: Payment verification data
└─ Example: payment_verification:ABC123...XYZ

pending_commission:{commission_id}
├─ Type: String (JSON)
├─ TTL: 90 days
├─ Content: Pending referral commission data
└─ Example: pending_commission:12345

paid_commission:{commission_id}
├─ Type: String (JSON)
├─ TTL: 1 year
├─ Content: Paid commission data with tx_hash
└─ Example: paid_commission:12345
```

---

### 5. Coupons

```
┌─────────────────────────────────────────────────────────────┐
│                      COUPON KEYS                             │
└─────────────────────────────────────────────────────────────┘

coupon:{code}
├─ Type: String (JSON)
├─ TTL: Based on coupon validity
├─ Content: Coupon details (discount, max_uses, etc.)
└─ Example: coupon:WELCOME50
└─ Value: {
│     "discount_percent": 50.0,
│     "max_uses": 100,
│     "used_count": 25,
│     "valid_until": "2026-01-01T00:00:00Z"
│   }

coupon_usage:{code}:{chat_id}
├─ Type: String (boolean flag)
├─ TTL: Based on coupon validity
├─ Content: "1" if user already used this coupon
└─ Example: coupon_usage:WELCOME50:123456789 → "1"
```

---

### 6. Sets & Collections

```
┌─────────────────────────────────────────────────────────────┐
│                  SETS & COLLECTIONS                          │
└─────────────────────────────────────────────────────────────┘

active_subscribers
├─ Type: SET
├─ TTL: None (persistent)
├─ Content: Set of chat_ids with active subscriptions
├─ Operations: SADD, SISMEMBER (O(1) membership check)
└─ Example: SADD active_subscribers 123456789

active_subscribers:{plan}
├─ Type: SET
├─ TTL: None (persistent)
├─ Content: Set of chat_ids with specific plan
└─ Example: active_subscribers:monthly
```

---

## TTL Strategy

### Short TTL (5 minutes)
- `conversation_state:{chat_id}`
- `token_price:{token_address}`
- `price:{token_address}`

### Medium TTL (30 minutes - 1 hour)
- `channel:{channel_id}:info` (30 min)
- `processed_token:{token_address}:{user_id}` (rejected: 1 hour)

### Long TTL (7 days - 1 year)
- `multiplier:{token_address}:{user_id}` (7 days)
- `processed_token:{token_address}:{user_id}` (accepted: 14 days)
- `used_transaction:{tx_hash}` (1 year)
- `paid_commission:{commission_id}` (1 year)

### No TTL (Persistent)
- `user_settings:{chat_id}`
- `subscription:{chat_id}` (expires based on subscription, not TTL)
- `referral_code:{code}`
- `user_referred_by:{chat_id}`

---

## Key Statistics

### Estimated Key Counts (100 users, 25K tokens)
- `user_settings:*`: ~100 keys
- `subscription:*`: ~100 keys
- `processed_token:*:*`: ~2.5M keys (25K tokens × 100 users)
- `multiplier:*:*`: ~2.5M keys
- `token_price:*`: ~25K keys
- `channel:*:info`: ~100-500 keys
- `coupon:*`: ~10-50 keys
- **Total:** ~5M+ keys

### Memory Usage
- Average key size: ~100-500 bytes
- Total memory: ~50-500 MB (depends on user count and token volume)

---

## Operations Pattern

### Read-Heavy Operations
- `user_settings:{chat_id}` - Read on every notification
- `subscription:{chat_id}` - Read on every notification
- `processed_token:{token_address}:{user_id}` - Read for deduplication

### Write-Heavy Operations
- `processed_token:{token_address}:{user_id}` - Written for every token
- `multiplier:{token_address}:{user_id}` - Updated on price changes
- `token_price:{token_address}` - Updated frequently

### Atomic Operations
- `SETNX` for idempotent writes
- `INCR` for counters
- `SADD` for set membership
- `EXPIRE` for TTL management

---

## Example Key Values (Sanitized)

### user_settings:123456789
```json
{
  "min_win_rate": 30,
  "min_avg_gain": 50,
  "min_fdv_at_call": 30000,
  "blockchain_filter": "both",
  "target_channel": null,
  "preset": "preset_2"
}
```

### subscription:123456789
```json
{
  "plan": "monthly",
  "status": "active",
  "expires_at": "2026-01-25T10:00:00Z",
  "payment_method": "solana",
  "tx_hash": "ABC123...XYZ"
}
```

### processed_token:ABC123...XYZ:123456789
```json
{
  "status": "accepted",
  "timestamp": "2025-12-25T10:00:00Z",
  "channels_checked": ["@channel1", "@channel2"],
  "channels_called_at": {
    "@channel1": "2025-12-25T10:00:00Z"
  }
}
```

---


