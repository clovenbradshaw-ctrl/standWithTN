# API Instructions: Content Ingestion & Snapshots

> Server-side requirements for the Stand With Tennessee event-sourced data layer

---

## Endpoint Overview

### POST `/ingest_swt_content`

Single endpoint for all data mutations. Accepts both **activities** (diffs) and **snapshots** (full state).

#### Schema

```json
{
  "id": "integer (auto-generated)",
  "created_at": "timestamp",
  "agent": "text",
  "uuid": "uuid",
  "payload": "json",
  "operator": "enum (INS, ALT, NUL)",
  "target": "text",
  "set": "text"
}
```

#### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Auto-generated primary key |
| `created_at` | timestamp | When this record was created |
| `agent` | text | Who/what created this (e.g., "admin", "system", "web-client") |
| `uuid` | uuid | Unique identifier for the event/snapshot |
| `payload` | json | The data - diff for activities, full state for snapshots |
| `operator` | enum | `INS` (create), `ALT` (update), `NUL` (delete) |
| `target` | text | What's being acted upon (record_id for activities, frame name for snapshots) |
| `set` | text | The context/frame - see below |

#### `set` Values (Frames)

| Set Value | Description |
|-----------|-------------|
| `organizations` | Organization records |
| `campaigns` | Campaign records |
| `submissions` | User submissions |
| `services` | Services offered |
| `regions` | Geographic regions |
| `languages` | Language support |
| `events` | Event listings |
| `siteData` | Site configuration (hero text, etc.) |
| `siteTheme` | Theme configuration (colors, typography) |
| `content` | Website articles/pages |
| `snapshots` | **Point-in-time state snapshots** |

---

## Two Record Types

### 1. Activities (Diffs)

Individual mutations that record what changed. Append-only, immutable.

```json
{
  "uuid": "evt_abc123",
  "agent": "admin",
  "target": "org_456",
  "set": "organizations",
  "operator": "INS",
  "payload": {
    "data": {
      "INS": {
        "id": "org_456",
        "fields": {
          "name": "TN Immigration Coalition",
          "region": "Nashville",
          "status": "pending"
        }
      }
    },
    "frame": "organizations",
    "ordinal": 42,
    "record_id": "org_456",
    "event_type": "given"
  },
  "created_at": "2026-01-29T10:30:00Z"
}
```

**Operators:**
- `INS` - Create new record
- `ALT` - Update existing record (contains old_value, new_value)
- `NUL` - Delete/archive record (contains reason, snapshot of deleted state)

### 2. Snapshots (Full State)

Computed point-in-time state. Created server-side. Replaces previous snapshot for same target.

```json
{
  "uuid": "snap_all_1706529000000",
  "agent": "system",
  "target": "all",
  "set": "snapshots",
  "operator": "INS",
  "payload": {
    "data": {
      "organizations": [ /* all org records */ ],
      "campaigns": [ /* all campaign records */ ],
      "submissions": [ /* all submission records */ ],
      "services": [ /* all service records */ ],
      "regions": [ /* all region records */ ],
      "languages": [ /* all language records */ ],
      "events": [ /* all event records */ ],
      "siteData": { /* site configuration */ },
      "siteTheme": { /* theme configuration */ },
      "content": [ /* all content/articles */ ]
    },
    "frame": "all",
    "record_counts": {
      "organizations": 15,
      "campaigns": 8,
      "submissions": 42,
      "services": 12,
      "regions": 5,
      "languages": 3,
      "events": 10,
      "content": 25
    },
    "computed_at": "2026-01-29T10:35:00Z",
    "last_activity_ordinal": 156
  },
  "created_at": "2026-01-29T10:35:00Z"
}
```

---

## Server-Side Snapshot Generation

### Trigger Conditions

Create a new snapshot when **either** condition is met:

1. **User session ends** - Receive a session-end signal (beacon, explicit logout, or connection close)
2. **5 minutes of inactivity** - No new activities received for 5 minutes after the last activity

### Implementation Logic

