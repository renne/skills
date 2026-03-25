---
name: activity-logging
description: NetBird activity and audit logging — audit event log, tracked event types, unknown user/email issue and fix, and SIEM event streaming (cloud-only). Use when investigating audit trail gaps, understanding what events are logged, fixing unknown email in audit logs, or setting up real-time event streaming to Datadog, Amazon S3, Firehose, or a custom HTTP endpoint.
---
# NetBird Activity Logging

Sources:
- https://docs.netbird.io/manage/activity
- https://docs.netbird.io/manage/activity/event-streaming

## Overview

NetBird logs every management-plane action to an **audit event log**, visible at:

- **Cloud**: `app.netbird.io/events/audit`
- **Self-hosted**: dashboard → **Activity → Audit Log**

Logging is enabled by default; no configuration is required.

---

## Tracked Event Categories

| Category | Typical events |
|----------|---------------|
| **Peers** | peer added, removed, renamed, group changed, SSH enabled/disabled, login expiry changed |
| **Users** | user created, deleted, role changed, blocked, invited, service user created |
| **Groups** | group created, deleted, peer added/removed from group |
| **Policies** | policy created, updated, deleted, enabled/disabled |
| **Routes / Networks** | route created, updated, deleted; network resource added/removed |
| **Setup Keys** | key created, revoked, used for enrollment |
| **Nameservers** | nameserver added, updated, deleted |
| **Personal Access Tokens** | token created, deleted |
| **Service Users** | service user created, deleted |
| **Integrations** | IdP sync configured/changed; SIEM stream enabled/disabled |

All events include a timestamp, actor (user email or service account name), and affected resource.

---

## The `unknown` Email / Name Issue

If the actor for an event shows as **`unknown`** (or a blank email address), the **encryption key** used to encrypt the actor's identity in the database has been lost or corrupted.

### Root Cause

NetBird encrypts actor emails in the event store. If the server restarts after the key changes or the key is deleted/lost, previously-encrypted names cannot be decrypted.

### Fix for Self-Hosted: Combined Container (`netbird-server` / `config.yaml`)

Add or verify the `server.store.encryptionKey` field in `config.yaml`:

```yaml
server:
  store:
    encryptionKey: "<hex-encoded-32-byte-key>"
```

Generate a key:
```bash
openssl rand -hex 32
```

**Back this key up**. If it is lost, all actor names in the audit log become permanently unreadable.

### Fix for Self-Hosted: Legacy Multi-Container Setup (`management.json`)

Set the `DataStoreEncryptionKey` field in `management.json`:

```json
{
  "DataStoreEncryptionKey": "<base64-or-hex-key>"
}
```

Restart the management container after updating the key.

---

## Event Streaming (Cloud-Only)

> **Availability**: Cloud-hosted NetBird only. Not available in self-hosted deployments.

Real-time event streaming pushes audit events to external SIEM and observability platforms as they occur.

### Supported Destinations

| Platform | Notes |
|----------|-------|
| **Datadog** | Sends events via Datadog Logs API |
| **Amazon S3** | Stores events as JSON objects in a bucket |
| **Amazon Data Firehose** | Streams events to Firehose delivery streams (→ S3, Redshift, Elasticsearch, etc.) |
| **Generic HTTP** | POST JSON payloads to any webhook endpoint |

### Configuration

Dashboard → **Activity → Event Streaming → Add Integration**:

1. Choose destination type.
2. Provide credentials (API key, AWS IAM role ARN, or HTTP endpoint URL + optional auth header).
3. Enable the integration.

Events begin streaming immediately after enabling. Historical events are **not** backfilled.

### Planned Features

Connection events (peer A connected to peer B, with duration and bytes) are on the roadmap but not yet available as of the last documentation update.

---

## References

- [Audit Log](https://docs.netbird.io/manage/activity)
- [Event Streaming](https://docs.netbird.io/manage/activity/event-streaming)
