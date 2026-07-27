# grug-llm

grug-llm lets a small local LLM (Qwen 14B on a consumer GPU, say) solve hard problems by trading latency for reliability. Instead of asking the model to reason harder in one shot, it runs the model in a long, tool using loop inside a disposable Docker sandbox, until it has gathered enough evidence to actually prove its answer. An hour, or overnight, is fine.

The philosophy is "stupid but diligent." A 14B model rarely has a flash of insight. It can be extremely persistent, never forget anything, and run thousands of cheap experiments before giving up. Persistence is undervalued.

## Core idea

Don't optimize for chain of thought. Optimize for an external research loop:

```
complex problem
  -> generate uncertainty
  -> split into smaller questions
  -> test the cheapest/highest value hypothesis
  -> update state
  -> repeat
  -> simple final decision
```

The final step is usually trivial, often a one line diff, because the hard part was never deducing the answer. It was building an environment where the answer becomes obvious. In most software engineering problems the difficulty is epistemic (which file, which variable, which cause), not mechanical. grug-llm spends compute reducing that uncertainty instead of on longer reasoning traces.

Two framings make this easy to reason about. It resembles the scientific method more than deduction: don't think your way to the truth, design the experiment that eliminates the most possibilities. And it resembles a theorem prover more than a chatbot: accumulate empirical proof (reproductions, tests, benchmarks) until every claim is justified, not merely asserted.

The output should read like a lab report, not a chat reply: a conclusion, a confidence level, a list of checks that passed, and an explicit list of what's still uncertain.

It's meant to be run as a batch job, not a conversation:

```bash
grug-llm investigate \
  --model qwen14b \
  --repo https://github.com/example/project \
  --question "Why does this benchmark regress on ARM?" \
  --max-runtime 8h

grug-llm status 42
grug-llm report 42
```

## Design principles

- **Python owns control flow, the LLM doesn't.** This keeps the system predictable and makes timeouts, iteration limits, and loop detection easy to add.
- **Every investigation gets a fresh, disposable container**, only `/workspace` mounted. Nothing persists except what's on disk.
- **State lives in small files, not one journal**: `question.md`, `hypotheses.md`, `evidence.md`, `todo.md`. The model never has to remember iteration 40, and only retrieves what's relevant to its current decision.
- **Observations are immutable, beliefs are not.** Raw tool output is never rewritten; interpretation lives separately and can change.
- **Confidence is numeric, not vibes.** Every hypothesis carries a confidence score. The loop stops only once confidence clears a threshold and no required verifications remain.
- **The tool set starts small**: search, fetch, git clone, grep, find, read, write, run. A general shell comes later, if at all. Small models plan better with a constrained action space.
- **The system is model agnostic.** State, tools, verification, and scheduling are the product. The LLM underneath is swappable as better local models appear.

## Related work

OpenHands, SWE-agent, and Aider are the closest existing projects, but all three optimize for autonomy or benchmark score (solve the issue, fast), not for patient, evidence-heavy verification. Aider in particular assumes a human drives and the LLM assists, rather than investigating unattended for hours. grug-llm's niche: a modest model that isn't in a hurry, and isn't trusted until it has tried and failed to prove itself wrong.

## Plan of action

### Phase 0: MVP with Python, Docker and an LLM

Prove the loop works, nothing more.

- Python CLI, takes a question and optional repo URL.
- One disposable Ubuntu container with internet access and the repo cloned in.
- One markdown journal the model reads and appends to each iteration.
- Minimal tools: `search`, `fetch`, `git_clone`, `grep`, `find`, `read`, `write`, `run`.
- Loop: read journal, ask for the single highest priority question and one action, execute it, append the result, repeat.
- Stop when the model reports no unverified hypotheses left, or a max iteration count is hit.

### Phase 1: Structured, persistent state

Replace the single journal with separate files, and give the model a real belief-tracking format.

- Split into `question.md`, `hypotheses.md`, `evidence.md`, `todo.md`, `summary.md`.
- Every hypothesis gets a status: untested, supported, contradicted, or unknown.
- Raw output goes into an append-only `observations/` directory; interpretation lives in a separate, editable file.
- Every turn states current confidence, the largest remaining uncertainty, and the next action, rather than just "what's next."
- Cache proven facts so the model never re-derives what it's already verified.

### Phase 2: Verification and review

One reviewing mechanism, used at two points, replaces what could otherwise become three overlapping ones (a critic, a judge, and an adversary).

- A fresh model instance reviews the current state, looking only for factual errors, unsupported claims, or contradicting evidence, and producing objections, never rewrites.
- Run it periodically during the investigation, to catch stalling early.
- Run it once more at the end, in a fresh container with the repo re-cloned, given only the question and the final answer, told to prove it wrong. Success reopens the investigation.
- Objections go back into the todo list; the loop continues until none remain.
- Replace "DONE" with a structured stop signal: `confidence`, `remaining_uncertainties`, `required_verifications`. Stop only when confidence clears a threshold and required verifications is empty.
- Final output: conclusion, confidence, a checklist of what was verified, and what's still uncertain.

### Phase 3: Efficiency and focus

Make the loop cheaper and harder to waste.

- Retrieve only the relevant handful of workspace files per decision, instead of feeding the whole journal into every prompt.
- Compress periodically: collapse old observations into a summary plus references.
- Detect loops: if consecutive actions are near-identical, interrupt and force a different avenue.
- Require every tool call to state its goal and how each possible result would change current beliefs. This is a cheaper, more honest substitute for asking the model to self-score a probability of success, which small models can't do reliably.

### Phase 4: Specialized roles

Split the single prompt into a small pipeline sharing the same model: planner, researcher, programmer, tester, editor. Small models do better with a narrow job than with one prompt doing everything, and since only one role runs at a time, this doesn't multiply GPU time.

Running several independent investigators from scratch and comparing conclusions is a legitimate way to raise confidence further, but it costs a real multiple of GPU time on a single card. Worth revisiting once the single-investigator loop is solid.

### Phase 5: Scheduling and checkpointing

Turn the script into something people actually leave running overnight.

- Job-style CLI: `investigate`, `status`, `report`, not a live chat interface.
- Periodic checkpoints: elapsed time, actions taken, tests run, hypotheses confirmed or disproven, estimated time remaining.
- Graceful pausing when the investigation genuinely can't proceed (needs network access, a newer compiler, human input), instead of silently looping.
- A full audit trail as output: hypotheses, evidence, commands run, failed approaches, remaining uncertainty, final answer, so a failed run can be inspected instead of restarted.

### Phase 6: Learning across investigations

Improve over time without a bigger model.

- After each run, write out lessons learned (patterns, mistakes, useful commands and repos) into a persistent `investigations/` directory, keyed by topic. A distilled experience log, not a vector store.
- Let future investigations query it before starting from scratch.
- Judge the system by the quality of the investigation, not just whether the final patch passes tests. A passing patch reached through a wasteful, poorly justified path is a lucky pass, not a real success.
