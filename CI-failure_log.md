# CI Failure Diagnosis Laboratory — Log Manifest

This document records the baseline empirical failure cases across stages of the CI pipeline lifecycle to train and validate diagnostic agent beliefs.

---

## Case 001: Deterministic Test Defect

* **Failure:** CI pipeline failed during the test execution stage.
* **Observed Evidence:**
  * pytest executed successfully
  * `test_add` failed
  * Exception: `AssertionError`
  * Expected value: `5`
  * Actual value: `4`
  * Failing test: `test_app.py::test_add`
  * Exit code: `1`
* **Hidden State:** $s_1$ — Application / deterministic test defect
* **Root Cause:** Incorrect expected value assertion in test suite: `assert add(2, 2) == 5`.
* **Diagnostic Action:** Inspect pytest failure traceback; compare expected value against actual returned value.
* **Fix:** Correct test assertion expectation from `5` to `4`.
* **Result:** Test passed; CI completed successfully.
* **Agent Learning:** Test-stage failure with clear deterministic assertion mismatch is direct evidence for application/test logic defects.

---

## Case 002: Pipeline Configuration Defect

* **Failure:** CI pipeline failed during the workflow validation/setup stage.
* **Observed Evidence:**
  * Workflow engine failed prior to execution or at step evaluation
  * Invalid property or unsupported runner configuration
  * pytest did not execute
  * Failure occurred before dependency installation and testing
* **Hidden State:** $s_2$ — Pipeline configuration defect
* **Root Cause:** Workflow YAML contained an invalid/unsupported property key (e.g., `execute` instead of `run`) in `.github/workflows/ci.yml`.
* **Diagnostic Action:** Check workflow schema validation logs; isolate malformed key/value before analyzing application files.
* **Fix:** Revert invalid key to a valid GitHub Actions schema directive (`run: pytest`).
* **Result:** Workflow configuration parsed successfully and runner jobs initialized.
* **Agent Learning:** Failures occurring prior to job execution or during runner step parsing reject application defects ($s_1$) immediately.

---

## Case 003: Dependency Version Conflict

* **Failure:** CI pipeline failed during the dependency installation stage.
* **Observed Evidence:**
  * Step: `pip install -r requirements.txt` failed
  * Exit code: `1`
  * Resolver token: `ERROR: ResolutionImpossible`
  * Conflict: user requested `requests==2.28.0` and `requests==2.32.0`
  * Pytest never ran
* **Hidden State:** $s_3$ — Upstream dependency drift / dependency defect
* **Root Cause:** Mutually exclusive version pins specified for the same package in `requirements.txt`.
* **Diagnostic Action:** Inspect dependency resolver step output; extract package names from `ResolutionImpossible` block.
* **Fix:** Remove conflicting duplicate requirement pin from `requirements.txt`.
* **Result:** `pip install` solved the dependency graph without errors.
* **Agent Learning:** Package resolution errors during the setup/install stage eliminate $s_1$ and narrow belief strictly to dependency definitions ($s_3$).

---

## Case 004: Missing Runtime Dependency

* **Failure:** CI pipeline failed during the test collection stage.
* **Observed Evidence:**
  * Pytest exited with code `2` (Interrupted during collection)
  * Summary token: `ERROR collecting test_app.py`
  * Traceback import line: `import requests`
  * Exception: `ModuleNotFoundError: No module named 'requests'`
  * `0` test items executed
* **Hidden State:** $s_3$ — Upstream dependency drift / missing dependency
* **Root Cause:** Code imported `requests`, but package was omitted from `requirements.txt` and absent in the runner environment.
* **Diagnostic Action:** Cross-reference pytest exit code `2` (collection failure) with the missing module name extracted from the import traceback.
* **Fix:** Add `requests` to `requirements.txt`.
* **Result:** Test collection succeeded and test cases executed.
* **Agent Learning:** Pytest exit code `2` specifically denotes collection/environment issues, not assertion failures. `ModuleNotFoundError` during collection confirms environment omission over code regression.

---

## Case 005: Runner Memory Exhaustion

* **Failure:** CI pipeline failed during test execution due to memory ceiling breach.
* **Observed Evidence:**
  * Pytest exited with code `1` in `0.05s`
  * Exception: `MemoryError`
  * Location: `memory_test.py::test_runner_memory`
  * Contrast: Caught cleanly via process ceiling (`RLIMIT_AS`) vs. platform watchdog cancellation (`The operation was canceled`)
