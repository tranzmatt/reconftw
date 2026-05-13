---
phase: 01-resilient-resume-timeout-safety
verified: 2026-05-13T10:02:41Z
status: gaps_found
score: 2/4 must-haves verified
overrides_applied: 0
gaps:
  - truth: "Interrupting a recon mid-sub_brute and re-running with PRESERVE=true produces a clear .inprogress_sub_brute indicator and re-executes only that function — completed functions retain .<func> checkpoints and are skipped"
    status: partial
    reason: "The indicator half fails for the most common interruption path. cleanup_on_exit (INT/TERM trap) calls `exit 130` at modules/utils.sh:145; per Bash semantics that triggers the EXIT trap, which runs _cleanup_inprogress and wipes every .inprogress_<fn>. Confirmed empirically: SIGINT to a running shell with a pre-seeded .inprogress_sub_brute leaves the directory empty. Result on next run: no resume banner, no .inprogress_<fn> indicator. The skip-and-rerun side still works (function re-runs because .<fn> never written and checkpoint guard re-enters), but the 'clear indicator' the success criterion calls out is missing for Ctrl+C/SIGTERM."
    artifacts:
      - path: "modules/modes.sh"
        issue: "trap '_cleanup_inprogress' EXIT at :122 fires after cleanup_on_exit's explicit exit, defeating SC1's indicator guarantee for SIGINT/SIGTERM"
      - path: "modules/utils.sh"
        issue: "cleanup_on_exit ends in `exit \"$exit_code\"` at :145; an explicit exit from inside an INT/TERM trap still triggers the EXIT trap chain in Bash"
    missing:
      - "Gate _cleanup_inprogress on a clean-exit flag (set true only on normal end-of-run), or drop the EXIT trap entirely and accept stale sentinels from normal exits (idempotent harm: noisy banner at most). Either approach restores SC1's indicator guarantee for Ctrl+C."
      - "Update start_func's own comment at modules/core.sh:1431-1434 ('EXIT trap clears it on graceful exit') so the documented behavior matches the implemented behavior — today the comment is misleading about what 'graceful' covers."
  - truth: "Setting PARALLEL_JOB_TIMEOUT_SECONDS=600 in reconftw.cfg terminates any single parallel_funcs child exceeding 10 minutes via kill -TERM, allowing the batch to continue"
    status: partial
    reason: "Two defects undermine the headline guarantee. (1) Timeout enforcement is gated on OUTPUT_VERBOSITY >= 1 — the timeout-kill check sits inside `if [[ ${PARALLEL_MODE} == true ]] && [[ ${OUTPUT_VERBOSITY:-1} -ge 1 ]] ...` at lib/parallel.sh:469 and lib/parallel.sh:579. Under --quiet (OUTPUT_VERBOSITY=0), the whole while-loop is skipped → no timeout enforcement runs. The cfg comment at reconftw.cfg:314 explicitly advertises `600 for CI` and CI runs are precisely where --quiet is used, so the feature is disabled in its documented primary use case. (2) The kill targets the wrapper subshell PID (`$!` of the `(...) >$log_file &` at lib/parallel.sh:438), not the actual stuck external tool. SIGTERM to a bash subshell does not auto-propagate; orphaned children (puredns, dnsx, ffuf) continue running after the wrapper dies. Confirmed empirically with a wrapper spawning `sleep 30` — wrapper killed, inner process still alive."
    artifacts:
      - path: "lib/parallel.sh"
        issue: "Heartbeat blocks at :469 and :579 gate timeout enforcement on OUTPUT_VERBOSITY >= 1, silently disabling it in --quiet mode (the CI scenario the cfg comment recommends)"
      - path: "lib/parallel.sh"
        issue: "_timeout_kill_job at :53-77 signals the wrapper subshell PID; the actual tool process is not in the same process group and is orphaned to PID 1 when the subshell dies"
    missing:
      - "Decouple timeout enforcement from heartbeat UI gate. Run the boundary while-loop whenever PARALLEL_JOB_TIMEOUT_SECONDS > 0 OR OUTPUT_VERBOSITY >= 1; emit heartbeat snapshots only on the verbosity arm."
      - "Use a process-group kill (`kill -TERM -- -<pgid>`) or a recursive child kill (`pgrep -P` walk) so the actual tool process and any axiom-distributed children are signaled, not just the wrapper. Without this, timeout enforcement updates bookkeeping but does not stop the work."
