# AutoKaggle — System Design

*Status: draft for review · 2026-11-20 · working name "AutoKaggle"*

## 1. Intent

AutoKaggle is an open-source agent harness. You point it at a Kaggle computer-vision competition and it does three things:

1. It produces a **competitive solution**, and ideally a winning one.
2. It produces **research worth reading**: what was tried, what worked, what didn't, the ablations behind those calls, and the papers and prior solutions that drove each idea.
3. It **improves its own strategy** as it works, in the spirit of Harness² (recursive harness self-improvement), while every self-edit is checked against real scores.

Inspirations:
- Karpathy's AutoResearch: tight edit, train, keep-if-better loops.
- AIDE: tree search over candidate solutions.
- AutoResearchClaw: staged pipeline with dynamic human-in-the-loop (HITL).
- AutoKernel and OpenResearch.
- Google's Harness²: recursive improvement of the harness, not the model.

The ideas behind these come from math and general research. AutoKaggle applies them to ML/CV competitions. Kaggle is the starting format because it is standardized: fixed data, a fixed metric, a leaderboard, and a large archive of past competitions and winning write-ups. Other open challenges (e.g. WRIVA CVGL) are expected to plug in later through the same adapter interface.

### Priority order
1. Find the most performant solution.
2. Generate insightful research.
3. Be a tool other people find useful.

When these conflict, the higher one wins.

### Core belief
Kaggle vision competitions are won by:
- trustworthy validation;
- many cheap iterations;
- well-informed ideas, many taken from past winners;
- ensembling and test-time augmentation (TTA);
- the discipline not to chase the public leaderboard.

The harness turns each of these habits into machinery. Agents provide the ideas and the code. The harness provides the discipline.

## 2. Goals, non-goals, anti-goals

**Goals**
- End-to-end autonomy from competition slug to final submission, with optional human gates.
- Adapt to any compute environment: laptop GPU, a single workstation GPU, multi-GPU, or cloud. Behaviour should degrade gracefully on smaller hardware.
- Provider-agnostic LLM access. Claude comes first, then any model via OpenRouter.
- Every score, claim, and decision is reproducible and traceable to an experiment record.
- Measurable self-improvement, within a competition first and across competitions later.

**Non-goals**
- Training or fine-tuning the LLMs themselves. Model weights stay frozen and only the harness changes.
- Non-vision Kaggle tracks (tabular, NLP, RL) at first. The architecture should not rule them out.
- A hosted SaaS product. AutoKaggle is a tool you run.

**Anti-goals.** The project has failed if any of these happen, even if the code works.
- **Leaderboard overfitting:** climbing the public leaderboard while the private score collapses.
- **Reward hacking:** an agent "improves" the score by touching labels, the metric, the validation split, or the test set.
- **Rule violations:** multiple accounts, private sharing, disallowed external data or weights, or leaderboard probing.
- **Opaque wins:** a high score that nobody, including the harness, can explain or reproduce.
- **Polluted memory:** lessons from one competition silently misleading the next one.
- **Runaway cost:** token or GPU spend beyond the budget the user set.

## 3. Constraints

- **Model access is text-only.** Models return text. Every tool is implemented and executed by the harness. No provider-hosted tools, no provider agent runtimes, and no reliance on Claude Code or the Claude Agent SDK as the harness. *(User decision.)*
- **Orchestration uses LangChain/LangGraph.** *(User decision.)*
- **The first provider is Anthropic.** OpenRouter follows as the second provider path. *(User decision.)*
- **The reference dev machine** is Windows 11 with one RTX 5070 Ti (16 GB). The primary supported runtime is Linux, with WSL2 on Windows. Native Windows is best effort. *(My call: Docker, NCCL, and most CV tooling are Linux-first.)*
- **Kaggle-side limits:**
  - Competition rules must be accepted by a human in the browser before the API can download data.
  - There are daily submission caps.
  - Code competitions require inference to run in a Kaggle notebook offline, under time and hardware limits.
  - External data and pretrained-weight rules vary by competition.
- **Disk.** Vision datasets range from tens to hundreds of GB, and benchmark suites multiply that.

## 4. Approaches considered

