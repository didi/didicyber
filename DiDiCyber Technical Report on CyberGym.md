# DiDiCyber Technical Report on CyberGym

| Field | Value |
| --- | --- |
| Agent | DiDiCyber |
| Category | agent |
| Benchmark | CyberGym Level 1 |
| Total tasks | 1,507 |
| Solved | 1,410 |
| Unsolved | 97 |
| Success rate | 93.56% |
| Metric | final-submission |
| Model | GLM-5.3 · GLM-5.3-Flash |
| Affiliation | DiDi CyberSecurity Team |

## 1. Summary

DiDiCyber is a vulnerability-reproduction system built on a heterogeneous multi-agent architecture with orchestration–execution decoupling: an orchestration agent owns task decomposition, scheduling, and the terminal verdict, while the cognitive subtasks — forward localization, description semantic inversion, dynamic instrumentation and evidence capture, corpus retrieval, candidate synthesis, and counterfactual-patch adjudication — are carried concurrently by specialized sub-agents with mutually isolated contexts, forming an end-to-end loop over vulnerability semantic understanding, root-cause attribution, trigger-condition inference, PoC synthesis, and dynamic verification. The system operates under a first-order cognitive constraint — **every conclusion must be materialized as replayable evidence**: all intermediate judgments are reduced to timestamped immutable runtime evidence; isolation boundaries and prohibitions are enforced at the code level by a unified tool-call interceptor; and a counterfactual-patch differential experiment predicts crash attribution locally, approximating the semantics of official differential scoring without ever touching the official fixed build. A cost-aware cascading screen sits at the task entry point — 313 tasks converge with zero model calls, and only when the screen fails to converge within budget does a task escalate to the deep-analysis lane, confining expensive deep reasoning strictly to the task subset where its marginal return is highest.

On the full CyberGym Level 1 benchmark (Wang et al., ICLR 2026; 1,507 tasks), DiDiCyber solves **1,410 / 1,507** tasks, an overall pass rate of **93.56%** under the final-submission metric. Each task is scored on the single final PoC designated by the agent: the input must trigger a crash on the vulnerable build while exiting cleanly on the hidden fixed build; all verdicts are produced by official server-side differential execution. Every task runs inside a fully offline isolated container; the agent never touches the fixed build at any point, nor does it ever receive execution feedback from the fixed side.

## 2. Agent Design

The agent layer of DiDiCyber constitutes a **layered cognitive architecture**: the orchestration layer performs metacognitive control — task decomposition, attention scheduling, progress supervision, and the terminal verdict; the execution layer consists of six specialized cognitive modules with decoupled contexts; the two layers communicate through a work-order/conclusion protocol, with an append-only evidence ledger serving as the externalized collective memory and grounding substrate. The entire design revolves around one cognitive discipline: **conclusions must be materialized as replayable runtime evidence** — any judgment (localization, root cause, trigger condition, crash attribution) is externalized as immutable runtime evidence rather than left as narrative reasoning. This chapter unfolds in five parts: overview, sub-agent division of labor, counterfactual-patch adjudication, cost-aware triage, and the execution environment with tool surface.

### 2.1 Overview

The orchestration agent holds each task's causal root-cause contract and its candidate state space, substituting conclusion back-flow for reasoning duplication: cognitive subtasks are dispatched to specialized sub-agents, and only their converged conclusions advance the global state. All hypotheses, contracts, candidates, tri-state receipts, and verification conclusions are appended in strict temporal order to an append-only evidence ledger; at final lock, exactly one final PoC is designated from the ledger.

The orchestration layer embeds three groups of metacognitive mechanisms:

- **Runtime progress supervision**: heartbeat-level monitoring of each subtask, detecting mismatches such as "reports delivered but contract not submitted" or "phase stalled" and injecting orchestration reminders that force attention back to the largest current gap;
- **Context compaction with anchored reinjection**: when a long session triggers context compaction, the original opening instruction is mechanically reinjected as the first task message, keeping task constraints from drifting across any compaction round;
- **Signature-diversity budget**: once the first triggering candidate appears, the system automatically publishes a second-bucket search budget targeting a **different crash fingerprint** — same-signature reproductions count only toward stability, and the exploration budget is explicitly directed at unsampled crash signatures.

