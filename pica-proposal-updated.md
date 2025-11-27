# PicaOS Outlook Integration - System Proposal

## Executive Summary
Integration system for syncing calendar events between our platform and Microsoft Outlook using PicaOS as the integration backend. Initial phase focuses on **pushing events TO Outlook** with **real-time toast notifications** (via existing app-events service) on sync completion, with future expansion for **pulling events FROM Outlook**. Leverages existing platform infrastructure for notifications.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND                                      │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   AuthKit    │  │  Integration │  │  Connection  │  │   Toast    │ │
│  │   Connect    │  │   Settings   │  │    Status    │  │ Notification│ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
         │                   │                  │                 ▲
         │ Connect           │ Configure        │ Check Status    │ 
         ▼                   ▼                  ▼                 │ 
                                                                  │ Notification
                                                                  │    (via app-events)
┌─────────────────────────────────────────────────────────────────────────┐
│                         NestJS API Backend                              │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              Connection Controller                               │  │
│  │  POST   /api/pica/connections/store                              │  │
│  │  GET    /api/pica/connections/:id/status  ← NEW! Real-time      │  │
│  │         (Calls PicaOS /vault/connections API)                    │  │
│  │  DELETE /api/pica/connections/:id                                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │         Integration Settings Controller                          │  │
│  │  GET    /api/pica/integrations                                   │  │
│  │  POST   /api/pica/integrations/:id/enable                        │  │
│  │  POST   /api/pica/integrations/:id/disable                       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Schedule Service                              │  │
│  │  - Create/Update/Delete events - ??                                  │  │
│  │  - Publishes to outlook.sync.requested on changes               │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Kafka Producer/Consumer                      │    │
│  │  - Publishes to outlook.sync.requested (sync requests)          │    │
│  │  - Consumes outlook.sync.completed (sync results)               │    │
│  │  - Publishes to app-events (user notifications)                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
         │                                                     ▲
         │ Produce Sync Events                                │ Consume Results
         ▼                                                     │
┌─────────────────────────────────────────────────────────────────────────┐
│                            Kafka Cluster                                │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐       │
│  │ outlook.sync     │  │ outlook.sync     │  │ outlook.webhook  │       │
│  │ .requested       │  │ .completed       │  │ .received        │       │
│  │ (Push to Outlook)│  │ (Results)        │  │ (Future: Pull)   │       │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘       │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────────────────────────────┐   │
│  │ outlook.sync.dlq │  │ app-events (Existing service)            │   │
│  │ (Failed syncs)   │  │ - User notifications (toast messages)    │   │
│  └──────────────────┘  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
         │                                                     │
         │ Consume                                            │ Produce
         ▼                                                     │
┌─────────────────────────────────────────────────────────────────────────┐
│                   Outlook Sync Worker Service(s)                        │
│                         (Horizontally Scalable)                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │               Kafka Consumer (Consumer Group)                    │  │
│  │  - Consumes from outlook.sync.requested                         │  │
│  │  - Processes BOTH initial syncs AND retries (same topic)        │  │
│  │  - Maintains idempotency with deduplication keys                │  │
│  │  - Multiple instances for horizontal scaling                    │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Retry Scheduler (Cron)                        │  │
│  │  - Runs every minute (ONLY on leader instance)                  │  │
│  │  - Query: SELECT WHERE next_retry_at <= NOW()                   │  │
│  │  - Republishes to Kafka: outlook.sync.requested                 │  │
│  │  - Clears next_retry_at in DB                                   │  │
│  │  - Uses leader election to ensure single execution              │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Sync Orchestrator                             │  │
│  │  - Rate limiting per connection                                 │  │
│  │  - Batch processing (per user, up to 20 operations)             │  │
│  │  - Idempotency checking                                         │  │
│  │  - On failure: SCHEDULE retry (set next_retry_at in DB)         │  │
│  │  - Does NOT retry immediately                                   │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              PicaOS API Client (Simple HTTP Wrapper)            │  │
│  │  - POST /v1/passthrough/$batch (batch operations)               │  │
│  │  - POST /v1/passthrough (individual operations)                 │  │
│  │  - Headers: x-pica-secret, x-pica-connection-key                │  │
│  │  - Error handling and classification                            │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
         │                          ▲
         │ API Calls to PicaOS      │ Retry cron republishes to Kafka
         ▼                          │ (consumed by same workers)
         │                                                     │
         │ Sync Events + Status Checks                        │ Results
         ▼                                                     │
┌─────────────────────────────────────────────────────────────────────────┐
│                         PicaOS Platform                                 │
│                   https://api.picaos.com/v1                             │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │   Vault          │  │  Passthrough │  │   Webhooks   │            │
│  │  /vault/         │  │   /proxy/    │  │  (Future)    │            │
│  │  connections ←───┼──┼──────────────┼──┼──────────────┤            │
│  │                  │  │              │  │  Status Check │            │
│  │  GET /vault/     │  │  POST /proxy │  │  from API     │            │
│  │  connections     │  │  /microsoft  │  │  Backend      │            │
│  │  ?userId=X       │  │  -outlook/   │  │              │            │
│  │  (Returns state) │  │  calendar/   │  │              │            │
│  └──────────────────┘  └──────────────┘  └──────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
         │                                                     
         │ OAuth & API Calls                                  
         ▼                                                     
┌─────────────────────────────────────────────────────────────────────────┐
│                       Microsoft Outlook API                             │
│                   (Microsoft Graph API)                                 │
│                                                                         │
│  - Calendar Events CRUD                                                │
│  - Real-time change notifications (webhooks)                           │
│  - Batch operations support                                            │
└─────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────┐
│                         PostgreSQL Database                             │
│                      Schema: pica_integrations                          │
│                                                                         │
│  [Tables detailed in Database Design section below]                    │
└─────────────────────────────────────────────────────────────────────────┘
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
Frontend: GET /api/pica/connections/:id/status
    ↓
Connection Controller
    ↓
Check cache (< 5 min old?)
    ├─ Yes → Return cached status from DB
    │
    └─ No → Call PicaOS API
        ↓
    GET /v1/vault/connections?userId={pica_user_id}
        ↓
    PicaOS returns: { state: "operational", email: "user@example.com", ... }
        ↓
    Update DB with fresh state
        ↓
    Return to Frontend
        ↓
    Display status badge (Green ✓ / Yellow ⚠ / Red ✗)
```

**User views integration settings or encounters sync error:**

```typescript
// Frontend request
GET /api/pica/connections/{connectionId}/status

// Backend Controller Implementation
@Get('connections/:id/status')
async getConnectionStatus(
  @Param('id') connectionId: string,
  @CurrentUser() user: User
) {
  // 1. Get connection from DB
  const connection = await db.findConnection(connectionId, user.id);
  
  if (!connection) {
    throw new NotFoundException('Connection not found');
  }

  // 2. Get real-time status from PicaOS
  const picaStatus = await picaClient.getConnectionStatus(connection.pica_user_id);
  const outlookConnection = picaStatus.connections.find(
    c => c.platform === 'microsoft-outlook'
  );

  if (!outlookConnection) {
    throw new NotFoundException('Outlook connection not found in PicaOS');
  }

  // 3. Map PicaOS state to our response
  const statusMap = {
    'operational': 'Connection is healthy',
    'degraded': 'OAuth token refresh in progress',
    'failed': 'Reconnection required',
    'unknown': 'Unable to determine status'
  };

  // 4. Update our DB with latest status (optional but recommended)
  await db.updateConnection(connectionId, {
    connection_status: outlookConnection.state,
    last_validated_at: new Date()
  });

  // 5. Return to frontend
  return {
    connectionId: connection.id,
    userId: user.id,
    integration: 'microsoft-outlook',
    status: outlookConnection.state,
    statusDescription: statusMap[outlookConnection.state],
    lastChecked: new Date().toISOString(),
    connectedAt: connection.connected_at,
    email: outlookConnection.email
  };
}
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

### Individual Sync Flow (Real-Time)

**When:** User creates/updates/deletes a schedule

```typescript
// NestJS API - Schedule Service
@Injectable()
export class ScheduleService {
  async onScheduleChanged(scheduleId: string, action: 'CREATE' | 'UPDATE' | 'DELETE') {
    // API calculates affected users
    const affectedUsers = await this.db.query(`
      SELECT 
        u.id as user_id,
        u.pica_user_id,
        c.id as connection_id,
        c.pica_connection_key,
        s.external_event_id,
        s.sync_version
      FROM schedule_sync_status s
      JOIN user_pica_integration_connections c 
        ON s.user_pica_integration_connections_id = c.id
      WHERE s.schedule_id = $1 
        AND c.is_active = true
        AND c.deleted_at IS NULL
    `, [scheduleId]);
    
    // Publish ONE message PER USER (API provides all details)
    for (const user of affectedUsers) {
      const idempotencyKey = `${scheduleId}:${user.user_id}:v${user.sync_version}:${action}`;
      
      await this.kafka.publish('outlook.sync.requested', {
        type: 'individual_sync',
        scheduleId,
        userId: user.user_id,
        picaUserId: user.pica_user_id,
        connectionId: user.connection_id,
        action,
        externalEventId: user.external_event_id,
        priority: 'high',
        idempotencyKey,
        timestamp: new Date().toISOString()
      });
    }
  }
}
```

