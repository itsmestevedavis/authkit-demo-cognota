# PicaOS Outlook Integration - System Proposal

## Executive Summary
Integration system for syncing calendar events between our platform and Microsoft Outlook using PicaOS as the integration backend. Initial phase focuses on **pushing events TO Outlook** with **real-time toast notifications** (via existing app-events service) on sync completion, with future expansion for **pulling events FROM Outlook**. Leverages existing platform infrastructure for notifications.

---

## System Architecture

```
- API's represent business concepts and entities, do a class diagram to represent the entities, what gets impacted when we create a connection 

We need to differentiate betweeen when the admin disables an integration for the entire
org and when a users disconnects their own account on purpose. We also need to know
if we should be deleting the connection_key from pica when the users disconnects
or just set a flag on the table.

1. Initial Sync
 - Sync to the current users outlook any event are a facilitator

2. Create New Event
 - Sync the event to the current users and any other assigned facilitator's outlook

┌─────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND                                      │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐   │
│  │   AuthKit    │  │  Integration │  │  Connection  │  │   Toast    │   │
│  │   Connect    │  │   Settings   │  │    Status    │  │ Notifcation│   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
         │                   │                  │                 ▲
         │ Connect           │ Configure        │ Check Status    │ 
         ▼                   ▼                  ▼                 │ 
                                                                  │ Notification for Connection Completed
                                                                  │ 
┌────────────────────────────────────────────────────────────────────────┐
│                         NestJS API Backend                             │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              Connection Controller                               │  │
│  │  POST   /integration/connections/store (userId from context)     │  │
│  │  GET    /connections/status (userId from context)                │  │
│  │         (Calls PicaOS /vault/connections API)                    │  │
│  │  DELETE /connections (userId from context)                       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │         Integration Settings Controller                          │  │
│  │  GET    /available-integrations                                  |  |
|  |  GET    /organization-integrations-settings                      │  │
│  │  POST   /integrations/:id (id is integration id)                 │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Schedule Service                             │   │
│  │  GET     /sync/:taskId/status                                   │   │
|  |  POST    /sync (userId from context, for sync)                  |   |
|  |  POST    /sync/:scheduleId (single event)                       |   |
|  |  PUT     /sync/:scheduleId                                      |   |
|  |  DELETE  /sync/:scheduleId                                      |   |
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Kafka Producer/Consumer                      │   │
│  │  - Publishes to outlook.sync.requested (sync requests)          │   │
│  │  - Consumes outlook.sync.completed (sync results)               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
         │                                                     ▲
         │ Produce Sync Events                                 │ Consume Results
         ▼                                                     │
┌─────────────────────────────────────────────────────────────────────────┐
│                            Kafka Cluster                                │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐                             |
│  │ outlook.sync     │  │ outlook.sync     │                             |
│  │ .requested       │  │ .completed       │                             |
│  │ (Push to Outlook)│  │ (Results)        │                             |
│  └──────────────────┘  └──────────────────┘                             |
│                                                                         │
│  ┌──────────────────┐                                                   |
│  │ outlook.sync.dlq │                                                   |
│  │ (Failed syncs)   │                                                   |
│  └──────────────────┘                                                   |
└─────────────────────────────────────────────────────────────────────────┘
         │                                                     │
         │ Consume                                             │ Produce
         ▼                                                     │
┌───────────────────────────────────────────────────────────────────────┐
│                   Outlook Sync Worker Service(s)                      │
│                         (Horizontally Scalable)                       │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │               Kafka Consumer (Consumer Group)                   │  │
│  │  - Consumes from outlook.sync.requested                         │  │
│  │  - Determines if it is a batch sync or individual               │  │
│  │  - Processes BOTH initial syncs AND resync (same topic)         │  │
│  │  - Maintains idempotency with deduplication keys                │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Retry Scheduler                              │  │
│  │  - Query: SELECT WHERE next_retry_at <= NOW()                   │  │
│  │  - Republishes to Kafka: outlook.sync.requested (individual)    │  │
│  │  - Clears next_retry_at in DB                                   │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Sync Orchestrator                            │  │
│  │  - Batch processing (per user, up to 20 operations)             │  │
│  │  - Idempotency checking                                         │  │
│  │  - On failure: SCHEDULE retry (set next_retry_at in DB)         │  │
│  │  - Does NOT retry immediately                                   │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              PicaOS API Client (Simple HTTP Wrapper)            │  │
│  │  - POST /v1/passthrough/$batch (batch operations)               │  │
│  │  - POST /v1/passthrough (individual operations)                 │  │
│  │  - Headers: x-pica-secret, x-pica-connection-key                │  │
│  │  - Error handling and classification                            │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Detailed Flow

### Overview of All Flows

```
┌─────────────────────────────────────────────────────────────────────┐
│  KEY SYSTEM FLOWS                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. CONNECTION SETUP (One-time)                                     │
│     User → AuthKit → Frontend → API → PicaOS → Store in DB         │
│                                                                     │
│  2. EVENT SYNC (Automatic on schedule change)                       │
│     Schedule Change → Kafka → Worker → PicaOS → Outlook → Notify  │
│                                                                     │
│  3. RETRY FAILED SYNCS (Background cron every minute)               │
│     Cron → Find Due Retries → Republish to Kafka → Worker Retries  │
│                                                                     │
│  4. CONNECTION STATUS CHECK (On-demand)                             │
│     Frontend → API → PicaOS Vault API → Return State → Display     │
│                                                                     │
│  5. CONNECTION HEALTH CHECK (Hourly cron)                           │
│     Cron → Check All Connections → PicaOS → Update DB → Notify     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Phase 1: Connection Setup

