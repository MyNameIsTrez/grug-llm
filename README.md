# grug-llm

grug-llm is an overnight investigation engine for small local LLMs. Instead of expecting a weak model to reason its way to the answer, it repeatedly gathers evidence in a disposable sandbox until the answer is justified by experiments, not intuition.

The point is access, not speed. Running an agent that autonomously executes terminal commands for hours currently costs a capable GPU or a hosted API budget. grug-llm is for people who want to use hardware they already own: a laptop's crappy GPU, or a phone charging on the nightstand. The tradeoff is waiting until morning for answers to well-scoped, complicated questions.

## How it works

```
complicated problem
  -> generate uncertainty
  -> split into smaller questions
  -> test the cheapest/highest value hypothesis
  -> update state
  -> repeat
  -> simple final decision
```

Most agent projects optimize for solving a task in as few LLM calls as possible. grug-llm optimizes for the opposite: maximizing evidence gathered per dollar, using compute that would otherwise sit idle. Intelligence is expensive, persistence is cheap.

The difficulty in most investigations is epistemic (which file, which source, which cause), not mechanical, so the final step is usually trivial once you get there: a one line diff, a well-placed paragraph. The stronger mental model is a theorem prover, not a chatbot: every claim needs an explicit justification, and confidence comes only from what's been verified.

The real risk isn't the model forgetting, it's the model choosing bad experiments. Given fifty candidate commands, a good engineer picks one; a small model might pick six mediocre ones, and doing that for hours quietly burns the overnight budget. Improving experiment selection matters more than improving reasoning, which is why even the MVP ranks candidate commands instead of running whatever the model proposes first.

Run as a batch job, not a conversation:

```bash
$ grug-llm investigate \
    --model qwen14b \
    --prompt "https://github.com/openresty/luajit2 - optimize unpack() by getting it out of LuaJIT's NYI (not-yet-implemented) list" \
    --max-runtime 8h
Started investigation 42

$ grug-llm status 42
$ grug-llm report 42
```

## Design

- Python owns control flow, not the LLM: timeouts, iteration limits, and loop detection stay predictable.
- Every investigation runs in a fresh, disposable sandbox with nothing but `/workspace`.
- State lives in small files (`question.md`, `hypotheses.md`, `evidence.md`, `todo.md`), so the model never has to remember iteration 40.
- Observations are immutable, beliefs are not: raw tool output never gets rewritten, interpretation does.
- Confidence is derived, not reported: the loop tracks verified/contradicted/pending claim counts and stops once every hypothesis has a status.
- Experiment selection is a first-class constraint from day one: candidate commands are cheaply ranked by expected information gain, and commands whose output already sits in `evidence.md` are refused before running.
- `search` and `fetch` are the only bespoke tools (a search API, and HTML-to-text extraction). Everything else — git clone, grep, `python -c`, curl — goes through one real, unrestricted shell. Seeing stdout/stderr/exit code after every command corrects a bad command faster than a whitelist prevents one.

## Plan of action

0. **MVP.** Python CLI, one disposable container, one markdown journal, `search`/`fetch`/`run`, a basic ranking-plus-dedup pass before execution, loop until no unverified hypotheses remain.
1. **Structured state.** Split the journal into `hypotheses.md`, `evidence.md`, `todo.md`, an append-only `observations/`, and track claim counts instead of vibes.
2. **Verification.** A fresh model instance periodically reviews for unsupported claims, then tries once more to prove the final answer wrong before it ships.
3. **Efficiency.** Retrieve only relevant files per decision, compress old observations, sharpen the ranking heuristic, require every tool call to state its goal and expected belief update.
4. **Scheduling.** Job-style CLI (`investigate`, `status`, `report`), periodic checkpoints, graceful pausing, a full audit trail.
5. **Learning across runs.** Distill procedural knowledge per topic ("LuaJIT NYI investigations usually need X, Y, Z"), not raw transcripts; judge runs by evidence gathered per hour, not just whether the patch passed.
6. **Mobile.** Termux gives an unrestricted shell without root; smaller models (3B-8B) need the loop even more than a 14B does. Open problem: thermal duty cycling (burst inference, sleep, gate on temperature/charge) to survive an 8 hour run.