| | Approach | Trade-off | Verdict |
|---|---|---|---|
| A | **Claude Code as the harness.** The loop is a `program.md` plus skills, subagents, and hooks. | Almost nothing to build, but weak control over long runs, scheduling, and budgets. Hard to reproduce, and tied to one vendor's runtime. | Rejected. It conflicts with the text-only and provider-agnostic constraints. |
| B | **Deterministic orchestrator with agents as workers.** Code owns stages, scoring, scheduling, and budgets. Agents do the creative work. | More upfront building, but reproducible, benchmarkable, and resumable, and the provider is swappable. | **Chosen**, with LangGraph as the orchestrator and Harness²-style self-improvement built in. |
| C | **A single lead-researcher agent** that decides everything and spawns subagents freely. | Most flexible, but nondeterministic, prone to budget blowups, and hard to evaluate. You can't tell harness failure from model variance. | Rejected as the core. Its flexibility comes back through the Strategist and the self-improvement loop, both bounded by B. |

## 5. Design principles (invariants)

1. **The harness computes every score.** Agents never report metrics. Evaluation code, labels, and fold assignments are read-only to agents and live outside their writable workspace.
2. **Local CV is the truth.** The public leaderboard is a sanity signal, tracked as a CV/leaderboard correlation. It is never something to optimize.
3. **Everything is an experiment record.** If it isn't in the ledger, it didn't happen.
4. **Small diffs over rewrites.** Solutions follow a standard skeleton, and agents change it through reviewable diffs.
5. **Cheap first, expensive last.** Ideas climb a fidelity ladder. Full runs are reserved for ideas that have earned them.
6. **Self-edits must pay rent.** A change to the harness is adopted only on measured improvement, never on an agent's say-so.
7. **Interfaces are designed for the full system, even where an early phase implements only part of them.**

## 6. System overview

```
                ┌──────────────── Human (dynamic gates: TUI / web / notifications) ──────┐
                ▼                                                                         │
 Challenge ──► Stage machine (LangGraph, checkpointed, resumable)                         │
 Adapter        Understand → Validate → Research → Baseline → Explore ⇄ Exploit           │
 (Kaggle,       → Ensemble → Submit → Write-up       (+ Research re-entry, PIVOT)         │
  MLE-bench,         │ dispatches jobs to                                                 │
  later: others)     ▼                                                                    │
   Agent layer (text only): Strategist · Researcher · Coder · Debugger · Analyst ·        │
                            Reviewer · Writer · Improver                                  │
                     │ emit structured tool requests; harness validates and executes      │
                     ▼                                                                    │
   Tool layer: workspace/diff · run-job · query-ledger · search-literature ·              │
               view-images · inspect-predictions · kaggle-cli                             │
                     │                                                                    │
   ┌─────────────────┼────────────────────┬─────────────────────┬──────────────────┐      │
   ▼                 ▼                    ▼                     ▼                  ▼      │
 Compute layer   Experiment ledger    Research library    Harness store       Budget &    │
 (profiler,      (runs, commits,      (idea cards with    (versioned prompts, observability┘
  fidelity        fold scores, OOF    evidence, sources)   skills, policies;
  ladder, queue,  preds, costs)                            edited only by the
  runners)                                                 self-improvement loop)
```

## 7. Components

### 7.1 Challenge adapters

The adapter is the only layer that knows where a challenge comes from. It produces a **competition card** and hands over the data.

**Competition card** fields:
- task type (classification, regression, detection, segmentation, retrieval, …);
- metric definition and a local metric implementation;
- submission format and validator;
- data layout and size;
- rules (external data, pretrained weights, code-competition runtime/hardware/internet limits);
- submission cap and deadline.

**Adapters:**
- **Kaggle adapter:**
  - Uses the official Kaggle API/CLI for download, submission, and leaderboard reads.
  - Detects whether rules have been accepted. If not, it halts with a clear instruction for the human.
  - For code competitions, it packages inference as a Kaggle notebook plus a weights dataset (phase 3).
- **MLE-bench adapter:** offline competitions with local grading and medal thresholds. It is used to evaluate the harness itself (§9), never as a live target.
- **Future adapters** (WRIVA CVGL, DrivenData, CVPR workshop challenges) implement the same interface.

**Metric verification.** The local metric must reproduce Kaggle's numbers on a known case before modelling starts. That known case can be a sample submission, a documented example, or a first real submission.

### 7.2 Stage machine

Built as a LangGraph graph with a persistent checkpointer, so any run can be killed and resumed. Each stage has entry criteria, exit criteria, a default gate policy, and the agents it may use.