**Step 1: User Initiates Connection**
```
Frontend → PicaOS AuthKit Modal → User Authenticates → Frontend receives token
```

**Step 2: Backend Stores Connection**
```javascript
POST /api/pica/connections/store
Body: {
  userId: "uuid",
  integrationId: "outlook",
  authKitToken: "temp_token"
}

Backend Process (in Connection Controller):
1. Direct API call to PicaOS: GET /v1/vault/connections?userId={userId}
   Headers: { Authorization: Bearer {PICA_API_KEY} }
2. Extract connectionKey from response
3. Encrypt and store in user_pica_integration_connections table
4. Set is_active = true, connected_at = now()
5. Return connection status to frontend
```

### Phase 3: Check Connection Status (Frontend calls when needed)

**Flow Diagram:**
```
User Opens Settings Page
    ↓
Frontend: GET /connections/status
    ↓
Connection Controller
    ↓
Call PicaOS API
    ↓
GET /v1/vault/connections?userId={pica_user_id}
    ↓
PicaOS returns: { state: "operational", email: "user@example.com", ... }
    ↓
Return to Frontend
    ↓
Display status badge (Green ✓ / Yellow ⚠ / Red ✗)
```

### Phase 2: Event Sync (Push to Outlook)

**Trigger Points:**
- Schedule created/updated/deleted (automatic)
- User manually triggers sync (optional)
- Scheduled batch sync (e.g., every 5 minutes for failed syncs)

**Flow:**
```
1. Schedule Service detects change (e.g., "Team Meeting" updated)
   ↓
2. Find all users who have this schedule synced to Outlook
   SELECT user_id, pica_user_id FROM schedule_sync_status
   WHERE schedule_id = 'schedule-123' AND sync_status = 'synced'
   ↓
3. Publish ONE Kafka message PER USER to: outlook.sync.requested
   Message (for each user): {
     eventId: "uuid",
     scheduleId: "schedule-123", 
     userId: "alice-id",           // ← Specific user
     picaUserId: "pica-alice",      // ← For PicaOS API calls
     action: "CREATE" | "UPDATE" | "DELETE",
     eventData: {...},
     externalEventId: "outlook-alice-event-123", // Their Outlook event
     idempotencyKey: "schedule-123:alice-id:v2:UPDATE",
     attemptNumber: 0,
     timestamp: ISO8601
   }
   // If 3 users have this synced, 3 messages are published
   ↓
4. Sync Worker consumes message (processes one user at a time)
   ↓
5. Looks up pica_connection_key for this specific user
   SELECT pica_connection_key FROM user_pica_integration_connections
   WHERE pica_user_id = 'pica-alice'
   ↓
6. Checks idempotency (already processed this sync for this user?)
   Check: schedule-123:alice-id:v2:UPDATE
   ↓
7. Makes direct HTTP call to PicaOS Passthrough API
   POST https://api.picaos.com/v1/passthrough
   Headers: { 
     "x-pica-secret": "{PICA_SECRET}",
     "x-pica-connection-key": "{alice_connection_key}",  // Alice's connection
     "Content-Type": "application/json"
   }
   Body: {
     method: "PATCH",
     url: "/me/calendar/events/{external_event_id}",
     body: { event data transformed to Outlook format }
   }
   ↓
8. Handles response:
   - Success: Update schedule_sync_status for THIS user, publish to outlook.sync.completed
   - Retryable Error: Retry with backoff (only for this user)
   - Permanent Error: Publish to DLQ, notify THIS user
   ↓
9. API Backend receives outlook.sync.completed
   ↓
10. Updates sync status in DB for THIS user
    UPDATE schedule_sync_status 
    SET sync_status = 'synced', last_sync_at = NOW()
    WHERE schedule_id = 'schedule-123' AND user_id = 'alice-id'
   ↓
11. Publishes notification to app-events Kafka topic
    Message: {
      userId: "alice-id",  // Only notify Alice
      type: "success",
      title: "Synced to Outlook",
      message: "Calendar event synced to Outlook"
    }
   ↓
12. App-Events Service delivers toast to Alice

// This entire flow runs independently for each user (Alice, Bob, Charlie)
// If Bob's sync fails, Alice and Charlie are unaffected
```

