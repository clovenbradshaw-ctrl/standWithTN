# Activity Stream Model with Epistemic Operators (EO)

> Architecture reference for event-sourced content management in Stand With Tennessee

---

## Overview

This document describes the activity stream architecture using **Epistemic Operators (EO)**—a formal grammar for expressing all data mutations as semantic operations. This model underpins the event-sourced data layer, enabling audit trails, causal reasoning, and schema-free evolution.

---

## The Core Architecture

### Universal Action Grammar

```
operator(target, context, frame)
```

Every mutation is an operation with:

| Component | Description |
|-----------|-------------|
| **target** | What's being acted upon (entity, value, set) |
| **context** | Where this operation occurs (source table, relationship, scope) |
| **frame** | Interpretive lens (K-frame: definition version, validation rules, time horizon) |

### Minimal Schema (The `operations` Table)

```sql
CREATE TABLE operations (
  id TEXT PRIMARY KEY DEFAULT (uuid()),
  ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  -- The EO operator (9 possible values)
  op TEXT NOT NULL CHECK (op IN ('INS','DES','SEG','CON','SYN','ALT','SUP','REC','NUL')),

  -- The universal payload
  target JSONB NOT NULL,    -- what's being operated on
  context JSONB NOT NULL,   -- where/how it's happening
  frame JSONB,              -- optional interpretive frame

  -- Indexable extractions (computed)
  source_table TEXT GENERATED ALWAYS AS (context->>'table') STORED,
  entity_id TEXT GENERATED ALWAYS AS (target->>'id') STORED
);
```

**Six real columns, two computed indexes.** Everything else lives in JSON.

---

## The Nine Operators as Event Types

| Operator | Code | Symbol | Activity Store Meaning |
|----------|------|--------|------------------------|
| **NUL** | `NUL` | ∅ | Recognizing absence/deletion |
| **DES** | `DES` | ≝ | Naming/defining (designation) |
| **INS** | `INS` | ⊕ | Creating/instantiating |
| **SEG** | `SEG` | ⊢ | Segmenting/filtering |
| **CON** | `CON` | ⋈ | Connecting/relating |
| **ALT** | `ALT` | ⇌ | Alternating/transitioning state |
| **SYN** | `SYN` | ⊗ | Synthesizing/merging |
| **SUP** | `SUP` | ⧦ | Superposing/layering context |
| **REC** | `REC` | ↻ | Reconfiguring/learning |

### Operator Triads

Operators are organized in three semantic triads:

```
Identity:       NUL → DES → INS
                (absence → definition → existence)

Structure:      SEG → CON → SYN
                (partition → link → merge)

Interpretation: ALT → SUP → REC
                (transition → overlay → adapt)
```

---

## Key Design Principles

### 1. Operators are stable, K-frames are emergent

You don't evolve schema—you let meaning crystallize from operator patterns over time. The nine operators are universal; interpretation lives in the `frame` payload.

### 2. Semantic queryability

Instead of filtering by many verb variations, filter by operator class:

```javascript
// All state transitions
events.filter(e => e.op === "ALT")

// All creations
events.filter(e => e.op === "INS")

// All deletions/nullifications
events.filter(e => e.op === "NUL")
```

### 3. Causal pattern recognition

Track transformation sequences to reveal workflows:

```javascript
const pattern = events
  .filter(e => e.target.id === recordId)
  .map(e => e.op)
  .join("→")

// "INS→CON→SYN" reveals: created, linked, then merged
// "INS→ALT→ALT→NUL" reveals: created, modified twice, then deleted
```

---

## Event Structure

### Standard Event Shape

```javascript
{
  id: "uuid",
  ts: 1699564800000,
  op: "ALT",
  target: {
    id: "client_123",
    field: "status"
  },
  context: {
    table: "clients",
    old: "pending",
    new: "active"
  },
  frame: {
    version: "v2",
    validator: "status_rules"
  }
}
```

### Practical Examples

#### Creating an organization (INS)
```javascript
{
  op: "INS",
  target: { id: "org_456", type: "organization" },
  context: {
    table: "organizations",
    data: { name: "TN Immigration Coalition", region: "Nashville" }
  },
  frame: { source: "admin_dashboard" }
}
```

#### Updating campaign status (ALT)
```javascript
{
  op: "ALT",
  target: { id: "campaign_789", field: "status" },
  context: {
    table: "campaigns",
    old: "draft",
    new: "published"
  },
  frame: { actor: "admin_user" }
}
```

