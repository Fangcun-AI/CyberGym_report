# Fangcun Agent: An Agentic System for Automated Vulnerability Reproduction

**Agent**: `Fangcun Agent`
**Model**: DeepSeek V4 Flash 0731 (`deepseek-v4-flash`)
**Benchmark**: CyberGym Level 1
**Success Rate**: 91.5% (final-submission metric)
**Category**: agent

---

## Abstract

We present `Fangcun Agent`, an agentic system for automated vulnerability reproduction on CyberGym Level 1. The evaluated configuration uses DeepSeek V4 Flash 0731 as its base model within a ReAct agent framework with multi-agent orchestration capabilities. On the full CyberGym Level 1 benchmark comprising 1,507 real-world vulnerability tasks across 188 open-source projects, the system achieves a **91.5% strict win rate** under the final-submission metric — 1,379 of 1,507 tasks were solved with PoCs that trigger a sanitizer crash on the vulnerable build and remain clean on the patched build, as verified by the CyberGym server's differential execution.

The system uses a structured, evidence-driven analysis process with bounded independent hypothesis verification and candidate quality control. Together, these design principles enable the agent to progress from a natural-language vulnerability description to a verified PoC input through iterative investigation.

---

## 1. Introduction

### 1.1 Benchmark Overview

CyberGym is a large-scale, execution-based benchmark for evaluating AI agents' real-world cybersecurity capabilities [Wang et al., ICLR 2026]. It comprises 1,507 benchmark instances derived from real-world vulnerabilities discovered by OSS-Fuzz across 188 widely-used open-source projects. Every task corresponds to a real vulnerability that was discovered, reported, and patched in production software, ensuring that the evaluation reflects genuine security challenges.

At Level 1, the primary benchmark task, the agent receives a vulnerability description and the pre-patch (vulnerable) codebase. The objective is to produce a proof-of-concept (PoC) input that triggers a sanitizer crash on the pre-patch build but does not crash the post-patch (fixed) build. This requires the agent to bridge the gap from a natural-language description to concrete input bytes — a process demanding source-code comprehension, input-format reasoning, execution-feedback absorption, and iterative refinement.

### 1.2 Task Definition

Each CyberGym Level 1 task provides:

- A text description of the vulnerability (approximate location, bug type, root cause)
- The pre-patch codebase (`repo-vul.tar.gz`)
- A harness or fuzzer entry point
- A `submit.sh` script for PoC submission to the verification server

The agent must output a single final PoC file. During a task, a vulnerable-side flag indicates that a candidate reproduces a real crash, but it does not by itself constitute a strict win. Success is determined only after the benchmark independently performs server-side differential verification of the selected final PoC: it must crash the vulnerable binary (`vul_exit_code != 0`) and remain clean on the patched binary (`fix_exit_code == 0`). This differential requirement provides stronger attribution to the target vulnerability than vulnerable-side reproduction alone.

---

## 2. System Architecture

### 2.1 Overview

`Fangcun Agent` is built on a ReAct (Reasoning + Acting) agent framework with multi-agent orchestration. The system is designed around the principle that vulnerability reproduction is an evidence-driven process: the agent prioritizes concrete actions — submitting, reading, executing — over abstract deliberation, and every action should produce observable evidence that informs the next decision.

At a high level, the system combines evidence from available task materials with systematic source-code investigation, search-space mapping, and targeted PoC construction. This allows inexpensive evidence to inform deeper analysis without prescribing a single fixed investigation order.

During investigation, bounded sub-agents can independently evaluate competing vulnerability hypotheses from different analytical perspectives. Their findings are treated as evidence rather than final verdicts and are consolidated by the main agent to reduce repeated exploration. A crash validation mechanism filters candidate PoCs before final submission, rejecting crash types that are unlikely to pass differential verification. Once a flag is obtained, a review process selects the single best-matching PoC as the final submission. After each solved task, a knowledge record is saved to enable cross-task learning for future tasks.

### 2.2 Base Model

All LLM calls in the reported evaluation use DeepSeek V4 Flash 0731 (`deepseek-v4-flash`).

| Value | Parameter |
|-------|-----------|
| `deepseek-v4-flash` | Model |
| LangChain + LangGraph | Framework |

### 2.3 Key Components

The system comprises several interconnected components, each responsible for a distinct aspect of the vulnerability reproduction pipeline:

- **Main Agent Loop**: A ReAct-based reasoning loop that enforces single-action-per-iteration discipline. The agent is guided by a custom prompt with an action-first decision tree that prioritizes immediate PoC submission over extended reasoning, ensuring that the iteration budget is spent on concrete actions rather than ungrounded deliberation.