**Worker Processing:**

```typescript
async function processIndividualSync(message: IndividualSyncMessage) {
  const { scheduleId, userId, connectionId, action, idempotencyKey } = message;
  
  // 1. Check idempotency - already processed?
  const existing = await db.findOne('schedule_sync_status', {
    schedule_id: scheduleId,
    user_pica_integration_connections_id: connectionId,
    idempotency_key: idempotencyKey
  });
  
  if (existing) {
    logger.info('sync.duplicate_detected', { scheduleId, userId, idempotencyKey });
    return; // Skip - already processed
  }
  
  // 2. Store idempotency key BEFORE API call
  await db.update('schedule_sync_status', {
    idempotency_key: idempotencyKey,
    last_sync_action: action,
    updated_at: new Date()
  }, {
    schedule_id: scheduleId,
    user_pica_integration_connections_id: connectionId
  });
  
  // 3. Execute sync (individual API call)
  const connection = await getConnection(connectionId);
  await picaClient.updateOutlookEvent(
    connection.pica_connection_key,
    message.externalEventId,
    eventData
  );
  
  // 4. Update success status
  await updateSyncStatus(scheduleId, connectionId, 'synced');
}
```

### Batch Sync Flow (Background)

**When:** Retry cron runs every minute, finds users with pending retries

```typescript
// Worker Service - Retry Scheduler (Cron)
@Injectable()
export class RetryScheduler {
  @Cron('* * * * *') // Every minute
  async processPendingRetries() {
    // Only run on leader instance
    if (!this.leaderElection.isLeader()) return;
    
    // Worker queries to find users with pending retries
    const usersWithRetries = await this.db.query(`
      SELECT DISTINCT 
        user_pica_integration_connections_id,
        COUNT(*) as pending_count
      FROM schedule_sync_status
      WHERE next_retry_at IS NOT NULL
        AND next_retry_at <= NOW()
        AND retry_count < 5
        AND deleted_at IS NULL
      GROUP BY user_pica_integration_connections_id
      HAVING COUNT(*) > 0
    `);
    
    // Publish ONE batch message PER USER
    for (const user of usersWithRetries) {
      await this.kafka.publish('outlook.sync.requested', {
        type: 'batch_sync',
        connectionId: user.user_pica_integration_connections_id,
        priority: 'low',
        reason: 'retry',
        timestamp: new Date().toISOString()
      });
      
      logger.info('batch.scheduled', {
        connectionId: user.user_pica_integration_connections_id,
        pendingCount: user.pending_count
      });
    }
  }
}
```

**Worker Processing:**

```typescript
async function processBatchSync(message: BatchSyncMessage) {
  const { connectionId, reason } = message;
  
  // 1. Worker queries to find all pending syncs for this user
  const pendingSyncs = await db.query(`
    SELECT 
      s.*,
      sched.title,
      sched.start_time,
      sched.end_time
    FROM schedule_sync_status s
    JOIN schedule sched ON s.schedule_id = sched.id
    WHERE s.user_pica_integration_connections_id = $1
      AND s.next_retry_at <= NOW()
      AND s.deleted_at IS NULL
    ORDER BY s.next_retry_at ASC
    LIMIT 20  -- Microsoft Graph batch limit
  `, [connectionId]);
  
  if (pendingSyncs.length === 0) {
    return; // Nothing to do
  }
  
  // 2. Filter out already-processed syncs (idempotency check)
  const unprocessedSyncs = [];
  for (const sync of pendingSyncs) {
    const idempotencyKey = `${sync.schedule_id}:${sync.user_id}:v${sync.sync_version}:UPDATE:retry${sync.retry_count + 1}`;
    
    // Check if this retry attempt already has an idempotency key
    const alreadyProcessed = await db.findOne('schedule_sync_status', {
      schedule_id: sync.schedule_id,
      idempotency_key: idempotencyKey
    });
    
    if (alreadyProcessed) {
      logger.info('batch.item_already_processed', {
        scheduleId: sync.schedule_id,
        idempotencyKey
      });
      continue; // Skip this one
    }
    
    unprocessedSyncs.push({ ...sync, idempotencyKey });
  }
  
  if (unprocessedSyncs.length === 0) {
    logger.info('batch.all_processed', { connectionId });
    return; // All already processed
  }
  
  // 3. Store idempotency keys BEFORE batch API call
  for (const sync of unprocessedSyncs) {
    await db.update('schedule_sync_status', {
      idempotency_key: sync.idempotencyKey,
      last_sync_action: 'UPDATE',
      updated_at: new Date()
    }, {
      schedule_id: sync.schedule_id,
      user_pica_integration_connections_id: connectionId
    });
  }
  
  // 4. Build batch request (only unprocessed items)
  const batchRequests = unprocessedSyncs.map((sync, index) => ({
    id: String(index),
    method: 'PATCH',
    url: `/me/calendar/events/${sync.external_event_id}`,
    body: transformToOutlookFormat(sync),
    headers: { "Content-Type": "application/json" }
  }));
  
  // 5. Execute batch API call
  const connection = await getConnection(connectionId);
  try {
    const response = await picaClient.batchSync(
      connection.pica_connection_key,
      batchRequests
    );
    
    // 6. Handle each result individually (partial success supported)
    await handleBatchResponse(unprocessedSyncs, response.responses);
    
  } catch (error) {
    // Entire batch failed - handle gracefully
    await handleBatchFailure(unprocessedSyncs, error);
  }
}
```

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

### Benefits of Hybrid Approach

**API Publishes Individual Messages (Real-time):**
- ✅ Low latency - worker can process immediately
- ✅ API has context about what changed
- ✅ Clear accountability - one message = one sync
- ✅ Better observability and debugging

**Worker Publishes Batch Messages (Background):**
- ✅ Efficient batching - up to 20 operations per API call
- ✅ Reduces Kafka message volume (1 msg vs. 20 msgs)
- ✅ Optimizes background processing
- ✅ Worker can intelligently group retries

**Overall:**
- ✅ Best of both worlds (latency + efficiency)
- ✅ Clear separation of concerns
- ✅ Scalable for real-time and batch scenarios
- ✅ Idempotency protection at both levels

---

## Database Design - Enhanced Schema

### Suggested:

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
  
  -- Global availability toggle - controls if integration can be enabled by orgs
  -- Set to false to disable an integration system-wide
  is_available BOOLEAN DEFAULT true,
  
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
  sync_version INTEGER DEFAULT 1,
  
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
  event_hash TEXT DEFAULT NULL,
  
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

-- -----------------------------------------------------------------------------
-- Table: sync_audit_log
-- Purpose: Complete audit trail of all sync attempts and their outcomes
-- Notes: Used for debugging, monitoring, and compliance
--        Records EVERY sync attempt (including retries)
-- -----------------------------------------------------------------------------
CREATE TABLE pica_integrations.sync_audit_log (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Links to the sync status record this audit entry belongs to
  schedule_sync_status_id UUID NOT NULL,
  
  -- Action attempted: 'CREATE', 'UPDATE', or 'DELETE'
  action VARCHAR(20) NOT NULL,
  
  -- Result of the sync attempt: 'success', 'error', 'retrying'
  status VARCHAR(50) NOT NULL,
  
  -- Full request payload sent to PicaOS (for debugging)
  -- Contains the complete event data sent
  request_payload JSONB DEFAULT NULL,
  
  -- Full response received from PicaOS (success or error)
  -- Includes Microsoft Graph's response (via PicaOS passthrough)
  response_payload JSONB DEFAULT NULL,
  
  -- Error message if sync failed
  error_message TEXT DEFAULT NULL,
  
  -- Error code for classification (e.g., 'RATE_LIMIT', 'INVALID_TOKEN')
  -- Used for error analysis and automated handling
  error_code VARCHAR(100) DEFAULT NULL,
  
  -- Time taken to complete the sync operation (milliseconds)
  -- Used for performance monitoring
  duration_ms INTEGER DEFAULT NULL,
  
  -- Which retry attempt this was (1 = first attempt, 2+ = retries)
  attempt_number INTEGER DEFAULT 1,
  
  -- Timestamp of this audit entry
  created_at TIMESTAMPTZ DEFAULT NOW(),

  CONSTRAINT fk_audit_schedule_sync 
    FOREIGN KEY (schedule_sync_status_id) REFERENCES pica_integrations.schedule_sync_status(id)
);

-- Index for finding audit logs for a specific sync status
CREATE INDEX idx_audit_schedule_sync 
  ON pica_integrations.sync_audit_log(schedule_sync_status_id);

-- Index for time-based queries (e.g., recent errors, performance monitoring)
CREATE INDEX idx_audit_created 
  ON pica_integrations.sync_audit_log(created_at DESC);

-- Index for error analysis
CREATE INDEX idx_audit_status_error 
  ON pica_integrations.sync_audit_log(status, error_code) 
  WHERE status = 'error';

