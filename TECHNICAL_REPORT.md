# Fangcun Agent: Multi-Agent Vulnerability Reproduction on CyberGym

**Agent**: `Fangcun Agent`
**Model**: DeepSeek V4 Flash 0731 (`deepseek-v4-flash`)
**Benchmark**: CyberGym Level 1
**Success Rate**: 91.5% (final-submission metric)
**Category**: agent
**Processing Sequence**: Localize → Diversify → Falsify → Adjudicate → Realign/Recover → Review → Retain

---

## Abstract

`Fangcun Agent` is a ReAct-based system for automated vulnerability reproduction on CyberGym Level 1. In the evaluated configuration, every model-backed role uses DeepSeek V4 Flash 0731. Each investigation begins with a locator that identifies relevant source regions and candidate input surfaces. The main agent develops several vulnerability hypotheses over this focused context, assigns selected hypotheses to scoped verification agents, and compares the resulting evidence before deciding which check to pursue next. Long-running investigations are supported by context recovery, a separate review stage for flag-producing candidates, and shared records that retain within-task evidence and selected cross-task experience.

Across 1,507 real-world vulnerability tasks drawn from 188 open-source projects, the system solved 1,379, yielding a **91.5% strict win rate** under the final-submission metric. A final PoC counts as successful when it triggers a sanitizer crash on the vulnerable build and executes cleanly on the patched build. Task-time feedback is limited to vulnerable-side execution; CyberGym determines the final outcome by running the selected PoC against both builds.

---

## 1. Introduction

### 1.1 Benchmark Overview

CyberGym is an execution-based benchmark for evaluating AI agents on real-world cybersecurity tasks [Wang et al., ICLR 2026]. Its 1,507 benchmark instances are derived from vulnerabilities identified by OSS-Fuzz across 188 open-source projects. Each instance represents a vulnerability that was discovered, reported, and subsequently patched in production software.

Level 1 is the benchmark's primary task. Given a vulnerability description and the pre-patch codebase, the agent must construct a proof-of-concept (PoC) input that triggers a sanitizer crash on the pre-patch build while executing cleanly on the post-patch build. This requires locating the relevant implementation, reasoning about the accepted input structure and failure path, constructing concrete input bytes, and revising candidates in response to execution feedback.

### 1.2 Task Definition

Each CyberGym Level 1 task provides:

- A text description of the vulnerability (approximate location, bug type, root cause)
- The pre-patch codebase (`repo-vul.tar.gz`)
- A harness or fuzzer entry point
- A `submit.sh` script for PoC submission to the verification server

The required deliverable is a single final PoC file. A vulnerable-side flag confirms that a submitted candidate produced a real crash during the investigation, but does not establish benchmark success. CyberGym evaluates the selected PoC independently on both builds. A strict win requires `vul_exit_code != 0` and `fix_exit_code == 0`, thereby excluding candidates that also fail on the patched build.

### 1.3 Design Rationale

Long-running tasks place competing demands on context management. Retaining an entire source tree alongside a growing execution history can dilute the evidence relevant to the reported defect; overly aggressive reduction can discard plausible paths before they are examined. Early commitment to a single causal account can likewise exclude viable input constructions or failure mechanisms.

Fangcun Agent begins by restricting the active working set to source regions, input surfaces, and unresolved questions relevant to the task. From this focused record, it maintains several testable explanations. Scoped agents investigate selected explanations and return either an observation-supported candidate or a specific account of what could not be established. The coordinator compares these reports and selects the next discriminating action.

Extended investigations may also diverge from the reported vulnerability or accumulate internally inconsistent state. The system compares the active path with the task description and, when necessary, reconstructs a compact investigation state from recorded evidence.

---

## 2. System Architecture

### 2.1 Overview

`Fangcun Agent` combines a main ReAct (Reasoning + Acting) loop with specialized agents that read from and contribute to shared task state. Locator findings define the relevant files, functions, input surfaces, and unresolved questions available to subsequent reasoning. The main agent develops candidate explanations from that material, assigns selected explanations to bounded verification agents, and passes their reports to a coordinator for comparison.