---

## Sync Strategy: Individual vs. Batch Operations

### Message Types

The system uses **two distinct message types** for different scenarios:

```typescript
// Type 1: Individual Sync (Real-time)
interface IndividualSyncMessage {
  type: 'individual_sync';
  scheduleId: string;
  userId: string;
  picaUserId: string;
  connectionId: string;
  action: 'CREATE' | 'UPDATE' | 'DELETE';
  externalEventId?: string;
  priority: 'high';
  idempotencyKey: string;
  eventData?: any;  // Optional: include to avoid worker query
  timestamp: string;
}

// Type 2: Batch Sync (Background)
interface BatchSyncMessage {
  type: 'batch_sync';
  connectionId: string;
  userId: string;
  priority: 'low';
  reason: 'retry' | 'scheduled' | 'reconnection';
  timestamp: string;
}
```

### Hybrid Approach: Who Calculates What?

| Trigger | Who Calculates? | Message Type | Granularity | Why? |
|---------|-----------------|--------------|-------------|------|
| **Schedule Created** | API | Individual | 1 msg per user | Real-time, API has context |
| **Schedule Updated** | API | Individual | 1 msg per user | Low latency, immediate action |
| **Schedule Deleted** | API | Individual | 1 msg per user | Real-time, user expects fast sync |
| **Retry Failed Syncs** | Worker (Cron) | Batch | 1 msg per user | Background, batch-friendly |
| **User Reconnects** | API | Batch | 1 msg (user-level) | Sync all pending for user |
| **Scheduled Sync** | Worker (Cron) | Batch | 1 msg per user | Background, efficient batching |

### Idempotency Handling Comparison

| Aspect | Individual Sync | Batch Sync |
|--------|----------------|------------|
| **Idempotency Keys** | 1 key checked/stored | N keys checked/stored (one per schedule) |
| **Pre-filtering** | Simple existence check | Filter out processed schedules from batch |
| **Key Storage** | Store before API call | Store all keys before batch call |
| **Kafka Redelivery** | Skip if key exists | Filter and batch only unprocessed items |
| **Partial Success** | N/A (single operation) | Supported - each schedule tracked separately |
| **API Calls** | 1 individual call | 1 batch call (filtered schedules only) |

### Key Insight: Idempotency Per Schedule

```typescript
// Idempotency key format
`${scheduleId}:${userId}:v${version}:${action}[:retry${N}]`

// Examples:
"schedule-123:alice-id:v1:CREATE"              // Alice creates schedule
"schedule-123:alice-id:v2:UPDATE"              // Alice updates (version++)
"schedule-123:alice-id:v2:UPDATE:retry1"       // First retry
"schedule-123:bob-id:v1:CREATE"                // Bob's separate sync

// Why include userId?
// - Same schedule syncs to multiple users' calendars
// - Each user's sync is independent
// - Prevents cross-user deduplication errors
```

## Database Design - Enhanced Schema