#### Soft-deleting a submission (NUL)
```javascript
{
  op: "NUL",
  target: { id: "submission_101" },
  context: {
    table: "submissions",
    reason: "duplicate"
  },
  frame: { reversible: true }
}
```

---

## EO-IR: Intermediate Representation

For advanced use cases, **EO-IR** (Epistemic Operators Intermediate Representation) tracks epistemic types alongside operators:

| Type | Meaning | Example |
|------|---------|---------|
| **GIVEN** | External, immutable, observed | API response, user input |
| **MEANT** | Internal interpretations | Calculated status, derived category |
| **DERIVED** | Computed values | Aggregations, transformations |

### Provenance Chain Example

```javascript
{
  op: "SYN",
  target: { id: "report_001" },
  context: {
    table: "reports",
    inputs: ["campaign_789", "campaign_790"]
  },
  frame: {
    epistemic: "DERIVED",
    provenance: [
      { id: "campaign_789", type: "GIVEN" },
      { id: "campaign_790", type: "GIVEN" }
    ]
  }
}
```

This preserves provenance chains and enables query compilation to SQL while maintaining semantic metadata.

---

## Implementation in This Project

### Current Event Types in Use

The Stand With Tennessee application currently uses a subset of operators:

| Operator | Current Usage |
|----------|---------------|
| `INS` | Creating organizations, campaigns, submissions |
| `ALT` | Updating content, changing statuses |
| `NUL` | Deleting/archiving records |

### Integration Points

1. **LocalStorage DB** (`DB` object in `index.html`)
   - Event sourcing with `INS`, `ALT`, `NUL` operations
   - State computed from event replay

2. **Xano API** (`ContentAPI` object)
   - `/ingest_swt_content` - Event ingestion endpoint
   - `/swtSiteEvents` - Fetch event history

3. **Webhook Logging**
   - Activity stream events forwarded to n8n for external audit

---

## Operator Interaction Matrix

Understanding how operators compose:

```
INS + CON = Create and link (common for related entities)
INS + ALT = Create then modify (draft → published workflow)
SEG + SYN = Filter then merge (data pipeline pattern)
ALT + SUP = Change with context overlay (versioned updates)
NUL + REC = Delete with learning (adaptive cleanup)
```

---

## K-Frame Structure

K-frames (Knowledge frames) provide interpretive context without schema changes:

```javascript
frame: {
  // Version control
  version: "v2.1",
  schema: "2024-01",

  // Validation rules
  validator: "campaign_rules",
  constraints: ["required_fields", "date_range"],

  // Temporal context
  effective_from: "2024-01-01",
  effective_to: null,

  // Epistemic metadata
  certainty: 0.95,
  source: "user_input",

  // Actor/audit info
  actor: "admin_123",
  reason: "quarterly_update"
}
```

---

## Query Patterns

### Reconstruct entity state

```javascript
function getEntityState(entityId, events) {
  return events
    .filter(e => e.target.id === entityId)
    .filter(e => e.op !== "NUL")
    .sort((a, b) => a.ts - b.ts)
    .reduce((state, event) => {
      if (event.op === "INS") return event.context.data;
      if (event.op === "ALT") return { ...state, [event.target.field]: event.context.new };
      return state;
    }, {});
}
```

### Find all related entities

```javascript
function getConnections(entityId, events) {
  return events
    .filter(e => e.op === "CON")
    .filter(e => e.target.id === entityId || e.context.related === entityId)
    .map(e => e.target.id === entityId ? e.context.related : e.target.id);
}
```

### Audit trail for entity

```javascript
function getAuditTrail(entityId, events) {
  return events
    .filter(e => e.target.id === entityId)
    .sort((a, b) => a.ts - b.ts)
    .map(e => ({
      when: new Date(e.ts).toISOString(),
      action: e.op,
      details: e.context,
      actor: e.frame?.actor
    }));
}
```

---

## Future Extensions

### Planned Operators

- **SEG** for advanced filtering/partitioning of content
- **CON** for explicit relationship management between entities
- **SYN** for merging duplicate submissions
- **SUP** for contextual overlays (e.g., regional variations)
- **REC** for adaptive content recommendations

### Integration Opportunities

- Real-time collaboration via operator streams
- Conflict resolution using operator semantics
- Machine learning on operator patterns
- Cross-system event federation

---

## References

- Internal: `index.html` - `DB` object, `ContentAPI` object
- Endpoints: Xano API at `xvkq-pq7i-idtl.n7d.xano.io`
- Related: Event Sourcing patterns, CQRS architecture

---

*Document created: 2026-01-29*
*For Stand With Tennessee event-sourced architecture*
