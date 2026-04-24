# PostgreSQL Database Schema Diagram

## Entity Relationship Diagram (ERD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DATABASE SCHEMA                                 │
│                   Crypto Trading Analytics Bot                           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                              USERS                                       │
├─────────────────────────────────────────────────────────────────────────┤
│ PK  chat_id                    BIGINT                                    │
│     telegram_username          VARCHAR(255)                              │
│     first_name                 VARCHAR(255)                              │
│     referral_code              VARCHAR(50) UNIQUE                        │
│ FK  referred_by_chat_id         BIGINT → users.chat_id                   │
│     created_at                 TIMESTAMP WITH TIME ZONE                   │
│     last_active                TIMESTAMP WITH TIME ZONE                   │
│     total_subscriptions        INT                                       │
│     metadata                   JSONB                                     │
└─────────────────────────────────────────────────────────────────────────┘
         │
         │ 1:N
         │
         ├──────────────────────────────────────────────────────────────┐
         │                                                               │
         ▼                                                               ▼
┌──────────────────────────┐                    ┌──────────────────────────┐
│      SUBSCRIPTIONS       │                    │        PAYMENTS           │
├──────────────────────────┤                    ├──────────────────────────┤
│ PK  id                   │                    │ PK  id                   │
│ FK  chat_id              │                    │ FK  chat_id              │
│     plan                  │                    │ FK  subscription_id       │
│     status                │                    │     plan                  │
│     started_at            │                    │     amount_sol            │
│     expires_at            │                    │     amount_usd            │
│     tx_hash               │                    │     payment_method        │
│     amount_sol            │                    │     status                │
│     amount_usd            │                    │     tx_hash UNIQUE         │
│     payment_method        │                    │     invoice_id UNIQUE     │
│     coupon_code           │                    │     coupon_code           │
│     discount_percent      │                    │     created_at            │
│     created_at            │                    │     completed_at          │
│     updated_at            │                    │     metadata              │
└──────────────────────────┘                    └──────────────────────────┘
         │                                               │
         │ 1:N                                           │ 1:N
         │                                               │
         ▼                                               ▼
┌──────────────────────────┐                    ┌──────────────────────────┐
│  REFERRAL_COMMISSIONS       │                    │    COUPON_USAGE          │
├──────────────────────────┤                    ├──────────────────────────┤
│ PK  id                   │                    │ PK  id                   │
│ FK  owner_chat_id        │                    │     coupon_code           │
│ FK  buyer_chat_id        │                    │ FK  chat_id              │
│ FK  payment_id           │                    │ FK  payment_id           │
│ FK  subscription_id       │                    │ FK  subscription_id       │
│     level                 │                    │     plan                  │
│ FK  intermediate_chat_id │                    │     discount_percent      │
│     ref_code              │                    │     original_amount_sol  │
│     plan                  │                    │     discount_amount_sol   │
│     amount_sol            │                    │     final_amount_sol      │
│     commission_sol        │                    │     used_at               │
│     commission_percent   │                    │     metadata              │
│     status                │                    └──────────────────────────┘
│     tx_hash               │
│     created_at            │
│     paid_at               │
└──────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        ANALYTICS TABLES                                  │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐  ┌──────────────────────────┐  ┌──────────────────────────┐
│  USER_ACTIVITY_LOG       │  │   TOKEN_PERFORMANCE       │  │   SETTINGS_CHANGES       │
├──────────────────────────┤  ├──────────────────────────┤  ├──────────────────────────┤
│ PK  id                   │  │ PK  id                   │  │ PK  id                   │
│ FK  chat_id              │  │     token_address         │  │ FK  chat_id              │
│     activity_type        │  │     token_name            │  │     setting_name         │
│     details               │  │     token_symbol          │  │     old_value            │
│     created_at            │  │ FK  chat_id               │  │     new_value            │
└──────────────────────────┘  │     fdv_at_call           │  │     changed_at            │
                               │     fdv_at_peak            │  │     source              │
                               │     max_multiplier         │  └──────────────────────────┘
                               │     tracked_since          │
                               │     peak_reached_at        │  ┌──────────────────────────┐
                               │     notification_sent      │  │ USER_SETTINGS_SNAPSHOTS  │
                               │     created_at             │  ├──────────────────────────┤
                               └──────────────────────────┘  │ PK  id                   │
                                                             │ FK  chat_id              │
                                                             │     settings JSONB        │
                                                             │     snapshot_at           │
                                                             └──────────────────────────┘

┌──────────────────────────┐
│      NOTIFICATIONS        │
├──────────────────────────┤
│ PK  id                   │
│ FK  chat_id              │
│     notification_type    │
│     token_address         │
│     multiplier            │
│     message_text          │
│     sent_at               │
│     metadata              │
└──────────────────────────┘
```

## Table Relationships

### Core Tables
- **users** (1) ──→ (N) **subscriptions** - One user can have many subscriptions
- **users** (1) ──→ (N) **payments** - One user can have many payments
- **users** (1) ──→ (N) **referral_commissions** (as owner) - One user can earn many commissions
- **users** (1) ──→ (N) **referral_commissions** (as buyer) - One user can generate commissions for others
- **users** (1) ──→ (N) **users** (self-referral) - Users can refer other users (MLM structure)

### Payment Flow
- **payments** (1) ──→ (1) **subscriptions** - One payment creates one subscription
- **payments** (1) ──→ (N) **referral_commissions** - One payment can generate multiple commissions (Level 1 + Level 2)

### Analytics Tables
- **users** (1) ──→ (N) **user_activity_log** - Track all user activities
- **users** (1) ──→ (N) **token_performance** - Track token performance per user
- **users** (1) ──→ (N) **settings_changes** - Audit trail of settings changes
- **users** (1) ──→ (N) **user_settings_snapshots** - Periodic backups of user settings
- **users** (1) ──→ (N) **notifications** - History of all notifications sent

## Key Indexes

### Performance Indexes
- `idx_users_referral_code` - Fast referral code lookup
- `idx_subscriptions_chat_id` - Fast subscription queries per user
- `idx_subscriptions_expires_at` - Fast active subscription queries
- `idx_payments_tx_hash` - Fast payment verification
- `idx_commissions_status` - Fast pending commission queries
- `idx_token_perf_multiplier` - Fast top performers queries

## Views

### Pre-built Views
- **active_subscriptions** - Currently active subscriptions (not expired)
- **pending_commissions** - Unpaid referral commissions
- **revenue_summary** - Daily revenue by payment method
- **active_users_stats** - Most active users (last 30 days)
- **token_performance_summary** - Aggregated token metrics
- **settings_change_frequency** - Settings change patterns

---

**Total Tables:** 10 (5 core + 5 analytics)  
**Total Views:** 6  
**Total Indexes:** 30+  
**Database:** PostgreSQL 14+  
**Architecture:** Dual-write (Redis + PostgreSQL)