-- -----------------------------------------------------------------------------
-- Table: integration_rate_limits
-- Purpose: Tracks API rate limit usage per user connection
-- Notes: Prevents exceeding third-party API rate limits (e.g., Microsoft Graph)
-- -----------------------------------------------------------------------------
CREATE TABLE pica_integrations.integration_rate_limits (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- User connection this rate limit tracking applies to
  user_connection_id UUID NOT NULL,
  
  -- Start of the rate limit window (e.g., top of the hour)
  window_start TIMESTAMPTZ NOT NULL,
  
  -- End of the rate limit window
  window_end TIMESTAMPTZ NOT NULL,
  
  -- Number of API requests made in this window
  -- Incremented with each sync operation
  request_count INTEGER DEFAULT 0,
  
  -- Number of requests remaining (from API response headers)
  -- e.g., Microsoft Graph returns X-RateLimit-Remaining header
  limit_remaining INTEGER DEFAULT NULL,
  
  -- Timestamp when the rate limit resets (from API response headers)
  -- e.g., Microsoft Graph returns X-RateLimit-Reset header
  limit_reset_at TIMESTAMPTZ DEFAULT NULL,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),

  CONSTRAINT fk_rate_limit_connection 
    FOREIGN KEY (user_connection_id) REFERENCES pica_integrations.user_pica_integration_connections(id),
  
  -- Each connection can only have one rate limit record per time window
  UNIQUE(user_connection_id, window_start)
);

-- Index for checking current rate limit window for a connection
CREATE INDEX idx_rate_limits_window 
  ON pica_integrations.integration_rate_limits(user_connection_id, window_end);

-- Index for cleanup job (remove old windows)
CREATE INDEX idx_rate_limits_cleanup 
  ON pica_integrations.integration_rate_limits(window_end) 
  WHERE window_end < NOW();
```

### Database Schema Summary

**Key Tables & Their Roles:**

1. **`pica_integrations`** - Reference table of available integrations (Outlook, Google Calendar, etc.)
2. **`organization_integration_settings`** - Org-level toggle to enable/disable integrations
3. **`user_pica_integration_connections`** - User's connection to Outlook (stores encrypted connection key)
4. **`schedule_sync_status`** - Tracks sync state per schedule per user (the heart of the system)
5. **`sync_audit_log`** - Complete audit trail of all sync attempts (debugging & compliance)
6. **`integration_rate_limits`** - Rate limit tracking per user (prevent API throttling)

**Critical Design Decisions:**

| Decision | Rationale |
|----------|-----------|
| **One sync record per schedule per user** | Allows independent sync states (Alice synced, Bob failed) |
| **`idempotency_key` with UNIQUE constraint** | Prevents duplicate syncs, provides database-level protection |
| **`sync_version` increments on change** | Generates new idempotency keys when schedule data changes |
| **`next_retry_at` for scheduled retries** | Enables background retry job without immediate reprocessing |
| **Separate audit log table** | Unbounded growth without impacting sync_status table performance |
| **`user_pica_integration_connections_id` FK** | Direct link from sync to connection (no need for user_id lookup) |

**How Idempotency Works with the Schema:**

```sql
-- 1. Individual Sync: Check if already processed
SELECT * FROM schedule_sync_status 
WHERE schedule_id = $1 
  AND user_pica_integration_connections_id = $2
  AND idempotency_key = $3;
-- If exists → skip processing

-- 2. Batch Sync: Find unprocessed schedules
SELECT * FROM schedule_sync_status
WHERE user_pica_integration_connections_id = $1
  AND (
    -- New syncs (no key yet)
    idempotency_key IS NULL
    OR 
    -- Failed syncs ready for retry
    (next_retry_at <= NOW() AND retry_count < 5)
  )
LIMIT 20;
-- Only these go into batch API call

-- 3. Store idempotency key BEFORE API call
UPDATE schedule_sync_status
SET idempotency_key = $1,
    last_sync_action = $2,
    updated_at = NOW()
WHERE schedule_id = $3 
  AND user_pica_integration_connections_id = $4;
-- Protects against Kafka redelivery
```

---

## Multi-User Sync Explained

### Why schedule_sync_status needs user_id and pica_user_id

When a schedule/event involves multiple users (e.g., a team meeting), **each user with Outlook integration gets their own sync record**:

```
ONE SCHEDULE → MULTIPLE SYNC RECORDS (one per user)
```

**Example: Team Meeting with 3 Attendees**

```typescript
// 1. Your main schedule (one record)
public.schedule {
  id: "schedule-123",
  title: "Team Meeting",
  start_time: "2025-11-27 10:00:00",
  // attended by: Alice, Bob, Charlie
}

// 2. Each user gets their own sync status (three records)
pica_integrations.schedule_sync_status [
  {
    // Alice's sync
    schedule_id: "schedule-123",
    user_id: "alice-id",
    pica_user_id: "pica-alice",
    external_event_id: "outlook-alice-event-123", // Alice's Outlook
    sync_status: "synced" ✓
  },
  {
    // Bob's sync
    schedule_id: "schedule-123",
    user_id: "bob-id",
    pica_user_id: "pica-bob",
    external_event_id: "outlook-bob-event-456", // Bob's Outlook
    sync_status: "synced" ✓
  },
  {
    // Charlie's sync
    schedule_id: "schedule-123",
    user_id: "charlie-id",
    pica_user_id: "pica-charlie",
    external_event_id: "outlook-charlie-event-789", // Charlie's Outlook
    sync_status: "error" ✗  // Connection expired!
  }
]
```

**Result:**
- ✅ Alice sees the meeting in HER Outlook calendar
- ✅ Bob sees the meeting in HIS Outlook calendar
- ❌ Charlie's sync failed (needs to reconnect)

### Benefits of This Design

**1. Independent Sync States**
```sql
-- Same schedule, different states per user
SELECT user_id, sync_status FROM schedule_sync_status 
WHERE schedule_id = 'schedule-123';

-- Results:
-- alice-id   | synced
-- bob-id     | synced
-- charlie-id | error
```

**2. Easy User-Specific Queries**
```sql
-- "Show me all MY synced schedules"
SELECT s.* FROM schedule s
JOIN schedule_sync_status sss ON s.id = sss.schedule_id
WHERE sss.user_id = 'alice-id' 
  AND sss.sync_status = 'synced';

-- "Which users need to be updated when schedule changes?"
SELECT user_id, pica_user_id, external_event_id
FROM schedule_sync_status
WHERE schedule_id = 'schedule-123'
  AND sync_status = 'synced';
```

**3. Independent Sync Operations**
```typescript
// When schedule updates:
async function syncScheduleToAllUsers(scheduleId: string) {
  const syncRecords = await getSyncRecords(scheduleId);
  
  for (const record of syncRecords) {
    // Each user gets their own Kafka message
    await publishToKafka('outlook.sync.requested', {
      scheduleId,
      userId: record.user_id,
      picaUserId: record.pica_user_id,
      externalEventId: record.external_event_id, // Update their Outlook event
      action: 'UPDATE'
    });
  }
}
```

### Key Fields Explained

| Field | Purpose | Example |
|-------|---------|---------|
| `schedule_id` | Links to the shared schedule | `schedule-123` |
| `user_id` | Which user this sync belongs to | `alice-id` |
| `pica_user_id` | PicaOS user ID (for API calls) | `pica-alice` |
| `user_connection_id` | User's Outlook connection | `conn-abc` |
| `external_event_id` | Outlook event ID (unique per user) | `outlook-alice-event-123` |
| `sync_status` | Sync state for THIS user | `synced`, `error`, `pending` |

### Unique Constraint

```sql
UNIQUE(schedule_id, user_id)  -- One sync record per schedule per user
```

This ensures:
- ✅ Each user can only have ONE sync status per schedule
- ✅ No duplicate sync records
- ✅ Easy to query: "What's the sync status for this schedule and this user?"

---

## Key Changes & Improvements

### 1. **Database Schema Enhancements**

**Current Issues:**
- ❌ Column name inconsistency: `pica_selected_connection_key` vs `pica_connection_key`
- ❌ Missing connection status tracking
- ❌ No audit trail for sync operations
- ❌ Limited error tracking and retry logic
- ❌ No idempotency support
- ❌ Schedule table doesn't link to user connection

**Improvements:**
- ✅ Renamed to consistent naming: `pica_connection_key`
- ✅ Added `connection_status` enum field
- ✅ Added `sync_audit_log` table for complete audit trail
- ✅ Added `idempotency_key` for deduplication
- ✅ Added `event_hash` for efficient change detection
- ✅ Added `user_id` and `pica_user_id` to schedule_sync_status (supports multi-user sync)
- ✅ Added `user_connection_id` FK in schedule_sync_status
- ✅ Added UNIQUE constraint on (schedule_id, user_id) to ensure one sync per user per schedule
- ✅ Added `integration_rate_limits` table
- ✅ Added `metadata` JSONB fields for flexibility
- ✅ Added proper indexes for query performance

### 2. **Kafka Topic Structure**

**Current:**
- Generic "sync" topic

**Improved:**
```
outlook.sync.requested      - Sync requests from API
outlook.sync.completed      - Successful sync results
outlook.sync.failed         - Failed syncs (after all retries)
outlook.sync.dlq            - Dead letter queue
outlook.webhook.received    - Future: Incoming webhook events
outlook.batch.scheduled     - Future: Scheduled batch syncs