```
ON new_activity_received:
  1. Store the activity
  2. Reset inactivity timer for this session/agent
  3. Record last_activity_ordinal

ON inactivity_timeout (5 min) OR session_end:
  1. Query all activities up to last_activity_ordinal
  2. Compute current state by replaying activities (see algorithm below)
  3. Create snapshot record with set="snapshots", target="all"
  4. Store snapshot
```

### State Computation Algorithm

```javascript
function computeSnapshot(activities) {
  // Group activities by frame
  const frames = {};

  for (const activity of activities.sort((a, b) => a.ordinal - b.ordinal)) {
    const frame = activity.set;
    const recordId = activity.target;

    if (!frames[frame]) frames[frame] = {};

    if (activity.operator === 'INS') {
      // Create new record
      frames[frame][recordId] = {
        id: recordId,
        ...activity.payload.data.INS.fields,
        _created_at: activity.created_at
      };
    }
    else if (activity.operator === 'ALT') {
      // Update existing record
      if (frames[frame][recordId]) {
        const altData = activity.payload.data.ALT;
        frames[frame][recordId][altData.field] = altData.new_value;
        frames[frame][recordId]._updated_at = activity.created_at;
      }
    }
    else if (activity.operator === 'NUL') {
      // Delete record
      delete frames[frame][recordId];
    }
  }

  // Convert to arrays
  const snapshot = {};
  for (const [frame, records] of Object.entries(frames)) {
    snapshot[frame] = Object.values(records);
  }

  return snapshot;
}
```

---

## Query Endpoints Needed

### 1. GET Latest Snapshot

```
GET /snapshots/latest
```

Returns the most recent snapshot where `set = 'snapshots'` and `target = 'all'`.

**Response:**
```json
{
  "uuid": "snap_all_1706529000000",
  "payload": {
    "data": { /* full state */ },
    "computed_at": "2026-01-29T10:35:00Z",
    "last_activity_ordinal": 156
  }
}
```

### 2. GET Activities Since Snapshot

```
GET /activities?since_ordinal={ordinal}
```

Returns all activities with `ordinal > {ordinal}`. Used to apply recent changes on top of a snapshot.

**Response:**
```json
{
  "activities": [
    { "uuid": "evt_xyz", "ordinal": 157, ... },
    { "uuid": "evt_abc", "ordinal": 158, ... }
  ]
}
```

---

## Client Load Flow

```
1. Fetch latest snapshot:
   GET /snapshots/latest

2. If snapshot exists:
   - Hydrate UI with snapshot.payload.data
   - Note the last_activity_ordinal

3. Fetch any activities since snapshot:
   GET /activities?since_ordinal={last_activity_ordinal}

4. Apply activities on top of snapshot state (replay)

5. Ready for user interaction
```

This provides:
- **Fast initial load** - Single snapshot fetch instead of replaying all activities
- **Consistency** - Activities since snapshot ensure no data loss
- **Audit trail** - All activities preserved for history/debugging

---

## Session End Detection

Options for detecting when to generate a snapshot:

### Option A: Beacon API (recommended)

Client sends beacon on page unload:

```javascript
window.addEventListener('beforeunload', () => {
  navigator.sendBeacon('/api/session-end', JSON.stringify({
    agent: 'web-client',
    last_ordinal: DB.ordinal
  }));
});
```

Server receives this and triggers snapshot generation.

### Option B: Heartbeat + Timeout

Client sends periodic heartbeat (every 60s). Server generates snapshot when heartbeats stop + 5 min grace period.

### Option C: Activity-Based Timer

Server tracks last activity timestamp per session. Cron job checks for sessions with no activity in 5+ minutes and generates snapshots.

---

## Summary

| Aspect | Activities | Snapshots |
|--------|------------|-----------|
| **Who creates** | Client | Server |
| **When** | Every mutation | Session end or 5min inactivity |
| **Set value** | Frame name (organizations, etc.) | `"snapshots"` |
| **Target value** | Record ID | `"all"` |
| **Operator** | INS, ALT, NUL | INS only |
| **Payload** | Diff (single change) | Full state (all frames) |
| **Mutable** | No (append-only) | Replaced by newer snapshot |

---

*Document created: 2026-01-29*
*For Stand With Tennessee API implementation*