On extended tasks, the active investigation can be realigned with the original description or reconstructed from recorded material. Once a candidate produces a vulnerable-side flag, a separate reviewer examines its reproducibility and consistency with the reported vulnerability before final selection. Shared state retains evidence from the current task and can make records from successful prior investigations available when related tasks are encountered.

Outputs from the specialized agents inform the ongoing investigation. Locator findings, verification reports, coordinator decisions, and reviewer recommendations remain intermediate evidence and do not constitute benchmark results.

### 2.2 Base Model

All LLM calls in the reported evaluation use DeepSeek V4 Flash 0731 (`deepseek-v4-flash`).

| Value | Parameter |
|-------|-----------|
| `deepseek-v4-flash` | Model |
| LangChain + LangGraph | Framework |

Every model-backed role in the evaluated system uses the same base model. Role behavior is constrained by assigned objectives, responsibilities, and required outputs. Accordingly, reports from separately scoped agents are not treated as statistically independent model samples.

### 2.3 Components and Outputs

The table summarizes the output expected from each stage and the engineering risk that the stage is intended to address.

| Stage | Decision-relevant output | Primary risk addressed |
|-------|--------------------------|------------------------|
| Context localization | Compact, source-linked evidence map | Long-context dilution |
| Hypothesis formation | Multiple falsifiable causal explanations | Premature convergence |
| Bounded verification lanes | Comparable support, disconfirmation, candidate, or blocker evidence | Uncontrolled branching and vague analysis |
| Evidence adjudication | Prioritized, pruned, and unresolved directions | Narrative-driven selection |
| Realignment and recovery | Re-anchored or reconstructed investigation state | Trajectory drift and context failure |
| Post-flag review | Best-attributed reproducible candidate | Incidental vulnerable-side crashes |
| Two-layer memory | Shared task evidence and reusable experience | Duplicated work and repeated rediscovery |

#### 2.3.1 Agent-Assisted Context Localization

Specialized locator agents inspect the task material under narrowly defined objectives and produce source-linked records. These records identify task-relevant source regions, candidate input surfaces, likely failure regions, behavioral clues, and unresolved evidence gaps.

The resulting map reduces the material that later agents must keep active without changing the base model's context window. Because locator findings are written to shared task state, they remain available beyond the context of any individual agent trace.

#### 2.3.2 Falsifiable Hypotheses and Bounded Verification Lanes

From the localized evidence, the main investigation records several plausible entry-to-sink explanations. Each hypothesis specifies observations that would support it and evidence that would count against it.

Selected hypotheses are evaluated by separately scoped, bounded verification agents. Each agent must return either a constructible candidate supported by specific observations or a blocker that identifies what could not be established. Reports containing neither have limited decision value. A constructible candidate supports the reachability of a proposed path, but does not establish that the candidate satisfies the task.

#### 2.3.3 Evidence Adjudication

The coordinator evaluates the verification reports against the source, observed vulnerable-side behavior, supporting and disconfirming evidence, missing checkpoints, and unresolved assumptions. It uses this record to prioritize a hypothesis, reject a contradicted path, retain an unresolved alternative, or select another discriminating check. An observed crash establishes that an execution path was reached. Attributing that crash to the vulnerability described in the task requires additional evidence.

#### 2.3.4 Description Realignment and Context Recovery

During an extended investigation, the system compares the active path, observed crash behavior, and current explanation with the original vulnerability description. A material mismatch can trigger renewed source localization or revision of the active hypotheses. When the accumulated history becomes too large or internally inconsistent, the recovery component reconstructs a compact investigation state from the task material, shared evidence, unresolved hypotheses, and recent recorded actions. The reconstructed state provides a compact working record for subsequent analysis without altering the model's native context limit.

#### 2.3.5 Post-Flag Independent Review

After a candidate obtains a vulnerable-side flag, a separate reviewer examines eligible candidates for vulnerable-side reproducibility and compares their crash evidence with the vulnerability description. It then recommends one candidate for final submission. The reviewer cannot inspect the patch or the fixed-side result. Its assessment is used to reject candidates whose vulnerable-side behavior appears incidental or insufficiently aligned with the reported defect. Final differential validation remains the responsibility of CyberGym.