app-events                  - EXISTING: User notifications (consumed by app-events service)
```

### 3. **Idempotency & Deduplication**

**Strategy:**
```typescript
// Generate idempotency key (includes userId to make it unique per user!)
const idempotencyKey = `${scheduleId}:${userId}:${syncVersion}:${action}`;
// Example: "schedule-123:alice-id:v2:UPDATE"

// Before processing in worker
const existing = await checkIdempotencyKey(idempotencyKey);
if (existing) {
  logger.info('Duplicate event detected, skipping', {
    scheduleId,
    userId,
    action
  });
  return existing.result;
}

// Why include userId?
// - Same schedule syncs to multiple users' Outlook calendars
// - Each user's sync must be processed independently
// - Prevents cross-user sync deduplication errors
```

### 4. **Retry Logic with Exponential Backoff**

**Two-Part System:**
1. **Schedule retries** when sync fails (store `next_retry_at`)
2. **Execute retries** via background job that republishes to Kafka

```typescript
const retryConfig = {
  maxAttempts: 5,
  initialDelay: 1000, // 1 second
  maxDelay: 300000,   // 5 minutes
  backoffMultiplier: 2,
  
  retryableErrors: [
    'RATE_LIMIT_EXCEEDED',
    'SERVICE_UNAVAILABLE',
    'TIMEOUT',
    'NETWORK_ERROR'
  ],
  
  permanentErrors: [
    'INVALID_TOKEN',
    'CONNECTION_REVOKED',
    'PERMISSION_DENIED',
    'INVALID_EVENT_DATA'
  ]
};

function calculateNextRetry(attemptNumber: number): Date {
  const delay = Math.min(
    retryConfig.initialDelay * Math.pow(retryConfig.backoffMultiplier, attemptNumber),
    retryConfig.maxDelay
  );
  return new Date(Date.now() + delay);
}

// PART 1: When sync fails, schedule retry
async function handleSyncFailure(sync: SyncStatus, error: Error) {
  const errorType = classifyError(error);
  
  if (retryConfig.permanentErrors.includes(errorType)) {
    // Don't retry - user action needed
    await db.update('schedule_sync_status', {
      sync_status: 'error',
      last_error: error.message,
      retry_count: 999,  // Mark as non-retryable
      next_retry_at: null
    }, { id: sync.id });
    
    await appEvents.notifyConnectionError(sync.user_id);
    return;
  }
  
  if (retryConfig.retryableErrors.includes(errorType)) {
    const nextRetry = calculateNextRetry(sync.retry_count);
    
    await db.update('schedule_sync_status', {
      sync_status: 'error',
      last_error: error.message,
      retry_count: sync.retry_count + 1,
      next_retry_at: nextRetry  // ← Schedule for future retry
    }, { id: sync.id });
    
    logger.info('sync.retry_scheduled', {
      scheduleId: sync.schedule_id,
      userId: sync.user_id,
      attemptNumber: sync.retry_count + 1,
      nextRetryAt: nextRetry
    });
  }
}

// PART 2: Background job checks for due retries
async function retryFailedSyncs() {
  const dueSyncs = await db.query(`
    SELECT 
      ss.*,
      uc.pica_connection_key,
      uc.pica_user_id,
      s.title as schedule_title,
      s.start_time,
      s.end_time
    FROM schedule_sync_status ss
    JOIN user_pica_integration_connections uc 
      ON ss.user_pica_integration_connections_id = uc.id
    JOIN schedule s ON ss.schedule_id = s.id
    WHERE ss.next_retry_at IS NOT NULL
      AND ss.next_retry_at <= NOW()
      AND ss.retry_count < ${retryConfig.maxAttempts}
      AND ss.deleted_at IS NULL
    LIMIT 100  -- Process in batches
  `);
  
  if (dueSyncs.length === 0) return;
  
  logger.info('retry.job.processing', { count: dueSyncs.length });
  
  for (const sync of dueSyncs) {
    // Republish to Kafka for retry
    await kafka.publish('outlook.sync.requested', {
      scheduleId: sync.schedule_id,
      userId: sync.user_id,
      picaUserId: sync.pica_user_id,
      connectionKey: sync.pica_connection_key,
      externalEventId: sync.external_event_id,
      action: sync.last_sync_action || 'UPDATE',
      eventData: {
        title: sync.schedule_title,
        start_time: sync.start_time,
        end_time: sync.end_time
        // ... fetch full event data
      },
      attemptNumber: sync.retry_count + 1,
      isRetry: true,
      idempotencyKey: `${sync.schedule_id}:${sync.user_id}:v${sync.sync_version}:${sync.last_sync_action}:retry${sync.retry_count + 1}`,
      timestamp: new Date().toISOString()
    });
    
    // Clear next_retry_at (will be set again if this retry fails)
    await db.update('schedule_sync_status', {
      next_retry_at: null
    }, { id: sync.id });
  }
}

// Schedule retry job (runs every minute)
// IMPORTANT: Use leader election to ensure only ONE instance runs this
import cron from 'node-cron';
import { acquireLock, releaseLock } from './distributed-lock';

cron.schedule('* * * * *', async () => {
  // Only run on leader instance (prevents duplicate execution)
  const lockAcquired = await acquireLock('retry-scheduler', 60000); // 60s TTL
  
  if (!lockAcquired) {
    logger.debug('retry.job.skipped', { reason: 'Another instance is leader' });
    return;
  }
  
  try {
    await retryFailedSyncs();
  } catch (error) {
    logger.error('retry.job.failed', { error: error.message });
  } finally {
    await releaseLock('retry-scheduler');
  }
});

// Leader Election Options:

// Option 1: Redis-based lock
async function acquireLock(key: string, ttlMs: number): Promise<boolean> {
  const result = await redis.set(
    `lock:${key}`,
    process.env.INSTANCE_ID,
    'PX', ttlMs,  // Expire in milliseconds
    'NX'          // Only set if not exists
  );
  return result === 'OK';
}

// Option 2: PostgreSQL advisory lock
async function acquireLock(key: string): Promise<boolean> {
  const lockId = hashStringToInt(key);
  const result = await db.raw('SELECT pg_try_advisory_lock(?)', [lockId]);
  return result.rows[0].pg_try_advisory_lock;
}

// Option 3: Use NestJS @Schedule with leader election
import { Schedule, SchedulerRegistry } from '@nestjs/schedule';
import { LeaderElection } from './leader-election';

@Injectable()
export class RetryScheduler {
  constructor(private leaderElection: LeaderElection) {}
  
  @Schedule('* * * * *')  // Every minute
  async handleRetries() {
    if (!this.leaderElection.isLeader()) {
      return;  // Skip if not leader
    }
    
    await retryFailedSyncs();
  }
}

// Alternative: BullMQ with delayed jobs
import { Queue } from 'bullmq';

const retryQueue = new Queue('outlook-retry');

// When sync fails, add to queue with delay
await retryQueue.add('retry-sync', syncData, {
  delay: calculateDelay(attemptNumber)  // Milliseconds
});
```

**Visual Retry Flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│ MAIN SYNC FLOW (real-time)                                     │
└─────────────────────────────────────────────────────────────────┘
    Kafka: outlook.sync.requested
        ↓
    Sync Worker processes
        ↓
    [Calls PicaOS Passthrough]
        ↓
    ┌─────────────┬─────────────┐
    │   SUCCESS   │   FAILURE   │
    └─────────────┴─────────────┘
         │              │
         │              ↓ (If retryable error)
         │         UPDATE schedule_sync_status:
         │         - retry_count = 1
         │         - next_retry_at = NOW() + 2 seconds
         │         - sync_status = 'error'
         │              ↓
         │         [WAIT - retry NOT immediate]
         │              ↓
┌────────┴──────────────┴─────────────────────────────────────────┐
│ RETRY FLOW (background job)                                     │
└──────────────────────────────────────────────────────────────────┘
    Cron runs every minute
        ↓
    Query: SELECT * WHERE next_retry_at <= NOW()
        ↓
    Found 3 due retries
        ↓
    For each retry:
        ↓
    Republish to Kafka: outlook.sync.requested
    (with isRetry: true, attemptNumber: 2)
        ↓
    Clear next_retry_at = null
        ↓
    [Back to main sync flow]
        ↓
    Sync Worker processes retry
        ↓
    ┌─────────────┬─────────────┐
    │   SUCCESS   │   FAILURE   │
    └─────────────┴─────────────┘
         │              │
         ✓              ↓ (If retryable)
    Mark synced    Schedule next retry
                   - retry_count = 2
                   - next_retry_at = NOW() + 4 seconds
                        ↓
                   [Repeat until success or max retries]
```

**Key Points:**
- ❌ Retries are NOT immediate - they're scheduled
- ✅ Retry worker runs every minute to find due retries
- ✅ Retries use the same Kafka topic (outlook.sync.requested)
- ✅ Same sync worker processes both initial syncs and retries
- ✅ Exponential backoff via next_retry_at calculation
- ✅ Max 5 attempts total

### 5. **Rate Limiting**