* **Hidden State:** $s_4$ — Runner resource exhaustion
* **Root Cause:** Uncontrolled heap expansion loop exceeded process address space limit.
* **Diagnostic Action:** Check exception class (`MemoryError`) or system exit signal (`137` / `Killed`); inspect process-level resource constraints.
* **Fix:** Constrain buffer allocations or terminate unbounded loops.
* **Result:** Memory consumption remained within allocated process limits.
* **Agent Learning:** Resource exhaustion manifests as either caught runtime allocations (`MemoryError`, exit code 1) or OS-level signals (`SIGKILL`, exit code 137). Neither should be diagnosed as assertion logic defects.

---

## Case 006: Missing Secret / Authentication Failure

* **Failure:** CI pipeline failed during the pre-test authentication/validation step.
* **Observed Evidence:**
  * Shell validation step exited with code `1`
  * Environment dump revealed empty variable: `API_TOKEN: `
  * Explicit log error: `FATAL: Secret INTERNAL_DEPLOY_KEY is not set or empty`
  * Pipeline halted before test suite execution
* **Hidden State:** $s_5$ — Authentication / access failure
* **Root Cause:** Workflow referenced a repository secret (`INTERNAL_DEPLOY_KEY`) that was not provisioned in repository settings.
* **Diagnostic Action:** Inspect environment variable mapping in step definition; verify repository secret configuration.
* **Fix:** Add missing secret to repository settings or mock token in CI configuration.
* **Result:** Environment variable received token value and step succeeded.
* **Agent Learning:** Empty environment variable assignments paired with pre-execution validation checks indicate access/configuration omissions, not code logic errors.

---

## Case 007: Flaky / Non-Deterministic Failure

* **Failure:** CI pipeline failed intermittently during test execution across identical commits.
* **Observed Evidence:**
  * Failing test: `test_app.py::test_flaky_timing`
  * Exception: `AssertionError: Transient network timing blip`
  * Condition: `assert (1789221799 % 2) == 0` via `time.time()`
  * Subsequent execution on the identical commit SHA passed
* **Hidden State:** $s_6$ — Flaky / non-deterministic failure
* **Root Cause:** Test relied on non-deterministic system clock state instead of deterministic logic or isolated fixtures.
* **Diagnostic Action:** Inspect git diff; if no logic changes exist, rerun pipeline on the exact commit SHA to detect state volatility.
* **Fix:** Refactor test to mock time dependencies or remove non-deterministic conditional branches.
* **Result:** Tests execute deterministically across consecutive runs.
* **Agent Learning:** Single-run log output of a flaky test is identical to a standard application defect ($s_1$). Verification requires an agent action: re-running on the identical commit SHA.

---

## Case 008: External Outage / Network Timeout

* **Failure:** CI pipeline failed during test execution due to unreachable network endpoint.
* **Observed Evidence:**
  * Step duration increased to `> 2.0s`
  * Traceback origin: `socket.py:create_connection`
  * Exception: `TimeoutError: timed out`
  * Failing test: `test_app.py::test_external_dependency`
* **Hidden State:** $s_7$ — External service / network outage
* **Root Cause:** Test attempted a direct TCP socket connection to an unreachable/unroutable IP (`10.255.255.1`) with a strict timeout.
* **Diagnostic Action:** Extract host/port from socket traceback; verify external endpoint accessibility or mock network boundaries.
* **Fix:** Mock network calls using local test fixtures or stubs.
* **Result:** Suite completed in milliseconds without depending on external network state.
* **Agent Learning:** Elevated execution time ending in `TimeoutError` or connection refused signals external service drift or network constraints, shifting belief away from unit logic defects.

---

## Case 009: Masked Pipeline Failure (False Green)

* **Failure:** CI pipeline reported success despite internal test assertion failures.
* **Observed Evidence:**
  * Job status in CI UI: `Passed (Green)`
  * Step exit code: `0`
  * Raw log content: `FAILED test_app.py::test_masking_demo - assert 1 == 2`
  * Pytest summary: `1 failed in 0.02s`
* **Hidden State:** $s_{N+1}$ / $s_2$ — Masked failure / pipeline configuration defect
* **Root Cause:** Command pipeline `pytest test_app.py | tee test.log` executed without `pipefail`, returning the exit code of `tee` (`0`) and discarding pytest's failure code (`1`).
* **Diagnostic Action:** Cross-reference orchestrator job exit code with test stdout strings (`FAILED`, `ERROR`).
* **Fix:** Configure shell with pipefail enabled (`shell: bash -eo pipefail {0}`).
* **Result:** Step correctly exits with code `1` whenever an assertion fails inside the pipeline.
* **Agent Learning:** The agent cannot treat job status `success` as proof of zero failures; it must inspect raw stdout to rule out swallowed exit codes.
