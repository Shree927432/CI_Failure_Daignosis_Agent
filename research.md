# Research Notes: CI Diagnosis Agent — Selecting the Next Test for a Software Failure

**Prepared:** 2026-09-11
**Project status:** beginner, pre-design
**Status tags used below:** ✅ verified (source found during research) · 🔎 needs a source · 🧪 needs an empirical test · ⚠️ assumption · ❓ unclear — decide before designing

---

## 1. Technical terms for this problem

Your problem sits at the intersection of several mature research areas. Learn these terms — they are your search vocabulary.

### Core software-testing terms
| Term | What it means |
|---|---|
| **Regression Test Selection (RTS)** | Choosing a subset of the test suite to run after a code change (e.g., only tests covering modified code). |
| **Test Case Prioritization (TCP)** | Reordering the test suite so fault-revealing tests run early. Standard metric: **APFD** (Average Percentage of Faults Detected). |
| **Test suite minimization / reduction** | Permanently dropping redundant tests. |
| **Test acceleration** | Umbrella term for RTS + TCP + parallelization to shorten CI feedback time. |
| **Test Impact Analysis (TIA)** | Industry name (Microsoft, Facebook/Meta tooling) for "which tests does this change affect?" |
| **Flaky test / test flakiness** | A test that non-deterministically passes/fails on identical code. Central confounder for any CI-diagnosis agent. |
| **Fault Localization (FL)** | Automatically pointing at the code location likely responsible for a failure. |
| **Spectrum-Based Fault Localization (SBFL)** | FL from pass/fail coverage spectra; classic formulas: Tarantula, Ochiai. |
| **Build failure triage / CI failure diagnosis** | Classifying why a CI build failed (real regression vs. flaky test vs. infrastructure). |
| **Build bisection / delta debugging** | Binary-searching over commits or code changes to isolate a failure's cause. |
| **Reproduction / rerun-based flakiness detection** | Re-executing a failed test N times; pass-after-fail ⇒ flaky. Google reruns up to ~10×; Microsoft Flakes reruns once by default. |

### AI / decision-making terms (your "decisions under incomplete information" framing)
| Term | What it means |
|---|---|
| **Partially Observable Markov Decision Process (POMDP)** | The formal model of sequential decisions where the true state (e.g., "which code is buggy") is hidden and only observed indirectly. |
| **Belief state** | A probability distribution over possible hidden states; updated as evidence (test results) arrives. |
| **Sequential diagnosis / active diagnosis** | Choosing the *next* observation/test to maximize information — exactly your problem. |
| **Value of Information (VoI) / information gain** | Scoring candidate actions by how much they reduce uncertainty. |
| **Next-best-test selection / test selection for fault localization** | Named variants of sequential diagnosis in the testing literature. |
| **Reinforcement Learning (RL) for TCP** | Learning prioritization policies from histories of test outcomes (RL-based HMM methods exist). |
| **Multi-armed bandit / active learning** | Cheaper frameworks than full RL for "which test gives most information per unit cost". |
| **LLM-based software-engineering agents** | Agents that read logs/code and act in a repo environment; key artifacts: **SWE-bench** (benchmark), **SWE-agent**, **Agentless**, **mini-SWE-agent**. |

---

## 2. Useful search queries

```
regression test selection survey
test case prioritization APFD survey
spectrum-based fault localization survey Tarantula Ochiai
sequential diagnosis fault localization next best test
POMDP software debugging test selection
flaky tests detection machine learning iDFlakies
flaky test classification LLM FlakyLens 2025
LLM agent CI failure diagnosis build log
LLM automated debugging SWE-bench agent
reinforcement learning test case prioritization
test impact analysis continuous integration
delta debugging build bisection git bisect
value of information active testing software
TravisTorrent Travis CI build failures dataset
"build failure" "root cause" machine learning GitHub Actions
```

Dataset/keyword add-ons: `Defects4J`, `Bugs.jar`, `Bears`, `TravisTorrent`, `GitHub Actions logs dataset`, `IDoFT (Illinois Dataset of Flaky Tests)`.

---

## 3 & 4. Relevant Reddit communities and why

*(Communities are long-standing, but verify current activity before investing time.)*

| Community | Why it is relevant |
|---|---|
| **r/softwaretesting** | Practitioners who live the pain your agent solves (flaky tests, triage). Best place to learn real workflows and failure stories. |
| **r/QualityAssurance** | Broader QA perspective; many members run or maintain CI suites and can describe what a "good next test" means to them. |
| **r/devops** | CI/CD pipeline owners; strong on GitHub Actions/GitLab/Jenkins, infra failures, and why builds break. |
| **r/continuousintegration** | Niche but directly on-topic for CI failure stories. Verify activity level. |
| **r/programming** | General audience for sanity-checking whether your framing of the problem matches real developer needs. |
| **r/ExperiencedDevs** | Senior engineers discuss test-suite design and CI cost trade-offs; useful for validation of assumptions. |
| **r/MachineLearning** | For the POMDP/belief-state/RL angle; post design questions and paper pointers. |
| **r/learnmachinelearning** | Beginner-friendly sub for your POMDP/bandit/agent fundamentals. |
| **r/reinforcementlearning** | If you model next-test choice as an RL problem, this is the specialist community. |
| **r/LocalLLaMA** | Very active on building LLM agents cheaply (log parsing, test reruns orchestrated by an LLM); good for implementation tactics. |

