# grug-llm

grug-llm lets a small local LLM solve hard problems by trading latency for reliability: a 14B model on a spare GPU, or eventually a 3B to 8B model on the phone already charging on your nightstand overnight. Instead of asking the model to reason harder in one shot, it runs the model in a long, tool using loop inside a disposable sandbox, until it has gathered enough evidence to actually prove its answer. An hour, or overnight, is fine.

The philosophy is "stupid but diligent." A small model rarely has a flash of insight. It can be extremely persistent, never forget anything, and run thousands of cheap experiments before giving up. Persistence is undervalued, and the smaller the model, the more of it you need.

The deeper motivation is access, not just cost. Running an agent that autonomously executes terminal commands for hours currently means either a capable GPU or a hosted API budget. grug-llm is for people who would rather not depend on either: who don't mind sending nothing to a third party server, don't mind their own electricity being far less efficient than a datacenter's, don't mind waiting until morning, and aren't expecting the model to design an architecture from scratch, just to grind patiently through one well-scoped question.

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

The final step is usually trivial once you get there, a one line diff in a codebase, a single well-placed paragraph in an essay, because the hard part was never deducing the answer. It was building an environment where the answer becomes obvious. In most investigations, whether debugging a benchmark regression or writing a well-supported essay, the difficulty is epistemic (which file, which source, which cause), not mechanical. grug-llm spends compute reducing that uncertainty instead of on longer reasoning traces.

Two framings make this easy to reason about. It resembles the scientific method more than deduction: don't think your way to the truth, design the experiment that eliminates the most possibilities. And it resembles a theorem prover more than a chatbot: accumulate empirical proof (reproductions, tests, benchmarks) until every claim is justified, not merely asserted.

The output should read like a lab report, not a chat reply: a conclusion, a confidence level, a list of checks that passed, and an explicit list of what's still uncertain.

It's meant to be run as a batch job, not a conversation. `investigate` starts the run in the background and immediately hands back a job ID, the same way submitting a job to a cluster scheduler does:

```bash
$ grug-llm investigate \
    --model qwen14b \
    --question "Why does https://github.com/ROCm/rocm-libraries its device_adjacent_find benchmark crash?" \
    --max-runtime 8h
Started investigation 42

$ grug-llm status 42
$ grug-llm report 42
```

## Design principles

- **Python owns control flow, the LLM doesn't.** This keeps the system predictable and makes timeouts, iteration limits, and loop detection easy to add.
- **Every investigation gets a fresh, disposable container**, only `/workspace` mounted. Nothing persists except what's on disk.
- **State lives in small files, not one journal**: `question.md`, `hypotheses.md`, `evidence.md`, `todo.md`. The model never has to remember iteration 40, and only retrieves what's relevant to its current decision.
- **Observations are immutable, beliefs are not.** Raw tool output is never rewritten; interpretation lives separately and can change.
- **Confidence is numeric, not vibes.** Every hypothesis carries a confidence score. The loop stops only once confidence clears a threshold and no required verifications remain.
- **`search` and `fetch` are the only bespoke tools, and even those are thin.** `search` wraps a real search API, since there's no shell equivalent for "give me ranked web results"; scraping a search engine's result page with curl breaks the moment its markup changes or it blocks non-browser requests. `fetch` just extracts the readable text from a page before it hits the model's context, since raw HTML is mostly markup and context is the scarcest resource here. It's a convenience default, not a wall around curl.
- **Everything else is a real, unrestricted shell.** Git clone, grep, find, cat, `python -c`, curl, whatever, all go through one `run(command)`. The model already sees stdout, stderr, and exit code after every command, and that feedback loop corrects a bad command faster than a whitelist prevents one. Blacklisting something like `python -c` would remove one of the cheapest ways to calculate an exact number that might otherwise be hallucinated instead.
- **The system is model agnostic.** State, tools, verification, and scheduling are the product. The LLM underneath is swappable as better local models appear.

## Related work

None of the individual pieces here are new. Worth being upfront about that.

