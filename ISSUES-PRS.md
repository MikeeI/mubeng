# Issue and Pull Request Publication Status

This file is the sole source of truth for every finding's ID, delivery mode, lifecycle status, evidence, and location.
Read and update this ledger instead of inferring state from chat history, clone reports, or earlier reviews.
`FORMAT.md` owns research, drafting, implementation authorization, approval, and publication rules.

Next finding ID: ISSUE-2026-020


## Findings

### ISSUE-2026-001 — checker: isolate mutable state per proxy check

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; user impact not measured.
- Internal priority: High.
- Confidence: High.
- Type: Structure and resource lifecycle.
- Publication target: Undecided.
- Summary: Parallel checker workers mutate one retry client and one package-global `IPInfo`.
- Evidence: `internal/checker/checker.go:23-32,82-128` and `internal/checker/vars.go:3-6`.
- Shared change pressure: Retry state, result state, transport, and cleanup change with one proxy-check execution.
- Impact: Source proves cross-worker state can be mixed; occurrence frequency and affected output are not measured.
- Proposed direction: Let each `check` own its retry client, local `IPInfo`, transport, body, and cleanup.
- Risks and boundaries: Preserve retry count, timeout, country filtering, output, and per-check transport cleanup.
- Verification: Run two distinguishable proxy responses concurrently under `-race` and exercise read/decode failures.
- Missing publication evidence: Current `upstream/master`, focused reproduction, and upstream prior-art research.

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

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; endpoint behavior not observed.
- Internal priority: High.
- Confidence: High.
- Type: Validation.
- Publication target: Undecided.
- Summary: A decoded response can reach `LIVE` output without a 2xx status or nonempty `IPInfo.IP`.
- Evidence: `internal/checker/checker.go:108-128` and `internal/checker/ipinfo.go:5-15`.
- Shared change pressure: HTTP status, decoded identity, filtering, and LIVE classification form one success decision.
- Impact: Source proves error-shaped JSON can satisfy the current success path; endpoint occurrence is not measured.
- Proposed direction: Require a final 2xx response and nonempty `ip` before filtering or output.
- Risks and boundaries: Keep optional IPInfo fields optional and retain existing retry and country-filter behavior.
- Verification: Replay 4xx JSON, 2xx without `ip`, and a complete 2xx response; only the last may be LIVE.
- Missing publication evidence: Current `upstream/master`, endpoint-independent reproduction, and prior-art research.

### ISSUE-2026-005 — server: stream unchanged upstream response bodies

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; performance not measured.
- Internal priority: High.
- Confidence: Medium.
- Type: Algorithm and resource lifecycle.
- Publication target: Undecided.
- Summary: The response path buffers every body even though it only removes hop-by-hop headers.
- Evidence: `internal/server/handler.go:93-103,154-160` and the pinned `goproxy v1.7.2` contract.
- Shared change pressure: Response-body ownership and downstream copying change with the one passthrough path.
- Impact: Memory scales with payload size and first-byte delivery waits for EOF; representative cost is not measured.
- Proposed direction: Return the original body to `goproxy` and remove full buffering and premature close ownership.
- Risks and boundaries: Preserve header sanitation, content length, body closure, and future transformation boundaries.
- Verification: Proxy a slow chunked response and a large body; confirm first-chunk delivery before EOF.
- Missing publication evidence: Current `upstream/master`, pinned dependency verification, benchmark, and prior art.

### ISSUE-2026-006 — daemon: preserve all server options in service arguments

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; runtime divergence not reproduced.
- Internal priority: High.
- Confidence: High.
- Type: Mapping.
- Publication target: Undecided.
- Summary: Service arguments omit five parsed options that control server error and retry behavior.
- Evidence: `internal/daemon/daemon.go:13-42` and `internal/server/handler.go:56-88,190-280`.
- Shared change pressure: Direct and service starts must map the same server-owned options into one runtime contract.
- Impact: Source proves daemon defaults replace explicit CLI values; an installed service was not inspected.
- Proposed direction: Forward both enabled booleans and all three numeric limits through `cfg.Arguments`.
- Risks and boundaries: Exclude checker and recursive daemon flags and preserve exact parser names and values.
- Verification: Compare rendered service arguments and parsed options with an equivalent direct invocation.
- Missing publication evidence: Current `upstream/master`, service reproduction, platform scope, and prior-art research.

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