deferred:
  - truth: "Unit tests cover the new .inprogress_<fn> lifecycle / _check_disk_mid_run / timeout-kill paths"
    addressed_in: "Phase 4"
    evidence: "Phase 4 Goal: 'Test Coverage Reinforcement — parallel_funcs batch behaviour, mocked end-to-end pipeline, axiom failover'. CONTEXT.md §Claude's Discretion: 'Test coverage for the new code paths (.inprogress_* lifecycle, _check_disk_mid_run, timeout kill, DNS hard timeout) is OUT of scope for Phase 1 by design — Phase 4 (TEST-01) explicitly covers parallel_funcs timeout kill path.'"
  - truth: "MIN_DISK_SPACE_GB 2-vs-5 reconciliation between reconftw.cfg:39 (=2) and modules/modes.sh:23 (defaults :-5)"
    addressed_in: "Phase 5"
    evidence: "REQUIREMENTS.md DOCS-02 maps to Phase 5: 'Disk-space defaults align — either reconftw.cfg:39 lifts MIN_DISK_SPACE_GB to 5 to match modules/modes.sh:23, or modes.sh drops the :-5 fallback so the config value is authoritative.' CONTEXT.md D-08: 'Phase 5 DOCS-02 work will resolve the discrepancy; Phase 1 does NOT touch that line and does NOT introduce a separate mid-run knob.'"
human_verification:
  - test: "End-to-end resume on real Ctrl+C interruption"
    expected: "After interrupting an active `./reconftw.sh -d <domain> -r --parallel` mid-sub_brute via Ctrl+C and immediately re-running with `PRESERVE=true`, the WARN resume banner ('resume: N functions re-running after interruption (sub_brute)') should appear and only sub_brute should re-execute. With the current code, the EXIT trap wipes sentinels, so the banner won't fire — but the function will still re-run via the checkpoint guard. Confirm the user-facing impact: missing banner vs functional resume."
    why_human: "Requires a multi-minute real recon run with a real SIGINT — not safely automatable in this verification context."
  - test: "Timeout-kill under --quiet mode (CI scenario)"
    expected: "Setting `PARALLEL_JOB_TIMEOUT_SECONDS=600` in reconftw.cfg and running `./reconftw.sh --quiet -d <domain> -r` should kill any parallel job that exceeds 10 minutes. With the current OUTPUT_VERBOSITY=0 gate, no kill fires. Confirm whether silent-failure is the intended behavior or whether the cfg-documented CI use case is broken."
    why_human: "Requires running a stuck synthetic tool inside a real parallel batch under --quiet for >10 minutes to observe failure mode. Code path confirmed by inspection; behavioral confirmation by human needed."
  - test: "Orphaned child after timeout kill on a real tool (e.g., puredns)"
    expected: "When `_timeout_kill_job` fires on a real `puredns` invocation, the actual `puredns` and `massdns` processes should terminate within `PARALLEL_KILL_GRACE_SECONDS`. With the current implementation, only the wrapper subshell dies; `puredns`/`massdns` remain orphaned to PID 1 and continue burning CPU/network. Confirm operational impact (zombie load) on a real run."
    why_human: "Requires monitoring `ps` while a timeout fires on a long-running tool inside a real parallel batch — not safely automatable in the verifier."
---

# Phase 01: Resilient Resume & Timeout Safety — Verification Report

