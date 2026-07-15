# MPS Training Patterns (Apple Silicon)

Proven patterns for training small LMs on Apple Silicon MPS from custom JSONL corpora.
Derived from the Ch6 GeoReason experiment (GPT-2 124M + TinyLlama-1.1B LoRA on Keele energy data).

## Training Script Template

Key structure for a `train_lm.py` that streams + shuffles + packs JSONL:

```python
import random, json, torch
from transformers import GPT2LMHeadModel, GPT2Tokenizer
from transformers import get_cosine_schedule_with_warmup
from torch.utils.data import IterableDataset, DataLoader

SEED = 42
SEQ_LEN = 512
BATCH_SIZE = 8
GRAD_ACCUM = 4        # effective batch = BATCH_SIZE * GRAD_ACCUM
LR = 2e-5             # safe for GPT-2 full FT on MPS
WARMUP_PCT = 0.02     # 2% of optimizer steps

class ShuffledPackingDataset(IterableDataset):
    """Read all sentences, shuffle with fixed seed, pack to seq_len."""
    def __init__(self, path, tokenizer, seq_len, seed=42):
        self.path = path; self.tokenizer = tokenizer
        self.seq_len = seq_len; self.seed = seed

    def __iter__(self):
        rng = random.Random(self.seed)
        sentences = []
        with open(self.path) as f:
            for line in f:
                sentences.append(json.loads(line)["text"])
        rng.shuffle(sentences)
        buffer = []
        for text in sentences:
            ids = self.tokenizer.encode(text, add_special_tokens=False)
            buffer.extend(ids)
            while len(buffer) >= self.seq_len + 1:
                chunk = buffer[:self.seq_len + 1]
                buffer = buffer[self.seq_len:]
                yield {
                    "input_ids": torch.tensor(chunk[:-1], dtype=torch.long),
                    "labels": torch.tensor(chunk[1:], dtype=torch.long)
                }
```

## Critical Bugs Found and Fixed

### 1. Unshuffled Corpus -> Misleading Loss Curves
**Symptom**: Loss crashes from 14.77 to 0.63 in 500 steps, then RISES to 1.51 by step 2000.
**Root cause**: Corpus ordered by meter -> date. First 500 steps train on near-zero residence meters only. Easy memorization -> fake low loss. Later steps hit different buildings -> loss rises.
**Fix**: Shuffle ALL sentences before packing with fixed seed (42). Load full corpus into memory, shuffle, then pack. Same seed for A and B so comparison is fair.

### 2. Scheduler Steps vs Batch Steps Mismatch
**Symptom**: Cosine schedule barely decays - model finishes training at ~75% of max LR.
**Root cause**: `num_training_steps` was set to `batch_steps` (e.g., 6347), but `scheduler.step()` only fires every `GRAD_ACCUM` batches.
**Fix**:
```python
batch_steps = token_budget // (BATCH_SIZE * SEQ_LEN)
opt_steps = batch_steps // GRAD_ACCUM
scheduler = get_cosine_schedule_with_warmup(
    optimizer,
    num_warmup_steps=warmup_steps,
    num_training_steps=opt_steps  # correct: optimizer calls, not batches
)
```

### 3. Warmup as Percentage, Not Fixed Steps
**Fix**: Compute warmup dynamically as percentage of optimizer steps:
```python
warmup_steps = max(1, int(opt_steps * 0.02))  # 2% = ~32 steps for 1,586 total
```

### 4. Duplicate Model Load for Initial Loss Hangs MPS
**Fix**: Compute initial loss on the SAME model before training:
```python
model.eval()
with torch.no_grad():
    test_batch = next(iter(loader))
    test_batch = {k: v.to(DEVICE) for k, v in test_batch.items()}
    initial_loss = model(**test_batch).loss.item()
model.train()
```

## LoRA on MPS (1.1B model, batch_size=2, grad_accum=8)

```python
from peft import LoraConfig, get_peft_model, TaskType

lora_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    task_type=TaskType.CAUSAL_LM,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],
)
model = get_peft_model(model, lora_config)
# TinyLlama-1.1B: 4.5M trainable / 1.1B total = 0.41%
# LR 2e-4 (typical for LoRA, higher than full FT)
# Speed: ~550-995 tok/s on M1 Max MPS (slow - overnight runs expected)
```

## MPS Performance Notes

| Model | Batch | Speed (tok/s) | Notes |
|-------|-------|---------------|-------|
| GPT-2 124M | 8x4 | 5,700->4,000 | Thermal throttle after 30+ min |
| TinyLlama-1.1B LoRA | 2x8 | 360->995 | Much slower, ~3.6h for 13M tokens |

- `PYTORCH_ENABLE_MPS_FALLBACK=1` required
- fp32 only (MPS bf16 unstable)
- Speed degrades over sustained runs (thermal throttling on MacBook/Mini)
- MPS is NOT bitwise deterministic - weight SHA differs between identical reruns, but loss values reproduce exactly

## Output Artifacts Per Condition

```
models/cond{A,B,B2}/
  model.safetensors     # GPT-2 weights
  config.json
  tokenizer.json
  TRAIN_STATS.json      # condition, lr, tokens_seen, initial/final loss, wall_time, SHA
  loss.jsonl            # step, loss, lr per step

models/condC_lora/
  adapter_model.safetensors  # LoRA weights only
  adapter_config.json
  TRAIN_STATS.json
  loss.jsonl
```

## TRAIN_STATS.json Schema

```json
{
  "condition": "A|B|B2|C",
  "model": "openai-community/gpt2",
  "params": 124000000,
  "device": "mps",
  "seed": 42,
  "seq_len": 512, "batch_size": 8, "grad_accum": 4,
  "lr": 2e-5, "warmup_steps": 31,
  "tokens_budget": 26000000, "tokens_seen": 25997312,
  "batch_steps": 6347,
  "initial_loss": 11.12, "final_loss": 0.89,
  "wall_time_s": 4444.5,
  "weight_sha256": "bb5fc52e...",
  "corpus": "/path/to/corpus.jsonl"
}
```