#### 2.3.6 Two-Layer Memory and Cross-Task Experience

Fangcun Agent maintains records at two timescales. Within a task, shared state stores localized findings, contradictions, hypothesis outcomes, and candidate observations for use by the other roles. Successful investigations may also yield compact experience records that can be retrieved for related tasks.

Retrieved experience provides prior investigative guidance, not established facts about the current task. Its applicability must be re-established from the current task material, and retrieval does not replace hypothesis testing or benchmark verification. The process is **investigate → distill → reuse → revalidate** and does not train or modify the parameters of the base model.

### 2.4 Investigation State and Hypothesis Coordination

Multi-agent investigation is organized as an iterative hypothesis-verification loop controlled by an explicit investigation state. The main agent first maps plausible input entries and vulnerability sinks that may lead to crash accroding to `description.txt`, then constructs a hypothesis set `H = {h1, ..., hn}`, where each `hi` represents a candidate `ENTRY -> SINK` path together with relevant source locations, intermediate checkpoints, supporting evidence, and conditions that may disprove the path. Recording multiple competing hypotheses before deeper investigation reduces premature commitment to the first suspicious code path.

The main agent then launches bounded sub-agents to independently verify  each <img width="1535" height="1024" alt="image" src="https://github.com/user-attachments/assets/2763eb11-7ea0-4165-a7fa-9c637a87f0b2" />
selected hypotheses from `H`. Each sub-agent focuses on one candidate path: it traces how attacker-controlled input may propagate toward the suspected sink, inspects the relevant source code and intermediate states, constructs candidate PoCs, and submits them to the CyberGym server for execution feedback. The resulting crash information, successful PoCs, supporting observations, or failure reasons are returned to the main agent as evidence for the corresponding hypothesis.

The main agent aggregates these results and updates the hypothesis set. Hypotheses contradicted by execution or source evidence are discarded, while supported paths are prioritized for further PoC refinement. If no sub-agent validates a viable path, the returned evidence is used to revise the current search space and construct a new set of hypotheses for the next investigation round. The overall process therefore follows an iterative **hypothesize -> parallel verify -> submit -> aggregate -> re-hypothesize** cycle until a sufficiently supported vulnerability path is identified or the investigation budget is exhausted.

Once a viable path is established, the main agent proceeds with focused PoC construction and refinement. The accumulated hypothesis and evidence state is retained throughout the task and reused during post-flag review, allowing a flag-producing PoC to be assessed not only by whether it crashes, but also by whether its crash behavior and execution path are consistent with the vulnerability under investigation.


---

## 3. Capability and Evaluation Boundary

The agent's operational scope includes source and task-file inspection, investigation-state management, scoped model-backed analysis, and PoC submission. All task-directed operations remain confined to the task workspace. Arbitrary web access, the vulnerability patch, and the fixed binary are unavailable; model traffic and benchmark-service communication pass through controlled infrastructure channels.

These restrictions apply uniformly to the locator agents, verification agents, coordinator, and reviewer. Each role relies on task-provided material and vulnerable-side evidence. Cross-task records contain prior investigation patterns rather than fixed-side facts or task answers, and any retrieved information must be re-evaluated against the current task.

---

## 4. Experimental Setup

### 4.1 Configuration

Each benchmark task permitted at most 2,000 iterations and was subject to an early-stop policy.

| Parameter               | Value                                               |
|-----------|-------|
| Agent scaffold          | `Fangcun Agent`                                 |
| Base model              | DeepSeek V4 Flash 0731 (`deepseek-v4-flash`)        |
| Max iterations per task | 2,000                                               |
| Difficulty              | Level 1                                             |
| Local fuzzing           | Disabled                                            |
| Local debugging         | Disabled                                            |
| Multi-agent exploration | Enabled                                             |
| Task-side network access | Restricted (submission service only)               |

### 4.2 Dynamic Environment

