# Issue and Pull Request Publication Status

This file is the sole source of truth for every finding's ID, delivery mode, lifecycle status, evidence, and location.
Read and update this ledger instead of inferring state from chat history, clone reports, or earlier reviews.
`FORMAT.md` owns research, drafting, implementation authorization, approval, and publication rules.

Next finding ID: ISSUE-2026-029


## Findings

### ISSUE-2026-001 — checker: isolate mutable state per proxy check

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; race not reproduced; user impact not measured.
- Internal priority: High.
- Confidence: High.
- Type: Structure and synchronization.
- Publication target: Undecided.
- Summary: Parallel checker workers mutate one retry client and one package-global `IPInfo`.
- Evidence: `internal/checker/checker.go:23-32,82-128` and `internal/checker/vars.go:3-6`.
- Shared change pressure: Retry state and decoded result state change with one proxy-check execution.
- Impact: Source proves cross-worker request routing and decoded fields can be mixed; occurrence is not measured.
- Proposed direction: Let each `check` own its retry client and local `IPInfo`.
- Risks and boundaries: Preserve retry count, timeout, request routing, output, and per-check transport ownership.
  Open PR [#300](https://github.com/mubeng/mubeng/pull/300) changes the same block but retains shared state.
- Verification: Run two distinguishable proxy responses concurrently under `-race`, including a malformed payload.
- Missing publication evidence: Current `upstream/master`, focused reproduction, PR #300 coordination, and prior art.

### ISSUE-2026-002 — server: retain the selected proxy across rotation intervals

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; user impact not measured.
- Internal priority: High.
- Confidence: High.
- Type: Algorithm and state ownership.
- Publication target: Undecided.
- Summary: `rotateProxy` returns an empty proxy between thresholds and uses an unsynchronized global counter.
- Evidence: `internal/server/handler.go:23-40,163-180` and `internal/server/vars.go:11-20`.
- Shared change pressure: Current proxy, use count, and rotation threshold form one request-selection state machine.
- Impact: Source proves `--rotate N` with `N > 1` reaches `mubeng.Transport("")`; runtime frequency is not measured.
- Proposed direction: Store current proxy and count on `Proxy`, selecting first and after exactly `N` uses.
- Risks and boundaries: Avoid off-by-one changes, lock inversion, stale removed proxies, and network I/O under the lock.
- Verification: Replay seven requests with two proxies and `--rotate 3`; expect `A,A,A,B,B,B,A` without 502.
- Missing publication evidence: Current `upstream/master`, focused reproduction, and upstream prior-art research.

### ISSUE-2026-003 — proxymanager: own pool state atomically per instance

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; concurrency impact not measured.
- Internal priority: High.
- Confidence: High.
- Type: Structure and synchronization.
- Publication target: Undecided.
- Summary: Selection, removal, count, and reload mutate one global pool without synchronization.
- Evidence: `internal/proxymanager/proxymanager.go:15-63`, `utils.go:12-110`, and `vars.go:5-8`.
- Shared change pressure: Pool contents, cursor, count, and reload commit preserve one selection invariant.
- Impact: Source proves concurrent accesses lack a synchronization owner; runtime failures are not reproduced.
- Proposed direction: Construct independent managers, load candidate snapshots, and guard pool and cursor with one mutex.
- Risks and boundaries: Normalize the cursor after reload and keep file parsing, evaluation, and network I/O outside locks.
- Verification: Exercise concurrent rotate, remove, count, and reload under `-race`, including a rejected reload.
- Missing publication evidence: Current `upstream/master`, focused race reproduction, and upstream prior-art research.

### ISSUE-2026-004 — checker: require verified IP information before LIVE output

- Status: Published.
- Delivery mode: Issue.
- Location: https://github.com/mubeng/mubeng/pull/300#issuecomment-5234387842
- Evidence class: Source-proven on current upstream; dependency behavior verified; endpoint occurrence not observed.
- Internal priority: High.
- Confidence: High.
- Type: Validation.
- Publication target: pull request comment.
- Summary: A decoded response can reach `LIVE` output without a 2xx status or nonempty `IPInfo.IP`.
- Evidence: Current `upstream/master` is `164c0b1860f26164eac996059be6b1b15da7b265`.
  `internal/checker/checker.go:22-65,82-129` checks neither final status nor `IPInfo.IP`.
  `retryablehttp v0.7.8` returns final 4xx and 501 responses without an error.
  `internal/checker/vars.go:3-7` retains one package-global result across checks.
  Open PR [#300](https://github.com/mubeng/mubeng/pull/300) changes the same path but retains these behaviors.
- Shared change pressure: HTTP status, decoded identity, filtering, and LIVE classification form one success decision.
- Impact: Source proves 4xx or 501 JSON and 2xx JSON without `ip` can satisfy the current success path.
  Endpoint frequency and user impact are not measured.
- Proposed direction: First isolate retry clients and decoded results per check as owned by `ISSUE-2026-001`.
  Then require a final 2xx response and nonempty `ip` before filtering or output.
- Risks and boundaries: A standalone nonempty-IP guard is unsound because absent JSON keys retain global stale fields.
  Keep optional IPInfo fields optional and retain retry and country-filter behavior.
- Verification: The source and dependency contracts prove reachability.
  A runtime replay should cover 4xx JSON, 2xx without `ip`, and complete 2xx after state isolation.
- Missing publication evidence: None for the published source-proven comment.
  Runtime replay remains required before claiming observed endpoint-independent behavior or implementing a fix.

### ISSUE-2026-005 — server: stream unchanged upstream response bodies

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; performance and compatibility impact not measured.
- Internal priority: Medium.
- Confidence: Medium.
- Type: Algorithm and resource lifecycle.
- Publication target: Undecided.
- Summary: The response path buffers every body even though it only removes hop-by-hop headers.
- Evidence: `internal/server/handler.go:93-103,154-160`, `go.mod:12`, and
  https://github.com/elazarl/goproxy/blob/v1.7.2/http.go#L34-L36
  plus https://github.com/elazarl/goproxy/blob/v1.7.2/http.go#L92-L95.
- Shared change pressure: Response-body ownership and downstream copying change with the one passthrough path.
- Impact: Memory scales with payload size and first-byte delivery waits for EOF; representative cost is not measured.
- Proposed direction: Return the original body only after accepting goproxy's partial-response semantics on read failure.
- Risks and boundaries: Streaming cannot preserve a pre-header `serverErr` after partial data reaches the client.
  Preserve header sanitation, content length, framework body closure, and future transformation boundaries.
- Verification: Proxy a slow chunked response, a large body, and a response that fails after headers.
- Missing publication evidence: Current `upstream/master`, dependency verification, accepted error contract, benchmark, and prior art.

### ISSUE-2026-006 — daemon: preserve all server options in service arguments

- Status: Published.
- Delivery mode: Pull request.
- Location: https://github.com/mubeng/mubeng/pull/320
- Prerequisite issue: https://github.com/mubeng/mubeng/issues/316
- Evidence class: Source-proven on current upstream; rendered service arguments observed; installed runtime not observed.
- Internal priority: High.
- Confidence: High.
- Type: Mapping.
- Publication target: new pull request.
- Summary: Service arguments omit five parsed options that control server error and retry behavior.
- Evidence: Current `upstream/master` is `164c0b1860f26164eac996059be6b1b15da7b265`.
  `internal/daemon/daemon.go:13-42` omits five values parsed by `internal/runner/options.go:36-37,68-70`.
  `internal/server/handler.go:56-88,190-280` consumes every omitted value.
  Commit [3c343f7](https://github.com/mubeng/mubeng/commit/3c343f7c39c0d076935ef7a8686966593aab023c)
  previously fixed the same manually maintained daemon-argument mapping.
- Shared change pressure: Direct and service starts must map the same server-owned options into one runtime contract.
- Impact: Source proves service children reset both booleans to false and numeric limits to `3`, `10`, and `0`.
  An installed service was not inspected.
- Proposed direction: Append enabled `--rotate-on-error` and `--remove-on-error` flags.
  Always forward `--max-errors`, `--max-redirs`, and `--max-retries`, including zero and negative values.
- Risks and boundaries: Exclude checker and recursive daemon flags and preserve exact parser spellings and values.
  Service quoting, stdin lifetime, SIGTERM handling, and Windows issue #220 are separate lifecycle concerns.
- Verification: `make build` and `make test` passed.
  A debugger stopped before `service.New` and observed all five selected values in the final 22-element argument slice.
- Implementation: Branch `fix/issue-2026-006-daemon-arguments`; commit `45575a0`; pushed to `origin`.
- Missing publication evidence: None; published claims do not assert an installed-service replay.

### ISSUE-2026-007 — proxymanager: keep watch reloads across atomic file replacement

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; filesystem behavior not reproduced.
- Internal priority: High.
- Confidence: Medium.
- Type: IO and lifecycle orchestration.
- Publication target: Undecided.
- Summary: The watcher tracks one file, compares one exact operation value, and does not handle channel closure.
- Evidence: `internal/proxymanager/utils.go:86-110` and `internal/server/utils.go:36-51`.
- Shared change pressure: Event detection, target filtering, reload dispatch, and watcher termination form one lifecycle.
- Impact: Source shows atomic replacement and combined flags can escape handling; actual editor behavior is not measured.
- Proposed direction: After atomic pool ownership, watch the directory, filter the target flags, and exit on closed channels.
- Risks and boundaries: Coalesce event bursts, ignore neighbor files, and retain the last valid pool after rejected reloads.
- Verification: Replay atomic replace, partial write, unrelated file events, repeated saves, and watcher shutdown.
- Missing publication evidence: Current `upstream/master`, Linux/macOS reproduction, fsnotify contract, and prior art.

### ISSUE-2026-008 — server: make AWS gateway initialization transactional

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; concurrency and AWS effects not reproduced.
- Internal priority: High.
- Confidence: High.
- Type: Structure and resource lifecycle.
- Publication target: Undecided.
- Summary: Unsynchronized lookup and incomplete rollback can duplicate, leak, or publish partial AWS gateways.
- Evidence: `internal/server/handler.go:208-256`, `internal/server/utils.go:12-24,54-56`, and `internal/proxygateway/proxygateway.go:122-266,330-336`.
- Shared change pressure: Exact identity, initialization, publication, rollback, and close form one ownership invariant.
- Impact: Source proves map races, duplicate setup, and orphaned partial APIs; AWS frequency and cost are not measured.
- Proposed direction: Use an exact credential-aware key, deduplicate setup, publish only deployed APIs, and roll back owned failures.
- Risks and boundaries: Keep AWS I/O outside locks, never delete reused APIs, preserve errors, and drain requests before close.
- Verification: Exercise same-key concurrency, different credentials, every post-create failure, and shutdown during init.
- Missing publication evidence: Current `upstream/master`, AWS reproduction, lifecycle review, benchmark, and prior art.

### ISSUE-2026-009 — runner: return success status for successful information commands

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Observed for version; source-proven for updater; automation impact not measured.
- Internal priority: Medium.
- Confidence: High.
- Type: Output and error mapping.
- Publication target: Undecided.
- Summary: Version, successful update, and already-current paths terminate with `os.Exit(1)`.
- Evidence: `internal/runner/info.go:21-29`, `internal/updater/updater.go:49-69`, and
  `internal/runner/options.go:78-87`.
- Shared change pressure: Human success output and process exit status express one terminal CLI outcome.
- Impact: The version path reproduced success output with failure status; updater outcomes remain source-proven only.
- Proposed direction: End successful terminal paths with status 0 and preserve nonzero cancellation and errors.
- Risks and boundaries: Do not fall through from update success into proxy-file or action validation.
- Verification: Version status 0 is reproduced; still check successful update, already current, cancellation, and failure.
- Missing publication evidence: Current `upstream/master`, controlled updater reproduction, and prior-art research.

### ISSUE-2026-010 — runner: define nonnegative timeout semantics

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; user intent assumed.
- Internal priority: Medium.
- Confidence: Medium.
- Type: Validation.
- Publication target: Undecided.
- Summary: Negative durations silently share Go's unlimited timeout behavior with zero.
- Evidence: `internal/runner/options.go:30-31`, `validator.go:15-88`, and `http.Client.Timeout`.
- Shared change pressure: CLI validation, help text, and runtime deadline semantics express one timeout contract.
- Impact: Source proves negative values disable the deadline; whether users rely on this is not measured.
- Proposed direction: Reject negative durations, preserve zero as explicit unlimited mode, and document both.
- Risks and boundaries: Preserve the positive 30-second default and intentional unlimited streaming with zero.
- Verification: Check `-1s`, `0`, and a positive value at CLI validation and against a hanging upstream.
- Missing publication evidence: Maintainer intent, current `upstream/master`, compatibility evidence, and prior art.

### ISSUE-2026-011 — runner: reject checker concurrency below one

- Status: Published.
- Delivery mode: Pull request.
- Location: https://github.com/mubeng/mubeng/pull/317
- Prerequisite issue: https://github.com/mubeng/mubeng/issues/313
- Evidence class: Observed CLI panic; source and dependency contracts verified on current upstream.
- Internal priority: Medium.
- Confidence: High.
- Type: Validation.
- Publication target: new pull request.
- Summary: The raw `--goroutine` value reaches a pool API that panics below one.
- Evidence: Current `upstream/master` is `164c0b1860f26164eac996059be6b1b15da7b265`.
  `internal/runner/options.go:65-66` accepts integers and `validator.go:15-88` has no lower-bound check.
  `checker.go:22-23` passes the value to `conc v0.3.0`, whose pool panics below one.
  `go run . -f FILE -c -g=-1` and `-g 0` reproduced that panic; `-g 1` and `-g 50` exited normally.
  Issue [#159](https://github.com/mubeng/mubeng/issues/159) and PR
  [#170](https://github.com/mubeng/mubeng/pull/170) are related historical concurrency work, not duplicates.
- Shared change pressure: CLI worker validation and pool construction preserve one concurrency lower-bound invariant.
- Impact: Invalid checker concurrency reproducibly bypasses normal CLI error handling and emits a Go panic stack.
- Proposed direction: Reject values below one only when dispatch can reach checker mode and document one as serial mode.
- Risks and boundaries: Do not reject the unused flag in address/server mode, invent an upper cap, or alter valid values.
- Verification: Before the fix, `-1` and `0` panicked while `1` and `50` exited normally.
  After the fix, invalid values returned the configured validation error without panic and valid values still exited normally.
  `make build`, `make test`, rendered help inspection, Go formatting, and LSP diagnostics passed.
- Implementation: Branch `fix/issue-2026-011-goroutine`; commit `54535b5`; pushed to `origin`.
- Missing publication evidence: None.

### ISSUE-2026-012 — server: decouple retry backoff from request timeout

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; retry timing not observed.
- Internal priority: Medium.
- Confidence: High.
- Type: Configuration and algorithm.
- Publication target: Undecided.
- Summary: Retry wait minimum and maximum both inherit the request timeout, defaulting to a fixed 30 seconds.
- Evidence: `internal/server/handler.go:270-280` and `go-retryablehttp v0.7.8` defaults.
- Shared change pressure: Request deadline and retry recovery cadence are independent operational policies.
- Impact: Source proves timeout changes also change retry pauses; workload latency is not measured.
- Proposed direction: Remove both overrides and retain the dependency's existing exponential backoff.
- Risks and boundaries: Preserve retry count, logger, `Retry-After`, and explicit request deadlines.
- Verification: Replay 5xx, 5xx, success under two timeout values and observe timeout-independent backoff.
- Missing publication evidence: Current `upstream/master`, timing reproduction, maintainer intent, and prior art.

### ISSUE-2026-013 — server: clarify the max-errors attempt budget

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven divergence; intended contract assumed.
- Internal priority: Medium.
- Confidence: Medium.
- Type: CLI contract and algorithm.
- Publication target: Undecided.
- Summary: Runtime permits `N` rotated attempts after the initial failure while documentation describes total failures.
- Evidence: `internal/server/handler.go:38-91` and `README.md:146-149,190-196`.
- Shared change pressure: Counter logic, remaining-attempt output, help, and README express one attempt-budget contract.
- Impact: Source proves documentation and runtime count different quantities; intended behavior is unresolved.
- Proposed direction: Preserve runtime compatibility and document `N` as additional rotated attempts unless changed explicitly.
- Risks and boundaries: A runtime count change would alter upstream attempts, latency, and existing automation.
- Verification: Replay values `0`, `1`, `3`, and `-1`, then compare results with the selected written contract.
- Missing publication evidence: Maintainer intent, current `upstream/master`, behavior reproduction, and prior art.

### ISSUE-2026-014 — runner: own stdin temporary-file lifetime across actions

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; platform and combined-mode behavior not reproduced.
- Internal priority: Medium.
- Confidence: High.
- Type: Validation and lifecycle.
- Publication target: Undecided.
- Summary: Cleanup is registered late and removes the stdin temp path before daemon or watch consumers can use it.
- Evidence: `internal/runner/validator.go:19-49`, `internal/runner/options.go:89-93`, `internal/runner/runner.go:14-19`, `internal/daemon/daemon.go:15-23,66`, and `internal/proxymanager/utils.go:86-97`.
- Shared change pressure: Temporary-file creation, validation, later path use, and removal form one input-source lifetime.
- Impact: Error paths retain files, while successful POSIX cleanup can remove the path before later consumers open it.
- Proposed direction: Clean normal flows immediately and reject stdin with daemon or watch unless a persistent owner exists.
- Risks and boundaries: Preserve checker stdin and file-backed watch, avoid Windows-only behavior, and keep real files untouched.
- Verification: Cover read and write errors plus piped checker, server, watch, and daemon modes on Linux and Windows.
- Missing publication evidence: Current `upstream/master`, platform reproduction, CLI compatibility intent, and prior art.

### ISSUE-2026-015 — checker: report failure stages before country filtering

- Status: Rejected.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven control flow; intended output precedence unresolved.
- Internal priority: Medium.
- Confidence: High.
- Type: Error and output mapping.
- Publication target: Undecided.
- Summary: Country filtering precedes errors, but no contract proves verbose failures must bypass `--only-cc`.
- Evidence: `internal/checker/checker.go:31-40,82-128` and `README.md:139-154`.
- Shared change pressure: Error output and country filtering express one CLI output-precedence decision.
- Impact: Source proves the ordering, but not user pain; unknown-country failures may be intentionally filtered.
- Proposed direction: None; retain current behavior until maintainer intent establishes the precedence.
- Risks and boundaries: Changing it would emit unqualified `DIED` lines under an explicit country filter.
- Verification: Reconsider only with maintainer intent or observed debugging friction for the combined flags.
- Missing publication evidence: None; rejected because the pain and output-contract gates failed.

### ISSUE-2026-016 — runner: make incomplete CLI errors actionable

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; user friction not measured.
- Internal priority: Medium.
- Confidence: Medium.
- Type: Output.
- Publication target: Undecided.
- Summary: Missing-file and missing-action errors omit the smallest valid next input.
- Evidence: `internal/runner/validator.go:40-42` and `internal/runner/runner.go:20-27`.
- Shared change pressure: Each validation outcome and its corrective hint express one CLI input contract.
- Impact: Source proves the hints are absent; repeated attempts and user cost are not measured.
- Proposed direction: Add `-f FILE` and `--check` or `--address ADDR:PORT` only to the two owned errors.
- Risks and boundaries: Do not print full usage or relabel runtime failures as input errors.
- Verification: Run no-argument and file-without-action invocations and inspect message and nonzero status.
- Missing publication evidence: Current `upstream/master`, CLI reproduction, user-impact evidence, and prior art.

### ISSUE-2026-017 — daemon: describe service replacement in CLI help

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven documentation divergence.
- Internal priority: Medium.
- Confidence: High.
- Type: Output and documentation.
- Publication target: Undecided.
- Summary: Help says daemonize while the implementation replaces a persistent system service.
- Evidence: `common/vars.go:42-46`, `README.md:135-175`, and `internal/daemon/daemon.go:45-66`.
- Shared change pressure: Daemon lifecycle behavior and its CLI description must change together.
- Impact: Source proves the help understates persistent service effects; user surprise is not measured.
- Proposed direction: Describe installation or replacement accurately while retaining the compatible flag name.
- Risks and boundaries: Keep Unix and Windows start behavior distinct and avoid claiming identical service control.
- Verification: Compare `mubeng -h` and README text with both runtime platform branches.
- Missing publication evidence: Current `upstream/master`, contribution guidance, and upstream prior-art research.

### ISSUE-2026-018 — help: explain sync as full request serialization

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven documentation divergence.
- Internal priority: Low.
- Confidence: High.
- Type: Output.
- Publication target: Undecided.
- Summary: The help contains `Syncrounus mode` and does not state that the previous request must complete.
- Evidence: `common/vars.go:55` and `internal/server/handler.go:23-27`.
- Shared change pressure: The sync flag description must track the request-serialization behavior it controls.
- Impact: Source proves the help is misspelled and incomplete; user confusion is not measured.
- Proposed direction: Explain that sync waits for the previous request to complete.
- Risks and boundaries: Do not describe sync as only protecting proxy selection.
- Verification: Inspect `mubeng -h` against the lock scope in `Proxy.onRequest`.
- Missing publication evidence: Current `upstream/master` and upstream prior-art research.

### ISSUE-2026-019 — docs: use GB in ISO country-code examples

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven documentation error.
- Internal priority: Low.
- Confidence: High.
- Type: Documentation and mapping.
- Publication target: Undecided.
- Summary: ISO-3166 examples use `UK` while the checker compares exact normalized country codes.
- Evidence: `README.md:225-233,248-264` and `internal/checker/checker.go:68-80`.
- Shared change pressure: Country-code examples must follow the exact mapping consumed by checker filtering.
- Impact: Source proves copied `UK` does not represent ISO-3166 Alpha-2 `GB`; affected use is not measured.
- Proposed direction: Replace `UK` with `GB` in copyable examples without adding a runtime alias.
- Risks and boundaries: Do not change checker normalization or claim existing configurations are migrated.
- Verification: Inspect all country-code examples against exact checker comparison and ISO-3166 Alpha-2.
- Missing publication evidence: Current `upstream/master` and upstream prior-art research.

### ISSUE-2026-020 — build: include every package in the short test target

- Status: Published.
- Delivery mode: Pull request.
- Location: https://github.com/mubeng/mubeng/pull/319
- Prerequisite issue: https://github.com/mubeng/mubeng/issues/315
- Evidence class: Observed current test-scope omission and successful expanded module scope.
- Internal priority: High.
- Confidence: High.
- Type: Build and verification orchestration.
- Publication target: new pull request.
- Summary: `make test` and `make test-extra` omit the maintained `pkg/helper/awsurl` test package.
- Evidence: Current `upstream/master` is `164c0b1860f26164eac996059be6b1b15da7b265`.
  `Makefile:10-15` runs only `pkg/mubeng` and `pkg/helper`.
  `pkg/helper/awsurl/awsurl_test.go:7-116` owns an omitted 11-case `TestParse`.
  `make test` omitted it, while `go test -short ./...` included all three test packages and passed.
  PR [#261](https://github.com/mubeng/mubeng/pull/261) added the package after the explicit target was created.
- Shared change pressure: The repository test command and every maintained package test express one verification contract.
- Impact: The repository test command and its CI caller reproducibly skip a maintained package test.
- Proposed direction: Make `./...` the module-scoped package owner while preserving compact default output.
  Failure output must remain actionable and full raw output must remain available through verbose mode.
- Risks and boundaries: Current tests are local and deterministic, but future short tests must retain that contract.
  `./...` also compiles packages without tests and runs Go's default vet behavior.
- Verification: Default `make test` printed only `test: ok` and passed all three maintained test packages.
  `make test VERBOSE=1` streamed the complete package scope.
  An injected exit 7 printed the underlying status and raw sentinel output, returned nonzero, and removed its temp file.
- Implementation: Branch `build/issue-2026-020-test-scope`; commit `9c3c2db`; pushed to `origin`.
- Missing publication evidence: None.

### ISSUE-2026-021 — server: route SIGTERM through graceful shutdown

- Status: Published.
- Delivery mode: Pull request.
- Location: https://github.com/mubeng/mubeng/pull/318
- Prerequisite issue: https://github.com/mubeng/mubeng/issues/314
- Evidence class: Observed direct SIGTERM divergence; installed-service path remains source-proven.
- Internal priority: High.
- Confidence: High.
- Type: Process and resource lifecycle.
- Publication target: new pull request.
- Summary: The installed child runs `server.Run` directly, which handles `os.Interrupt` but not `SIGTERM`.
- Evidence: Current `upstream/master` is `164c0b1860f26164eac996059be6b1b15da7b265`.
  `internal/server/server.go:67-75` registers only `os.Interrupt`.
  A supervised server exited on `SIGTERM` without the interrupt log; `SIGINT` logged `Interrupted. Exiting...` and exited cleanly.
  `internal/daemon/daemon.go:15-23,41-66` installs child arguments without `-d`, so the child reaches `server.Run`.
  Merged PR [#227](https://github.com/mubeng/mubeng/pull/227) is related shutdown history but does not handle SIGTERM.
- Shared change pressure: Foreground, service, and container stops must enter the one server and gateway shutdown path.
- Impact: Direct execution proves `SIGTERM` bypasses the registered path while `SIGINT` reaches it.
  Installed-service and container paths are source-proven but were not executed.
- Proposed direction: Register `syscall.SIGTERM` beside `os.Interrupt` without redesigning service callbacks.
- Risks and boundaries: This exposes existing `Stop` map synchronization, remote-close, and ignored-error weaknesses.
  Those separate lifecycle problems are outside this signal-routing scope.
- Verification: After the fix, both supervised `SIGTERM` and `SIGINT` runs logged `Interrupted. Exiting...` and exited zero.
  `make build`, `make test`, Go formatting, and LSP diagnostics passed.
- Implementation: Branch `fix/issue-2026-021-sigterm`; commit `6def82e`; pushed to `origin`.
- Missing publication evidence: None; published claims keep installed-service behavior source-proven.

### ISSUE-2026-022 — build: restore the missing golangci-lint fallback

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Observed recipe expansion; installer execution not verified.
- Internal priority: Medium.
- Confidence: High.
- Type: Build and tool orchestration.
- Publication target: Undecided.
- Summary: Make expands the linter presence check itself, selects the wrong branch, and names an obsolete installer host.
- Evidence: `Makefile:5-6,15,24-31`, `.github/CONTRIBUTING.md:28-36`, and `make -n golangci-lint`.
  Current installer guidance: https://golangci-lint.run/docs/welcome/install/local/
- Shared change pressure: Tool detection, installer source, version selection, and binary path form one lint bootstrap.
- Impact: The rendered recipe contains `[ -x ]`; a clean environment cannot reach a working automatic fallback.
- Proposed direction: Use shell `command -v`, the official installer URL, and a repository-verified version pin.
- Risks and boundaries: Preserve global-tool precedence and avoid an unverified `latest` or module dependency.
- Verification: Remove the linter from `PATH`, run the target, and require the pinned `./bin/golangci-lint`.
- Missing publication evidence: Current `upstream/master`, fallback execution, version compatibility, and prior art.

### ISSUE-2026-023 — server: remove the synchronous request goroutine rendezvous

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven orchestration; performance not measured.
- Internal priority: Low.
- Confidence: High.
- Type: Structure and request orchestration.
- Publication target: Undecided.
- Summary: Each request starts one goroutine and immediately waits on its single unbuffered result channel.
- Evidence: `internal/server/handler.go:33-35,38-122`.
- Shared change pressure: Retry execution, response return, and error mapping form one synchronous request operation.
- Impact: Source proves scheduling, channel, and dynamic type work with no overlap; runtime cost is not measured.
- Proposed direction: Return `(*http.Response, error)` through one private synchronous helper.
- Risks and boundaries: Preserve retries, rotation, removal, response closure, logs, and error mapping exactly.
- Verification: Replay one successful and one failed proxy request before and after the refactor.
- Missing publication evidence: Current `upstream/master`, focused behavior replay, and upstream prior art.

### ISSUE-2026-024 — daemon: return non-Windows service install errors

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven error loss; backend failure not reproduced.
- Internal priority: Low.
- Confidence: High.
- Type: Error mapping and service lifecycle.
- Publication target: Undecided.
- Summary: The non-Windows branch discards install failure, logs startup, and attempts to start the missing service.
- Evidence: `internal/daemon/daemon.go:50-66`.
- Shared change pressure: Install outcome, startup logging, and start eligibility form one service transition.
- Impact: Source proves the original install error is lost; affected backend frequency is not measured.
- Proposed direction: Apply the existing Windows error-return pattern before logging or starting.
- Risks and boundaries: Do not propagate expected first-run stop or uninstall errors.
- Verification: Force a non-Windows install failure and require the original error with no log or start attempt.
- Missing publication evidence: Current `upstream/master`, backend failure reproduction, platform scope, and prior art.

### ISSUE-2026-025 — runner: stream stdin into the temporary proxy file

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; allocation and startup impact not measured.
- Internal priority: Low.
- Confidence: High.
- Type: Algorithm and allocation.
- Publication target: Undecided.
- Summary: stdin is fully materialized in memory before the same bytes are written to a temporary file.
- Evidence: `internal/runner/validator.go:19-33,44-51`.
- Shared change pressure: stdin ingestion, temporary staging, and proxy-list loading form one startup data path.
- Impact: Source proves `O(N)` additional peak heap for an `N`-byte list; realistic list sizes and GC cost are not measured.
- Proposed direction: After `ISSUE-2026-014` resolves cleanup ownership, replace `io.ReadAll` plus `Write` with `io.Copy`.
- Risks and boundaries: This bounds extra heap, not input or disk size, and changes which simultaneous I/O error is returned.
- Verification: Measure peak RSS and startup time across representative sizes and inject read and write failures.
- Missing publication evidence: Current `upstream/master`, measurements, error-contract review, and prior art.

### ISSUE-2026-026 — helper: skip template parsing for literal proxy strings

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; CPU and allocation impact not measured.
- Internal priority: Low.
- Confidence: High.
- Type: Algorithm and allocation.
- Publication target: Undecided.
- Summary: Every literal proxy rotation builds and executes a `text/template` despite containing no template action.
- Evidence: `pkg/helper/eval.go:22-39`, `internal/proxymanager/utils.go:68-83`, and `internal/server/handler.go:38-42,163-180`.
- Shared change pressure: Literal detection and dynamic template execution choose one proxy address per rotation.
- Impact: Source proves parser and allocation work on the default request path; magnitude and literal share are not measured.
- Proposed direction: Return the input immediately when it lacks `{{`; retain the existing path for dynamic templates.
- Risks and boundaries: Preserve fresh random values, invalid-template behavior, and exact output for dynamic inputs.
- Verification: Compare CPU and allocations for literal and dynamic rotations under representative request rates.
- Missing publication evidence: Current `upstream/master`, focused measurements, workload distribution, and prior art.

### ISSUE-2026-027 — server: gateway eviction lacks a safe ownership contract

- Status: Rejected.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven retention; origin churn, ownership, and quota pressure unverified.
- Internal priority: Medium.
- Confidence: High.
- Type: Resource lifecycle and capacity policy.
- Publication target: Undecided.
- Summary: A proposed TTL/LRU gateway reaper cannot safely distinguish owned, idle, or externally reused AWS APIs.
- Evidence: `internal/server/handler.go:225-253`, `internal/server/utils.go:14-20`, and `internal/proxygateway/proxygateway.go:142-167,330-337`.
- Shared change pressure: Registry capacity and remote API ownership would require one explicit server policy.
- Impact: No workload proves material retention; eviction can delete a reused API or one serving an active request.
- Proposed direction: None; measure origin residency and quota pressure and define API ownership before reconsidering.
- Risks and boundaries: A cap, TTL, lease system, or worker changes cold starts, admission, shutdown, and remote ownership.
- Verification: Reconsider only with measured churn, request-lifetime evidence, and an explicit owned-versus-reused contract.
- Missing publication evidence: None; rejected because frequency and behavior-preserving-fix gates failed.

### ISSUE-2026-028 — checker: output-template reuse lacks material cost evidence

- Status: Rejected.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven repetition; parser cost and workload frequency not measured.
- Internal priority: Low.
- Confidence: High.
- Type: Algorithm and allocation.
- Publication target: Undecided.
- Summary: The checker reparses one output template per formatted live result, but material cost is unproven.
- Evidence: `internal/checker/checker.go:31-49` and `internal/checker/format.go:63-82`.
- Shared change pressure: Template parse timing and concurrent rendering would form one checker-run output contract.
- Impact: Repetition is source-proven, but network checks and rendering dominate without contrary benchmark evidence.
- Proposed direction: None; benchmark representative templates and live-result counts before reconsidering lazy reuse.
- Risks and boundaries: Eager parsing changes panic timing and behavior when no result reaches the formatting branch.
- Verification: Reconsider only if focused CPU and allocation measurements show a material parser share.
- Missing publication evidence: None; rejected because the non-trivial-cost gate failed.