### ISSUE-2026-008 — server: serialize AWS gateway cache creation and shutdown

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; AWS concurrency not reproduced.
- Internal priority: High.
- Confidence: High.
- Type: Structure and resource lifecycle.
- Publication target: Undecided.
- Summary: Gateway cache reads are unlocked, writes are locked, and shutdown has two ownership paths.
- Evidence: `internal/server/handler.go:208-256`, `proxy.go:15-34`, and `utils.go:12-24`.
- Shared change pressure: Gateway lookup, creation, cache insertion, and close preserve one resource-ownership invariant.
- Impact: Source proves a concurrent map access surface and duplicate creation window; AWS impact is not measured.
- Proposed direction: Use one creation mutex, stop HTTP handling first, snapshot the cache, and close once.
- Risks and boundaries: Avoid half-created entries, duplicate `Start`, lock-held close I/O, and shutdown races.
- Verification: Run concurrent same-key creation and shutdown; require one start, one cached value, and one close.
- Missing publication evidence: Current `upstream/master`, controlled AWS reproduction, lifecycle review, and prior art.

### ISSUE-2026-009 — runner: return success status for successful information commands

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; CLI status not reproduced.
- Internal priority: High.
- Confidence: High.
- Type: Output and error mapping.
- Publication target: Undecided.
- Summary: Version, successful update, and already-current paths terminate with `os.Exit(1)`.
- Evidence: `internal/runner/info.go:21-29`, `internal/updater/updater.go:49-69`, and `options.go:78-87`.
- Shared change pressure: Human success output and process exit status express one terminal CLI outcome.
- Impact: Source proves success is mapped to failure for shell callers; affected automation is not measured.
- Proposed direction: End successful terminal paths with status 0 and preserve nonzero cancellation and errors.
- Risks and boundaries: Do not fall through from update success into proxy-file or action validation.
- Verification: Check exit status for version, successful update, already current, cancellation, and update failure.
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

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; CLI panic not reproduced.
- Internal priority: Medium.
- Confidence: High.
- Type: Validation.
- Publication target: Undecided.
- Summary: The raw `--goroutine` value reaches a pool API that panics below one.
- Evidence: `internal/runner/options.go:65-66`, `validator.go:15-88`, and `checker.go:22-23`.
- Shared change pressure: CLI worker validation and pool construction preserve one concurrency lower-bound invariant.
- Impact: Source and the pinned `conc v0.3.0` contract prove a panic path; occurrence is not measured.
- Proposed direction: Reject `Goroutine < 1` in runner validation and keep one as the serial mode.
- Risks and boundaries: Do not invent an unmeasured upper cap or alter valid concurrency values.
- Verification: Check `--goroutine=-1`, `0`, `1`, and `50`; invalid values must return errors without panic.
- Missing publication evidence: Current `upstream/master`, direct CLI reproduction, dependency contract, and prior art.

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

### ISSUE-2026-014 — runner: reject stdin with server watch mode

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; CLI behavior not reproduced.
- Internal priority: Medium.
- Confidence: High.
- Type: Validation and lifecycle.
- Publication target: Undecided.
- Summary: Validation removes the stdin temp file before server watch setup can observe it.
- Evidence: `internal/runner/validator.go:19-49` and `internal/proxymanager/utils.go:86-97`.
- Shared change pressure: Input-source lifetime and watch compatibility form one runner-owned CLI invariant.
- Impact: Source proves stdin plus address-mode watch cannot retain an observable file; frequency is not measured.
- Proposed direction: Reject stdin with server `--watch` before temporary-file creation.
- Risks and boundaries: Preserve checker stdin and server watch behavior for real files.
- Verification: Compare piped and file-backed address-mode watch, plus piped checker mode.
- Missing publication evidence: Current `upstream/master`, direct CLI reproduction, and prior-art research.

### ISSUE-2026-015 — checker: report failure stages before country filtering

- Status: Hold.
- Delivery mode: Undecided.
- Location: Not published.
- Evidence class: Source-proven; diagnostic demand not measured.
- Internal priority: Medium.
- Confidence: High.
- Type: Error and output mapping.
- Publication target: Undecided.
- Summary: The checker discards errors and applies `--only-cc` before its verbose DIED branch.
- Evidence: `internal/checker/checker.go:31-40,82-128`.
- Shared change pressure: Failure classification, redaction, country filtering, and verbose output form one diagnostic path.
- Impact: Source proves verbose errors remain hidden with country filters; user debugging cost is not measured.
- Proposed direction: Handle errors first and emit stable redacted stages such as transport, request, read, or decode.
- Risks and boundaries: Do not expose proxy credentials, add normal-mode noise, or alter successful country filtering.
- Verification: Exercise each failure stage with verbose on and off, both with and without `--only-cc`.
- Missing publication evidence: Current `upstream/master`, focused diagnostic reproduction, and prior-art research.

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