---

## 5. Researchers and engineers to follow on X

**Handles verified via X during research:**
- **John Yang — @jyangballin** — Princeton; co-creator of SWE-bench and SWE-agent. ✅
- **Carlos E. Jimenez — @_carlosejimenez** — Princeton; co-creator of SWE-bench. ✅

**People verified as active researchers in this field (paper trail confirmed; X handles to verify before following — search their name + site:x.com):**
- **Jonathan Bell** — Northeastern University; flaky tests (large-scale longitudinal study with Google/Meta data). ✅ researcher
- **Chris Parnin** — North Carolina State University; CI/DevOps, fault localization, developer behavior. ✅ researcher
- **Wing Lam** — George Mason University; flaky-test root-causing at Microsoft scale (CloudBuild), iDFlakies. ✅ researcher
- **August Shi** — UT Austin; flaky tests, test-order dependence, mutation testing. ✅ researcher
- **Darko Marinov** — UIUC; flaky tests, NonDex, IDoFT dataset. ✅ researcher
- **Michael Hilton** — CMU; developer use/misuse of CI, TCP. ✅ researcher
- **Owain Parry** — Swansea; lead author of the ACM TOSEM flaky-test survey (the best single overview). ✅ researcher
- **Atif Memon** — University of Maryland; flaky-test ranking at Google scale. ✅ researcher
- **Saikat Dutta** — Cornell; FlakyLens (showed LLM flaky-test classification was over-estimated — cautionary for your project). ✅ researcher
- **Gregg Rothermel / Mary Jean Harrold lineage** — the classic TCP literature (seminal 1999 paper). ✅ for background reading

---

## 6. Design questions: hidden states, evidence, actions, errors

### Hidden states (what the agent cannot observe directly)
1. Which commit/line actually introduced the fault? (Usually unknown — that's why we're diagnosing.)
2. Is a given failure a *real regression*, a *flaky test*, or an *infrastructure* failure? Can the agent maintain a belief distribution over these three? ❓
3. How many distinct faults does the current failure hide? (One red test ≠ one bug.)
4. Each test's true flakiness rate — hidden, only estimable via reruns.
5. Uncommitted environment state (network, services, test order effects) in the CI worker.

### Evidence (observations the agent gets)
6. Which signals are available per run: pass/fail per test, logs, stack traces, coverage, timing, diff, history of past runs? ❓ (decide your input features)
7. How reliable is each evidence source? (Logs can mislead; flaky reruns give noisy labels.)
8. How many reruns are needed before a "pass" is trustworthy evidence of flakiness rather than luck? (Literature suggests ~5; see survey §5 — verify for your context.) 🔎
9. Does the agent get *coverage spectra*, or only binary pass/fail? This determines whether SBFL-style scoring is available. ❓

### Actions (what the agent can do)
10. Is the action space: (a) pick the next existing test to run, (b) rerun a failed test k times, (c) bisect commits, (d) request extra logging/instrumentation, (e) generate a new test? Each is a different project. ❓
11. Is there an explicit cost per action (CI minutes, queue delay)? If yes → optimize information *per cost*, not information alone. ❓
12. When does the agent stop? (Budget exhausted, belief exceeds threshold, fault localized?) ❓
13. Can the agent take compound actions (rerun + bisect together)?

### Errors (how the agent can be wrong)
14. False alarm cost: agent says "real bug" but it was flaky → wasted developer time.
15. Missed-bug cost: agent says "flaky" but it was a real regression → bug shipped. Asymmetric costs — how will you weight them? ❓
16. Wrong localization: agent ranks an innocent line as top suspect (SBFL is known to struggle). 🧪 test on Defects4J.
17. Goodhart risk: if the agent is judged on "tests that fail get found fast", it may learn to pick flaky-but-often-failing tests. ⚠️
18. Non-stationarity: tests change, new code lands, flake rates drift — does the agent update its beliefs online? ❓

---

## 7. Claims that need a source or a test

