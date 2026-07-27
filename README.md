# grug-llm

A command line tool that lets a small local LLM (for example Qwen 14B running on a consumer GPU) punch above its weight by trading latency for reliability. Instead of asking the model to reason harder in one shot, it runs the model in a long, patient, tool using loop inside a disposable Docker sandbox, until it has gathered enough evidence to actually prove its answer.

The working philosophy is "stupid but diligent." A 14B model will rarely have a flash of insight the way a frontier model can, but it can be extremely persistent, never forget anything, run thousands of cheap experiments overnight, and refuse to stop until every claim is backed by evidence. Persistence is undervalued.

## Core idea

Don't optimize for chain of thought. Optimize for an external research loop.

A frontier model often does:

```
complex problem -> large internal reasoning jump -> answer
```

grug-llm instead does:

```
complex problem
  -> generate uncertainty
  -> split into smaller questions
  -> test the cheapest/highest value hypothesis
  -> update state
  -> repeat
  -> simple final decision
```

The final step becomes trivial (often a one line diff) because the hard part was never "deducing the answer." The hard part was constructing an environment where the answer becomes obvious. The difficulty in most software engineering problems is epistemic, not mechanical: knowing the right file, the right variable, the right causal mechanism. grug-llm spends compute on reducing that uncertainty rather than on longer reasoning traces.

This connects to two older ideas worth naming. It resembles the scientific method more than classic reasoning: the agent doesn't try to think its way to the truth, it designs the experiment that eliminates the most possibilities. And it resembles a theorem prover more than a chatbot: instead of mathematical proof it accumulates empirical proof (reproductions, tests, benchmarks) until every claim is justified rather than merely asserted.

The resulting artifact should look less like a chat response and more like a scientific report: a conclusion, a confidence level, a list of checks that passed, and an explicit list of remaining uncertainty.

## Who this is for

Anyone who would rather trade time for reliability: developers who don't want to send proprietary code to a hosted API, hobbyists who don't want to pay for frontier API usage, researchers who care more about an inspectable evidence trail than about speed, or anyone happy to leave a laptop running overnight on a question like "audit this codebase" or "why does this benchmark regress on ARM."

grug-llm is meant to be run like a batch job, not a chat:

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

- **The Python controller owns control flow, not the LLM.** This keeps the system predictable and makes it easy to add timeouts, max iteration counts, and loop detection.
- **Every investigation gets a fresh, disposable container** (`docker run --rm`) with only a `/workspace` directory mounted. Nothing is assumed to persist except what's written to disk.
- **State lives in small, separate files**, not one giant journal: `question.md`, `hypotheses.md`, `evidence.md`, `todo.md`, `confidence.json`. This means the model never has to remember what happened forty iterations ago, and it can retrieve only the handful of files relevant to its next decision.
- **Observations are immutable, beliefs are not.** Raw command output and search results are never rewritten. Interpretations of that evidence live in a separate, mutable beliefs file.
- **Confidence is numeric, not vibes.** The model tracks a confidence score per hypothesis and only stops when confidence exceeds a threshold and no required verifications remain.
- **Tools are a small, fixed set at first**: search, fetch, git clone, grep, find, read, write, run, and a few test runners. A general shell is added later, once the smaller action space has proven the approach works. Constraining the action space tends to produce better planning from a smaller model.
- **The system is model agnostic.** The valuable part is persistent state, tools, verification, and scheduling. The LLM itself is a pluggable reasoning engine that can be swapped as better local models appear.

## Related work

The closest existing projects are OpenHands (formerly OpenDevin), SWE-agent, and Aider. All three are excellent, but they are optimized for autonomy or benchmark score ("solve this issue" as fast as possible), not for patient, evidence heavy verification. Aider in particular assumes a human is driving and the LLM assists, rather than the LLM investigating unattended for hours. grug-llm's niche is different: a modest local model that isn't in a hurry, and that isn't trusted until it has proven itself wrong a few times and failed to do so.

## Plan of action

### Phase 0: MVP with Python, Docker and an LLM

Get the smallest possible version of the loop working end to end.

- A Python CLI that takes a question and an optional repo URL.
- Launches a single disposable Ubuntu Docker container with internet access and the repo cloned in.
- A single markdown journal file the LLM reads and appends to each iteration.
- A minimal tool set exposed to the model: `search`, `fetch`, `git_clone`, `grep`, `find`, `read`, `write`, `run`.
- A basic loop: read journal, ask the model for the single highest priority unanswered question and one action, execute that action in the container, append the result to the journal, repeat.
- A simple stop condition: ask "are there any unverified hypotheses left," stop when the answer is no or a max iteration count is hit.
- Output: whatever final answer the model writes to the journal.