| Stage | Purpose | Exit criteria |
|---|---|---|
| **Understand** | Build the competition card. Run EDA: class balance, image sizes and modalities, metadata, sample images (agents look at them), and near-duplicates found with perceptual hashing. | Card and EDA report written. |
| **Validate** | Design the CV scheme (group/stratified k-fold), check for leakage (duplicates or shared groups across folds, train/test overlap), verify the metric, and test fold stability with a cheap model. | Frozen fold assignment, a verified metric, and an acceptable fold-score variance. **Gated by default.** |
| **Research** | Produce idea cards from the literature and past competitions (§7.8). The stage can be re-entered at any time. | An initial idea backlog, ranked. |
| **Baseline** | Build a strong standard pipeline end to end: a pretrained backbone, sensible augmentation, a validated submission file, and a first submission to get the first CV/leaderboard data point. | Baseline in the ledger, plus the first leaderboard point. |
| **Explore ⇄ Exploit** | The main loop (§7.7). | Budget exhausted, a plateau detected, or a human stops it. |
| **Ensemble** | Blend on OOF predictions (hill climbing or stacking), add TTA, and check runtime limits for code competitions. | Ensemble CV beats the best single model or is explicitly rejected. |
| **Submit** | Spend remaining submissions according to policy and select the two finals: one best-CV and one robust (diverse, low fold variance). | Finals selected. |
| **Write-up** | Produce the research report and a Kaggle-style solution write-up (§7.12). | Report generated. |

**Transitions.**
- The Strategist can request **PIVOT** (back to Research or Explore with a new direction) or **REFINE**.
- The stage machine decides whether the request is allowed, based on budget and gate policy.
- Every transition is logged with its reason.

### 7.3 Agent layer

Every agent is a role: a prompt, a set of skills, and an allowed tool set, all loaded from the harness store (§7.11), plus a model assignment.

| Role | Job | Typical tools |
|---|---|---|
| **Strategist** | Chooses what to try next, proposes pivots, and reads the ledger and backlog | query-ledger, read idea cards |
| **Researcher** | Searches sources and writes idea cards with evidence | search-literature, fetch-source |
| **Coder** | Implements ideas as small diffs against the solution skeleton | workspace/diff, run-job (smoke rung only) |
| **Debugger** | Fixes failed runs from logs and tracebacks | workspace/diff, read logs, run-job (smoke) |
| **Analyst** | Error analysis: views worst-predicted images, confusion structure, per-slice scores | view-images, inspect-predictions, query-ledger |
| **Reviewer** | Pre-flight check before expensive runs: leakage, bugs, rule compliance, licenses of external data and weights | read diff, read rules |
| **Writer** | Writes the research report and solution write-up from the ledger | query-ledger, read idea cards |
| **Improver** | Proposes edits to the harness itself (§7.11) | read traces, read harness store |

**Model gateway.**
- One interface turns a prompt and a schema into text.
- Config maps each role to a model, e.g. a strong model for the Strategist, Coder, and Improver and a cheap one for paper triage.
- Providers:
  - v1: Anthropic via LangChain's Anthropic integration.
  - Phase 2: OpenRouter via LangChain's OpenRouter integration.
- Multimodal input (images) is supported where the model allows it. The Analyst needs it.

**Text-only tool protocol.** *(My reading of "models only provide text.")*
- Each agent turn returns a structured JSON object: a short rationale plus either one or more tool requests (name and arguments) or a final answer.
- The harness validates the object against the role's tool schema, executes it, and returns the observation as text.
- Nothing executes on the provider side.
- Default transport is prompt-and-parse with schema validation and retry-on-invalid, because not every OpenRouter model supports native function calling. The builder may use native structured-output or function-calling APIs as a *transport* where available, as long as execution stays harness-side.

### 7.4 Tool layer

All tools are harness code with explicit permissions per role.
- **workspace/diff:** apply, revert, and inspect diffs in the solution repo. Writes outside the solution workspace are refused.
- **run-job:** submit a job at a given fidelity rung. Returns a job id, never a score directly.
- **query-ledger:** structured queries over experiments.
- **search-literature / fetch-source:** see §7.8.
- **view-images / inspect-predictions:** render samples, worst-k predictions, and per-class grids as images for multimodal models.
- **kaggle-cli:** leaderboard reads and submissions. Only the Submit stage holds this permission.

### 7.5 Workspace and solution skeleton

- Each competition run gets a git repo with a standard skeleton: data module, model, train loop, inference/TTA, and config. Agents work by diff, and every ledger run points at a commit.
- Separate skeletons exist per task family. v1 covers classification/regression. Detection and segmentation come in phase 3.
- Labels, fold assignments, and the metric live in a **read-only area** that the training code consumes but agents cannot edit. This enforces principle 1.

