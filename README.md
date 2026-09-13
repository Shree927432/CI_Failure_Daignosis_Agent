# Bayesian Information-Gain Agent for CI Failure Diagnosis

A probabilistic, sequential diagnostic AI agent for identifying the root cause of Continuous Integration (CI) pipeline failures.

**Problem Statement:**
> The agent observes raw CI failure logs and telemetry. It must select an active diagnostic probe (e.g., reading files, pinging endpoints, rerunning jobs) because the primary hidden root cause of the failure is not known.

---

## 🏗️ Architecture & Core Loop

Unlike standard static classifiers, this agent treats CI failure diagnosis as a sequential decision-making problem under uncertainty. The agent explicitly separates evidence structuring from probabilistic reasoning.

1. **Passive Baseline ($E_0$):** An LLM structurer parses raw unstructured CI terminal text into a deterministic JSON evidence vector.
2. **Belief Engine:** Maintains a probability distribution $P(s_i \mid E)$ over mutually exclusive primary root causes.
3. **Diagnostic Policy:** Evaluates the Expected Information Gain (EIG) vs. compute cost for available active actions.
4. **Active Probing:** Executes the highest-scoring diagnostic action to gather new evidence.
5. **Belief Update:** Updates the posterior probability using Bayes' theorem.
6. **Resolution:** Loops until a state-specific confidence threshold $p_i^*$ is reached, or escalates to human review.

---

## 🔍 Hidden State Space

To maintain mathematical mutual exclusivity, failures are classified by their **Primary Root Cause requiring human intervention**.

| State | Description | Resolution Intervention |
| :--- | :--- | :--- |
| **$s_1$** | **Code / Logic Defect:** Assertion mismatch, strict static typing errors, linting violations. | Developer code fix |
| **$s_2$** | **Pipeline Config:** Invalid YAML, deprecated actions, matrix incompatibilities. | Devops/Config fix |
| **$s_3$** | **Dependency Drift:** Missing imports, resolver conflicts, unpinned versions. | Lockfile update |
| **$s_4$** | **Resource Exhaustion:** OOM killer (137), memory leaks, runner timeouts. | Provisioning / Opt |
| **$s_5$** | **Authentication:** Missing secrets, invalid tokens, unauthorized registries. | Secret injection |
| **$s_6$** | **Flaky Test:** Time/concurrency non-determinism. | Retry mechanism |
| **$s_7$** | **Network Outage:** Upstream proxy 503, unroutable external endpoints. | SRE Intervention |

---

## 🛠️ Diagnostic Action Space ($\mathcal{A}$)

The agent selects the optimal action $a^*$ by maximizing Expected Information Gain per unit of Execution Cost: 
$$a^* = \arg\max_{a \in \mathcal{A}} \frac{EIG(a)}{Cost(a)}$$

| Action | Execution Target | Expected Cost ($C$) | Primary States Targeted |
| :--- | :--- | :--- | :--- |
| `read_file_snippet(path)` | Local Git tree | **1 (Low)** | $s_1$, $s_2$ |
| `get_git_diff(sha)` | Git history | **1 (Low)** | $s_1$, $s_2$, $s_3$ |
| `ping_endpoint(url)` | External network | **2 (Low)** | $s_7$ |
| `check_runner_metrics()` | CI Telemetry API | **5 (Medium)** | $s_4$ |
| `resolve_dependencies()` | Package Manager | **15 (High)** | $s_3$ |
| `rerun_job(sha)` | CI Pipeline API | **100+ (Critical)** | $s_6$, $s_7$ |

---

## ⚖️ Decision Policy & Thresholds

Instead of a rigid, global 90% confidence cutoff, this agent employs **Risk-Aware State-Specific Thresholds**. The threshold to declare a Flaky Test ($s_6$) is strictly higher than declaring a Network Outage ($s_7$) due to the severe compute waste (cost of false positive) associated with automated job reruns.

$$p_i^* = \frac{CFP_i}{CFP_i + CFN_i}$$

If actions are exhausted before reaching $p_i^*$, the agent triggers a **Human Escalation** function rather than forcing an uncertain prediction.

---

## 🧪 Evaluation Methodology

The agent's policy is evaluated against a static Regex Rule-Engine baseline using 50 held-out cases:
* **9 Synthetic Controlled Sandbox Cases:** Demonstrating isolated occurrences of $s_1$ through $s_7$.
* **41 Production Benchmark Cases:** Clean, un-truncated failure tails mined from open-source repositories (e.g., `uvicorn`, `axolotl`).