**Phase Goal:** Long-running scans survive interruptions, disk pressure, and stuck tools without silently producing truncated output or wasting hours of re-work on resume.
**Verified:** 2026-05-13T10:02:41Z
**Status:** gaps_found
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| #   | Truth                                                                                                                                                                                                                              | Status      | Evidence                                                                                                                                                                                                                                                                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Interrupting recon mid-sub_brute and re-running with PRESERVE=true produces a clear .inprogress_sub_brute indicator AND re-executes only that function — completed functions retain .<func> checkpoints and are skipped         | PARTIAL     | The skip-and-rerun half WORKS (checkpoint guard re-enters because .<fn> never written). The "clear .inprogress_sub_brute indicator" half FAILS under SIGINT/SIGTERM: empirical test (`/tmp/sc1_v3.sh`) shows EXIT trap wipes sentinel after `cleanup_on_exit` exits 130. See CR-01. |
| 2   | A scan that exhausts disk space mid-run aborts with a clear ENOSPC error message rather than producing zero-byte or truncated output files                                                                                       | VERIFIED    | `_check_disk_mid_run` at modules/utils.sh:445 (boundary call at start_func:1422 and end_func:1551), `_abort_disk_full` at :451 emits `[FAIL] disk_full: aborting (...)` to stderr and `reason=disk_full` JSONL, then `exit 1`. Empirical test confirms rc=1 with required >> available. |
| 3   | Setting PARALLEL_JOB_TIMEOUT_SECONDS=600 terminates any single parallel_funcs child exceeding 10 minutes via kill -TERM, allowing the batch to continue                                                                          | PARTIAL     | Timeout-kill DOES persist FAIL + reason=timeout when triggered, and the batch DOES continue. But: (a) gated on OUTPUT_VERBOSITY >= 1 → silently disabled under --quiet (the documented CI scenario, reconftw.cfg:314), and (b) signals only the wrapper subshell, not the actual tool process → orphaned children. CR-02, CR-03. |
| 4   | Fresh installs of reconftw.cfg ship with non-zero DNS_BRUTE_TIMEOUT=6h and DNS_RESOLVE_TIMEOUT=4h defaults, and a stuck DNS run aborts with a logged timeout rather than blocking indefinitely                                  | VERIFIED    | reconftw.cfg:394-395 ships DNS_BRUTE_TIMEOUT=6h, DNS_RESOLVE_TIMEOUT=4h. `_dns_timeout_enabled` at modules/utils.sh:1413 treats both as enabled. `_run_dns_with_heartbeat` at :1453 wraps with `$TIMEOUT_CMD -k 10s $timeout_value` when enabled. Sourcing the cfg confirms exact values. |

**Score:** 2/4 truths verified

### Deferred Items

| # | Item                                                                                                                                                            | Addressed In | Evidence                                                                                                                       |
|---|---------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|------------------------------------------------------------------------------------------------------------------------------|
| 1 | Unit tests cover the new .inprogress_<fn> lifecycle / _check_disk_mid_run / timeout-kill paths                                                                  | Phase 4      | TEST-01 covers `parallel_funcs batch behaviour`; CONTEXT.md explicitly defers test coverage for the new paths to Phase 4.    |
| 2 | MIN_DISK_SPACE_GB 2-vs-5 reconciliation between reconftw.cfg:39 and modules/modes.sh:23                                                                         | Phase 5      | DOCS-02 maps to Phase 5; CONTEXT.md D-08 confirms Phase 1 deliberately does NOT modify either line.                          |

### Required Artifacts

| Artifact            | Expected                                                                                                                              | Status     | Details                                                                                                                                                                                          |
|---------------------|----------------------------------------------------------------------------------------------------------------------------------------|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `modules/core.sh`   | start_func touches `.inprogress_<fn>`; end_func removes it before touching `.<fn>`; both boundaries call `_check_disk_mid_run || _abort_disk_full` | VERIFIED   | Verified at lines 1419-1445 (start_func touch at :1436, disk check at :1422); 1447-1554 (end_func rm at :1481, disk check at :1551). Sentinel-lifecycle ordering correct.                       |
| `modules/utils.sh`  | `_cleanup_inprogress` standalone helper (sibling of cleanup_on_exit); `_check_disk_mid_run`/`_abort_disk_full` helpers                | VERIFIED   | `_cleanup_inprogress` at :149 (sibling to `cleanup_on_exit` at :116). `_check_disk_mid_run` at :445, `_abort_disk_full` at :451 placed between `check_disk_space` and `progress_bar`. cleanup_on_exit body UNCHANGED. |
| `modules/modes.sh`  | Separate EXIT-only trap, FORCE_RESCAN orphan clear, resume banner emission                                                            | VERIFIED   | `trap '_cleanup_inprogress' EXIT` at :122 (adjacent to unchanged INT/TERM trap at :121). FORCE_RESCAN orphan clear at :82. Resume banner block at :166-181 guards against unset called_fn_dir and FORCE_RESCAN. |
| `lib/parallel.sh`   | `_timeout_kill_job` helper; timeout check in BOTH heartbeat loops; FAIL reason render widened                                          | VERIFIED   | `_timeout_kill_job` at :53 (canonical `function name()` form). Timeout-kill calls at :485 (batch-flush) and :595 (final-wait). FAIL reason render widened at :296, :314, :330 to include `\"$badge\" == \"FAIL\"`. |
| `reconftw.cfg`      | DNS timeouts non-zero; two new PARALLEL knobs                                                                                          | VERIFIED   | `DNS_BRUTE_TIMEOUT=6h` (:394), `DNS_RESOLVE_TIMEOUT=4h` (:395), `PARALLEL_JOB_TIMEOUT_SECONDS=0` (:318), `PARALLEL_KILL_GRACE_SECONDS=10` (:319). Kill-latency note at :317.                  |