**Per Connection:**
```typescript
class RateLimiter {
  async checkLimit(connectionId: string): Promise<boolean> {
    const window = await getCurrentWindow(connectionId);
    
    if (window.request_count >= LIMIT_PER_WINDOW) {
      await scheduleRetryAfter(window.limit_reset_at);
      return false;
    }
    
    await incrementCounter(connectionId);
    return true;
  }
}
```

### 6. **Change Detection**

```typescript
// Before syncing, check if event actually changed
const currentHash = hashEventData(eventData);
const lastSync = await getLastSyncStatus(scheduleId);

if (lastSync.event_hash === currentHash) {
  logger.info('No changes detected, skipping sync');
  return;
}
```

### 7. **Connection Health Monitoring**

```typescript
// Periodic health check job (runs every hour)
async function validateConnections() {
  const activeConnections = await getActiveConnections();
  
  for (const conn of activeConnections) {
    try {
      // Get connection status from PicaOS
      const picaStatus = await picaClient.getConnectionStatus(conn.pica_user_id);
      const outlookConnection = picaStatus.connections.find(
        c => c.platform === 'microsoft-outlook'
      );
      
      if (!outlookConnection) {
        await updateConnectionStatus(conn.id, 'error', 'Connection not found in PicaOS');
        continue;
      }

      // Update our DB with PicaOS state
      await updateConnectionStatus(conn.id, outlookConnection.state, null);
      await updateLastValidated(conn.id, new Date());
      
      // Notify user if connection has failed
      if (outlookConnection.state === 'failed') {
        await appEvents.notifyConnectionError(conn.user_id);
      }
      
      // Log degraded connections for monitoring
      if (outlookConnection.state === 'degraded') {
        logger.warn('connection.degraded', {
          userId: conn.user_id,
          connectionId: conn.id,
          picaUserId: conn.pica_user_id
        });
      }
      
    } catch (error) {
      logger.error('connection.health_check_failed', {
        userId: conn.user_id,
        connectionId: conn.id,
        error: error.message
      });
      await updateConnectionStatus(conn.id, 'unknown', error.message);
    }
  }
}

// Map PicaOS states to our internal connection_status
// 'operational' → 'active'
// 'degraded' → 'degraded' 
// 'failed' → 'error'
// 'unknown' → 'unknown'
```

### 8. **Security Enhancements**

**Connection Key Encryption:**
```typescript
// Store encrypted connection keys
import { encrypt, decrypt } from './crypto';

async function storeConnectionKey(userId: string, connectionKey: string) {
  const encrypted = encrypt(connectionKey, process.env.ENCRYPTION_KEY);
  await db.insert({ 
    user_id: userId, 
    pica_connection_key: encrypted 
  });
}

async function getConnectionKey(userId: string) {
  const record = await db.findOne({ user_id: userId });
  return decrypt(record.pica_connection_key, process.env.ENCRYPTION_KEY);
}
```

### 9. **Batch Processing via PicaOS Passthrough**

**🔄 Architecture: You Call PicaOS, PicaOS Calls Microsoft**

```
Your Backend 
    ↓ POST /v1/passthrough/$batch (with x-pica-connection-key)
PicaOS Passthrough API
    ↓ (maps connection key to OAuth token)
Microsoft Graph API: /$batch
    ↓ (batch response)
PicaOS passes response back to you
    ↓
Your Backend receives Microsoft's batch response
```

**What PicaOS Does:**
- ✅ Maps your connection key to user's OAuth token
- ✅ Calls Microsoft Graph API on your behalf
- ✅ Handles token refresh automatically
- ✅ Passes through Microsoft's responses unchanged

**What You Do:**
- ✅ Call PicaOS passthrough endpoints
- ✅ Provide user's connection key
- ✅ Handle Microsoft's error responses (passed through by PicaOS)

**Important: Batch operations are per-user (same connection key), not across users.**

```typescript
// Batch Strategy: Group by user, then batch their events
async function syncSchedulesToUsers(scheduleId: string, userIds: string[]) {
  // Process each user independently (cannot batch across users!)
  for (const userId of userIds) {
    await syncUserSchedules(userId);
  }
}

// For a single user with MULTIPLE schedules to sync
async function syncUserSchedules(userId: string) {
  const connection = await getUserConnection(userId);
  const pendingSyncs = await getPendingSyncsForUser(userId); // Max 20 per batch
  
  if (pendingSyncs.length === 0) return;
  
  // Use Microsoft Graph Batch API (via PicaOS passthrough)
  const batchRequests = pendingSyncs.map((sync, index) => ({
    id: String(index),
    method: sync.action === 'DELETE' ? 'DELETE' : 
            sync.external_event_id ? 'PATCH' : 'POST',
    url: sync.external_event_id 
      ? `/me/calendar/events/${sync.external_event_id}`
      : '/me/calendar/events',
    body: sync.action !== 'DELETE' ? transformToOutlookFormat(sync.eventData) : undefined,
    headers: {
      "Content-Type": "application/json"
    }
  }));
  
  try {
    // Call PicaOS passthrough API (PicaOS then calls Microsoft Graph)
    // POST https://api.picaos.com/v1/proxy/microsoft-outlook/$batch
    const response = await picaClient.batchSync(
      connection.pica_connection_key,  // PicaOS uses this to get OAuth token
      batchRequests
    );
    
    // response contains Microsoft Graph's batch response (passed through by PicaOS)
    // CRITICAL: Check each operation's status individually
    await handleBatchResponse(pendingSyncs, response.responses);
    
  } catch (error) {
    // Network error or PicaOS API error - all operations failed
    await handleBatchFailure(pendingSyncs, error);
  }
}

// Handle individual operation results from Microsoft Graph (via PicaOS)
async function handleBatchResponse(
  syncs: SyncStatus[], 
  responses: BatchResponse[]  // Microsoft's responses passed through PicaOS
) {
  for (let i = 0; i < responses.length; i++) {
    const sync = syncs[i];
    const response = responses[i];  // Each response is from Microsoft Graph
    
    if (response.status >= 200 && response.status < 300) {
      // Microsoft Graph operation succeeded!
      await db.update('schedule_sync_status', {
        sync_status: 'synced',
        external_event_id: response.body.id,
        last_sync_at: new Date(),
        last_error: null,
        retry_count: 0,
        next_retry_at: null
      }, { 
        schedule_id: sync.schedule_id, 
        user_id: sync.user_id 
      });
      
      await appEvents.notifySyncSuccess(sync.user_id, sync.schedule_title);
      
    } else if (response.status === 401 || response.status === 403) {
      // Permanent error - connection issue
      await db.update('schedule_sync_status', {
        sync_status: 'error',
        last_error: response.body.error.message,
        retry_count: 999 // Don't retry, needs reconnection
      }, { 
        schedule_id: sync.schedule_id, 
        user_id: sync.user_id 
      });
      
      await appEvents.notifyConnectionError(sync.user_id);
      
    } else if (response.status === 429 || response.status >= 500) {
      // Retryable error - schedule retry
      const nextRetry = calculateNextRetry(sync.retry_count);
      await db.update('schedule_sync_status', {
        sync_status: 'error',
        last_error: response.body.error.message,
        retry_count: sync.retry_count + 1,
        next_retry_at: nextRetry
      }, { 
        schedule_id: sync.schedule_id, 
        user_id: sync.user_id 
      });
      
    } else {
      // Other error (400, etc) - likely data issue
      await db.update('schedule_sync_status', {
        sync_status: 'error',
        last_error: response.body.error.message
      }, { 
        schedule_id: sync.schedule_id, 
        user_id: sync.user_id 
      });
      
      await appEvents.notifySyncFailure(
        sync.user_id, 
        sync.schedule_title, 
        response.body.error.message
      );
    }
  }
}

// Microsoft Graph Batch API Limits
const BATCH_LIMITS = {
  MAX_REQUESTS_PER_BATCH: 20,  // Microsoft Graph limit
  MAX_BATCH_SIZE_KB: 4096,     // 4MB total
  CONCURRENT_BATCHES: 5        // Process 5 batches in parallel max
};
```

### Batching Strategy: When to Batch vs Individual Calls

**Batch Operations (via $batch endpoint):**
✅ User has multiple schedules to sync at once (bulk import, retry job)
✅ Scheduled batch sync job (process all pending syncs for a user)
✅ User reconnects and needs to sync 10+ events

**Individual Operations:**
✅ Real-time sync (schedule just created/updated)
✅ Single schedule change
✅ Lower latency requirement (batch adds overhead)

**Example Scenarios:**

```typescript
// Scenario 1: Single schedule updated → Individual call
async function handleScheduleUpdate(scheduleId: string) {
  const users = await getUsersWithSync(scheduleId);
  
  for (const user of users) {
    // Each user gets individual API call (fast, real-time)
    await kafka.publish('outlook.sync.requested', {
      scheduleId,
      userId: user.user_id,
      useBatch: false  // Individual call
    });
  }
}

// Scenario 2: User reconnects → Batch all their pending syncs
async function handleUserReconnect(userId: string) {
  const pendingSyncs = await getPendingSyncsForUser(userId); // 15 schedules
  
  if (pendingSyncs.length > 1) {
    // Batch all their syncs together (efficient)
    await syncUserSchedulesBatch(userId, pendingSyncs);
  }
}

// Scenario 3: Hourly retry job → Batch per user
async function retryFailedSyncs() {
  const failedSyncs = await getFailedSyncs();
  
  // Group by user
  const byUser = groupBy(failedSyncs, 'user_id');
  
  for (const [userId, syncs] of Object.entries(byUser)) {
    if (syncs.length > 3) {
      // Batch multiple retries together
      await syncUserSchedulesBatch(userId, syncs);
    } else {
      // Just a few, do individually
      for (const sync of syncs) {
        await syncSingleSchedule(sync);
      }
    }
  }
}
```

