# AgentChan — Inter-Agent Communication Protocol

A durable, concurrency-safe messaging protocol for AI agents sharing a filesystem.
Designed for multi-agent coordination — Claude Code, Hermes Agent, Cursor, Codex, or any
LLM-based agent that can read/write files and run shell commands.

**Single Python file. Zero dependencies beyond Python 3 stdlib. No server, no daemon, no
config.**

## Why

When you run multiple AI agents in parallel terminals (or tmux panes), they need to talk
to each other. You could use tmux `send-keys` — fragile, race-prone, no history. You
could spin up a message broker — overkill for a laptop.

AgentChan gives you maildir-style message passing over a shared `/tmp` directory. Every
message is a distinct JSON file. Delivery is atomic. History is automatic and grep-friendly.
The registry tells you who's online.

## Quick Start

```bash
# Install
cp agentchan ~/.local/bin/agentchan
chmod +x ~/.local/bin/agentchan

# Agent A registers
agentchan register --name claude

# Agent B registers
agentchan register --name hermes

# See who's online
agentchan who

# Claude sends a task to Hermes
agentchan send --to hermes --type task --subject "train model" \
  "Please train GPT-2 on corpus.jsonl for 26M tokens"

# Hermes checks inbox
agentchan inbox

# Hermes reads + claims the oldest unread
agentchan read

# Hermes sends a threaded reply
agentchan send --to claude --re abc12345 --type result \
  "Done. Final loss 0.89. Model saved to models/condA/"

# Claude checks the thread
agentchan history --with hermes
```

## Layout

```
/tmp/agent-channel/          (configurable via $AGENT_CHANNEL_DIR)
  registry/<name>.json       presence: name, pane, pid, started, last_seen, status
  <name>/new/                delivered-but-unread messages (maildir "new")
  <name>/cur/                read/processed messages       (maildir "cur")
  <name>/tmp/                staging for atomic delivery
  broadcast/                 audit copy of every "*" message
  chat.log                   unified transcript of all messages
```

Message filename: `<epoch_ns>-<from>-<shortid>.json` (lexical sort == chronological).

## Message Schema

```json
{
  "id": "8-hex",
  "ts": "ISO-8601 UTC",
  "from": "claude",
  "to": "hermes | *",
  "type": "msg|task|reply|ack|status|query|result|error|greeting",
  "re": "<correlation id or null>",
  "subject": "short summary",
  "body": "free text",
  "meta": {}
}
```

## CLI Reference

### `register` — announce presence (do this FIRST)

```bash
agentchan register --name NAME [--pane %N] [--status online]
```

Sets your presence in the registry so others can find and notify you. The `--pane` is
your tmux pane ID (`echo $TMUX_PANE`) — used for nudging (tmux `send-keys` notification
on message delivery). If you're not in tmux, omit `--pane` and nudges are skipped.

### `who` — list all agents

```bash
agentchan who
```

Shows name, pane, status, last-seen age, and unread count for every registered agent.

### `send` — send a message

```bash
agentchan send --to NAME --type TYPE --subject "subject" "body text"
agentchan send --to NAME --re CORRELATION_ID "reply body"
agentchan send --to '*' --type broadcast "everyone sees this"
echo "body from stdin" | agentchan send --to NAME
```

Options:

| Flag | Description |
|------|-------------|
| `--to` | Recipient name or `*` for broadcast |
| `--type` | `msg`, `task`, `reply`, `ack`, `status`, `query`, `result`, `error`, `greeting` |
| `--subject` | Short summary |
| `--re` | Correlation ID for threading |
| `--from` | Override sender name |
| `--meta` | JSON string of structured payload |
| `--no-nudge` | Skip tmux pane notification |

### `inbox` — check for new messages

```bash
agentchan inbox               # human-readable summary
agentchan inbox --json        # full JSON of all unread messages
```

### `read` — read and claim messages

```bash
agentchan read                # oldest unread, claims it (moves new/ -> cur/)
agentchan read --id ID        # specific message by id
agentchan read --all          # all unread
agentchan read --peek         # read without claiming (stays in new/)
```

### `watch` — blocking receive loop

```bash
agentchan watch               # block until messages arrive, then read them
agentchan watch --once         # block for one message then exit
agentchan watch --interval 1.0  # poll every N seconds
```

### `history` — conversation log

```bash
agentchan history                  # all your messages
agentchan history --with claude    # thread with a specific agent
agentchan history --all-agents     # everything across all agents
```

### `await` — block until inbox non-empty

```bash
agentchan await
```

Blocks until at least one message arrives, prints a summary, then exits without claiming.
Useful for scripting: run in background, then invoke agent to handle messages.

## Message Types

| Type | When to use |
|------|-------------|
| `msg` | General message, default |
| `task` | Delegating work to another agent |
| `reply` | Generic reply |
| `ack` | Confirming receipt of a task |
| `status` | Progress update during long-running work |
| `query` | Asking a question |
| `result` | Delivering completed work |
| `error` | Reporting a failure |
| `greeting` | Initial handshake |
| `broadcast` | Message to all agents (use `--to '*'`) |

## Design Properties

**Durable.** Every message is a distinct file, never overwritten. Full history is always
available. Nothing is lost on restart.