**Key Metrics Tracked:** Macro Precision/Recall, Average Diagnostic Cost (Compute), Human-Review Rate, and False-Positive Flaky Assumptions.

---

## 📁 Required Repository Structure

```text
week1/
└── deliverables/
    └── student-project/
        ├── README.md
        ├── research-file.md
        ├── discussion-record.md
        ├── review-record.md
        ├── paper/
        │   ├── main.tex
        │   ├── references.bib
        │   ├── figures/
        │   └── preprint.pdf
        ├── src/
        ├── data/
        ├── experiments/
        ├── results/
        ├── decisions/
        │   └── probability-decision-record.md
        └── social/
            ├── linkedin-post.md
            └── x-thread.md

---

## Architecture

```text
                 ┌─────────────────────┐
                 │      CI Failure     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │  Evidence Collector │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │    Belief Engine    │
                 │     P(H | E)        │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Diagnostic Policy   │
                 │      EIG / Cost     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │  Diagnostic Action  │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Action Outcome    │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Bayesian Posterior  │
                 │       Update        │
                 └──────────┬──────────┘
                            │
                    ┌───────┴────────┐
                    │                │
             Confidence ≥ 90%    Otherwise
                    │                │
                    ▼                ▼
              Report Cause      Select Next
                                Diagnostic
                                  Action

                     If actions are exhausted
                              │
                              ▼
                       Human Escalation
```

The architecture is designed as a feedback loop: diagnostic outcomes become new evidence, which changes the posterior and therefore changes which action should be selected next.

---

## Evidence

The primary empirically validated V1 evidence variable is the **failed pipeline stage**.

Because the original dataset did not contain a standardized stage field, pipeline stages were derived from pipeline step names using high-confidence mappings.

The resulting stages included:

- Build
- Test
- Setup/Dependency
- Quality/Analysis
- Deploy/Publish
- Ambiguous
- Unknown

The broader research architecture also considers evidence such as:

- Pipeline logs
- Test results
- Recent code/test changes
- Dependency changes
- CI/environment configuration changes
- Differences from the last successful run
- Local build results
- Docker image/cache changes
- Previous incidents/runbooks
- Hosting information
- Server/session state

The V1 quantitative model does not have direct historical observations for all of these variables, so the failed pipeline stage serves as the first empirically validated evidence variable.

---

## Bayesian Belief Updating

The agent begins with an empirical prior estimated from the historical labeled dataset.

After the V1 hidden-state mapping, **305 of the original 375 records** were retained for quantitative probability estimation.

### Empirical Prior

| Hidden State | Prior |
|---|---:|
| Code | 17.35% |
| Test | 38.55% |
| Dependency | 29.24% |
| CI/Config | 12.85% |
| Something else | 2.00% |
| **Total** | **100.00%** |

These probabilities describe the included subset of the dataset and are not intended to represent universal CI failure probabilities.

After an action produces an observation, the agent applies Bayes' rule:

$$P(H \mid E,o,a)$$
=
$$
\frac{P(o \mid H,E,a)P(H \mid E)}
{P(o \mid E,a)}
$$

where:

$$
P(o \mid E,a)
=
\sum_h P(o \mid h,E,a)P(h \mid E)
$$

Conceptually:

```text
Current belief
      │
      ▼
Diagnostic outcome
      │
      ▼
Likelihood × Prior
      │
      ▼
Normalize
      │
      ▼
Updated posterior
```

---

## Information Gain

The uncertainty of the current belief distribution is measured using Shannon entropy:

$$
H(H \mid E)
=
-\sum_h P(h \mid E)\log P(h \mid E)
$$

For a particular action outcome, information gain is the reduction in entropy:

$$
IG(o)
=
H(H \mid E)
-
H(H \mid E,o,a)
$$

An outcome is therefore informative when it substantially reduces uncertainty about the root cause.

Before executing an action, the agent does not know which outcome will occur. It therefore calculates **Expected Information Gain (EIG)**:

$$
EIG(a)
=
\sum_o
P(o \mid E,a)IG(o)
$$

with:

$$
P(o \mid E,a)
=
\sum_h
P(o \mid h,E,a)P(h \mid E)
$$

---

## Action Selection

The V1 diagnostic policy combines information gain with diagnostic effort.

For each available action:

$$
Score(a)
=
\frac{EIG(a)}{Cost(a)}
$$

The action with the highest score is selected.

This is important because the most informative action is not necessarily the best action.

For example:

```text
Action A
High information gain
High cost
       │
       └── May be ranked lower

Action B
Moderate information gain
Low cost
       │
       └── May have better EIG/cost