| # | Claim | Status | How to verify |
|---|---|---|---|
| 1 | At Google, 41% of test targets that had both passed and failed were flaky; at Microsoft (CloudBuild), 26% of sampled builds had flaky failures. | ✅ sourced (ACM TOSEM flaky-test survey; Lam et al. ISSTA'19) | Cite in report |
| 2 | Flaky tests are the leading cause of false alarms in CI. | ✅ sourced (survey §4) | Cite |
| 3 | Rerunning a failed test ~5× is enough to manifest most flaky tests. | 🔎 | Read Lam et al. "Understanding Reproducibility..." (ISSRE'20); then test on your own CI data |
| 4 | "Selecting tests covering recently changed code finds faults faster" (RTS works). | 🧪 | Compare RTS vs. random ordering on your data; measure faults-found-per-minute |
| 5 | TCP improves APFD over default ordering. | 🔎 | Surveys exist (Yoo & Harman 2012); replicate on one open-source project |
| 6 | SBFL (Tarantula/Ochiai) ranks faulty code better than random. | 🧪 | Benchmark on Defects4J; report Top-1/Top-5 accuracy |
| 7 | An LLM can classify CI failures from logs with useful accuracy. | 🧪 | Small experiment: prompt LLM with N labeled build logs; measure accuracy vs. baseline |
| 8 | LLM flaky-test classification results reported in early papers were over-estimated. | ✅ sourced (FlakyLens, OOPSLA'25) | Read before trusting LLM baselines |
| 9 | Framing next-test choice as VoI/information-gain beats greedy coverage heuristics. | 🧪 | This is essentially your research hypothesis — design an A/B experiment |
| 10 | The agent's decisions are robust to noisy (flaky) labels. | 🧪 | Inject synthetic flakiness at rates 1–25% and measure decision quality decay |

**Assumptions to state explicitly in your write-up (⚠️):**
- A1. Failures are reproducible enough that reruns/bisection are informative. (Often false — flaky tests.)
- A2. There exists ground truth for "which test was the right next test." If you can't label this, evaluation is hard. ❓
- A3. Test outcomes are independent given the fault. (Order-dependent tests violate this.)
- A4. CI logs contain enough signal for diagnosis. (Sometimes stack traces point outside the buggy code entirely.)

---

## 8. What is not clear about the problem (resolve before designing)

1. **What is a "test"?** Unit test in a suite? Entire CI job? Which framework/language/repo scale? ❓
2. **What is the agent's action space** — choose among existing tests, rerun, bisect, generate a test, or all of these? (Different papers, different projects.) ❓
3. **Failure types in scope:** test failures only, or also compile/build failures and infra failures? ❓
4. **Available inputs:** logs only? diff? coverage? test history? Can you instrument the CI system? ❓
5. **Objective/metric:** minimize developer time-to-diagnosis? maximize fault-detection rate (APFD)? localize the faulty line (Top-k)? minimize CI cost? You cannot optimize all at once. ❓
6. **Ground truth:** how will you know the agent's choice was "right"? (Linked fix commit? Developer labels? Defects4J-style benchmark?) ❓
7. **Budget model:** is each test run free or costly? Hard wall-clock limit per episode? ❓
8. **Environment:** simulated (replay historical builds) or live (interact with a real repo's CI)? Live interaction needs infrastructure and safety rails. ❓
9. **Baseline comparisons:** what does the agent need to beat — random order, coverage-based TCP, rerun-until-stable, or a human triager? ❓
10. **Flakiness handling:** is the agent supposed to *detect* flakiness as part of diagnosis, or is flakiness out of scope? (Strongly recommend in scope — it's the dominant noise source.) ❓
11. **Agent architecture:** LLM-driven planner? Bayesian belief updater? RL policy? Hybrid? Choose after the metric is fixed. ❓
12. **Scope of "beginner":** a good first milestone is a *replay environment* over public CI data (TravisTorrent/GitHub Actions logs) with a simple information-gain policy — decide if that's the target. ❓

---

## Key verified sources (from this research session)

1. Parry, Kapfhammer, Hilton, McMinn — *A Survey of Flaky Tests*, ACM TOSEM 2021/2025 (dl.acm.org/doi/10.1145/3476105). The single best overview; all prevalence statistics above come from here.
2. Lam et al. — *Root Causing Flaky Tests in a Large-Scale Industrial Setting*, ISSTA 2019 (Microsoft CloudBuild study).
3. Lam et al. — *A Large-Scale Longitudinal Study of Flaky Tests*, PACMPL OOPSLA 2020.
4. Rahman, Dutta, Shi — *Understanding and Improving Flaky Test Classification* (FlakyLens), OOPSLA 2025.
5. Jimenez et al. — *SWE-bench*, ICLR 2024; swebench.com leaderboards; SWE-agent & mini-SWE-agent repos (Princeton/Stanford).
6. Yoo & Harman — *Regression testing minimization, selection and prioritization: a survey* (2012) — classic TCP/RTS survey.
7. Elbaum et al. — *Test Case Prioritization: An Empirical Study* (1999) — origin of APFD.
