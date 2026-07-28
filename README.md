# grug-llm

grug-llm is an overnight investigation engine for small local LLMs. Instead of expecting a weak model to reason its way to the answer, it repeatedly gathers evidence in a disposable sandbox until the answer is justified by experiments rather than intuition.

The point is access, not speed. Running an agent that autonomously executes terminal commands for hours currently costs either a capable GPU or a hosted API budget. grug-llm is for people who'd rather spend hardware they already own than money: the target is a phone already charging on the nightstand. The tradeoff is waiting until morning for an answer to something well-scoped.

## How it works

```
complex problem
  -> generate uncertainty
  -> split into smaller questions
  -> test the cheapest/highest value hypothesis
  -> update state
  -> repeat
  -> simple final decision
```

Most agent projects optimize for solving a task in as few LLM calls as possible. grug-llm optimizes for the opposite: maximizing evidence gathered per dollar and per watt. Intelligence is expensive, persistence is cheap, and the whole system bets that cheap persistence can substitute for it.

The difficulty in most investigations is epistemic (which file, which source, which cause), not mechanical, so the final step is usually trivial once you get there: a one line diff, a well-placed paragraph. The stronger mental model is a theorem prover, not a chatbot: every claim needs an explicit justification, and confidence comes only from what's been verified, never from what's merely asserted.

The real risk isn't the model forgetting, it's the model choosing bad experiments. Given fifty possible next commands, a good engineer picks one; a small model might pick six mediocre ones, and doing that for hours quietly burns the overnight budget. Improving experiment selection matters more than improving reasoning.

Run as a batch job, not a conversation:

```bash
$ grug-llm investigate \
    --model qwen14b \
    --question "Why does https://github.com/ROCm/rocm-libraries its device_adjacent_find benchmark crash?" \
    --max-runtime 8h
Started investigation 42

$ grug-llm status 42
$ grug-llm report 42
```

## Design

- Python owns control flow, not the LLM: timeouts, iteration limits, and loop detection stay predictable.
- Every investigation runs in a fresh, disposable sandbox with nothing but `/workspace`.
- State lives in small files, not one journal (`question.md`, `hypotheses.md`, `evidence.md`, `todo.md`), so the model never has to remember iteration 40.
- Observations are immutable, beliefs are not: raw tool output never gets rewritten, interpretation does.
- Confidence is derived, not reported: small models can't produce a calibrated `confidence = 0.83`, so the loop tracks verified/contradicted/pending claim counts instead and stops once every hypothesis has a status.
- `search` and `fetch` are the only bespoke tools (a real search API, and HTML-to-text extraction). Everything else, git clone, grep, `python -c`, curl, whatever, goes through one real, unrestricted shell. The model already sees stdout, stderr, and exit code after every command, which corrects a bad command faster than a whitelist prevents one.

## Plan of action

0. **MVP.** Python CLI, one disposable container, one markdown journal, `search`/`fetch`/`run` as tools, loop until no unverified hypotheses remain.
1. **Structured state.** Split the journal into `hypotheses.md`, `evidence.md`, `todo.md`, an append-only `observations/` directory, and track claim counts instead of vibes.
2. **Verification.** A fresh model instance periodically reviews for unsupported claims, and once more at the end tries to prove the final answer wrong before it ships.
3. **Efficiency and experiment selection.** Retrieve only relevant files per decision, compress old observations, detect repeated actions, and require every tool call to state its goal and expected belief update.
4. **Scheduling.** Job-style CLI (`investigate`, `status`, `report`), periodic checkpoints, graceful pausing, a full audit trail.
5. **Learning across runs.** Distill procedural knowledge per topic ("ROCm investigations usually need X, Y, Z"), not raw transcripts, and judge runs by evidence gathered per hour, not just whether the patch passed.
6. **Mobile.** Termux already gives an unrestricted shell without root. Smaller models (3B-8B) need the loop even more than a 14B does. The open problem is thermal: a duty cycle (burst inference, sleep tens of seconds, gate on temperature and charge state) keeps the battery from cooking during an 8 hour run.