- **The loop itself**: this is close to the "Ralph Wiggum" technique, an unattended loop that re-invokes a coding agent against a todo file until a completion signal appears, with state kept in the filesystem and git history instead of conversation memory. People already run these overnight and for days unattended. It gets persistence right but has no concept of a hypothesis, a confidence score, or an adversarial review pass. It's diligence without epistemics.
- **Execution-grounded hypothesis testing**: AgentForge (Planner/Coder/Tester/Debugger/Critic roles sharing a mandatory Docker sandbox, every patch required to survive sandboxed execution) and DebugHarness (an explicit hypothesis-testing loop driven by a debugger rather than a markdown file) are structurally close to Phase 1 and Phase 2 here. Agentless-style systems also describe verifying hypotheses inside a sandbox as their core loop.
- **Hypothesis, experiment, verification loops in science**: NovelSeek, AI co-scientist, Agent Laboratory, and the open-source scholar-loop project all run something close to this same shape (literature review, grounded hypothesis, real experiment, scoring against ground truth, self-critique, write-up) aimed at research papers instead of code.
- **General purpose agent frameworks (LangChain, CrewAI, OpenHands, SWE-agent, Aider)**: all assume a frontier or near-frontier model behind the API and optimize for solving a task in the fewest steps, with the model's own reasoning doing most of the work. Aider in particular assumes a human drives and the LLM assists, rather than investigating unattended for hours.
- **Desktop local agents (Open Interpreter, Cline)**: these run locally too, but interactively, on a laptop or desktop GPU, targeting mid-sized models (roughly 14B to 32B) and expecting a person to guide the model when it gets stuck. grug-llm is meant to run headless, unattended, and eventually on hardware much smaller than a desktop GPU.
- **Small models in industry (Phi, Gemma, Qwen, and similar)**: these are already tuned to run on phones, but for the opposite reason grug-llm wants them there. Industry uses them for low-latency, real-time tasks (autocomplete, summarization, quick queries) where speed matters and the task is shallow. grug-llm uses a small model for the opposite kind of task: slow, long-horizon, brute-force inquiry, where speed doesn't matter and depth does.

So the mechanism, sandboxed execution as ground truth, a hypothesis ledger, an overnight loop with a hard stop condition, exists in pieces across all of these. What's less crowded is the specific framing: nearly all of the above either assume a capable model or use a small model for something quick and shallow. grug-llm's premise is the opposite of both: the model is deliberately weak, the task is deliberately slow, and the loop's entire job is to turn hardware you already own, whether that's a spare GPU or a phone on the nightstand, into the reliability an API call would otherwise cost money for. That's a difference in target hardware and motivation, not in mechanism, and it's worth being honest that this is closer to "known techniques, aimed at an underserved case" than to something nobody has tried.

## Plan of action

### Phase 0: MVP with Python, Docker and an LLM

Prove the loop works, nothing more.

- Python CLI, takes a question. If the question contains a repo URL, a paper, a book, whatever, the model fetches or clones it itself as one of its first actions.
- One disposable Ubuntu container with internet access.
- One markdown journal the model reads and appends to each iteration.
- Minimal tools: `search(query)`, `fetch(url)`, and `run(command)` against a real, unrestricted shell.
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
- Run it once more at the end, in a fresh container with whatever the investigation needed re-fetched from scratch (a repo re-cloned, a page re-fetched, whatever applies), given only the question and the final answer, told to prove it wrong. Success reopens the investigation.
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

### Phase 7: Mobile and edge deployment

Move the same design onto a phone that's already plugged in overnight, since that's the largest source of idle compute most people own.

- Termux on Android already gives a real, unrestricted shell without root, so the `run(command)` design carries over unchanged; a Python process there can drive a local model (Ollama, MLC-LLM, or similar) exactly like it drives one on a desktop.
- Smaller models (3B to 8B) are dumber and need more iterations, more hypotheses, and more external verification than the desktop baseline. The loop isn't a nice-to-have here, it's the only thing that makes a model this size viable at all.
- Sustained heavy inference heats up a phone that's charging, and a battery held above roughly 40°C for hours degrades faster. This needs a duty cycle: burst an inference step, then sleep the process for tens of seconds, gated on device temperature and only running while the battery is above some threshold and not fast-charging. A throttled pace of roughly one action per minute still gives an 8 hour night several hundred iterations, plenty to test dozens of hypotheses without ever pushing the thermal limit.
- The existing workflow already matches the hardware: start `investigate`, lock the screen, wake up to a finished report.