- **Multi-Agent Exploration**: During investigation, the main agent can launch bounded sub-agents that independently investigate competing vulnerability hypotheses. Each sub-agent explores a specific analytical perspective and contributes findings to a shared evidence state. This exploration helps prevent premature fixation on a single code path — a common failure mode in vulnerability analysis.

- **Crash Validation**: A pre-submission filtering mechanism that rejects crash types unlikely to pass differential verification. Generic crashes (out-of-memory, stack overflow) typically crash both vulnerable and patched builds; the validation mechanism prefers semantic memory bugs that are more likely to be specifically fixed by the patch, conserving server verification calls and reducing false-positive submissions.

- **Cross-Task Knowledge Transfer**: After each solved task, a knowledge record is saved capturing the vulnerability description, relevant file paths, and workflow summary. At the start of each new task, similar prior records are recalled and injected into the initial context, enabling the agent to leverage structural patterns discovered in related tasks — particularly those within the same project family.

- **Post-Flag Review**: When a flag is obtained, a review process compares all flag-producing candidates against the vulnerability description and selects the single best-matching PoC as the final submission, using crash-type preferences and description-alignment heuristics.

---

## 3. Capability and Evaluation Boundary

The agent's task-scoped capabilities cover PoC submission, source-code and file inspection, evidence and state management, and bounded multi-agent investigation. Task operations are restricted to the task workspace and do not provide arbitrary web access, the vulnerability patch, or access to the fixed binary. Model and benchmark-service traffic use controlled infrastructure channels. This boundary supports evidence-driven investigation while preserving the integrity of the evaluation.

---

## 4. Experimental Setup

### 4.1 Configuration

The full benchmark run used a bounded configuration with a maximum of 2,000 iterations per task and an early-stop policy.

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

The agent operates against sanitized vulnerable Docker images in benchmark mode. With both local fuzzing and local debugging disabled, it cannot execute the vulnerable binary locally, attach a debugger, or run a fuzzer. During a task, candidate execution is mediated by the benchmark interface and the agent receives only attributable vulnerable-side feedback; neither the patch nor fixed-side behavior is exposed to the agent. After the agent selects its final PoC, the benchmark independently performs vulnerable-then-fixed differential validation for result accounting.

### 4.3 Scoring Metric

Results are reported under the **final-submission** metric (CyberGym FAQ Q3): the agent designates exactly one PoC as its final answer, and the task is counted as solved only if that specific PoC passes independent differential verification. A vulnerable-side flag obtained during investigation is candidate evidence, whereas a **strict win** is assigned only after the selected final PoC passes the benchmark's vulnerable-then-fixed evaluation. This is the stricter of the two CyberGym scoring methods and avoids rewarding brute-force submission strategies.

---

## 5. Results

### 5.1 Overall Performance

On the full CyberGym Level 1 benchmark comprising 1,507 real-world vulnerability tasks, the system achieved a **91.5% strict reproduction rate** under the final-submission metric. A strict win requires the agent's final PoC to trigger a sanitizer crash on the pre-patch binary (exit code ≠ 0) AND remain clean on the post-patch binary (exit code = 0), as verified server-side. Of the 1,507 tasks, 1,379 achieved strict wins, while 128 tasks did not obtain a flag and have no reported numeric exit codes.

All 1,379 flagged tasks have a non-zero vulnerable-build exit code and a fixed-build exit code of zero. For the remaining 128 tasks, the source records `Got flag = No` and uses `-` for both exit-code fields; it does not provide a finer-grained failure classification.

| Metric | Value |
|--------|-------|
| Tasks attempted | 1,507 |
| Strict wins | 1,379 |
| Win rate (strict) | 91.5% |
| No flag / exit code unavailable | 128 |
| Scoring metric | Final-submission |

> **Strict win** = PoC triggers sanitizer crash on pre-patch binary (exit ≠ 0) AND does not crash post-patch binary (exit = 0), as verified server-side. Both-crash = not counted as win.

### 5.2 Performance by Task Source

The CyberGym benchmark draws tasks from two sources: the ARVO dataset and OSS-Fuzz. Performance is consistent across both sources, with a modest gap favoring ARVO tasks.

| Source | Attempted | Wins | Win rate |
|--------|-----------|------|----------|
| arvo (all ranges) | 1,368 | 1,254 | 91.7% |
| oss-fuzz | 139 | 125 | 89.9% |
| **Total** | **1,507** | **1,379** | **91.5%** |

Within the ARVO subset, the observed win rate varies across task ID ranges and is lower in the higher-numbered ranges:

| ARVO ID range | Attempted | Wins | Win rate |
|---------------|-----------|------|----------|
| 0 – 999 | 5 | 5 | 100.0% |
| 1,000 – 9,999 | 170 | 164 | 96.5% |
| 10,000 – 29,999 | 428 | 400 | 93.5% |
| 30,000 – 49,999 | 347 | 314 | 90.5% |
| 50,000 – 69,999 | 418 | 371 | 88.8% |

The observed source-level gap is 1.7 percentage points. No causal analysis was performed, so these results do not isolate the effects of dataset construction, harness structure, or project mix.

### 5.3 Exit Code Distribution

The following tables summarize the distribution of `vul_exit_code` and `fix_exit_code` across all 1,507 instances, as required by the CyberGym submission guidelines (2026-08-04 version).

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

The 128 unsolved tasks share the same recorded outcome: `Got flag = No`, `vul check exit code = -`, and `fix check exit code = -`. Because no numeric exit codes or termination reasons are provided for these rows, the data do not support subdividing them into budget-exhaustion, both-crash, or no-trigger categories.

| Observed outcome | Count |
|------------------|:------|
| Strict win (`vul != 0`, `fix = 0`) | 1,379 |
| No flag; exit codes unavailable | 128 |

### 5.5 Cost and Efficiency

The available telemetry provides per-task averages for token usage, LLM requests, wall-clock time, and estimated cost. Each task records an average of 92,398,505 input tokens, reads 92,232,172 tokens from cache, writes 166,334 tokens to cache, and generates 279,349 output tokens. A task makes 354.6 LLM requests on average (range: 50–1,930), has a mean wall-clock time of 56.3 minutes (approximately 3,378 seconds), and has an estimated mean cost of **$13.30**.

| Metric | Value |
|--------|:------|
| Avg reported input tokens per task | 92,398,505 |
| Avg cache-read tokens per task | 92,232,172 |
| Avg cache-creation tokens per task | 166,334 |
| Avg output tokens per task | 279,349 |
| Avg estimated USD cost per task | $13.30 |
| Avg wall-clock time per task | 56.3 min (approximately 3,378 sec) |
| Avg LLM requests per task | 354.6 (range: 50–1,930) |

The wall-clock distribution is strongly right-skewed: the median task completes in 28.3 minutes, while the mean is 56.3 minutes and the 99th percentile reaches 4.53 hours.

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

The most significant limitation is the absence of local target dynamic analysis. With both local fuzzing and local debugging disabled, the agent relies on source and file analysis together with controlled vulnerable-side execution feedback. This is sufficient for most tasks but may be inadequate for vulnerabilities requiring understanding of runtime control flow or memory layout.

The available efficiency telemetry includes aggregate per-task means, the LLM-request range, and selected wall-clock percentiles. Token-usage ranges, medians, and percentile distributions are not available, and the wall-clock minimum is not reported. No component-level ablation was performed, so the results characterize the system as a whole rather than the independent causal contribution of any single mechanism.

---

## 7. Conclusion

`Fangcun Agent` demonstrates that an agentic system evaluated with a single base model can achieve a 91.5% strict reproduction rate on CyberGym Level 1 — solving 1,379 of 1,507 real-world vulnerability tasks with PoCs that pass differential verification. In the evaluated configuration, structured evidence gathering, bounded independent hypothesis verification, and candidate quality control form a practical workflow for reliable vulnerability reproduction at scale.

---

## References

1. **Wang et al.** "CyberGym: Evaluating AI Agents' Real-World Cybersecurity Capabilities at Scale." ICLR 2026. [CyberGym Website](https://www.cybergym.io/cybergym/)
2. **CyberGym Submission Guidelines.** Version 2026-08-04. [GitHub](https://github.com/sunblaze-ucb/cybergym/blob/main/SUBMISSION.md)
3. **CyberGym FAQ.** [GitHub](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md)

---

## Appendix A: Submission Report (YAML)

```yaml
agent_name: Fangcun Agent
success_rate: 91.5%
link: https://github.com/Fangcun-AI/CyberGym_report
category: agent
models:
  - name: deepseek-v4-flash
    input_tokens: 92,398,505
    cache_read_tokens: 92,232,172
    cache_creation_tokens: 166,334
    output_tokens: 279,349
    est_usd_cost: $13.3
    time_cost_sec: 3378.1
    llm_requests: 354.6
```

## Appendix B: Network and Dynamic Environment Declaration

- **Network access**: Arbitrary task-side internet access is disabled. Model traffic routes through controlled infrastructure, and the only task-side network target is the local CyberGym submission service.

- **Dynamic environment**: The agent operates against sanitized vulnerable Docker images in benchmark mode. Local fuzzing and local debugging are both disabled. The agent cannot execute the vulnerable binary locally and receives only vulnerable-side execution feedback during the task; the patch and fixed-side behavior remain unavailable until the benchmark's independent final validation.

  
