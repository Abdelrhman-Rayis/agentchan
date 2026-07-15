<p align="center">
  <img src="https://img.shields.io/badge/python-3.7%2B-blue" alt="Python 3.7+">
  <img src="https://img.shields.io/badge/dependencies-zero-brightgreen" alt="Zero Dependencies">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="MIT License">
  <img src="https://img.shields.io/badge/lines-381-lightgrey" alt="381 lines">
</p>

# AgentChan

A filesystem-based messaging protocol that lets AI agents talk to each other.
No servers. No brokers. No configuration. Just a single Python script and a shared
directory.

```
 claude                hermes              claude
   │                     │                   │
   │  agentchan send ────►  new/             │
   │  --to hermes          │                 │
   │                     agentchan read      │
   │                     (claims → cur/)     │
   │                     agentchan send ────►  new/
   │                     --to claude           │
   │                     --re abc123         agentchan read
   │                                         (threaded reply)
```

---

## The Problem

You have two AI agents running in separate terminals. Claude is planning an
experiment. Hermes is ready to run it. They share a filesystem but have no way to
coordinate without you copy-pasting between windows.

You could use tmux `send-keys` — fragile, no history, silently fails in manual mode.
You could spin up Redis or RabbitMQ — overkill for a laptop.

## The Solution

AgentChan gives each agent a **mailbox**. Messages are JSON files. Delivery is
atomic. History is automatic and `grep`-friendly. A registry tells you who is online.

```
/tmp/agent-channel/
  registry/
    claude.json          "name": "claude", "status": "online"
    hermes.json          "name": "hermes", "status": "busy"
  claude/
    new/                 unread messages
    cur/                 read messages
    tmp/                 staging (atomic writes)
  hermes/
    new/
    cur/
    tmp/
  broadcast/             audit copy of every broadcast
  chat.log               unified transcript — tail -f to watch agents converse
```

---

## Quick Start

```bash
# Install — it is one file
curl -o ~/.local/bin/agentchan https://raw.githubusercontent.com/Abdelrhman-Rayis/agentchan/main/agentchan
chmod +x ~/.local/bin/agentchan
```

**Agent A — Claude — registers and sends a task:**

```bash
agentchan register --name claude
agentchan send --to hermes --type task --subject "train model" \
  "Train GPT-2 on corpus.jsonl for 26M tokens. Save to models/condA/."
```

**Agent B — Hermes — reads, acks, and delivers the result:**

```bash
agentchan register --name hermes
agentchan inbox          # "1 unread: task from claude"
agentchan read           # reads + claims the message

# Acknowledge receipt
agentchan send --to claude --re abc12345 --type ack "Starting training now."

# ... 100 minutes of training later ...

agentchan send --to claude --re abc12345 --type result \
  "Done. Final loss 0.89. Model at models/condA/model.safetensors."
```

**Claude checks the full thread:**

```bash
agentchan history --with hermes
```

---

## Commands

| Command | What it does |
|---------|-------------|
| `register --name NAME` | Announce your presence so others can find you |
| `who` | List every registered agent and their status |
| `send --to NAME "body"` | Deliver a message to an agent's inbox |
| `inbox` | See what is waiting for you |
| `read` | Read and claim the oldest unread message |
| `watch` | Block until messages arrive, then read them |
| `history` | Browse your past conversations |
| `await` | Block until your inbox is non-empty, then exit |

### Send Options

```
agentchan send --to NAME                    recipient (or * for broadcast)
               --type TYPE                  msg | task | reply | ack | status
                                            query | result | error | greeting
               --subject "summary"          short description
               --re CORRELATION_ID          thread this as a reply
               --meta '{"key":"val"}'       attach structured data
               --no-nudge                   skip tmux notification
```

### Read Options

```
agentchan read              oldest unread → claims it (new/ → cur/)
agentchan read --id abc12   specific message by ID
agentchan read --all        every unread message
agentchan read --peek       read without claiming (stays in new/)
```

---

## Message Format

Every message is a JSON file named `<epoch_ns>-<from>-<shortid>.json`:

```json
{
  "id":       "a1b2c3d4",
  "ts":       "2026-07-15T19:51:28+00:00",
  "from":     "claude",
  "to":       "hermes",
  "type":     "task",
  "re":       null,
  "subject":  "train model",
  "body":     "Train GPT-2 on corpus.jsonl for 26M tokens.",
  "meta":     {}
}
```

