# AgentChan Protocol Specification v1.0.0

## Overview

AgentChan is a filesystem-based inter-agent communication protocol. Agents share a
directory tree under a configurable root (default `/tmp/agent-channel`). All operations
are atomic file renames — no locks, no servers, no daemons.

## Directory Layout

```
$AGENT_CHANNEL_DIR/
  registry/
    <name>.json              Agent presence record
  <name>/
    new/                     Unread messages (maildir "new")
    cur/                     Read/processed messages (maildir "cur")
    tmp/                     Staging for atomic delivery
  broadcast/                 Audit copy of broadcast messages
  chat.log                   Unified human-readable transcript
```

## Presence Registry

Each registered agent writes a JSON record to `registry/<name>.json`:

```json
{
  "name": "hermes",
  "pane": "%0",
  "pid": 15193,
  "started": "2026-07-15T19:51:28+00:00",
  "last_seen": "2026-07-15T19:51:28+00:00",
  "status": "online"
}
```

Fields:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent's registered name |
| `pane` | string | tmux pane ID for nudging (empty if not in tmux) |
| `pid` | int | Process ID at registration time |
| `started` | ISO-8601 | First registration timestamp (preserved across re-registrations) |
| `last_seen` | ISO-8601 | Last activity timestamp (updated on inbox, read, send, watch) |
| `status` | string | Free-form status (e.g., `online`, `busy`, `training`) |

Registration is atomic: write to a temp file in `registry/`, then `os.rename` into place.

## Message Format

Messages are JSON files. Filename: `<epoch_ns>-<from>-<shortid>.json`

```json
{
  "id": "a1b2c3d4",
  "ts": "2026-07-15T19:51:28+00:00",
  "from": "claude",
  "to": "hermes",
  "type": "task",
  "re": null,
  "subject": "train model",
  "body": "Please train GPT-2 on corpus.jsonl for 26M tokens.",
  "meta": {}
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string (8 hex) | Yes | Unique message ID (uuid4 hex truncated to 8 chars) |
| `ts` | ISO-8601 UTC | Yes | Send timestamp |
| `from` | string | Yes | Sender agent name |
| `to` | string | Yes | Recipient agent name, or `*` for broadcast |
| `type` | enum | Yes | Message type (see below) |
| `re` | string or null | No | Correlation ID for threading (the `id` of the message being replied to) |
| `subject` | string | No | Short human-readable summary |
| `body` | string | No | Free-text message body |
| `meta` | object | No | Arbitrary structured payload (JSON object) |

### Message Types

| Type | Semantic |
|------|----------|
| `msg` | General-purpose message |
| `task` | Work delegation — recipient is expected to act |
| `reply` | Generic reply |
| `ack` | Confirmation of receipt ("I got your task, starting work") |
| `status` | Progress update during long-running work |
| `query` | Question for the recipient |
| `result` | Delivery of completed work |
| `error` | Failure report |
| `greeting` | Initial handshake / presence announcement |

## Delivery Protocol

### Sending

1. Sender writes the JSON message to `<to>/tmp/<filename>` using `atomic_write` (temp file + `fsync` + `os.rename`).
2. Sender atomically moves the file from `tmp/` to `new/` — this is the "publish" step.
3. If the sender is delivering to `*` (broadcast), a copy is also written to `broadcast/`.
4. A human-readable line-block is appended to `chat.log`.
5. If the recipient has a registered `pane`, sender nudges via `tmux send-keys -t <pane>`.

### Receiving

1. Recipient lists files in `<name>/new/` sorted by filename (lexical order = chronological).
2. Recipient reads a message file.
3. Recipient atomically moves the file from `new/` to `cur/` — this is the "claim" step.
4. Peek mode (read without claiming) skips step 3.

### Broadcast

Broadcast messages (`to: "*"`) are delivered to every registered agent except the sender.
A permanent audit copy is stored in `broadcast/`.

## Threading

Messages are threaded via the `re` field. When replying to a message, set `re` to the
original message's `id`. This creates a linked chain:

```
task  (id: abc123, re: null)
  → ack   (id: def456, re: abc123)
    → status (id: ghi789, re: abc123)
      → result (id: jkl012, re: abc123)
```

The `history --with <agent>` command reconstructs threads by matching `re` chains.

## Atomicity Guarantees

- **Write atomicity:** temp file in `tmp/` → `fsync` → `os.rename` to `new/`. On the
  same filesystem, `rename` is atomic per POSIX. A reader sees either the old state (no
  file) or the complete new file — never a partial write.
- **Claim atomicity:** `os.rename` from `new/` to `cur/`. Same guarantee: a concurrent
  reader either sees the message in `new/` or not at all.
- **`chat.log` appends:** Small `O_APPEND` writes are atomic on POSIX. Concurrent agents'
  log lines stay ordered at byte granularity.

## Security Model

AgentChan trusts the filesystem. Any process with write access to `$AGENT_CHANNEL_DIR`
can read, send, or forge messages. The protocol assumes a single-user machine where
all agents run under the same UID.

For multi-user or containerized environments, set `$AGENT_CHANNEL_DIR` to a directory
with appropriate permissions (e.g., a shared volume with `0700`).

## Versioning

This is Protocol v1.0.0. Future versions will be noted in the registry and message
schemas via a `version` field. Backward compatibility: readers should ignore unknown
fields.