### 7.6 Experiment ledger

The ledger is the single source of truth for the scheduler, the ensembler, the self-improvement loop, and the write-up.

| Entity | Key fields |
|---|---|
| Experiment | id, parent id (tree), idea card id, commit, config, seed, fidelity rung, status, failure reason |
| Result | per-fold scores, mean/std, OOF prediction file, test prediction file, train curves |
| Cost | GPU-seconds, wall time, peak VRAM, LLM tokens and dollars per role |
| Submission | experiment(s), public LB score, timestamp, CV at the time |
| Decision | stage transitions, gate outcomes, pivots, harness edits, each with a rationale |

Storage: a local database plus an artifact directory. The choice of database and layout is left to the builder.

### 7.7 Compute layer and search

**Hardware profiler.** On startup and on change, it measures GPU count, memory, and throughput, and caches a fit table (backbone × image size × batch). The table answers "what fits and how long will it take."

**Fidelity ladder.**

| Rung | Definition | Purpose |
|---|---|---|
| R0 smoke | a few batches, tiny subset | catch crashes, shape errors, NaNs |
| R1 proxy | data subset, reduced resolution, 1 fold, short schedule; cached backbone features where possible | rank many ideas cheaply |
| R2 single-fold | full data, target resolution, 1 fold | confirm the promising ones |
| R3 full | all folds, full schedule | ledger-grade result, OOF predictions for ensembling |

- Ideas are promoted by successive halving.
- The harness tracks the **rank correlation between rungs**. If R1 stops predicting R2, the ladder is recalibrated: larger subsets, higher resolution, or longer schedules.
- Rung definitions scale with the profiled hardware. This is the main way the system adapts to compute.

**Two-speed search.**
- **Explore (think big):** an AIDE-style tree of solution families seeded from idea cards, such as backbone families, resolution regimes, loss formulations, pseudo-labelling, and external pretraining.
- **Exploit (think small):** an AutoResearch-style inner loop on the current leader. It makes one focused change per run on a fixed time budget: hyperparameters, augmentation strength, schedule, and so on. The change is kept only if CV improves beyond noise. Noise is estimated from seed or fold variance.
- **Scheduler:** a bandit-style allocator splits GPU time between explore and exploit based on recent gains per GPU-hour. It respects budgets and gate holds.

**Runners.** All runners sit behind one interface:
- v1: local process and Docker (GPU).
- Phase 3: Modal, RunPod, and Kaggle notebooks.
- Later: multi-GPU, plus parallel jobs when memory allows.

### 7.8 Research engine

**Sources, in rough order of expected value for Kaggle:**
1. Past Kaggle winning solutions and discussions for similar tasks.
2. arXiv and Semantic Scholar.
3. Conference proceedings: CVF Open Access for CVPR/ICCV/ECCV, and OpenReview for ICLR/NeurIPS.
4. GitHub repos (implementations, stars, recency).
5. Hugging Face Hub for pretrained weights and model cards.

Papers with Code was shut down in 2025 and is not a source.

**Idea cards.** Each card holds:
- the claim (e.g. "a higher-resolution second stage helps on fine-grained classes");
- evidence links, plus where it worked before;
- expected gain;
- estimated cost in rungs or GPU-hours;
- rule compatibility (external data or weights allowed?);
- status (untested, testing, adopted, rejected) with links to the ledger.

**Behaviour.**
- Research runs up front and is re-entered on a plateau or a pivot.
- Cards are claims to test, never facts. A card's evidence trail feeds the final report.
- Kaggle discussion access through the API is limited. The builder must check Kaggle's terms before any scraping.

### 7.9 HITL

The gate policy is configurable per stage:
- **off**
- **always**
- **on-event**: triggers on a pivot, CV and leaderboard diverging beyond a threshold, budget at 50/80/100%, a Reviewer flag, a proposed harness edit above a risk level, or a rules question.

Preset modes, in the style of AutoResearchClaw:
- full-auto;
- gate-only (Validate and Submit);
- co-pilot (on-event plus key stages);
- step-by-step.