**Key Insight: Batching is PER-USER, not across users**

```
❌ CANNOT DO:
  Batch: [
    Alice's Schedule 1,
    Bob's Schedule 1,
    Charlie's Schedule 1
  ]
  // Different connection keys!

✅ CAN DO:
  Batch for Alice: [
    Alice's Schedule 1,
    Alice's Schedule 2,
    Alice's Schedule 3
  ]
  // Same connection key
  
  Batch for Bob: [
    Bob's Schedule 1,
    Bob's Schedule 2
  ]
  // Bob's connection key
```

### Error Handling: Partial Success in Batches

**Critical: One failure in a batch does NOT fail the others!**

This is Microsoft Graph's behavior, which PicaOS passes through to you unchanged.

```typescript
// Example: Call PicaOS with batch (PicaOS calls Microsoft, returns their response)
const response = await picaClient.batchSync(connectionKey, batchRequests);

// Microsoft Graph returns mixed results (via PicaOS):
const batch = [
  { id: 'schedule-1', action: 'CREATE' },  // ✓ Success
  { id: 'schedule-2', action: 'UPDATE' },  // ✗ 404 (event deleted in Outlook)
  { id: 'schedule-3', action: 'CREATE' },  // ✓ Success
];

// Response from Microsoft Graph:
{
  responses: [
    { id: '0', status: 201, body: { id: 'event-123' } },     // schedule-1 ✓
    { id: '1', status: 404, body: { error: { ... } } },      // schedule-2 ✗
    { id: '2', status: 201, body: { id: 'event-456' } }      // schedule-3 ✓
  ]
}

// Your code must handle each individually:
// - schedule-1: Mark as synced, notify user ✓
// - schedule-2: Schedule retry or mark as error ✗
// - schedule-3: Mark as synced, notify user ✓

// User sees 2 success toasts and 1 error toast
```

### Batch vs Individual: Performance Comparison

