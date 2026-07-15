# Let Your AI Agents Talk to Each Other in Minutes

I had two terminals open. Claude Code was in one, planning a machine learning
experiment. Hermes was in the other, ready to run it. And they had absolutely no way
to talk to each other.

I could copy-paste. I could type messages by hand. Claude could draft a task and I
could ferry it across like a human clipboard. But that defeated the point. The whole
reason I had two agents running was so they could coordinate without me in the
middle.

So I wrote a 381-line Python script called AgentChan.

## What It Is

AgentChan is a filesystem-based messaging protocol for AI agents. It sits in
`/tmp/agent-channel/` and works like an old-school maildir: each agent gets an inbox
with `new/` and `cur/` directories. Messages are JSON files. Delivery is atomic —
write to a temp file, `fsync`, `os.rename` into the inbox. No partial reads, no
double-processing, no race conditions.

There is no server. No broker. No configuration file. It is a single Python script
with zero dependencies beyond the standard library. If your agents share a
filesystem, they can talk.

## How It Works

An agent registers with a name:

```
agentchan register --name claude
agentchan register --name hermes
```

Now each agent knows the other exists. `agentchan who` lists everyone, their status,
and how many messages are waiting for them.

Sending a message is a one-liner:

```
agentchan send --to hermes --type task --subject "train model" \
  "Train GPT-2 on corpus.jsonl for 26M tokens."
```

The recipient checks their inbox, reads the oldest unread, and claims it:

```
agentchan inbox     # "1 unread: task from claude"
agentchan read      # displays the message, moves it to cur/
```

Replies are threaded with `--re <correlation-id>`. A task gets an ack. Long-running
work sends status updates. Completion sends a result. The whole conversation is a
linked chain stored in flat JSON files you can inspect with `cat` and `jq`.

A unified `chat.log` sits at the root — every message across every agent, appended
in human-readable blocks. `tail -f /tmp/agent-channel/chat.log` and you are watching
your agents converse in real time.

## Why Files Instead of Sockets

This was a deliberate choice. A message broker means another process to manage. A
socket means port assignments and connection handling and reconnection logic.
Filesystem operations mean your agents communicate the same way they read configs and
write output — through paths they already have access to.

The trade-off is latency. A file-based inbox isn't going to do microsecond dispatch.
But agents work in seconds-to-minutes. When Claude delegates a 100-minute training
run to Hermes, the overhead of a filesystem rename is noise.

The upside is durability. Nothing is held in memory. If both agents crash, the
messages are still there on disk in `new/` and `cur/`. Restart and pick up where you
left off.

## The Nudge Problem

Files are passive. An agent that doesn't check its inbox won't know a message
arrived. If both agents are inside tmux, AgentChan nudges the recipient's pane with
`tmux send-keys` — it types a short notification into their terminal. It is a
best-effort delivery signal, not a guarantee.

For agents outside tmux, the receiving side runs `agentchan watch` and blocks until a
message arrives. It is polling, but the polling interval is configurable and the
overhead is one `glob` call per tick.

## What It Got Used For

The first real test was a two-agent ML training session. Claude planned the
experiment: four conditions, GPT-2 on Apple Silicon MPS, 26 million tokens each.
Hermes executed: dataset loading, training loops, loss logging.

The conversation was around 30 messages over two hours. Claude sent tasks. Hermes
acked them. Training ran for 100 minutes and Hermes sent periodic status updates —
step count, current loss, learning rate. At one point Claude spotted that the corpus
was unshuffled and sent a mid-run correction. Hermes stopped, fixed the shuffle,
restarted, and carried on. This is the kind of back-and-forth you cannot do with a
shell script.

The experiment uncovered some real issues — messages piling up during long training
runs, the `read --id` flag silently failing on substring mismatches, hand-written
messages not appearing in the JSON inbox output. These are documented in the README
under Known Issues, with filesystem-level fallbacks.

## What It Is Not

AgentChan is not a task queue. There is no scheduling, no priority, no worker pools.
It does not handle authentication or encryption — it trusts the filesystem, so it
assumes a single-user machine.

It is not a replacement for RabbitMQ or Redis pub/sub. It is a replacement for
copy-pasting between terminal windows when you have more than one AI agent running
and you want them to coordinate without you.

## The Code

Everything is in one file. The whole CLI is built on `argparse` with subcommands:
`register`, `who`, `send`, `inbox`, `read`, `watch`, `history`, `await`. Each
command is a function around 10-30 lines. The protocol spec is a separate markdown
document with the message schema and atomicity guarantees. The examples directory has
real session transcripts and training patterns from the MPS experiment.

It is on GitHub at [github.com/Abdelrhman-Rayis/agentchan](https://github.com/Abdelrhman-Rayis/agentchan). MIT licensed.

I built this because I needed it. If you run multiple AI agents and find yourself
being the human router between them, you might need it too.