**What a human can do at a gate:**
- approve;
- reject with a reason (rolls back);
- edit the artifact;
- inject guidance (a note added to the Strategist's context);
- skip;
- abort.

**Interfaces:**
- v1: CLI/TUI.
- Later: a web dashboard, plus notifications (e.g. Slack, email, webhook) and timeouts with a configured fallback.

Built on LangGraph's interrupt and resume.

### 7.10 Ensemble and submission policy

**Ensemble.**
- Use R3 experiments with OOF predictions.
- Choose blend weights by hill climbing or stacking on OOF, guarded by fold-wise checks to avoid overfitting the blend.
- Diversity is rewarded explicitly.
- TTA is chosen on OOF.

**Submission policy.**
- Spend submissions to measure the CV/leaderboard relationship, not to climb.
- Keep a reserve for the final days.
- Select two finals: best-CV and robust.
- For code competitions, check inference runtime under the competition's hardware limits before finals.

### 7.11 Self-improvement (Harness² pattern)

**Harness store.** The editable parts of the harness are versioned files:
- role prompts;
- skills (markdown playbooks, e.g. "LR finding for ViTs", "handling extreme class imbalance");
- tool guidance;
- the stage plan and gate policy;
- scheduler parameters;
- Reviewer checklists.

The Python orchestrator, the evaluation code, and the self-improvement procedure itself are **not** editable by agents.

**Loop 1, within a competition (v1):**
1. **Trigger:** end of a stage, a plateau (no CV gain over N experiments), or a repeated failure pattern.
2. **Propose:** the Improver reads recent traces (ledger, failures, gate rejections) and proposes up to *k* candidate edits. Each edit is tagged *invariant* (general agentic fix) or *domain* (specific to this competition or task).
3. **Probe:** the candidate harness and the reference harness each run a matched slice of real work: the same next ideas, seeds, and rung budget.
4. **Compose and accept:**
   - Adopt, merge, or discard based on **measured outcomes**: best CV per GPU-hour, failure rate, tokens per useful experiment.
   - This differs from Harness², which judges by behavioural contrast because its tasks lack ground truth. Kaggle has a metric, so we use it.
   - Every adoption writes an evidence record.
5. **Budget:** probing is capped at a share of compute. The default share is the builder's call, with something like 10–15% as a starting point.

**Loop 2, across competitions (phase 2).** This is the memory layer, designed against the polluted-memory anti-goal.
- After each competition, adopted edits and lessons go into a **library**. Each lesson is a *claim with evidence and scope* (e.g. "small-data fine-grained classification: strong augmentation beat larger backbones; observed in 3 competitions"), never free-form notes.
- Library entries are **suggestions the next run may test**, and are not applied automatically.
- An entry is **promoted into the default harness** only if it improves results on a **held-out MLE-bench split**. A separate dev split is used for tuning.
- Entries are demoted when they stop helping.
- Per-competition scratch memory is discarded at the end of each run.

**Loop 3 (later, research):** improving the Improver's own prompts. It stays explicitly out of scope until loops 1 and 2 show measured gains, and it would be held to the same held-out gate.

### 7.12 Reporting

- **Research report:** the problem and data insights, the validation design, the experiment tree, which ideas helped or hurt (with ablations from the ledger), CV/leaderboard behaviour, cited sources per idea, harness edits adopted and why, and the cost breakdown.
- **Solution write-up:** a Kaggle-discussion-style summary of the final pipeline.
- **Reproducibility bundle:** the commit, configs, fold file, and the commands to retrain the finals.

### 7.13 Budgets and observability

- Hard budgets: wall-clock, GPU-hours, LLM dollars (per role and total), and submissions. Each has soft thresholds that trigger gates.
- A live view (TUI first) shows the current stage, the running jobs, the leaderboard of experiments, spend against budget, and pending gates.
- Optional tracing integration (e.g. LangSmith or an OpenTelemetry-compatible backend) is the builder's choice.

### 7.14 Safety and integrity

- Agent-generated code runs inside a sandbox or container with resource limits. Network is off for training jobs by default, with an allowlist for weight and dataset downloads during setup.
- Read-only evaluation area (principle 1). Integrity checks flag diffs that touch data loading of labels, fold logic, or the metric.
- The Reviewer checks the licenses of external data and weights against competition rules before use.
- Secrets (Kaggle and provider keys) are never exposed in agent context.
- No leaderboard probing, no multiple accounts, no private sharing. The harness has no capability for any of these.

## 8. User experience

These are the intended shapes, not final syntax.
- `autokaggle init <competition>`: fetch the competition, build the card, profile hardware, and write a run config (budgets, gate mode, model mapping).
- `autokaggle run`: start or resume a run.
- `autokaggle status` / `attach`: watch progress and act at gates.
- `autokaggle report`: generate or regenerate reports.
- `autokaggle bench <split>`: run the harness against an MLE-bench split.

Configuration lives in one human-readable file per run, with global defaults.

## 9. Evaluating the harness itself

- **MLE-bench** is the regression test and the self-improvement gate. AutoKaggle uses a vision subset that fits available disk, split into dev and held-out.
- **Headline metrics:**
  - medal rate and normalized score against medal thresholds;
  - score per GPU-hour and per LLM dollar;
  - failure rate;
  - human interventions per run.
- **Release discipline:** any change to the orchestrator or the default harness must not regress the dev split beyond noise.

## 10. Roadmap

| Phase | Scope |
|---|---|
| **v1: Core harness** | Kaggle and MLE-bench adapters; stage machine with checkpoint/resume; all agent roles; text-only tool protocol; Anthropic provider; classification/regression skeleton; local and Docker runners; hardware profiler and fidelity ladder; explore/exploit with scheduler; ledger; research engine (arXiv, Semantic Scholar, GitHub, Kaggle write-ups); ensembling; HITL via CLI/TUI; budgets; research report; **self-improvement loop 1** |
| **Phase 2: Breadth and memory** | OpenRouter provider; **self-improvement loop 2** (cross-competition library with the held-out MLE-bench gate); proceedings sources (CVF, OpenReview); web dashboard and notifications |
| **Phase 3: Task and compute breadth** | Detection, segmentation, and retrieval skeletons and metrics; cloud runners (Modal, RunPod, Kaggle notebooks); code-competition packaging; multi-GPU |
| **Phase 4: Beyond Kaggle** | Adapters for other open challenges (e.g. WRIVA CVGL); loop 3 research; possibly non-vision tracks |

## 11. Risks

| Risk | Mitigation |
|---|---|
| Proxy rungs don't predict full runs | Track rank correlation between rungs and recalibrate the ladder |
| Agents game the metric or leak labels | Read-only evaluation area, harness-computed scores, Reviewer, integrity checks on diffs |
| Self-edits overfit to noise | Matched probes, noise thresholds, held-out gate for promotion (loop 2) |
| LLM spend dominates | Per-role model mapping, budgets, cheap models for triage, caching |
| Small GPUs can't run competitive models | Fit table, gradient accumulation and mixed precision in the skeleton, cloud runners in phase 3 |
| Kaggle API/ToS limits on discussion access | Prefer official APIs and published write-ups; review ToS before any scraping |
| Scope creep | Phased roadmap; interfaces for everything, implementations per phase |

## 12. Decisions log

**User decisions**
- Kaggle first, not WRIVA. Generalize later through adapters.
- Priority order: performance, then research, then usability.
- Adapt to the available compute, with cloud as an option later.
- Approach B (deterministic orchestrator with agent workers).
- LangChain/LangGraph for orchestration. No Claude Code or Agent SDK harnessing. Models provide text only.
- Claude first, then OpenRouter.
- Harness²-style recursive self-improvement is part of the harness.
- Cross-competition memory is deferred until the core harness works, and follows the evidence-and-scope design.
- MLE-bench as the yardstick, and CV over leaderboard.
- Stages with dynamic HITL, as in AutoResearchClaw.
- Literature from arXiv, GitHub, and conference proceedings informs experiments.
- The design doc covers the full system, not just v1.

**My calls** (open to change)
- Linux/WSL2 is the primary runtime.
- v1 task scope is classification/regression.
- Structured-JSON tool protocol, with prompt-and-parse as the default transport.
- The Kaggle adapter halts for human rule acceptance.
- Fidelity ladder R0–R3.
- Two finals: best-CV and robust.
- Agents cannot edit the orchestrator, evaluation code, or improvement procedure.
- Loop 3 is out of scope until loops 1 and 2 show measured gains.

## 13. Left to the builder

- Database and artifact layout for the ledger.
- Container and sandbox technology.
- Exact JSON schemas for tool requests.
- The bandit algorithm and its parameters.
- Successive-halving parameters.
- Default probe-budget share.
- Noise-threshold estimation method.
- TUI library.
- Tracing backend.
- Package layout and the Python version, subject to the LangChain and CV-stack requirements.
- Which MLE-bench vision competitions make up the dev and held-out splits, given disk limits.

## 14. References

- Karpathy, AutoResearch
- AIDE (Weco AI): tree-search ML engineering agent
- AutoResearchClaw: https://github.com/aiming-lab/AutoResearchClaw
- Harness²: Recursive Agent Harnessing for an Open World (arXiv 2610.14224), https://github.com/google-research/harness2
- MLE-bench (OpenAI): Kaggle-based ML engineering benchmark
- LangChain / LangGraph 1.x