Goal of this phase is only to prove the basic "loop plus disposable sandbox" mechanism works, not to make it good.

### Phase 1: Structured, persistent state

Replace the single journal with the multi file workspace and make state easier for the model to reason about.

- Split state into `question.md`, `hypotheses.md`, `evidence.md`, `todo.md`, `scratch.md`, `summary.md`.
- Give every hypothesis a status: untested, supported, contradicted, or unknown, plus a list of evidence needed to resolve it.
- Make observations immutable: raw command/tool output goes into an append only `observations/` directory, while interpretation lives in a separate, editable beliefs file.
- Replace "what should you do next" with an explicit uncertainty and confidence prompt: current confidence in the answer, the largest remaining uncertainty, and the next action.
- Cache proven facts so the model doesn't re-derive things it has already verified.

### Phase 2: Verification and review

Add the machinery that turns a plausible answer into a well supported one. One reviewing mechanism, used at two points in the investigation, covers what would otherwise be three overlapping ideas (a skeptical reviewer, a separate "judge," and an adversarial re-check).

- The review prompt is always the same job on a fresh instance of the model: find factual errors, unsupported claims, missing experiments, or contradicting code paths, and only produce objections, never rewrite the answer itself.
- Run it periodically during the investigation, against the current state, to catch stalling or repeated work early.
- Run it once more at the end, against a fresh container with the repo re-cloned and only the question and final answer visible, with explicit instructions to prove the answer wrong. If it succeeds, reopen the investigation with the objection as a new hypothesis.
- Feed all objections back into the todo list and continue the loop until none remain.
- Replace the simple "DONE" signal with a structured stop condition, for example a JSON object with `confidence`, `remaining_uncertainties`, and `required_verifications`, and only allow termination when confidence is above a threshold and required verifications is empty.
- Final output becomes a structured report: conclusion, confidence, a checklist of what was verified (reproduced, fixed, regression tests pass, benchmark unchanged, docs match behavior), and explicit remaining uncertainty.

### Phase 3: Efficiency and focus

Make the loop cheaper and less prone to wasting an 8 hour budget on nothing.

- Build a retrieval layer over the workspace instead of stuffing the whole journal into every prompt. Ask the model to retrieve only the handful of most relevant pieces of evidence for its current decision.
- Add periodic compression: every N iterations, collapse old observations into an executive summary with references, to keep working context small.
- Add loop detection: compare consecutive actions, and if the controller sees repeated or near identical actions, interrupt and force the model to pick a fundamentally different avenue.
- Require every tool call to state its goal, expected observation, and how each possible result would change current beliefs. This is a cheaper and more honest substitute for asking the model to estimate its own information gain: a small model can state intent reliably, it cannot reliably self-score a probability of success.
- Have the controller (not the model) prefer cheap, read-only actions over expensive ones when both are queued, using simple static rules rather than model-estimated costs.

### Phase 4: Specialized roles

Split the single agent prompt into a small pipeline that shares the same underlying model: planner, researcher, programmer, tester, editor. Small models tend to do much better with a narrow task than with one prompt doing everything, and unlike running several full investigations side by side, this doesn't multiply GPU time, since only one role is active at once.

Running multiple independent investigators from scratch and comparing their conclusions is a legitimate way to raise confidence further, but it costs a multiple of total GPU time (three investigators means roughly three times the wall clock, on a single consumer GPU). Worth revisiting once the single-investigator loop above is solid and the added confidence is worth the added wait.

### Phase 5: Scheduling, checkpointing and the job model

Turn the tool from a script into something people actually leave running overnight.

- CLI subcommands that treat investigations like build jobs: `investigate`, `status`, `report`, rather than a live chat interface.
- Checkpointing: periodically write out elapsed time, number of actions performed, repos cloned, tests executed, hypotheses confirmed/disproven, and an estimated time to finish.
- Graceful pausing: if the investigation genuinely cannot proceed (needs network access to an issue tracker, a newer compiler, or human input), it should say so explicitly rather than silently looping.
- Full audit trail as a first class output: question, hypotheses, evidence collected, commands executed, failed approaches, remaining uncertainty, final answer, so a human can inspect a failed run and find the mistake quickly instead of starting over.

### Phase 6: Learning across investigations

Make the system improve over time without needing a bigger model.

- After each investigation, write out lessons learned: patterns, mistakes, useful commands, useful repositories, into a persistent `investigations/` directory keyed by topic. This is a distilled experience database, not a vector store.
- Let future investigations query this experience database before starting from scratch.
- Evaluate the system by the quality of the investigation itself (was evidence gathered efficiently, were hypotheses actually tested), not only by whether the final patch happens to pass tests, since a passing patch reached through a wasteful or poorly justified trajectory is a lucky pass rather than a real success.