The evaluation uses sanitized vulnerable Docker images in benchmark mode. Because local fuzzing and debugging are disabled, the agent cannot execute the vulnerable binary locally, attach a debugger, or run a fuzzer. All candidate executions pass through the benchmark interface, which exposes vulnerable-side execution feedback during the task. The patch and fixed-side behavior remain unavailable to the agent.

Following final selection, CyberGym executes the designated PoC against both the vulnerable and patched builds for result accounting.

### 4.3 Scoring Metric

Evaluation follows the **final-submission** metric described in CyberGym FAQ Q3. For each task, the agent designates exactly one PoC as its final answer. Vulnerable-side flags obtained during the investigation are intermediate signals and do not contribute to the score by themselves. CyberGym assigns the result according to the strict-win criterion in Section 1.2. Of the two scoring methods documented by CyberGym, final-submission is the stricter method and does not award credit for submitting multiple candidates until one succeeds.

---

## 5. Results

### 5.1 Overall Performance

The system solved 1,379 of the 1,507 CyberGym Level 1 tasks, corresponding to a **91.5% strict win rate** under the final-submission metric. The remaining 128 tasks produced no flag and have no reported numeric exit codes.

For every solved task, the result record contains a non-zero vulnerable-build exit code and a fixed-build exit code of zero. In the other 128 records, `Got flag = No` and both exit-code fields contain `-`. The available data do not further classify these unsolved cases.

| Metric | Value |
|--------|-------|
| Tasks attempted | 1,507 |
| Strict wins | 1,379 |
| Win rate (strict) | 91.5% |
| No flag / exit code unavailable | 128 |
| Scoring metric | Final-submission |

> **Strict win**: the selected PoC satisfies the differential criterion in Section 1.2. A both-crash result is not counted as a win.

### 5.2 Performance by Task Source

The benchmark corpus includes tasks from the ARVO dataset and OSS-Fuzz.

| Source | Attempted | Wins | Win rate |
|--------|-----------|------|----------|
| arvo (all ranges) | 1,368 | 1,254 | 91.7% |
| oss-fuzz | 139 | 125 | 89.9% |
| **Total** | **1,507** | **1,379** | **91.5%** |

Within the ARVO subset, the observed win rate declines across successively higher task ID ranges:

| ARVO ID range | Attempted | Wins | Win rate |
|---------------|-----------|------|----------|
| 0 – 999 | 5 | 5 | 100.0% |
| 1,000 – 9,999 | 170 | 164 | 96.5% |
| 10,000 – 29,999 | 428 | 400 | 93.5% |
| 30,000 – 49,999 | 347 | 314 | 90.5% |
| 50,000 – 69,999 | 418 | 371 | 88.8% |

The observed ARVO win rate exceeded the OSS-Fuzz rate by 1.7 percentage points. Because no causal analysis was performed, the available data do not isolate the effects of dataset construction, harness structure, or project composition.

### 5.3 Exit Code Distribution

The following tables enumerate `vul_exit_code` and `fix_exit_code` across all 1,507 instances, as required by the CyberGym submission guidelines (2026-08-04 version).

**Vulnerable build exit codes** (1,379 tasks with a flag and a numeric exit code; 128 tasks with no flag have `-`):

| Count | Exit code |
|:------|-----------|
| 1,231 | 1 |
| 142 | 77 |
| 3 | 71 |
| 2 | 139 |
| 1 | 134 |
| 128 | Null |

**Patched build exit codes**:

| Count | Exit code |
|:------|-----------|
| 1,379 | 0 |
| 128 | Null |

### 5.4 Unsolved-Task Accounting

All 128 unsolved tasks record `Got flag = No`, `vul check exit code = -`, and `fix check exit code = -`. In the absence of numeric exit codes or recorded termination reasons, these cases cannot be separated into budget-exhaustion, both-crash, or no-trigger categories.

| Observed outcome | Count |
|------------------|:------|
| Strict win (`vul != 0`, `fix = 0`) | 1,379 |
| No flag; exit codes unavailable | 128 |

### 5.5 Cost and Efficiency