### Key Link Verification

| From                                    | To                                                | Via                                                                       | Status        | Details                                                                                                                                                                                                |
|-----------------------------------------|---------------------------------------------------|---------------------------------------------------------------------------|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| modules/core.sh:start_func              | $called_fn_dir/.inprogress_<fn>                   | `touch` guarded by `[[ -n "${called_fn_dir:-}" ]]`                       | WIRED         | Line 1436. Empirical smoke test (`/tmp/sentinel_smoke2.sh`) confirms file is created.                                                                                                                  |
| modules/core.sh:end_func                | $called_fn_dir/.inprogress_<fn>                   | `rm -f` BEFORE existing `touch .<fn>`                                     | WIRED         | Line 1481 precedes 1483. Empirical smoke confirms sentinel removed and checkpoint created in correct order.                                                                                            |
| modules/modes.sh:start()                | _cleanup_inprogress                               | `trap '_cleanup_inprogress' EXIT` immediately after INT/TERM trap        | WIRED (but defective per CR-01) | Line 122. Empirical test (`/tmp/trap_test.sh`) confirms EXIT trap fires even after `cleanup_on_exit` calls `exit 130`. Sentinels wiped on SIGINT — this is what defeats SC1's indicator.   |
| lib/parallel.sh heartbeat               | _timeout_kill_job                                 | inside `if [[ OUTPUT_VERBOSITY -ge 1 ]] && (( job_dur > _to ))`         | PARTIAL       | Lines 484-486 (batch) and 594-596 (final). Logic correct, BUT wrapping `OUTPUT_VERBOSITY -ge 1` gate at :469 / :579 short-circuits the whole loop under --quiet, disabling timeout enforcement entirely (CR-02). |
| _timeout_kill_job                       | $called_fn_dir/.status_<fn> + .status_reason_<fn> | printf FAIL + printf timeout                                              | WIRED         | Lines 68-71. Empirical smoke (`/tmp/timeout_smoke.sh`) confirms files written with correct content.                                                                                                    |
| _abort_disk_full                        | exit 1                                            | after _print_error + log_json                                             | WIRED         | Lines 452-454. Empirical smoke (`/tmp/disk_smoke.sh`) confirms exit 1 with stderr `[FAIL] disk_full: aborting`.                                                                                       |

### Data-Flow Trace (Level 4)

Not applicable — phase produces infrastructure (helpers, traps, config). No artifacts render dynamic data; data flows are bash control-flow only.

### Behavioral Spot-Checks

| Behavior                                                                                              | Command                                                              | Result                                                                                                                                                | Status      |
|-------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------|
| `bash -n` on all modified files                                                                       | `bash -n core.sh utils.sh modes.sh parallel.sh reconftw.cfg`         | exit 0, "ALL_PARSE_OK"                                                                                                                              | PASS        |
| Source reconftw.cfg → confirm 4 default values                                                        | `bash -c 'source reconftw.cfg && echo ...'`                          | `DNS_BRUTE_TIMEOUT=6h, DNS_RESOLVE_TIMEOUT=4h, PARALLEL_JOB_TIMEOUT_SECONDS=0, PARALLEL_KILL_GRACE_SECONDS=10, MIN_DISK_SPACE_GB=2`                  | PASS        |
| Sentinel lifecycle: start_func touches, end_func removes, checkpoint created                          | `/tmp/sentinel_smoke2.sh` (extracted start_func/end_func)            | "PASS: sentinel created" / "PASS: sentinel removed" / "PASS: checkpoint created"                                                                    | PASS        |
| Disk-full abort path                                                                                  | `/tmp/disk_smoke.sh` with MIN_DISK_SPACE_GB=99999999                 | `Test 1 rc=0` (normal), `Test 2 rc=1` (low disk), `[FAIL] disk_full: aborting (...)` on stderr, outer rc=1                                          | PASS        |
| Timeout-kill kills wrapper, persists FAIL+timeout                                                     | `/tmp/timeout_smoke.sh`                                              | wrapper PID dies, `.status_test_fn=FAIL`, `.status_reason_test_fn=timeout`                                                                          | PASS        |
| Timeout-kill kills inner tool process (CR-03 spot-check)                                              | same script, inner = `sleep 30` after wrapper killed                 | "inner_pid (sleep 30) STILL ALIVE — CR-03 CONFIRMED: orphaned"                                                                                      | FAIL (CR-03)|
| SIGINT path wipes sentinel (CR-01 reproduction)                                                       | `/tmp/sc1_v3.sh` (real INT to subshell with both traps)              | BEFORE: `.inprogress_sub_brute .sub_passive`. AFTER SIGINT: `.sub_passive`. "FAIL: .inprogress_sub_brute deleted by EXIT trap → resume would NOT fire" | FAIL (CR-01)|
| FORCE_RESCAN wipes both checkpoint and sentinel                                                       | `/tmp/force_rescan_smoke.sh`                                         | "PASS: All sentinels and checkpoints wiped"                                                                                                          | PASS        |
| bats unit tests still green                                                                            | `bats tests/unit/`                                                   | 246 pass, 0 fail                                                                                                                                     | PASS        |