```

The policy therefore attempts to maximize useful information while accounting for diagnostic effort.

After each action:

```text
Select action
      ↓
Observe outcome
      ↓
Update posterior
      ↓
Recalculate EIG/cost
      ↓
Select next action
```

---

## Diagnostic Actions

The agent can consider actions such as:

- Inspect pipeline logs
- Inspect the failed pipeline stage
- Compare with the last successful run
- Inspect recent code/test changes
- Inspect dependency changes
- Inspect CI/environment configuration
- Search previous incidents/runbooks
- Inspect Docker image/cache changes
- Inspect server/session state

The actions are **not executed in a fixed order**. Their ranking depends on the current belief state and expected diagnostic value.

---

## Decision Policy

The V1 system reports a root cause when:

$$
P(H_i \mid E) \geq 0.90
$$

If useful candidate actions are exhausted without reaching this threshold, the case is escalated for human review.

This creates an explicit distinction between:

- **High-confidence diagnosis**
- **Insufficient evidence**
- **Human escalation**

The system is therefore designed as a diagnostic aid rather than an autonomous replacement for engineering judgment.

---

## Evaluation

The original replication dataset contains **375 labeled CI failure jobs**.

The V1 quantitative model uses:

- **305 records** for quantitative probability estimation
- **40 held-out cases** for policy evaluation

The evaluation compares three policies.

### 1. Historical Probability Baseline

Uses historical probabilities to produce a diagnosis without performing sequential diagnostic actions.

### 2. Cost-only Policy

Ranks diagnostic actions according to diagnostic cost and performs lower-cost actions first.

### 3. EIG/cost Policy

Ranks actions using:

$$
\frac{EIG(a)}{Cost(a)}
$$

After each action, its outcome is incorporated into the posterior and the remaining actions are re-evaluated.

---

## Results

Results over the 40 held-out evaluation cases:

| Metric | EIG/cost | Cost-only | Baseline |
|---|---:|---:|---:|
| **Accuracy** | **55%** | 50% | 25% |
| **Macro Precision** | **65.23%** | 63.06% | 6.25% |
| **Macro Recall** | **55%** | 50% | 25% |
| **Human-review rate** | 35% | 37.5% | 0% |
| **Average diagnostic cost** | **6.46** | 6.93 | 0 |
| **Average actions** | **4.68** | 5.23 | 0 |
| **Report precision** | **84.6%** | 80% | 25% |

Compared with the cost-only policy, EIG/cost:

- Improved accuracy by **5 percentage points**
- Improved macro recall by **5 percentage points**
- Improved macro precision by **2.17 percentage points**
- Reduced human-review rate by **2.5 percentage points**
- Reduced average diagnostic cost from **6.93 → 6.46**
- Reduced average actions from **5.23 → 4.68**
- Increased report precision from **80% → 84.6%**

The results suggest that combining expected information gain with diagnostic cost can improve the trade-off between diagnostic effort and root-cause identification under the constructed simulation environment.

---

## Failure Analysis

The evaluation identified several important failure modes.

### 1. Confident but Incorrect Diagnosis

In one case, the agent became confident in **Dependency** even though the true root cause was **CI/Config**.

This highlights the danger of becoming highly confident from imperfect evidence.

### 2. Misleading Evidence

Several cases showed early evidence shifting the posterior toward an incorrect hidden state.

Later evidence moved the posterior in the correct direction, but the incorrect hypothesis had already accumulated enough probability that the true cause could not recover sufficiently.

### 3. Correct Direction but Insufficient Confidence

One case reached **88.3% probability for CI/Config**, but the 90% stopping threshold prevented the agent from reporting it.

This demonstrates that a fixed threshold can cause the system to escalate even when the posterior is strongly favoring the correct cause.

### 4. Code Recall

Both EIG/cost and cost-only achieved **0% recall for Code** on the evaluation set.

This indicates that the V1 diagnostic outcome models did not adequately distinguish Code failures from the other root causes in this evaluation.

---

## Important Limitation

The most important limitation of V1 is that the original dataset does **not directly contain diagnostic-action outcomes**.

The dataset contains labeled CI failures, but it does not directly record what would have happened after actions such as:

- Inspecting dependencies
- Comparing with a previous successful run
- Inspecting server state
- Inspecting logs

However, EIG requires probabilities of the form:

$$
P(o \mid H,E,a)
$$

Therefore, V1 estimates these probabilities using **action-specific observable proxies derived from historical step and sub-category fields**.

The resulting action outcomes are simulated.

> **The reported results should therefore be interpreted as an evaluation of the proposed decision policy under the constructed simulation environment, not as production-level diagnostic performance.**

---

## Human-in-the-Loop Design

The agent deliberately includes human escalation.

If the agent exhausts useful diagnostic actions without achieving sufficient confidence, it does not force a root-cause prediction.

This is particularly important because:

> High posterior probability does not guarantee that the diagnosis is correct.

The system maintains an audit trail containing information such as:

- Evidence
- Hidden states
- Beliefs
- Actions
- Likelihoods
- Posterior probabilities
- Outcomes
- Decisions
- Version information

This provides a record of the agent's reasoning for an individual case.

---

## Future Work

### Jensen-Shannon Divergence Policy

V1 uses entropy reduction through EIG. A planned extension is to explore **Jensen-Shannon Divergence (JSD)** for action selection.

The motivation is that entropy/EIG measures overall uncertainty reduction, while JSD can measure how strongly alternative posterior distributions differ from one another.

For possible posterior distributions $P_o$:

$$
JSD(P_{o_1},P_{o_2},\ldots)
=
H\left(\sum_o w_oP_o\right)
-
\sum_o w_oH(P_o)
$$

where:

$$
w_o=P(o\mid E,a)
$$

A future policy can investigate:

$$
\frac{JSD(a)}{Cost(a)}
$$

as an alternative or complement to EIG/cost.

The goal is to prioritize actions that better distinguish between competing root-cause hypotheses rather than merely reducing overall uncertainty.

### State-Specific Stopping Thresholds

Instead of using a fixed 90% threshold for every root cause, future work proposes thresholds based on the cost of false positives and false negatives.

For hidden state $H_i$:

$$
p_i^*
=
\frac{CFP_i}{CFP_i+CFN_i}
$$

This allows different root causes to have different decision thresholds based on their domain risk.

### Richer Evidence Processing

A future architecture includes an LLM between the evidence collector and belief engine.

The intended role is to structure messy or unstructured CI evidence into representations that the probabilistic model can process.

The LLM is **not intended to directly determine the root cause**.

Instead:

```text
Raw CI Evidence
       ↓
      LLM