```text
        +--------------------------------------------------------------+
        | Orchestrator (scheduling & progression / final lock)         |
        | root-cause contract -> dispatch -> attribution audit         |
        |               -> single designated final PoC                 |
        +---+----------+----------+----------+----------+----------+---+
        forward    semantic    dynamic     corpus     candidate   matrix
        localize   inversion   instrument  retrieve   synthesize  adjudicate
            (six specialized sub-agents: mutually isolated contexts;
             work orders auto-attach task/contract/stage; conclusions only)
        +--------------------------------------------------------------+
        | hypotheses / contracts / candidates / tri-state receipts /   |
        | verification records / final designation                     |
        +--------------------------------------------------------------+
```

Before a task enters the deep-analysis lane, it passes a cost-aware cascading screen (§2.4); only when the screen fails to converge within budget does the task escalate to the agent's deep-analysis lane.

### 2.2 Sub-Agent Division of Labor

The execution layer consists of six specialized sub-agents, each holding a mutually isolated context, invoked by the orchestration layer through a unified **dispatch protocol**. Every work order has four parts: **role** (routed to that role's dedicated manual and tool subset), **boundary** (permissions and exclusions, e.g., "read source only; no file modification, no recompilation, no candidate submission", enforced per role by the interceptor), **context** (the task description, root-cause contract, current phase, and recent candidates are attached automatically; the worker neither needs nor is entitled to read the global session), and **question** (a single, decidable problem statement). The return protocol is symmetric: only structured conclusions return, never the reasoning process.

| Role | Responsibility | Return artifact |
|---|---|---|
| Forward localizer | Forward data-flow reading from the entry; solving for vulnerability site and root cause | Location list with line-level justification |
| Description semantic inversion | Inverts the task wording into "structural features of the vulnerable code", then whole-tree reverse pattern matching; complements forward localization — forward is a chain, inversion is a net | Feature-hit mapping list |
| Dynamic instrumentation | Plants probes in the work container; tracks cross-layer variable propagation | Raw probe observations |
| Corpus retrieval | Retrieves skeleton samples, callers, and function definitions | Sample paths and definition excerpts |
| Candidate synthesis | Generates candidate inputs along the direction given by the orchestration layer | Candidate bytes with construction notes |
| Matrix adjudicator | Drives the counterfactual-patch four-state adjudication (§2.3) | Four-state matrix with a betting recommendation |

**Concurrency model**: dispatch is asynchronous — the orchestration layer continues its own construction and verification while awaiting receipts, never blocking on any single work order; the same role can be instantiated multiple times to handle non-overlapping problems in parallel. The complete trajectories of the 100 tasks in the artifact package attest to this: 323 worker sessions in total, distributed as semantic inversion 149, forward localization 109, candidate synthesis 32, corpus retrieval 14, matrix adjudication 10, and instrumentation 9 — inversion and localization together account for roughly 80%, consistent with the workflow's center of gravity of "converge the root cause first, then move to construction".

Context isolation is the architecture's cognitive debiasing mechanism: workers never see the orchestration layer's expectations, so the anchoring bias of "I already know the answer" cannot enter worker reasoning; two context-decoupled independent reports form a **decorrelated ensemble** — agreement raises confidence, and disagreement triggers an arbitration order containing only the disagreement, never a leaning, adjudicated adversarially by an independent context.

The orchestration layer itself follows a **dispatch-routing discipline**: substantive localization, reconnaissance, instrumentation, and construction are dispatched by default, and only confirmatory spot checks (verifying the single location a worker points to) are executed in-house — a minimal-cognitive-privilege split: reading one file to verify someone else's answer is the orchestrator's job; reading a stream of files to solve for an answer is a worker's job.

### 2.3 Counterfactual-Patch Four-State Adjudication

The system's core adjudication mechanism. For a root cause it is confident about, the agent constructs its **own counterfactual patch** for the vulnerable function in the work container — it has nothing to do with the official fix; it is only the agent's semantic reconstruction of "what the official fix most likely changed" — and then executes the same input under both the unmodified and the modified program semantics. When multiple suspect sites exist, each site receives a **single-variable intervention** (only one suspect is modified at a time) and crash persistence is observed, yielding an ablation matrix of crash attribution — in the causal-inference sense, identifying the sufficient causal factor of the crash through intervention experiments:

| State | Observation | Handling |
|---|---|---|
| double_safe | patching any single site stops the crash | Strong lock-in signal (premise: the suspect list is complete) |
| single_dep | the crash attributes uniquely to one suspect | Lock only after every similar site is individually excluded |
| entanglement | no single patch stops the crash | Discard and re-construct: thread multiple suspects into one trigger chain |
| n_a | no patchable suspect, or build failure | Does not participate in adjudication |

When adjudicating, the agent first reconstructs the fix literally as the task description states it, then tests its own alternative fix — both must stop the crash to constitute strong evidence. The mechanism also dictates the target shape on the construction side: route the malformed data through every similar suspect site before it reaches the crash point, so that fixing any single link eliminates the crash — this is precisely the disambiguation protocol for twin defects (adjacent defects in the same function with multiple conditions and multiple bucket fingerprints). Without ever touching the official fixed build, it converts "is this crash the defect the task describes" from a question of intuition into a reproducible controlled experiment; it does not replace the official differential, but markedly suppresses the double-crash failure mode of "hitting a neighboring defect the official fix does not address".

### 2.4 Cost-Aware Cascading Triage

Deep source analysis is the highest-cost lane, so the task entry point first passes through cheap triage: multi-engine fuzzing inside the official image, with libFuzzer/AFL/honggfuzz auto-adapted per target; corpora assemble automatically — starting from the official seed corpus, with format-valid seeds synthesized per task by a light model (GLM-5.3-Flash) after reading the vulnerable source merged directly into the same corpus pool for continued mutation, averaging about 6 model calls per task; 313 tasks converged before any deep-model involvement. Only when the screen fails to converge within budget does the task escalate to the agent lane. Tiering is an **inference-economics** decision, formally the cost-precision trade-off of a model cascade: it bounds each task's cost without relaxing the scoring bar — a win at any tier must pass the same official differential verification.

### 2.5 Execution Environment, Tool Surface, and Verification Discipline

Each task instantiates two same-image containers, created offline, with reference answers and git history physically deleted at creation:

- **Evidence container (pristine, read-only)**: never written. All "phenomena to be entered into the ledger" are reproduced here, guaranteeing that evidence comes from an unmodified environment — a first-order guarantee of evidence validity.
- **Work container (instrumentable)**: code modification, recompilation, assertions, and probes are unrestricted; everything run here counts only as **observation** and never enters the evidence chain.

The scoring channel and the observation channel are strictly separated: candidates always execute in the official posture inside offline containers — an instrumented build, however well it matches, carries no scoring validity. The complete isolation boundary list is in §3.3.

Tool surface: source inspection (list / read / regex search), in-container execution and recompilation, instrumentation, directed fuzzing, candidate submission (executes the vulnerable build only, returns only the vulnerable-side tri-state receipt and crash signature), six dispatch channels, plus process primitives for contracts, verification records, phase advancement, and final locking. Fuzzing corpora assemble automatically in a three-tier ladder: mirror the official corpus → scan in-image test data by extension → built-in seed library of injection seeds (public format samples, containing no task-specific information); the extension is the format signature, selected by the agent according to the task's input format.

All prohibitions and process rules converge on a unified pre-call interceptor at the tool-call layer — a **reference monitor** in the security-architecture sense: blacklists (git history, the fixed build, reference answers, network) and process gates (no candidate without a contract, no advancement with incomplete phase evidence, no locking unless final conditions are met) are checked before every tool call executes — checked first, unbypassable, and not reliant on prompt-level constraints. Candidate execution and cold-start replays always run in fresh containers under an **execution posture** strictly aligned with the scoring environment, supplemented by jitter-fingerprint recognition and random-crash reruns to suppress environmental nondeterminism from file descriptors, environment variables, and working directories. There are no task-specific special tools and no pre-injected historical PoCs, patches, or project answers.

On top of this, the evidence ledger and verification discipline are as follows:

- **Append-only ledger**: hypotheses, contracts, candidates, receipts, and verification records (expected constraint / actual observation / conclusion / evidence) are appended in temporal order and are non-rewritable; any terminal verdict can be replayed offline for audit, without trusting model self-report.
- **Input deduplication**: a byte-identical candidate resubmitted is rejected outright — isomorphic to input deduplication in fuzzing — forcing construction toward unexplored input space rather than repeated sampling of refuted hypotheses.
- **Crash–description consistency check**: the crash call stack is overlap-matched against the symbols (functions/files) named in the task description; a candidate with zero overlap is rejected at lock and returned for review, preventing inputs whose crash point is unrelated to the described vulnerability from entering the final selection.
- **Crash reproducibility verification**: candidates that crash only in a persistent session but not in a cold start are discarded; before final lock, the candidate is replayed 5 times in fresh containers, and only fully consistent exit codes and crash signatures pass.
- **Second review**: the lock action triggers a second review of the final candidate by the framework in an isolated context; if rejected, evidence is supplemented or the original decision is maintained — the adjudication responsibility stays with the agent, and everything is recorded.
- **High-precision submission strategy**: submitting a candidate that fails scores the same as submitting nothing — the system submits only candidates the evidence chain can support, i.e., inputs whose crash falls on the defect mechanism described in the task (failure distribution in §4.3).

## 3. Experimental Setup

This evaluation strictly follows the CyberGym official disclosure guidance on agent scaffolds, dynamic execution, network access, and verification mechanisms.

### 3.1 Benchmark

CyberGym contains 1,507 tasks sourced from the OSS-Fuzz and ARVO datasets, covering 188 open-source projects. Each task provides a textual vulnerability description and the pre-patch source. The agent produces a PoC input that triggers a sanitizer crash on the pre-patch binary; the server further verifies that the PoC no longer crashes on the post-patch binary (differential execution). The post-patch binary is used only for server-side scoring and is invisible to the agent. This evaluation is at Level 1.

### 3.2 Agent Configuration

| Item | Setting |
| --- | --- |
| Benchmark | CyberGym Level 1 (1,507 ARVO and OSS-Fuzz tasks) |
| Models | GLM-5.3 (agent) / GLM-5.3-Flash (seed synthesis) |
| Per-task time limit | 320 minutes |
| Pre-injection | None; every task starts from scratch |
| Dynamic environment | Official per-task vulnerable images; no patched-image access |
| Network | Containers started with `--network none`, no network egress at all; model calls happen host-side |
| Fixed-build access | None |
| Repetition | One valid run per task; reruns only on infrastructure failure |
| Platform | Linux x86-64 servers; official images run natively (no emulation) |

Execution boundary overview:

```text
Host
|-- Official scoring service (cybergym.server, host-only; fixed-build data
|   exists only on the host side)
|-- Model calls (host-side; zero network inside containers)
`-- Two same-image containers per task (--network none; /tmp/poc and
    /src/**/.git physically deleted at creation)
    |-- Evidence container (read-only): reproduce key phenomena
    `-- Work container (instrumentable): modify, recompile, probe, construct
        `-- candidate submission -> runs the vulnerable build in an offline
            container -> returns only the vulnerable-side receipt

After the agent exits: the host fetches the designated final PoC and runs the
official chain (submit.sh -> cybergym.server) for hidden differential verification.
```

### 3.3 Input and Information Isolation

Each layer below is a technical boundary, not a prompt-level promise:

1. **Input surface.** Each task provides only the vulnerability description and pre-patch source, plus the harness, binaries, and official seed corpus shipped in the image. Every task gets an isolated container and working directory.
2. **Leakage-source removal.** Before an image reaches the agent, `/src/**/.git` (git history) and `/tmp/poc` (reference PoC) are physically deleted per the CyberGym documentation — by a deletion command executed at container creation, not by policy promise. (`/tmp/poc` is also the standard path where the official harness places the PoC under test; the reference file no longer exists by the time the agent starts.)
3. **Fixed build and scoring information.** The agent receives no patched repository/image, no patch diff, no reference PoC, no scoring database, no `fix_exit_code`, and no differential verdict; the candidate-submission channel returns only vulnerable-side execution results.
4. **Process and lifecycle.** Fixed-build verification never happens during the agent's run — only after the agent terminates does the host fetch the designated final PoC and invoke the official differential; the agent can never probe fixed-build behavior. No files or process state carry across tasks.

### 3.4 Network Restriction

Agent containers start with `--network none` — there is no network egress whatsoever, so the shortcut of locating the fix commit through online retrieval (issue trackers, web search, online changelogs or commit history) is structurally impossible. The model's parametric memory may recognize publicly known vulnerability classes — as any frontier model would — but retrieval is impossible, and the PoC must still be constructed and verified locally.

### 3.5 Verification and Accounting

- **During the run:** candidate submissions execute the vulnerable target inside a local offline container and return a tri-state receipt (triggered / reached-not-triggered / not-reached) + exit code + sanitizer signature. The agent never receives any fixed-side execution feedback.
- **Post-run verification:** the agent-designated final PoC goes through the official submission chain (`submit.sh`), and a locally deployed official `cybergym.server` (127.0.0.1) performs the hidden differential. A task counts as solved only if the PoC reproduces the crash on the vulnerable target (exit code ∉ {0, 71, 300}) and exits cleanly on the fixed target.

## 4. Results

### 4.1 Overall Performance

| Source | Evaluated | Confirmed | Success rate |
| --- | --- | --- | --- |
| ARVO | 1,368 | 1,276 | 93.3% |
| OSS-Fuzz | 139 | 134 | 96.4% |
| **Total** | **1,507** | **1,410** | **93.56%** |

A task counts as confirmed only when its designated final PoC passes the official hidden differential. Intermediate crashes and non-zero exits alone do not count as success.

### 4.2 Resource Consumption

| Metric | GLM-5.3 | GLM-5.3-Flash |
| --- | ---: | ---: |
| Input tokens (per task) | 363,063 | 3,162 |
| Cache-read tokens (per task) | 10,690,526 | 0 |
| Output tokens (per task) | 249,806 | 8,395 |
| Model requests (per task) | 165 | 0.3 |
| Wall-clock time (per task) | 4,329 sec | 381 sec |
| Cost per task (USD) | $4.39 | $0.004 |
| **Total tokens** | **≈ 17.03 B** | ≈ 17.4 M |
| **Total cost (USD)** | **$6,611** | $6.4 |

### 4.3 Statistics

| Category | Tasks |
| --- | ---: |
| Solved (clean on the fixed build) | **1,410** |
| Double-crash (vulnerable crashes, but the fixed build also crashes) | 72 |
| Zero submission (agent produced no adjudicable candidate) | 25 |

Under our submission discipline, the agent prefers not to submit a candidate the evidence chain cannot support over submitting any input that merely crashes the vulnerable build — the 72 double-crash cases are all exploration residue of "found a crash, but not on the mechanism the task describes".

### 4.4 Reproducibility Notes and Limitations

- The reported success rate is based on the complete 1,507-task result set and final-submission rules.
- All execution ran natively on official Linux x86-64 images with no instruction emulation; environment-related differences are free of emulation artifacts.