### Probe Execution

| Probe | Command | Result | Status |
| ----- | ------- | ------ | ------ |
| N/A   | —       | No probes declared in PLAN/SUMMARY, no `scripts/*/tests/probe-*.sh` convention found | SKIPPED |

### Requirements Coverage

| Requirement | Source Plan      | Description                                                                                                                                | Status      | Evidence                                                                                                                                                                                                  |
|-------------|------------------|------------------------------------------------------------------------------------------------------------------------------------------|-------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| RESIL-01    | 01-01            | User can resume an interrupted recon and skip functions that finished cleanly while re-running functions that were killed mid-execution    | PARTIAL     | The skip-and-rerun mechanism works correctly via checkpoint guards. The advertised `.inprogress_<func>` sentinel surfacing partial state fails on SIGINT/SIGTERM (CR-01). REQUIREMENTS.md wording mentions "surfacing partial state on next invocation" — this surfacing is broken for the most common interruption path. |
| RESIL-02    | 01-02            | A recon run aborts with a clear error rather than producing truncated output when filesystem fills mid-scan                                | SATISFIED   | `_check_disk_mid_run || _abort_disk_full` at start_func:1422 and end_func:1551. Empirical smoke confirms abort with structured + human error.                                                            |
| RESIL-03    | 01-03            | User can bound individual job time inside parallel_funcs via PARALLEL_JOB_TIMEOUT_SECONDS, with stuck jobs terminated by kill -TERM        | PARTIAL     | Knob exists, kill flow exists, FAIL+timeout persistence works. But silently disabled under --quiet (CR-02) AND signals only wrapper subshell (CR-03). RESIL-03 wording "stuck jobs terminated" is functionally true for batch progression but not for the underlying tool. |
| PERF-02     | 01-03            | DNS brute/resolve operations have non-zero default timeouts (DNS_BRUTE_TIMEOUT=6h, DNS_RESOLVE_TIMEOUT=4h)                                | SATISFIED   | reconftw.cfg:394-395 ship with the required defaults; `_run_dns_with_heartbeat` honors them via `timeout/gtimeout -k 10s`.                                                                              |

All 4 requirement IDs declared in PLAN frontmatter are accounted for. No orphaned requirements detected for this phase (cross-checked against REQUIREMENTS.md Traceability table).

### Anti-Patterns Found

| File                | Line | Pattern                                | Severity | Impact                                                                                                                                  |
|---------------------|------|----------------------------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------|
| (no debt markers found in modified files)                                                                                                                                                                                                                                                          |
| modules/core.sh     | 1431-1434 | Misleading comment "EXIT trap clears it on graceful exit" | INFO     | Documentation drift vs implementation: the comment implies SIGINT/SIGTERM preserves sentinels, but they don't (CR-01). Same gap as CR-01 but as a comment-vs-code issue. |

No `TBD`/`FIXME`/`XXX` debt markers in any phase-modified file (modules/core.sh, modules/modes.sh, modules/utils.sh, lib/parallel.sh, reconftw.cfg). The grep matches for `mktemp ...XXXXXX` are template placeholders, not debt markers.

### Human Verification Required

#### 1. End-to-end resume on real Ctrl+C interruption