Evidence Structuring
       ↓
Bayesian Belief Engine
       ↓
Probabilistic Diagnosis
       ↓
Diagnostic Policy
```

This keeps evidence interpretation separate from probabilistic diagnosis and action selection.

---

## Research Contributions

The project contributes:

- A CI failure diagnosis formulation where root cause is represented as a hidden state.
- A five-state probabilistic root-cause model.
- A sequential Bayesian belief-update process.
- An information-theoretic diagnostic policy using EIG/cost.
- A comparison against cost-only and historical-probability policies.
- A failure analysis of incorrect, misleading, and low-confidence diagnoses.
- A human-escalation mechanism for cases where evidence is insufficient.
- Future extensions based on Jensen-Shannon Divergence and risk-aware stopping thresholds.

---

## Reproducibility Notes

The V1 evaluation should be interpreted with the following constraints:

- The original dataset contains 375 labeled CI failure jobs.
- 305 records were retained after applying the V1 hidden-state mapping.
- Evaluation was performed on 40 held-out cases.
- Diagnostic action outcomes were simulated because the original dataset does not contain direct action-outcome observations.
- The 90% stopping threshold was selected heuristically.
- The evaluation size is small, so the reported metrics have substantial sampling uncertainty.

These constraints should be considered before interpreting the results as evidence of production performance.

---

## Project Structure

The exact repository structure is implementation-dependent.

A typical implementation may separate:

```text
agent/
├── belief/
├── policy/
├── actions/
├── evidence/
├── evaluation/
└── simulation/
```

Update this section to match the actual repository structure.

---

## Research Paper

This repository accompanies the research work:

**A Bayesian Information-Gain Agent for CI Failure Diagnosis**

Author: **Panduru Grishm Siddharth**  
Independent Researcher

---

## AI Use

AI tools were used during the project for coding and debugging assistance, mathematical explanations, research organization, editing and writing assistance, and data analysis.

The resulting code, analysis, decisions, and written material were reviewed by the author.

---

## Disclaimer

This is a research prototype.

The V1 action-outcome probabilities are simulated rather than directly observed from real diagnostic interactions, and the evaluation uses only 40 held-out cases.

The system should therefore **not be treated as an autonomous authority for production incident decisions without further validation**.

It is intended to support engineers by providing probabilistic reasoning, sequential diagnostic recommendations, and an auditable reasoning trail.

---

## License

Add the repository's chosen license here.
