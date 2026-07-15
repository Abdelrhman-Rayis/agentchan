# Example: Multi-Agent ML Training with AgentChan

This is a worked example from a real two-agent experiment: Claude Code (planner)
delegates GPT-2 training on Apple Silicon MPS to Hermes Agent (executor) via AgentChan.

## Session Walkthrough

1. **Claude sends task:**
   ```
   agentchan send --to hermes --type task --subject "train condA" \
     "Train GPT-2 on corpus.jsonl for 26M tokens. Save to models/condA/."
   ```

2. **Hermes reads and acks:**
   ```
   agentchan read
   agentchan send --to claude --re abc12345 --type ack "Received, starting training."
   ```

3. **Hermes starts training** (100+ minutes on M1 Max MPS).

4. **Hermes sends checkpoints:**
   ```
   agentchan send --to claude --re abc12345 --type status "Step 500/6347, loss 2.90"
   agentchan send --to claude --re abc12345 --type status "Step 2000/6347, loss 1.45"
   ```

5. **Claude interrupts mid-run** (detects unshuffled corpus):
   ```
   agentchan send --to hermes --re abc12345 --type task \
     "STOP. Corpus is unshuffled — loss rise is fake. Shuffle with seed=42 and restart."
   ```

6. **Hermes stops, fixes, restarts, acks:**
   ```
   agentchan send --to claude --re abc12345 --type ack "Fixed. Shuffled with seed=42. Restarting condA."
   ```

7. **Hermes delivers result:**
   ```
   agentchan send --to claude --re abc12345 --type result \
     "condA complete. Final loss 0.89. Model at models/condA/model.safetensors."
   ```

## Key Patterns

- **Threading with `--re`** keeps all messages in one conversation.
- **`--type ack`** confirms receipt before starting long work.
- **`--type status`** keeps the planner informed during long runs.
- **`--type result`** signals completion with deliverables.
- **Planner can interrupt and redirect** mid-task via new `--type task` with `--re`.

## Training-Specific Patterns

See [mps-training-patterns.md](mps-training-patterns.md) for MPS-specific script templates
and [ml-training-pitfalls.md](ml-training-pitfalls.md) for pitfalls discovered during
this experiment.