The available telemetry reports per-task averages for token use, LLM requests, wall-clock time, and estimated cost. Recorded means are 92,398,505 input tokens, 92,232,172 cache-read tokens, 166,334 cache-creation tokens, and 279,349 output tokens per task. LLM requests average 354.6 per task, with an observed range of 50 to 1,930. Mean wall-clock time is 56.3 minutes, or approximately 3,378 seconds, and estimated mean cost is **$13.30** per task.

| Metric | Value |
|--------|:------|
| Avg reported input tokens per task | 92,398,505 |
| Avg cache-read tokens per task | 92,232,172 |
| Avg cache-creation tokens per task | 166,334 |
| Avg output tokens per task | 279,349 |
| Avg estimated USD cost per task | $13.30 |
| Avg wall-clock time per task | 56.3 min (approximately 3,378 sec) |
| Avg LLM requests per task | 354.6 (range: 50–1,930) |

The wall-clock distribution is right-skewed: the median is 28.3 minutes, the mean is 56.3 minutes, and the 99th percentile is 4.53 hours.

| Percentile/statistic | Wall-clock time |
|----------------------|:----------------|
| p25 | 18.1 min (1,084 sec) |
| p50 | 28.3 min (1,698 sec) |
| p75 | 54.1 min (approximately 3,246 sec) |
| p99 | 4.53 h (16,290 sec) |
| Maximum | 4.91 h (approximately 17,676 sec) |
| Mean | 56.3 min (approximately 3,378 sec) |

---

## 6. Limitations

The evaluation excluded local target dynamic analysis. With local fuzzing and debugging disabled, candidate construction relied on source and file inspection together with controlled vulnerable-side execution feedback. The combined system solved most tasks under this configuration, although the interface may be inadequate when reproduction depends heavily on runtime control flow or memory layout. The aggregate results do not identify which failures, if any, fall into that category.

The available efficiency telemetry consists of aggregate per-task means, the LLM-request range, and selected wall-clock percentiles. Token-use ranges, medians, and percentiles are unavailable, and no wall-clock minimum was reported.

All model-backed roles use the same base model, and the evaluation includes no component-level ablations. Accordingly, the results do not establish statistical independence among verification agents or quantify the individual contribution of localization, multiple-hypothesis verification, evidence comparison, review, recovery, or shared records. The reported win rate applies to the combined system.

---

## 7. Conclusion

`Fangcun Agent` solved 1,379 of 1,507 CyberGym Level 1 tasks, yielding a 91.5% strict win rate under independent server-side differential verification. This result characterizes the complete evaluated configuration; the contribution of individual components was not measured.

---

## References

1. **Wang et al.** "CyberGym: Evaluating AI Agents' Real-World Cybersecurity Capabilities at Scale." ICLR 2026. [CyberGym Website](https://www.cybergym.io/cybergym/)
2. **CyberGym Submission Guidelines.** Version 2026-08-04. [GitHub](https://github.com/sunblaze-ucb/cybergym/blob/main/SUBMISSION.md)
3. **CyberGym FAQ.** [GitHub](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md)

---

## Appendix A: Submission Report (YAML)

```yaml
agent_name: Fangcun Agent
success_rate: 0.915
link: https://github.com/Fangcun-AI/CyberGym_report
category: agent
models:
  - name: deepseek-v4-flash
    input_tokens: 92398505
    cache_read_tokens: 92232172
    cache_creation_tokens: 166334
    output_tokens: 279349
    est_usd_cost: 13.30
    time_cost_sec: 3378.1
    llm_requests: 354.6
```

## Appendix B: Network and Dynamic Environment Declaration

- **Network access**: Arbitrary task-side internet access is disabled. Model traffic routes through controlled infrastructure, and the only task-side network target is the local CyberGym submission service.

- **Dynamic environment**: The agent operates against sanitized vulnerable Docker images in benchmark mode. Local fuzzing and local debugging are both disabled. The agent cannot execute the vulnerable binary locally and receives only vulnerable-side execution feedback during the task; the patch and fixed-side behavior remain unavailable until the benchmark's independent final validation.

  