**Safe.** Delivery is atomic: write to `tmp/`, then `os.rename` into `new/`. Concurrent
senders can never tear a message. Reads *claim* a message by atomic rename (`new/` →
`cur/`), so no double-processing.

**Addressed.** Every message has `from`, `to` (or `*` for broadcast), and an optional
correlation ID (`re`) for threading.

**Observable.** Registry is human-readable JSON. Messages are JSON. `chat.log` is a
unified transcript. Any person or agent can inspect the channel with `cat`, `ls`, `jq`,
or `tail -f`.

**Notified.** On send, agentchan nudges the recipient's tmux pane (if `--pane` was
registered) so nobody polls blindly. Outside tmux, use `agentchan watch` for blocking
receive.

## Worked Example: Multi-Agent ML Training

A concrete session from a real two-agent experiment: Claude (planner) delegates model
training to Hermes (executor).

1. **Claude sends task:** `agentchan send --to hermes --type task --subject "train condA" "Train GPT-2 on corpus.jsonl for 26M tokens. Save to models/condA/."`

2. **Hermes reads:** `agentchan read` — sees the task.

3. **Hermes acks:** `agentchan send --to claude --re <task_id> --type ack "Received, starting training."`

4. **Hermes starts long work** (100+ minutes of model training on MPS).

5. **Hermes sends checkpoints:** `agentchan send --to claude --re <task_id> --type status "Step 500/6347, loss 2.90"`

6. **Claude may interrupt mid-run** with a corrective task (e.g., "corpus is unshuffled, stop and restart"). Hermes stops, fixes, restarts, acks.

7. **Hermes delivers result:** `agentchan send --to claude --re <task_id> --type result "Done. Final loss 0.89. Model at models/condA/model.safetensors. TRAIN_STATS.json attached."`

8. **Claude verifies** against acceptance criteria, plans the next phase.

## Protocol Etiquette

1. **Register once at startup.** The pane is how others notify you.
2. **Reply with `--re <id>`** so threads stay linked.
3. **Use `--type ack`** to confirm receipt of a task; `--type result` when done.
4. **Always `read`** (not `--peek`) once you've handled a message, so it leaves your inbox.
5. **Never overwrite files** in the channel by hand — always go through `agentchan`.

## Fallback: Raw Filesystem Access

When `agentchan` CLI misbehaves (it's a young tool), you can always fall back to raw
filesystem operations:

```bash
# Read all unread messages
cat /tmp/agent-channel/<name>/new/*.json

# Claim a message by hand
mkdir -p /tmp/agent-channel/<name>/cur/
mv /tmp/agent-channel/<name>/new/<msg>.json /tmp/agent-channel/<name>/cur/

# Write a message directly (PREFER agentchan send for atomicity)
# But if you must:
cat > /tmp/agent-channel/<to>/new/$(date +%s%N)-manual-$(uuidgen | cut -c1-8).json << 'EOF'
{ "id": "...", "ts": "...", "from": "...", "to": "...", ... }
EOF
```

## Direct tmux Communication (when agentchan is unavailable)

```bash
# Send a message to the peer's pane
tmux send-keys -t <session>:<window>.<pane> "your message here" Enter

# Read the peer's response from scrollback
tmux capture-pane -t <session>:<window>.<pane> -p -S -80

# Check pane layout
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} title="#{pane_title}" cmd="#{pane_current_command}"'
```

**WARNING:** tmux `send-keys` silently fails when the peer is in manual mode (e.g., Claude
Code `manual mode on`). The text is typed onto the prompt line but never submitted. Use
agentchan instead for peers in manual mode.

## Known Issues

- `agentchan read --id X` may silently return nothing even when the message file exists in
  `new/`. ID matching is substring-based and can miss. **Fallback:** use raw filesystem
  access (`cat new/*.json` then `mv` to `cur/`).
- `agentchan read --all` can return `(nothing to read)` despite `.json` files in `new/`.
  This has been observed when messages were hand-written rather than sent via `agentchan
  send`. **Fallback:** same raw filesystem access.
- `agentchan inbox --json` can return `[]` while `.json` files sit in `new/`. Always
  verify with `ls /tmp/agent-channel/<name>/new/` if inbox looks empty.
- **Pane title leaks into agent name:** If you register without `--name`, agentchan uses
  the tmux pane title. A pane titled "GVK6XG7190" (a GPU/model ID) becomes the agent
  name. Always register explicitly: `agentchan register --name <your-name>`.
- **Messages accumulate during agent absence:** During long training runs
  (hours/overnight), 5-10 messages from different threads may pile up. Use
  `cat /tmp/agent-channel/<name>/new/*.json` to read everything at once, then
  `mv /tmp/agent-channel/<name>/new/*.json /tmp/agent-channel/<name>/cur/` to claim all.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AGENT_CHANNEL_DIR` | `/tmp/agent-channel` | Channel root directory |
| `AGENT_NAME` | hostname | Override sender name |
| `TMUX_PANE` | (auto-detected) | tmux pane for nudging |

## Requirements

- Python 3.7+
- tmux (optional, for pane nudging)
- Shared filesystem between agents

## License

MIT — see [LICENSE](LICENSE).

## Author

AgentChan was developed by Abdelrhman Rayis as part of the Hermes Agent project by
Nous Research, with contributions from real multi-agent ML training experiments.
