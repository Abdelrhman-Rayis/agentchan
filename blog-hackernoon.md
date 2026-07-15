# Let Your AI Agents Talk to Each Other in Minutes

I had two terminals open. Claude Code was in one, planning a machine learning
experiment. Hermes was in the other, ready to run it. And they had absolutely no way
to talk to each other.

Here is every approach I tried, in order, and where each one broke.

---

## Approach 1: The Human Clipboard

The obvious starting point. Claude drafts a task. I read it. I switch to Hermes'
terminal. I type it in. Hermes responds. I read the response. I switch back to
Claude. I type it in. Repeat.

This works for one message. Maybe two. It does not scale past a dozen exchanges,
and it completely defeats the point of having two agents. The whole reason I had
two terminals running was so the agents could coordinate without me in the middle,
not so I could become a manual message bus with extra steps.

Copy-paste is faster than retyping but solves none of the structural problems.
Messages have no IDs. There is no history beyond terminal scrollback. If I look
away for five minutes and both agents output something, the context is buried under
diagnostic noise. I am the bottleneck, and I am a slow one.

---

## Approach 2: tmux send-keys

If both agents are running inside tmux panes, you can type directly into another
pane's terminal:

```bash
# Claude sends a task to Hermes in pane %1
tmux send-keys -t mysession:0.1 \
  "Train GPT-2 on corpus.jsonl for 26M tokens. Save to models/condA/." Enter
```

This is genuinely better. No human fingers involved. The text lands on Hermes'
prompt and submits as if the user typed it. You can script it. You can chain
commands.

To read a response, you pull from scrollback:

```bash
tmux capture-pane -t mysession:0.1 -p -S -80
```

And to see who is where:

```bash
tmux list-panes -a
```

This gets you perhaps 70% of the way there. For quick one-offs between two panes,
it is fast and needs no additional tooling.

**Where it breaks:**

First, `send-keys` has no delivery confirmation. You type into the void and hope
the recipient was listening. If Hermes was mid-output, your message lands in the
middle of a log line and gets swallowed.

Second, there is no message structure. No IDs, no threading, no types. Every
message is raw text dumped onto a prompt. If Claude sends three tasks in quick
succession, Hermes has no way to know which reply goes with which request.

Third, and most critically, **tmux send-keys silently fails when the peer is in
manual mode.** Claude Code enters a `⏸ manual mode on` state when it needs user
approval for shell commands. In that state, `send-keys` types the text onto the
prompt line but the Enter key is intercepted and never submitted. The message sits
there, invisible, until someone manually disables manual mode and presses Enter.
You think you sent a task. The other agent never saw it.

---

## Approach 3: A Shared File

The next logical step: write messages to a file that both agents watch.

```bash
# Claude writes a task
echo '{"from":"claude","to":"hermes","body":"train model"}' > /tmp/tasks/hermes.json

# Hermes polls for it
while true; do
  if [ -f /tmp/tasks/hermes.json ]; then
    cat /tmp/tasks/hermes.json
    mv /tmp/tasks/hermes.json /tmp/tasks/hermes.done.json
    break
  fi
  sleep 2
done
```

Now you have durability. Messages survive crashes. You can `cat` the history. You
are no longer tied to tmux. This is the kernel of a real protocol.

**Where it breaks:**

A single file means one message at a time. A second message overwrites the first
before it is read. Multiple senders race on the same file. There is no registry so
you hardcode agent names. Polling is wasteful. And a `mv` from one agent can race
with a `cat` from another — partial reads, double-processed messages, lost tasks.

The idea is right. The implementation needs work.

---

## Approach 4: AgentChan

This is where I ended up. AgentChan takes the shared-file idea and makes it
production-grade without making it complicated. It is a 381-line Python script
with no dependencies. It sits in `/tmp/agent-channel/` and runs entirely over the
filesystem.

Every agent gets a maildir-style inbox:

```
/tmp/agent-channel/
  registry/claude.json    →  { "name": "claude", "status": "online" }
  registry/hermes.json    →  { "name": "hermes", "status": "busy" }
  claude/new/             →  unread messages
  claude/cur/             →  read messages
  hermes/new/
  hermes/cur/
  chat.log                →  unified transcript of everything
```

An agent registers once:

```bash
agentchan register --name claude
agentchan register --name hermes
```

Sending is a one-liner:

```bash
agentchan send --to hermes --type task --subject "train model" \
  "Train GPT-2 on corpus.jsonl for 26M tokens."
```

The recipient checks and claims:

```bash
agentchan inbox     # "1 unread: task from claude"
agentchan read      # displays the message, atomically moves new/ → cur/
```

Replies are threaded:

```bash
agentchan send --to claude --re abc12345 --type result \
  "Done. Final loss 0.89."
```

### What makes it different from the shared-file approach

**Atomic delivery.** A message is written to a temp file in `tmp/`, `fsync`'d,
then `os.rename`'d into `new/`. On the same filesystem, `rename` is atomic per
POSIX. A reader sees either no file or the complete file — never a partial write.
Concurrent senders cannot tear a message.

**Atomic claiming.** Reading moves the file from `new/` to `cur/` with another
atomic rename. No double-processing. No lost messages.

**Multiple messages.** Each message is a distinct file named
`<epoch_ns>-<from>-<shortid>.json`. Lexical sort equals chronological order. A
hundred messages can sit in the inbox at once — no overwrites, no races.

**Registry.** `agentchan who` tells every agent who else is online, their status,
when they were last seen, and how many unread messages are waiting. No hardcoded
names.

**Threading.** Every message carries an optional `re` field — the ID of the
message it replies to. `agentchan history --with claude` reconstructs the full
thread.

**Notifications.** If the recipient registered with a tmux pane, AgentChan nudges
them via `send-keys` on delivery. It is the best part of Approach 2, automated. If
the peer is in manual mode and the nudge silently fails, the message is still safe
in the inbox — it will be found on the next `agentchan inbox` or `watch`.

**Unified transcript.** `tail -f /tmp/agent-channel/chat.log` shows every message
across every agent in real time, in human-readable blocks. You can watch your
agents converse like reading a group chat.

---

## What It Got Used For

The first real test was a two-agent ML training session. Claude planned the
experiment: four conditions, GPT-2 on Apple Silicon MPS, 26 million tokens each.
Hermes executed: dataset loading, training loops, loss logging.

Thirty messages over two hours. Claude sent tasks. Hermes acked them. Training ran
for 100 minutes and Hermes sent periodic status updates — step count, current
loss, learning rate. At one point Claude spotted the corpus was unshuffled and sent
a mid-run correction. Hermes stopped, fixed the shuffle, restarted, and carried on.
This is coordination you cannot script in advance.

The experiment surfaced real issues — messages piling up during long runs, the
`read --id` flag silently failing on substring mismatches, hand-written messages
not appearing in the JSON inbox output. These are documented in the README under
Known Issues, with filesystem-level fallbacks.

---

## Comparison

| | Copy-Paste | tmux send-keys | Shared File | AgentChan |
|---|---|---|---|---|
| No human in loop | ✗ | ✓ | ✓ | ✓ |
| Delivery guarantee | ✗ | ✗ | ✗ | ✓ (atomic) |
| Multiple messages | ✗ | ✗ | ✗ | ✓ |
| Threading | ✗ | ✗ | ✗ | ✓ |
| Registry / discovery | ✗ | ✗ | ✗ | ✓ |
| History | ✗ | scrollback only | ✓ | ✓ |
| Works outside tmux | ✓ | ✗ | ✓ | ✓ |
| Survives manual mode | — | ✗ | ✓ | ✓ |
| Dependencies | none | tmux | none | Python 3.7+ |

---

## What AgentChan Is Not

It is not a task queue. There is no scheduling, no priority, no worker pools, no
authentication, no encryption. It trusts the filesystem and assumes a single-user
machine.

It does not replace RabbitMQ or Redis pub/sub. It replaces copy-pasting between
terminal windows when you have multiple AI agents and want them to coordinate
without you.

---

## The Code

Everything is in one file. The CLI is `argparse` with subcommands: `register`,
`who`, `send`, `inbox`, `read`, `watch`, `history`, `await`. Each command is a
function of 10 to 30 lines. The protocol spec is a separate document with the
message schema and atomicity guarantees. The examples directory holds real session
transcripts and training patterns.

It is on GitHub at
[github.com/Abdelrhman-Rayis/agentchan](https://github.com/Abdelrhman-Rayis/agentchan).
MIT licensed.

I went through four approaches so you can skip to the last one. If you run
multiple AI agents and find yourself being the human router between them, you
probably need this.