```typescript
-- -----------------------------------------------------------------------------
-- Table: pica_integrations
-- Purpose: Master registry of available third-party integrations (Outlook, Google Calendar, etc.)
-- Notes: This is a reference table - typically seeded with available integrations
-- -----------------------------------------------------------------------------
CREATE TABLE pica_integrations.pica_integrations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Platform identifier used internally (e.g., 'microsoft-outlook', 'google-calendar')
  -- Used in code and API endpoints
  platform_name VARCHAR(100) NOT NULL UNIQUE,
  
  -- Human-readable name shown in UI (e.g., 'Microsoft Outlook')
  friendly_name VARCHAR(255) NOT NULL,
  
  -- PicaOS platform identifier for API calls
  -- Used in PicaOS AuthKit and passthrough API calls
  pica_platform_id VARCHAR(100) NOT NULL,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ DEFAULT NULL  -- Soft delete timestamp
);

CREATE INDEX idx_pica_integrations_platform ON pica_integrations.pica_integrations(platform_name) 
  WHERE deleted_at IS NULL;

-- -----------------------------------------------------------------------------
-- Table: organization_integration_settings
-- Purpose: Tracks which integrations are enabled at the organization level
-- Notes: Organization must enable an integration before users can connect
-- -----------------------------------------------------------------------------
CREATE TABLE pica_integrations.organization_integration_settings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Organization that owns this integration setting
  organization_id UUID NOT NULL,
  
  -- Which integration this setting applies to (references pica_integrations table)
  pica_integration_id UUID NOT NULL,
  
  -- Whether the integration is enabled for this organization
  -- Users can only connect if their org has enabled the integration
  is_enabled BOOLEAN DEFAULT false,
  
  -- User who enabled the integration (for audit trail)
  enabled_by UUID DEFAULT NULL,
  
  -- User who disabled the integration (for audit trail)
  disabled_by UUID DEFAULT NULL,
  
  -- Timestamp when integration was enabled
  enabled_at TIMESTAMPTZ DEFAULT NULL,
  
  -- Timestamp when integration was disabled
  disabled_at TIMESTAMPTZ DEFAULT NULL,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ DEFAULT NULL,

  CONSTRAINT fk_org_integration_organization 
    FOREIGN KEY (organization_id) REFERENCES public.organizations(id),
  CONSTRAINT fk_org_pica_integrations 
    FOREIGN KEY (pica_integration_id) REFERENCES pica_integrations.pica_integrations(id),
  CONSTRAINT fk_org_integration_enabled_by 
    FOREIGN KEY (enabled_by) REFERENCES public.users(id),
  CONSTRAINT fk_org_integration_disabled_by 
    FOREIGN KEY (disabled_by) REFERENCES public.users(id),
  
  -- Each organization can only have one setting per integration
  UNIQUE(organization_id, pica_integration_id)
);

CREATE INDEX idx_org_settings_org_enabled 
  ON pica_integrations.organization_integration_settings(organization_id, is_enabled) 
  WHERE deleted_at IS NULL;

-- -----------------------------------------------------------------------------
-- Table: user_pica_integration_connections
-- Purpose: Stores individual user connections to third-party integrations
-- Notes: Contains encrypted OAuth connection keys from PicaOS
-- -----------------------------------------------------------------------------
CREATE TABLE pica_integrations.user_pica_integration_connections (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- User who owns this connection
  user_id UUID NOT NULL,

  -- Stable PicaOS user identifier - used for all PicaOS API calls
  -- Generated once and never changes, even if user reconnects
  -- This is YOUR stable identifier that you send to PicaOS
  pica_user_id UUID NOT NULL DEFAULT uuid_generate_v4(),
  
  -- Which integration this connection is for (Outlook, Google Calendar, etc.)
  pica_integration_id UUID NOT NULL,

  -- Encrypted connection key from PicaOS - used to authenticate API calls
  -- PicaOS maps this to the user's OAuth token
  -- Sent in x-pica-connection-key header for all passthrough API calls
  pica_connection_key TEXT,
  
  -- Whether this connection is currently active and can be used for sync
  is_active BOOLEAN DEFAULT false,
  
  -- Timestamp when user first connected via AuthKit
  connected_at TIMESTAMPTZ DEFAULT NULL,
  
  -- Timestamp when user disconnected or connection failed
  disconnected_at TIMESTAMPTZ DEFAULT NULL,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ DEFAULT NULL,

  CONSTRAINT fk_user_integration_user 
    FOREIGN KEY (user_id) REFERENCES public.users(id),
  CONSTRAINT fk_user_pica_integrations 
    FOREIGN KEY (pica_integration_id) REFERENCES pica_integrations.pica_integrations(id),
  
  -- Each user can only have one active connection per integration with a given pica_user_id
  UNIQUE(user_id, pica_user_id, pica_integration_id)
);

CREATE INDEX idx_user_connections_user_active 
  ON pica_integrations.user_pica_integration_connections(user_id, is_active) 
  WHERE deleted_at IS NULL;

CREATE INDEX idx_user_connections_status 
  ON pica_integrations.user_pica_integration_connections(connection_status) 
  WHERE deleted_at IS NULL AND is_active = true;

-- -----------------------------------------------------------------------------
-- Table: schedule_sync_status
-- Purpose: Tracks sync status for each schedule/event per user
-- Notes: One record per schedule per user - allows independent sync states
--
-- IDEMPOTENCY HANDLING:
-- - idempotency_key is checked BEFORE processing any sync (individual or batch)
-- - For individual syncs: Worker checks if key exists, skips if already processed
-- - For batch syncs: Worker filters OUT schedules with existing keys, batches rest
-- - Key format: scheduleId:userId:vN:action[:retryN]
-- - UNIQUE constraint prevents race conditions between workers
-- -----------------------------------------------------------------------------
CREATE TABLE pica_integrations.schedule_sync_status (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- The schedule/event being synced
  schedule_id UUID NOT NULL,
  
  -- User's integration connection (links to their Outlook connection)
  user_pica_integration_connections_id UUID NOT NULL,
  
  -- The external event ID in the third-party system (e.g., Outlook event ID)
  -- NULL until first successful sync, then used for updates/deletes
  external_event_id TEXT DEFAULT NULL,

  -- Timestamp of last successful sync
  last_sync_at TIMESTAMPTZ DEFAULT NULL,
  
  -- Last operation performed: 'CREATE', 'UPDATE', or 'DELETE'
  last_sync_action VARCHAR(20) DEFAULT NULL,
  
  -- Version counter - increments with each schedule change
  -- Used to generate unique idempotency keys when schedule data changes
  -- sync_version INTEGER DEFAULT 1,
  
  -- Last error message if sync failed
  last_error TEXT DEFAULT NULL,

  -- Number of times sync has been retried (only increments on failure)
  retry_count INTEGER DEFAULT 0,

  -- Scheduled time for next retry attempt (NULL if not scheduled)
  -- Used by retry cron job to find due retries
  -- Set by worker when sync fails with retryable error
  next_retry_at TIMESTAMPTZ DEFAULT NULL,
  
  -- IDEMPOTENCY KEY - Critical for deduplication
  -- Format: scheduleId:userId:vN:action[:retryN]
  -- Examples:
  --   "schedule-123:alice-id:v1:CREATE"         (initial create)
  --   "schedule-123:alice-id:v2:UPDATE"         (after data change)
  --   "schedule-123:alice-id:v2:UPDATE:retry1"  (first retry)
  --
  -- When checked:
  --   - BEFORE processing individual sync (skip if exists)
  --   - BEFORE adding to batch (exclude from batch if exists)
  --
  -- When stored:
  --   - BEFORE making API call (protects against redelivery)
  --   - For individual syncs: store immediately before API call
  --   - For batch syncs: store ALL keys before batch API call
  --
  -- UNIQUE constraint provides database-level race condition protection
  idempotency_key VARCHAR(255) UNIQUE,

  -- Hash of event data - used to detect if event actually changed
  -- Avoids unnecessary sync API calls if data hasn't changed
  event_hash TEXT DEFAULT NULL, - include facilitators as well
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ DEFAULT NULL,

  CONSTRAINT fk_schedule_sync_schedule 
    FOREIGN KEY (schedule_id) REFERENCES public.schedule(id),
  CONSTRAINT fk_schedule_sync_connection 
    FOREIGN KEY (user_pica_integration_connections_id) 
      REFERENCES pica_integrations.user_pica_integration_connections(id),
  
  -- Ensure one sync status per schedule per user connection
  UNIQUE(schedule_id, user_pica_integration_connections_id)
);

-- Index for finding all users who need syncing when a schedule changes (individual sync)
-- Used by API when publishing individual sync messages
CREATE INDEX idx_schedule_sync_schedule 
  ON pica_integrations.schedule_sync_status(schedule_id) 
  WHERE deleted_at IS NULL;

-- Index for finding all syncs for a specific user connection (batch sync queries)
-- Used by worker when pulling pending syncs for batch processing
CREATE INDEX idx_schedule_sync_connection 
  ON pica_integrations.schedule_sync_status(user_pica_integration_connections_id) 
  WHERE deleted_at IS NULL;

-- Index for retry cron job - finds user connections with pending retries
-- Used to identify which users need batch retry messages published
CREATE INDEX idx_schedule_sync_retry 
  ON pica_integrations.schedule_sync_status(next_retry_at) 
  WHERE deleted_at IS NULL AND next_retry_at IS NOT NULL;

-- Composite index for fast lookup of specific schedule + user connection
-- Used for idempotency checks in both individual and batch operations
CREATE INDEX idx_schedule_sync_schedule_connection 
  ON pica_integrations.schedule_sync_status(schedule_id, user_pica_integration_connections_id) 
  WHERE deleted_at IS NULL;
```

