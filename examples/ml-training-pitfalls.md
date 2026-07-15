# ML Training Pitfalls (discovered via AgentChan multi-agent experiment)

Captured from a Hermes+Claude agentchan multi-agent experiment (2026-07-12/13).
GPT-2 124M and TinyLlama-1.1B on M1 Max MPS, 26M-token budgets, 4 conditions.

## Pitfall 1: Unshuffled corpus -> fake low loss then rise

**Symptom**: Training loss drops dramatically (14.77 -> 0.63 in 500 steps), then steadily
RISES (-> 1.51 at step 2000). Looks like diverging, but the model is fine.

**Root cause**: Corpus ordered by meter then date. First 500 steps train only on near-zero
residence meters (trivially predictable). Later steps encounter different
meters/values/resources -> loss rises. This is non-i.i.d. batching from ordered data, not
instability.

**Fix**: Shuffle sentences with fixed seed (42) before packing into sequences. Loss curve
becomes healthy monotone decrease. For knowledge-probing experiments this is a validity
requirement: the model must see a uniform mix.

**Detection**: Check corpus ordering early. `head corpus/*.jsonl` vs `tail corpus/*.jsonl`
- if line 1 and line 455000 are different meters, it is ordered.

## Pitfall 2: Scheduler steps count mismatch

**Symptom**: Cosine schedule barely decays. Model trains on 6,347 batches but scheduler
only steps every GRAD_ACCUM=4 -> 1,587 optimizer steps. If `num_training_steps=6347`, the
scheduler only reaches ~25% of its decay.

**Root cause**: `num_training_steps` was set to batch count, not optimizer step count.

**Fix**: `opt_steps = batch_steps // GRAD_ACCUM` and pass `num_training_steps=opt_steps`
to the scheduler. Warmup should be 1-2% of opt_steps, not batch_steps.

## Pitfall 3: Duplicate model load hangs on MPS

**Symptom**: Training hangs after model load, no output. `multiprocessing
resource_tracker` warning appears.

**Root cause**: Script loaded a second model copy on CPU for initial loss computation
while first copy was on MPS. Multiprocessing semaphore leak.

**Fix**: Compute initial loss on the same model instance (set eval mode, compute loss,
set train mode). No duplicate load needed.

## Pitfall 4: Cross-corpus loss is not comparable

**Symptom**: condA (ontology) final loss 0.89 vs condB (raw) 1.70. Tempting to claim
"structure wins."

**Reality**: Corpus A is templated prose ("in the X zone of building Y (Type), about N
kWh") - inherently more predictable. Lower loss is expected regardless of whether
structure helps. Only a held-out QA evaluation can test the hypothesis.

**Rule**: Never compare training loss across different corpora. The only valid comparison
is same-corpus, same-config (e.g., A vs B2 on identical content).

## Pitfall 5: MPS non-determinism

**Symptom**: Retraining condA with identical seed gives identical loss (11.12->0.89) but
different weight SHA-256.

**Rule**: Loss curves reproduce to tolerance. Weight SHAs do not. Document this, do not
treat SHA mismatch as a bug. Acceptable for research artefacts.