| Scenario | Individual Calls | Batch API | Winner |
|----------|------------------|-----------|--------|
| 1 schedule, 3 users | 3 calls (parallel) | N/A (can't batch) | Individual |
| 1 user, 15 schedules | 15 calls (serial) | 1 call | Batch |
| Real-time update | 1 call, ~200ms | 1 batch, ~300ms | Individual |
| Bulk sync | N calls, N×200ms | N/20 batches | Batch |
| Rate limit friendly | Uses quota fast | Saves quota | Batch |

```

### 9a. **Simple PicaOS HTTP Client (No Heavy Service Layer)**

```typescript
// Simple utility class for PicaOS API calls - no complex service layer needed
class PicaClient {
  private baseUrl = 'https://api.picaos.com/v1';
  private apiKey = process.env.PICA_API_KEY;

  // Get user's connection key after AuthKit authentication
  async getConnectionKey(userId: string): Promise<string> {
    const response = await fetch(`${this.baseUrl}/vault/connections?userId=${userId}`, {
      headers: { 'Authorization': `Bearer ${this.apiKey}` }
    });
    const data = await response.json();
    return data.connections[0].connectionKey;
  }

  // Make passthrough API call to Outlook
  async createOutlookEvent(connectionKey: string, eventData: any) {
    const response = await fetch(`${this.baseUrl}/proxy/microsoft-outlook/calendar/events`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'X-Connection-Key': connectionKey,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(eventData)
    });

    if (!response.ok) {
      const error = await response.json();
      throw new PicaError(response.status, error);
    }

    return response.json();
  }

  // Update event in Outlook
  async updateOutlookEvent(connectionKey: string, eventId: string, eventData: any) {
    const response = await fetch(
      `${this.baseUrl}/proxy/microsoft-outlook/calendar/events/${eventId}`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'X-Connection-Key': connectionKey,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(eventData)
      }
    );

    if (!response.ok) throw new PicaError(response.status, await response.json());
    return response.json();
  }

  // Delete event from Outlook
  async deleteOutlookEvent(connectionKey: string, eventId: string) {
    const response = await fetch(
      `${this.baseUrl}/proxy/microsoft-outlook/calendar/events/${eventId}`,
      {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'X-Connection-Key': connectionKey
        }
      }
    );

    if (!response.ok) throw new PicaError(response.status, await response.json());
  }

  // Batch sync multiple operations for a SINGLE user via PicaOS passthrough
  async batchSync(connectionKey: string, requests: BatchRequest[]) {
    // Microsoft Graph limits: 20 requests per batch
    if (requests.length > 20) {
      throw new Error('Batch size exceeds Microsoft Graph limit of 20 requests');
    }

    // Call PicaOS passthrough endpoint (PicaOS will call Microsoft Graph)
    // This is NOT a direct call to Microsoft - it goes through PicaOS
    const response = await fetch(
      `${this.baseUrl}/proxy/microsoft-outlook/$batch`,  // PicaOS passthrough
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,       // Your PicaOS API key
          'X-Connection-Key': connectionKey,              // User's connection (PicaOS maps to OAuth token)
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ requests })  // Microsoft Graph batch format
      }
    );

    if (!response.ok) {
      // PicaOS API call itself failed (network, invalid connection key, etc)
      throw new PicaError(response.status, await response.json());
    }

    // PicaOS call succeeded, returns Microsoft Graph's batch response
    const data = await response.json();
    
    // data contains Microsoft's responses:
    // { responses: [ { id, status, body }, ... ] }
    // - status comes from Microsoft Graph
    // - body contains Microsoft's success/error details
    return data;
  }

  // Get connection status from PicaOS
  async getConnectionStatus(userId: string) {
    const response = await fetch(
      `${this.baseUrl}/vault/connections?userId=${userId}`,
      {
        method: 'GET',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`
        }
      }
    );

    if (!response.ok) throw new PicaError(response.status, await response.json());
    
    const data = await response.json();
    
    // Return the connection details including state
    return {
      connections: data.connections.map((conn: any) => ({
        connectionKey: conn.connectionKey,
        platform: conn.platform,
        state: conn.state, // 'operational', 'degraded', 'failed', 'unknown'
        email: conn.metadata?.email,
        connectedAt: conn.connectedAt,
        lastValidated: conn.lastValidated
      }))
    };
  }
}

// That's it! Just a thin HTTP wrapper - use directly in controllers and workers
const picaClient = new PicaClient();
```

### 10. **User Notifications via app-events (Existing Service)**

```typescript
// Simple Kafka producer for app-events topic
class AppEventsProducer {
  private producer: KafkaProducer;
  
  // Notify user of sync success
  async notifySyncSuccess(userId: string, scheduleTitle: string) {
    await this.producer.send({
      topic: 'app-events',
      messages: [{
        key: userId,
        value: JSON.stringify({
          userId,
          eventType: 'notification',
          notification: {
            type: 'success',
            title: 'Synced to Outlook',
            message: `"${scheduleTitle}" has been added to your Outlook calendar`,
            duration: 5000
          },
          timestamp: new Date().toISOString()
        })
      }]
    });
  }
  
  // Notify user of sync failure
  async notifySyncFailure(userId: string, scheduleTitle: string, error: string) {
    await this.producer.send({
      topic: 'app-events',
      messages: [{
        key: userId,
        value: JSON.stringify({
          userId,
          eventType: 'notification',
          notification: {
            type: 'error',
            title: 'Sync Failed',
            message: `Unable to sync "${scheduleTitle}" to Outlook`,
            action: { label: 'Retry', scheduleId: '...' }
          },
          timestamp: new Date().toISOString()
        })
      }]
    });
  }
  
  // Notify user of connection issues
  async notifyConnectionError(userId: string) {
    await this.producer.send({
      topic: 'app-events',
      messages: [{
        key: userId,
        value: JSON.stringify({
          userId,
          eventType: 'notification',
          notification: {
            type: 'warning',
            title: 'Outlook Connection Issue',
            message: 'Please reconnect your Outlook account',
            action: { label: 'Reconnect', url: '/settings/integrations' }
          },
          timestamp: new Date().toISOString()
        })
      }]
    });
  }
}

// Usage in your sync completion handler
const appEvents = new AppEventsProducer();
await appEvents.notifySyncSuccess(userId, scheduleTitle);

// Note: The existing app-events service handles:
// - Consuming these messages
// - Delivering via WebSocket to frontend
// - Toast rendering in the UI
// You just publish to the topic!

// Message Format for app-events:
// {
//   userId: string          - Required: User to notify
//   eventType: string       - Required: 'notification' for toast messages
//   notification: {
//     type: string          - Required: 'success' | 'error' | 'warning' | 'info'
//     title: string         - Required: Notification title
//     message: string       - Required: Notification message
//     duration?: number     - Optional: Display duration in ms (default 5000)
//     action?: {            - Optional: Action button
//       label: string
//       url?: string
//       scheduleId?: string
//     }
//   },
//   timestamp: string       - Required: ISO8601 timestamp
// }
```

---

## API Endpoints

### Connection Management
```typescript
POST   /api/pica/connections/store        // Store connection after AuthKit
GET    /api/pica/connections              // List user's connections
GET    /api/pica/connections/:id/status   // Get real-time connection status from PicaOS
DELETE /api/pica/connections/:id          // Disconnect
POST   /api/pica/connections/:id/validate // Check connection health (background job)
```

**GET /api/pica/connections/:id/status Response:**
```typescript
{
  connectionId: "uuid",
  userId: "uuid",
  integration: "microsoft-outlook",
  status: "operational" | "degraded" | "failed" | "unknown",
  statusDescription: string,
  lastChecked: "2025-11-27T10:00:00Z",
  connectedAt: "2025-11-20T10:00:00Z",
  email: "user@example.com" // From PicaOS metadata
}
```

**Status Definitions (from PicaOS):**
- `operational` - Connection is healthy and working correctly
- `degraded` - OAuth token refresh failed, automatic retry pending
- `failed` - Connection is inactive and requires manual reconnection
- `unknown` - State could not be determined

### Integration Settings (Org-level)
```typescript
POST   /api/pica/integrations/:id/enable   // Enable for org
POST   /api/pica/integrations/:id/disable  // Disable for org
GET    /api/pica/integrations               // List available integrations
GET    /api/pica/integrations/:id/settings // Get org settings
PUT    /api/pica/integrations/:id/settings // Update org settings
```

### Sync Management (Minimal - mostly automatic)
```typescript
POST   /api/pica/sync/schedule/:scheduleId     // Manually trigger sync (if needed)
POST   /api/pica/sync/retry/:scheduleId        // Retry failed sync (admin only)
```

### Notifications
```typescript
// No API endpoints needed - handled by existing app-events service
// Your sync completion handler publishes to 'app-events' Kafka topic
// App-events service delivers toast notifications to users automatically
```

---

## Error Handling Strategy

### Error Classification

```typescript
enum ErrorType {
  // Retryable - Will retry automatically
  RATE_LIMIT = 'RATE_LIMIT_EXCEEDED',
  TIMEOUT = 'TIMEOUT',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  NETWORK_ERROR = 'NETWORK_ERROR',
  
  // Permanent - Will not retry, needs user action
  INVALID_TOKEN = 'INVALID_TOKEN',
  CONNECTION_REVOKED = 'CONNECTION_REVOKED',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  INVALID_DATA = 'INVALID_EVENT_DATA',
  
  // Unknown - Will retry limited times
  UNKNOWN = 'UNKNOWN_ERROR'
}

function classifyError(error: any): ErrorType {
  if (error.status === 429) return ErrorType.RATE_LIMIT;
  if (error.status === 401) return ErrorType.INVALID_TOKEN;
  if (error.status === 403) return ErrorType.PERMISSION_DENIED;
  if (error.status === 400) return ErrorType.INVALID_DATA;
  if (error.status >= 500) return ErrorType.SERVICE_UNAVAILABLE;
  if (error.code === 'ETIMEDOUT') return ErrorType.TIMEOUT;
  return ErrorType.UNKNOWN;
}
```

### User Notifications

```typescript
interface NotificationStrategy {
  RATE_LIMIT: { notify: false, action: 'auto-retry' },
  INVALID_TOKEN: { notify: true, action: 'reconnect-required' },
  CONNECTION_REVOKED: { notify: true, action: 'reconnect-required' },
  PERMISSION_DENIED: { notify: true, action: 'check-permissions' },
  INVALID_DATA: { notify: true, action: 'check-event-data' }
}
```

---

## Future Enhancements (Phase 2: Pull from Outlook)

### 1. Webhook Support

```typescript
POST /api/pica/webhooks/outlook
// Receives notifications from PicaOS when Outlook events change

// Add to schedule_sync_status table:
webhook_subscription_id TEXT DEFAULT NULL
webhook_last_received_at TIMESTAMPTZ DEFAULT NULL
```

### 2. Conflict Resolution

When pulling events from Outlook:

```typescript
enum ConflictStrategy {
  OUTLOOK_WINS = 'outlook_wins',      // Always use Outlook data
  LOCAL_WINS = 'local_wins',          // Always use our data
  NEWEST_WINS = 'newest_wins',        // Compare timestamps
  MANUAL_REVIEW = 'manual_review'     // Flag for user review
}

// Add to organization_integration_settings.settings
{
  "conflict_resolution": "newest_wins",
  "sync_direction": "bidirectional"
}
```

### 3. Incremental Sync

```typescript
// Track last sync time per user
interface SyncCursor {
  user_connection_id: string;
  last_full_sync_at: Date;
  last_incremental_sync_at: Date;
  sync_token: string; // Microsoft Graph delta token
}
```

---

## Monitoring & Observability

### Key Metrics to Track

1. **Sync Success Rate**
   - Successful syncs / Total sync attempts
   - Target: > 99%

2. **Sync Latency**
   - Time from event change to Outlook sync
   - Target: < 5 seconds (p95)

3. **Error Rate by Type**
   - Track each error type separately
   - Alert on spike in permanent errors

4. **Connection Health**
   - Active connections by state (operational, degraded, failed)
   - Connection status check latency
   - Failed connection validations
   - Disconnection rate
   - Connections requiring reconnection

5. **Kafka Metrics**
   - Consumer lag (outlook.sync topics)
   - Message processing time
   - DLQ message count
   - app-events publish success rate

### Logging Strategy

```typescript
// Structured logging
logger.info('sync.started', {
  scheduleId,
  userId,
  action: 'CREATE',
  idempotencyKey,
  attemptNumber: 1
});

logger.info('sync.completed', {
  scheduleId,
  userId,
  action: 'CREATE',
  externalEventId: 'outlook-event-123',
  durationMs: 245
});

logger.error('sync.failed', {
  scheduleId,
  userId,
  action: 'CREATE',
  errorType: 'RATE_LIMIT_EXCEEDED',
  willRetry: true,
  nextRetryAt: '2025-11-26T10:30:00Z'
});

logger.info('notification.published', {
  userId,
  scheduleId,
  notificationType: 'success',
  topic: 'app-events',
  timestamp: new Date().toISOString()
});

logger.info('connection.status_checked', {
  userId,
  connectionId,
  picaState: 'operational',
  cached: false,
  durationMs: 120
});
```

---

## Deployment Considerations

### Environment Variables

```bash
# PicaOS
PICA_API_URL=https://api.picaos.com/v1
PICA_API_KEY=your_api_key
PICA_AUTH_KIT_URL=https://authkit.picaos.com

# Kafka
KAFKA_BROKERS=localhost:9092
KAFKA_CONSUMER_GROUP=outlook-sync-workers
KAFKA_TOPIC_PREFIX=outlook.sync

# Database
DATABASE_URL=postgresql://...
CONNECTION_KEY_ENCRYPTION_KEY=...

# Rate Limiting
SYNC_RATE_LIMIT_PER_USER=100  # requests per hour
SYNC_BATCH_SIZE=10
SYNC_BATCH_DELAY_MS=1000

# Retry Configuration
SYNC_MAX_RETRIES=5
SYNC_INITIAL_RETRY_DELAY_MS=1000
SYNC_MAX_RETRY_DELAY_MS=300000

# Worker Instance Configuration
INSTANCE_ID=${HOSTNAME:-worker-1}  # Unique ID for this instance
REDIS_URL=redis://localhost:6379   # For distributed locks (leader election)
```

### Scaling Strategy

**NestJS API Backend:**
- Horizontal scaling behind load balancer
- Stateless (except WebSocket connections)

**Outlook Sync Worker Service:**
- Deploy multiple instances in consumer group
- Kafka handles partition distribution for consumers
- Scale based on consumer lag
- **Retry Scheduler (cron):**
  - Runs in ALL instances but only executes on leader
  - Uses distributed lock (Redis/PostgreSQL) for leader election
  - If leader crashes, another instance takes over
  - Adds minimal overhead to non-leader instances

**Deployment Example:**
```
Instance 1 (Leader):
  ├─ Kafka Consumer ✓ (processes messages)
  └─ Retry Cron ✓ (ACTIVE - republishes retries)

Instance 2:
  ├─ Kafka Consumer ✓ (processes messages)
  └─ Retry Cron ⏸ (standby - skips execution)

Instance 3:
  ├─ Kafka Consumer ✓ (processes messages)
  └─ Retry Cron ⏸ (standby - skips execution)
```

**Database:**
- Read replicas for status queries
- Connection pooling
- Regular vacuum/analyze on high-write tables

---

## Testing Strategy

### Unit Tests
- PicaClient HTTP wrapper error handling
- Connection status mapping (PicaOS states → internal states)
- Idempotency key generation
- Retry logic calculations
- Event hash generation
- Event data transformation (your format → Outlook format)
- Status caching logic

### Integration Tests
- End-to-end sync flow with mocked HTTP responses
- Kafka message production/consumption (including app-events)
- Database transactions
- Notification message format validation

### E2E Tests
- Full user journey: connect → create schedule → verify notification published to app-events
- Error scenarios: connection revoked, rate limited, error notification published
- Retry scenarios and notification message validation

---

## Migration Plan

### Phase 1: Foundation (Week 1-2)
- [ ] Create database schema
- [ ] Set up Kafka topics
- [ ] Create simple PicaOS HTTP client wrapper
- [ ] Build connection management API endpoints
  - [ ] POST /api/pica/connections/store
  - [ ] GET /api/pica/connections/:id/status (new!)
  - [ ] DELETE /api/pica/connections/:id

### Phase 2: Sync Engine (Week 3-4)
- [ ] Kafka producer in API
- [ ] Kafka consumer worker
- [ ] Retry logic and error handling
- [ ] Idempotency support

### Phase 3: Notifications & Monitoring (Week 5-6)
- [ ] Integrate with app-events Kafka topic for notifications
- [ ] Publish sync completion messages to app-events
- [ ] Audit logging
- [ ] Metrics and alerting

### Phase 4: Production Rollout (Week 7-8)
- [ ] Pilot with internal users
- [ ] Gradual rollout to customers
- [ ] Performance tuning
- [ ] Documentation

---

### Frontend Usage Examples

**Integration Settings Page:**
```typescript
// Check connection status when user views settings
const ConnectionStatus: React.FC = () => {
  const [status, setStatus] = useState(null);
  
  useEffect(() => {
    fetchConnectionStatus();
  }, []);
  
  const fetchConnectionStatus = async () => {
    const response = await fetch('/api/pica/connections/my-connection-id/status');
    const data = await response.json();
    setStatus(data);
  };
  
  return (
    <div>
      <h3>Outlook Integration</h3>
      {status?.status === 'operational' && (
        <Badge color="green">✓ Connected</Badge>
      )}
      {status?.status === 'degraded' && (
        <Badge color="yellow">⚠ Reconnecting...</Badge>
      )}
      {status?.status === 'failed' && (
        <>
          <Badge color="red">✗ Connection Failed</Badge>
          <Button onClick={handleReconnect}>Reconnect</Button>
        </>
      )}
      <p>{status?.email}</p>
    </div>
  );
};
```

**After Sync Error:**
```typescript
// When user receives error notification, check connection status
socket.on('notification', async (notification) => {
  if (notification.type === 'error' && notification.message.includes('Outlook')) {
    // Check connection status to see if reconnection is needed
    const status = await checkConnectionStatus();
    
    if (status.status === 'failed') {
      // Show reconnect button in toast
      toast.error('Outlook connection lost', {
        action: { label: 'Reconnect', onClick: () => reconnectOutlook() }
      });
    }
  }
});
```

**Periodic Health Check (Optional):**
```typescript
// Check connection status every 5 minutes while user is active
useInterval(() => {
  checkConnectionStatus();
}, 5 * 60 * 1000);
```

### When to Check Connection Status

**Real-time Checks (Call PicaOS API):**
- ✅ User views integration settings page
- ✅ User receives sync error notification
- ✅ Before manual sync trigger
- ✅ After reconnection attempt

**Background Checks (Cron job):**
- ✅ Hourly health check for all active connections
- ✅ After OAuth token refresh failures detected

**Avoid:**
- ❌ On every sync operation (use cached status from DB)
- ❌ Too frequently (respect API rate limits)
- ❌ For inactive/disconnected connections

### Caching Strategy

```typescript
// Cache connection status in your DB
// Only call PicaOS API when:
// 1. User explicitly requests status
// 2. Last check was > 5 minutes ago
// 3. Sync operation failed with auth error

@Get('connections/:id/status')
async getConnectionStatus(@Param('id') connectionId: string) {
  const conn = await db.findConnection(connectionId);
  
  // Use cached status if recent
  const cacheAge = Date.now() - conn.last_validated_at.getTime();
  if (cacheAge < 5 * 60 * 1000 && conn.pica_state !== 'unknown') {
    return {
      ...formatConnectionStatus(conn),
      cached: true
    };
  }
  
  // Fetch fresh status from PicaOS
  const freshStatus = await picaClient.getConnectionStatus(conn.pica_user_id);
  await db.updateConnection(connectionId, {
    pica_state: freshStatus.state,
    last_validated_at: new Date()
  });
  
  return {
    ...formatConnectionStatus(freshStatus),
    cached: false
  };
}
```

### Status-based UI Behavior

**operational:**
- Show green badge
- Allow sync operations
- No action needed

**degraded:**
- Show yellow warning badge
- Allow sync operations (PicaOS is retrying automatically)
- Show message: "Reconnecting to Outlook..."
- No user action needed yet

**failed:**
- Show red error badge
- Block sync operations
- Show "Reconnect" button
- User must reconnect via AuthKit

**unknown:**
- Show gray badge
- Allow sync operations (assume healthy)
- Log for investigation

### Error Handling for Status Endpoint

```typescript
// Handle PicaOS API errors gracefully
@Get('connections/:id/status')
async getConnectionStatus(@Param('id') connectionId: string) {
  try {
    const picaStatus = await picaClient.getConnectionStatus(conn.pica_user_id);
    // ... normal flow
  } catch (error) {
    // If PicaOS API is down or rate limited, return cached status
    if (error.status === 429 || error.status >= 500) {
      logger.warn('pica.api.unavailable', {
        connectionId,
        error: error.message
      });
      
      // Return last known status from DB
      return {
        ...formatConnectionStatus(conn),
        status: conn.pica_state || 'unknown',
        cached: true,
        warning: 'Status may be stale, PicaOS API unavailable'
      };
    }
    
    // If connection not found in PicaOS (user disconnected externally)
    if (error.status === 404) {
      await db.updateConnection(connectionId, {
        is_active: false,
        pica_state: 'failed',
        disconnected_at: new Date()
      });
      
      return {
        connectionId,
        status: 'failed',
        statusDescription: 'Connection no longer exists in PicaOS',
        requiresReconnection: true
      };
    }
    
    throw error;
  }
}
```

---

## Questions for Discussion

1. **Connection Key Storage**: Should we encrypt connection keys at rest? (Recommended: Yes)

2. **Sync Trigger Strategy**: 
   - Real-time on every schedule change?
   - Batched (e.g., every 5 minutes)?
   - Hybrid (real-time for manual changes, batched for automated)?

3. **Org-level vs User-level Settings**: 
   - Should orgs be able to enforce sync policies?
   - Can users opt-out if org has enabled integration?

4. **Rate Limiting**: 
   - Per-user limits?
   - Per-org limits?
   - Global limits?

5. **Conflict Resolution**: 
   - For Phase 2, what's the default strategy?
   - Should this be configurable per org/user?

6. **Data Retention**:
   - How long to keep audit logs?
   - Archive strategy for old sync records?

7. **Multi-calendar Support**:
   - Initially support only primary calendar?
   - Future: Let users choose which calendar?

8. **Event Filtering**:
   - Sync all schedule types or allow filtering?
   - Privacy concerns for sensitive events?

---

## Cost Considerations

### PicaOS Costs
- Connection storage
- API request volume
- Webhook subscriptions (Phase 2)

### Infrastructure Costs
- Kafka cluster (or managed Kafka)
- Additional database storage for audit logs
- Worker compute resources

### Optimization Strategies
- Batch sync to reduce API calls
- Change detection to avoid unnecessary syncs
- Efficient indexing to minimize query costs
- Archive old audit logs to cheaper storage

---

## Success Metrics

### Technical Metrics
- Sync success rate > 99%
- p95 sync latency < 5 seconds
- Connection uptime > 99.9%
- Zero data loss

### Business Metrics
- User adoption rate
- Active connections per org
- Average events synced per user per day
- Support ticket volume

### User Experience Metrics
- Time from schedule creation to Outlook appearance
- Time from sync completion to app-events publish
- User-reported sync issues
- app-events publish success rate (notification delivery handled by app-events service)
- Reconnection frequency

---

## Conclusion

This design provides a robust, scalable foundation for Outlook integration using PicaOS. Key improvements over the initial design include:

1. ✅ Enhanced database schema with proper tracking and audit trails
2. ✅ Comprehensive error handling and retry logic
3. ✅ Idempotency and deduplication support
4. ✅ Rate limiting and connection health monitoring
5. ✅ Real-time toast notifications via existing app-events service
6. ✅ Clear separation of concerns (API, Worker, Database)
7. ✅ Scalability built-in from day one
8. ✅ Foundation for bi-directional sync (Phase 2)
9. ✅ Leverages existing infrastructure (app-events, Kafka)

The system is designed to be:
- **Reliable**: Retry logic, idempotency, audit trails
- **Scalable**: Horizontal scaling of workers, efficient batch processing
- **Maintainable**: Simple architecture with minimal abstraction layers
- **Secure**: Encrypted connection keys, proper error handling
- **Observable**: Metrics, logging, real-time notifications
- **User-friendly**: Instant feedback via toast notifications on sync completion
- **Simple**: Direct API calls to PicaOS, leverages existing app-events service
- **Integrated**: Works seamlessly with existing platform infrastructure

Next steps: Review this proposal, discuss open questions, and begin Phase 1 implementation.