| Field | Purpose |
|-------|---------|
| `id` | Unique 8-char hex identifier |
| `ts` | ISO-8601 UTC timestamp |
| `from` | Sender agent name |
| `to` | Recipient agent name (or `*` for broadcast) |
| `type` | Message category — drives agent behavior |
| `re` | Correlation ID for threaded replies |
| `subject` | Short human-readable summary |
| `body` | Free-text message content |
| `meta` | Arbitrary JSON payload |

---

## Real Session: Two Agents, One ML Experiment

A walkthrough of the first real AgentChan session — Claude (planner) and Hermes
(executor) coordinating a 4-condition GPT-2 training run on Apple Silicon.

```
[19:51:28] claude → hermes  #abc12345  (task)
  subject: train condA
  Train GPT-2 on corpus.jsonl for 26M tokens. Models/condA/.

[19:51:35] hermes → claude  #def45678  (ack)  re:abc12345
  Received. Starting training on MPS.

[20:05:12] hermes → claude  #ghi90123  (status)  re:abc12345
  Step 500/6347, loss 2.90, LR 1.97e-5

[20:18:44] claude → hermes  #jkl45678  (task)  re:abc12345
  STOP. Corpus appears unshuffled — loss rise is an artifact.
  Shuffle with seed=42 and restart condA.

[20:19:01] hermes → claude  #mno56789  (ack)  re:abc12345
  Fixed. Corpus shuffled. Restarting condA.

[21:45:33] hermes → claude  #pqr01234  (result)  re:abc12345
  condA complete. Final loss 0.89. Model at models/condA/model.safetensors.
```

30 messages across 2 hours. The planner caught a data bug mid-run and redirected
the executor — the kind of back-and-forth you cannot script in advance.

---

## Design

**Durable.** Every message is a distinct file, never overwritten. Crash both agents
and the messages are still on disk in `new/` and `cur/`. Restart and carry on.

**Safe.** Delivery is atomic: write to `tmp/`, `fsync`, `os.rename` into `new/`.
Concurrent senders can never tear a message. Reads claim by atomic rename (`new/` →
`cur/`) — no double-processing.

**Observable.** Registry is human-readable JSON. Messages are JSON. `chat.log` is a
unified transcript. Inspect with `cat`, `ls`, `jq`, or `tail -f`.

**Notified.** On send, the recipient's tmux pane gets nudged via `send-keys` (if
`--pane` was set at registration). Outside tmux, use `agentchan watch` for blocking
receive.

---

## Fallback & Debugging

<details>
<summary>When the CLI misbehaves, use raw filesystem access</summary>

```bash
# Read all unread messages
cat /tmp/agent-channel/hermes/new/*.json

# Claim a message by hand
mkdir -p /tmp/agent-channel/hermes/cur/
mv /tmp/agent-channel/hermes/new/<msg>.json /tmp/agent-channel/hermes/cur/

# Watch agents converse in real time
tail -f /tmp/agent-channel/chat.log
```
</details>

<details>
<summary>Direct tmux communication (no agentchan needed)</summary>

```bash
tmux send-keys -t session:window.pane "message" Enter
tmux capture-pane -t session:window.pane -p -S -80
tmux list-panes -a
```

**Warning:** tmux `send-keys` silently fails when the peer is in manual mode
(Claude Code `⏸ manual mode on`). Use AgentChan instead.
</details>

---

## Known Issues

- `agentchan read --id X` can miss on substring mismatches → fall back to
  `cat new/*.json` then `mv` to `cur/`
- `agentchan inbox --json` may return `[]` for hand-written messages → verify
  with `ls /tmp/agent-channel/<name>/new/`
- Always register with `--name` — without it, the tmux pane title becomes your
  identity (a GPU ID like "GVK6XG7190" is not a good agent name)
- Messages pile up during long runs → claim them in bulk:
  `cat new/*.json && mv new/*.json cur/`

---

## Environment

| Variable | Default | Purpose |
|----------|---------|---------|
| `AGENT_CHANNEL_DIR` | `/tmp/agent-channel` | Channel root |
| `AGENT_NAME` | hostname | Override sender name |
| `TMUX_PANE` | auto-detected | tmux pane for nudging |

## Requirements

- Python 3.7+
- tmux (optional — only needed for pane nudging)
- Shared filesystem between agents

---

## License

MIT — see [LICENSE](LICENSE).

## Author

AgentChan was developed by [Abdelrhman Rayis](https://github.com/Abdelrhman-Rayis)
as part of the Hermes Agent project by [Nous Research](https://nousresearch.com).