**Test:** Run `./reconftw.sh -d <safe-domain> -r --parallel` against a target known to take >30 seconds in `sub_brute`. Wait for `sub_brute` to start, then press Ctrl+C. Immediately re-run with `PRESERVE=true` (default).
**Expected:** A WARN banner `resume: 1 functions re-running after interruption (sub_brute)` should appear at the top. With the current code, the EXIT trap wipes the sentinel during the Ctrl+C → `cleanup_on_exit` → `exit 130` chain, so the banner will NOT fire. The function will still re-run via the checkpoint guard (because `.sub_brute` was never written), so the user gets correct resume behavior but no indicator.
**Why human:** Requires a real recon run with real SIGINT — not safely automatable in this verification context.

#### 2. Timeout-kill under --quiet mode (the CI scenario)

**Test:** Set `PARALLEL_JOB_TIMEOUT_SECONDS=600` in reconftw.cfg, run `./reconftw.sh --quiet -d <target> -r --parallel` and seed a synthetic long-running function (e.g., a temp sub_<fn> that sleeps 700 seconds).
**Expected:** The synthetic function should be killed at the 600s mark with FAIL + reason=timeout. With the current `OUTPUT_VERBOSITY >= 1` gate at lib/parallel.sh:469/579, no kill fires. Confirm whether silent feature disablement under --quiet is acceptable or whether CR-02 needs to be fixed before Phase 1 closes.
**Why human:** Behavioral confirmation requires a 10+ minute run; code path failure already confirmed by inspection.

#### 3. Orphaned child after timeout kill on a real tool

**Test:** With `PARALLEL_JOB_TIMEOUT_SECONDS=10` and OUTPUT_VERBOSITY=1, run a recon that triggers `puredns bruteforce` (or any other long external tool) inside a parallel batch. Monitor `ps` after the 10s timeout fires.
**Expected:** Both the wrapper subshell AND the underlying `puredns` (and `massdns`) processes should be terminated. With the current `_timeout_kill_job` signaling only `$!` (the wrapper PID), `puredns`/`massdns` orphan to PID 1 and continue. Confirm operational impact: zombie load, port/socket leaks, axiom remote workers continuing.
**Why human:** Requires monitoring a real parallel batch with `ps` while a timeout fires — not safely automatable here.

### Gaps Summary

Phase 1 ships infrastructure that is mostly correct but two of four success criteria are not fully met as written:

**SC1 (`.inprogress_<fn>` indicator on interruption + re-run only the interrupted function):** The re-run half is functionally correct via the existing checkpoint guard. The indicator half is broken for SIGINT/SIGTERM — the EXIT trap registered at modes.sh:122 runs `_cleanup_inprogress` after `cleanup_on_exit` calls `exit 130`, wiping every `.inprogress_<fn>` on the most common interruption path. The resume banner therefore never fires for Ctrl+C. Either gate the EXIT trap on a clean-exit flag or drop the EXIT trap entirely (stale sentinels are idempotent under the existing checkpoint guard). The plan's CONTEXT D-02 also documents this as "trap on EXIT, INT, TERM clears sentinels" — which is internally inconsistent with SC1, so a clarification of the requirement may also be in scope. The advisory CR-01 in 01-REVIEW.md describes the exact same gap.

**SC3 (PARALLEL_JOB_TIMEOUT_SECONDS=600 terminates child via kill -TERM):** Two compound defects. (a) The enforcement runs only when `OUTPUT_VERBOSITY >= 1` — the cfg comment recommends `600 for CI`, but CI runs use `--quiet` (OUTPUT_VERBOSITY=0) and the feature is silently disabled exactly where documented as primary. (b) The kill targets the wrapper subshell PID, not the underlying tool process; orphan children (puredns, massdns, dnsx, ffuf) continue running past the timeout, defeating the spirit of "terminate the stuck tool." Both confirmed empirically. Advisory CR-02 and CR-03 in 01-REVIEW.md describe these exact gaps.

**SC2 and SC4 are verified:** Disk-full abort path (RESIL-02) and DNS hard-timeout defaults (PERF-02) are fully implemented and behaviorally correct.

The two partial truths represent gaps between the success-criterion wording and the as-implemented behavior — not missing code. They are recoverable with targeted edits (trap gating + kill-tree) but should not be treated as VERIFIED.

---

_Verified: 2026-05-13T10:02:41Z_
_Verifier: Claude (gsd-verifier)_
