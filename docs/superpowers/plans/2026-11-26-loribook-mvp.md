# LoriBook MVP (Phase 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Linux `lori` CLI that runs `.lori` runbooks (checks first, start from any step, questions that create variables, approvals) in a bubblewrap sandbox with injected credentials, and records every attempt in an append-only WartLog history in `.lori_history`.

**Architecture:** A pure core crate (model, variables, dependency and stale logic, events, run orchestration) with three plug-in traits: `RunStore` (WartLog), `Runner` (sandbox) and `Operator` (the terminal). The CLI is a thin shell. Task 1 is a spike that decides whether WartLog can answer the history queries; the trait keeps the backend swappable.

**Tech Stack:** Rust (edition 2021, toolchain 1.97 is installed), `serde`, `serde_yaml`, `sha2`, `anyhow`, `thiserror`, `clap`, `tempfile`; `wartlog-store`, `wartlog-model` and `wartlog-query` pinned as git dependencies at rev `09a03b5d45e47d680a12bc69d9bdb0efeda1498e` of `https://github.com/tmcqueen/wartlog`; bubblewrap (`bwrap`) at run time.

**Spec:** `docs/superpowers/specs/2026-11-26-loribook-design.md`

## Global Constraints

- Rust only, one codebase, Linux only for the MVP; no Docker dependency.
- A runbook is a YAML file named `<name>.lori`; history lives in a `.lori_history` directory next to it. Run state is never written to the runbook.
- History is append-only: events are only ever appended, never edited or deleted. A stale step is a new event.
- Secrets are never stored: a secret value reaches a step through the environment only, is never pasted into script text or a command line, and captured output is scrubbed before it is appended.
- Dependencies are derived from variable use, never declared. `${foo}` is environment substitution; `${{foo}}` is exact (pasted) substitution; a secret variable allows only `${foo}`. `$${` writes a literal `${` (my call, spec item Open 1).
- A step is one (check, action) pair. Every script step and check-only step has a required `check`.
- A skipped-by-choice step is never shown as passed. Starting a run never approves an approval step.
- The sandbox is default-deny. In the MVP network is all or nothing per step (bubblewrap cannot filter by host); a host allowlist is deferred and must say so wherever network is declared.
- No run state in git; `.lori_history/` is added to `.gitignore`.
- Never commit data from the worked example (a client's names, ids, hostnames). Fixtures are synthetic.
- One active run per store (an advisory lock on the history directory).

## Review Focus

1. A runbook with no steps, duplicate step ids, or a `--from` naming a step that does not exist: a clear error, not a panic (Task 2, Task 7).
2. A script that prints a huge amount of output, or invalid UTF-8: capture is capped at 1 MiB per attempt with a visible truncation marker, and bad bytes are replaced, never a crash (Task 6).
3. A script that hangs, or the operator pressing Ctrl-C mid-step: the step has a timeout (default 600 s), the sandbox process group is killed, and the attempt ends as `interrupted` or `timed_out`, never left open (Task 6, Task 9).
4. An injected secret that is split across two reads of the output, or appears inside a longer line: scrubbed whenever it occurs in the final captured text (Task 8).
5. The runbook file is edited between runs: the run records a content hash, and history for a step shows which attempts came from an earlier version (Task 7).

---

## File Structure

```
Cargo.toml                         workspace (spikes/ excluded)
.gitignore                         add .lori_history/
crates/loribook-core/src/
  lib.rs                           re-exports
  model.rs                         Runbook, Step, StepKind, parsing, structural validation
  vars.rs                          template scanning, dependency graph, variable validation
  events.rs                        Event, EventKind, RunId, StepId
  state.rs                         derive RunState and variable values from events; stale logic
  store.rs                         RunStore trait, MemoryStore
  runner.rs                        Runner trait, StepRequest, StepOutcome, verdict and output rules
  creds.rs                         CredentialProvider trait, env/command/prompt providers, scrub()
  engine.rs                        Operator trait, run orchestration
crates/loribook-wartlog/src/lib.rs WartLogStore: RunStore over wartlog-store
crates/loribook-sandbox/src/lib.rs BwrapRunner (and HostRunner for tests)
crates/lori-cli/src/main.rs        `lori` binary, terminal Operator
crates/lori-cli/tests/             acceptance test and fixtures
spikes/wartlog-mapping/            throwaway spike (Task 1)
```

---

### Task 1: Spike: can WartLog answer "every attempt of step X"?

Throwaway, labeled as such, outside the workspace. Its only output is a decision recorded in `docs/superpowers/notes/2026-11-26-wartlog-spike.md`.

**Files:**
- Create: `spikes/wartlog-mapping/Cargo.toml`, `spikes/wartlog-mapping/src/main.rs`
- Create: `docs/superpowers/notes/2026-11-26-wartlog-spike.md`

**Interfaces:**
- Produces: a go or no-go for WartLog behind `RunStore`, with the exact record shape and query text to use (Task 5 depends on it).

- [ ] **Step 1: Create `spikes/wartlog-mapping` with git dependencies on `wartlog-model`, `wartlog-store` and `wartlog-query` at the pinned rev, plus `tempfile` and `anyhow`.** `cargo build` in that directory succeeds.
- [ ] **Step 2: In `main.rs`, write 20,000 records into a temp store with `wartlog_store::Writer`.** Shape: source `SourceKind` custom/ndjson with attr `runbook=<hash>`; per record `attrs` `run`, `step`, `attempt`, `kind`; `trace_id` derived from the run id; 2,000 runs of 10 steps. Try both mappings: (a) every event a log record, (b) step attempts as `span` records plus log records for output. `Writer::open` with `WriterOptions { background_index: false, ..Default::default() }`; call `close()`.
- [ ] **Step 3: Open `Snapshot::open(path)` and run `wartlog_query::run(src, &snap, &QueryOptions::default())` for each of: all events of one run (by trace id); all attempts of step `s4` across runs; the latest `answer` event for variable `x`.** Print each query's row count and elapsed time. Plan inspection (`wartlog_query::exec::plan`) should show whether pruning by term or trace id happened.
- [ ] **Step 4: Run `cargo run --release` and record the numbers and the chosen mapping in the note.** Decision rule: go if each query returns the right rows in under 100 ms on 20,000 records without a full scan; otherwise no-go and Task 5 uses a different backend behind the same trait (an append-only file of length-prefixed JSON events, fsynced per append).
- [ ] **Step 5: Commit** `git add spikes docs/superpowers/notes && git commit -m "spike: WartLog mapping for run history"`.

---

### Task 2: Workspace and the runbook model

**Files:**
- Create: `Cargo.toml` (workspace, `members = ["crates/*"]`, `exclude = ["spikes"]`), `crates/loribook-core/Cargo.toml`, `src/lib.rs`, `src/model.rs`
- Modify: `.gitignore` (add `.lori_history/`, `target/`)
- Test: `crates/loribook-core/tests/model.rs`

**Interfaces:**
- Produces:
```rust
pub struct StepId(pub String);                        // stable id, unique per runbook
pub enum StepKind { Script, CheckOnly, Question, Manual, Gate }
pub struct Question { pub var: String, pub ty: VarType, pub placeholder: Option<String>, pub default: Option<String>, pub secret: bool }
pub enum VarType { Text, Number, Choice(Vec<String>) }
pub struct Mount { pub path: String, pub writable: bool }
pub struct Expect { pub exit: i32, pub pattern: Option<String>, pub lines: Vec<String> }  // default exit 0
pub struct OutputRule { pub var: String, pub secret: bool, pub rule: Extract }
pub enum Extract { Regex(String), AfterMarker(String), JsonPath(String) }
pub struct Step {
  pub id: StepId, pub label: String, pub description: String, pub plan: Option<String>,
  pub kind: StepKind, pub question: Option<Question>,
  pub check: Option<String>, pub script: Option<String>, pub approve: bool,
  pub needs: Vec<String>, pub credentials: Vec<String>, pub network: bool, pub mounts: Vec<Mount>,
  pub expect: Expect, pub outputs: Vec<OutputRule>, pub timeout_secs: Option<u64>,
}
pub struct Runbook { pub name: String, pub steps: Vec<Step>, pub hash: String /* sha256 hex of the file bytes */ }
pub fn parse_runbook(bytes: &[u8]) -> Result<Runbook, ModelError>;
pub enum ModelError { Yaml(String), Empty, DuplicateId(String), MissingCheck(String), MissingQuestion(String), BadKind(String) }
```
YAML field names are the lowercase field names above; `kind` defaults to `script`.

- [ ] **Step 1: Write failing tests in `tests/model.rs`:** `parses_minimal_runbook` (one script step with check and script, hash is 64 hex chars); `rejects_empty_runbook` (`ModelError::Empty`); `rejects_duplicate_ids` (`DuplicateId("a")`); `script_step_requires_check` (`MissingCheck("a")`); `question_step_requires_question_block` (`MissingQuestion`); `gate_and_question_need_no_check`.
- [ ] **Step 2: Run `cargo test -p loribook-core --test model`; expect compile failure.**
- [ ] **Step 3: Implement `parse_runbook` in `model.rs` with serde derive and a validation pass.** The hash is over the raw bytes.
- [ ] **Step 4: Run the tests; expect PASS.**
- [ ] **Step 5: Commit** `feat(core): runbook model and parsing`.

---

### Task 3: Variables and the dependency graph

**Files:**
- Create: `crates/loribook-core/src/vars.rs`
- Test: `crates/loribook-core/tests/vars.rs`

**Interfaces:**
- Consumes: `Runbook`, `Step`, `StepId` (Task 2).
- Produces:
```rust
pub enum Subst { Env, Exact }
pub struct VarUse { pub var: String, pub subst: Subst }
pub fn scan_template(text: &str) -> Vec<VarUse>;          // honors $${ as a literal
pub fn render_exact(text: &str, values: &BTreeMap<String,String>) -> String; // pastes ${{v}}, leaves ${v} for the environment
pub struct Analysis { pub setters: BTreeMap<String, StepId>, pub deps: BTreeMap<StepId, Vec<StepId>>, pub diagnostics: Vec<Diagnostic> }
pub enum Diagnostic {
  UsedBeforeSet { var: String, step: StepId },   // error
  Undefined { var: String, step: StepId },       // error: nothing sets it (the builder would insert a question)
  SetNeverUsed { var: String, step: StepId },    // warning
  SecretPasted { var: String, step: StepId },    // error
  FreeTextPasted { var: String, step: StepId },  // warning
}
impl Diagnostic { pub fn is_error(&self) -> bool }
pub fn analyze(rb: &Runbook) -> Analysis;
pub fn downstream(a: &Analysis, steps: &[StepId]) -> Vec<StepId>; // transitive dependents, document order
```
Variables are set by question steps and by `outputs` rules. A step uses variables found in `check`, `script`, `plan`.

- [ ] **Step 1: Write failing tests:** `scan_finds_env_and_exact`; `literal_dollar_escape` (`$${x}` yields no use); `used_before_set_is_error`; `undefined_variable_is_error`; `set_never_used_is_warning`; `secret_exact_substitution_is_error`; `free_text_exact_is_warning_but_choice_and_number_are_not`; `deps_follow_use` (step 3 uses a var from step 1 → `deps[3] == [1]`); `downstream_is_transitive` (3 depends on 2, 2 on 1 → `downstream([1]) == [2,3]`).
- [ ] **Step 2: Run; expect failure.**
- [ ] **Step 3: Implement `scan_template` as a single left-to-right scan (no regex), then `analyze`.**
- [ ] **Step 4: Run; expect PASS.**
- [ ] **Step 5: Commit** `feat(core): variables, dependencies, diagnostics`.

---

### Task 4: Events and derived state

**Files:**
- Create: `crates/loribook-core/src/events.rs`, `crates/loribook-core/src/state.rs`
- Test: `crates/loribook-core/tests/state.rs`

**Interfaces:**
- Consumes: `StepId`, `Analysis`, `downstream` (Tasks 2 and 3).
- Produces:
```rust
pub struct RunId(pub String);                         // uuid v7 text
pub struct Event { pub seq: u64 /*0 until stored*/, pub ts_ns: i64, pub run: RunId, pub kind: EventKind }
pub enum EventKind {
  RunStarted { runbook_hash: String, mode: RunMode, from: Option<StepId> },
  StepStarted { step: StepId, attempt: u32 },
  CheckResult { step: StepId, attempt: u32, passed: bool, exit: i32, output: String },
  ScriptResult { step: StepId, attempt: u32, exit: i32, output: String, ended: Ended, verdict: bool },
  ValueExtracted { step: StepId, var: String, value: Option<String> },  // None for a secret
  Answered { step: StepId, var: String, value: Option<String>, secret: bool },
  Approved { step: StepId, who: String, plan: String },
  Attested { step: StepId, who: String, note: String },
  StepSkipped { step: StepId },                       // skipped by choice
  StepVerified { step: StepId },                      // check passed, nothing run
  StepMarkedStale { step: StepId, because: String },
  RunEnded { status: RunStatus },
}
pub enum RunMode { Step, Auto, CheckOnly }
pub enum Ended { Exited, TimedOut, Interrupted, Truncated }
pub enum RunStatus { Completed, Failed, Abandoned, Interrupted }
pub enum StepState { Pending, Verified, Succeeded, Answered, Attested, Skipped, Failed, Stale }
pub struct RunState { pub steps: BTreeMap<StepId, StepState>, pub values: BTreeMap<String,String>, pub status: Option<RunStatus> }
pub fn derive(rb: &Runbook, a: &Analysis, events: &[Event]) -> RunState;   // events of ONE run plus earlier answers via `prior`
pub fn current_values(events: &[Event]) -> BTreeMap<String,String>;        // latest non-secret Answered/ValueExtracted per var, across runs
pub fn changed_vars(prev: &BTreeMap<String,String>, now: &BTreeMap<String,String>) -> Vec<String>;
```
Rule: re-answering with the same value emits nothing; a different value marks only steps that use the variable, plus everything downstream, `Stale`.

- [ ] **Step 1: Write failing tests:** `verified_then_skipped_states`; `skipped_is_never_succeeded`; `same_answer_marks_nothing_stale`; `changed_answer_stales_users_and_downstream_only` (4 steps; step 1 sets `x`, step 2 uses it, step 3 depends on 2, step 4 unrelated → 2 and 3 stale, 4 untouched); `secret_values_never_in_current_values`; `later_attempt_replaces_failed_state`.
- [ ] **Step 2: Run; expect failure.** **Step 3: Implement `derive` and `current_values` as pure folds over events.** **Step 4: Run; expect PASS.**
- [ ] **Step 5: Commit** `feat(core): events and derived run state`.

---

### Task 5: RunStore trait, in-memory store, WartLog store

**Files:**
- Create: `crates/loribook-core/src/store.rs`, `crates/loribook-wartlog/Cargo.toml`, `crates/loribook-wartlog/src/lib.rs`
- Test: `crates/loribook-core/tests/store_contract.rs` (generic contract), `crates/loribook-wartlog/tests/wartlog_store.rs`

**Interfaces:**
- Consumes: `Event`, `RunId`, `StepId` (Task 4).
- Produces:
```rust
pub trait RunStore {
  fn append(&mut self, e: Event) -> anyhow::Result<u64>;                 // returns assigned seq; durable on return
  fn all_events(&self) -> anyhow::Result<Vec<Event>>;                    // oldest first
  fn run_events(&self, run: &RunId) -> anyhow::Result<Vec<Event>>;
  fn step_events(&self, step: &StepId) -> anyhow::Result<Vec<Event>>;   // every run, oldest first
}
pub struct MemoryStore;                                                   // for tests
pub fn store_contract<S: RunStore>(make: impl Fn() -> S);                 // shared assertions, `pub` in core's `testing` module behind a `testing` feature
// loribook-wartlog:
pub struct WartLogStore;
impl WartLogStore { pub fn open(dir: &Path) -> anyhow::Result<WartLogStore>; }  // creates dir mode 0700, takes the store lock
```
`WartLogStore` uses the mapping and query text recorded by Task 1. If Task 1 was no-go, implement `FileStore` (length-prefixed JSON, fsync per append) in this crate under the same trait instead, and skip the WartLog dependency.

- [ ] **Step 1: Write `store_contract` assertions:** appended events come back in order with increasing seq; `run_events` isolates runs; `step_events` spans runs; a second `open` of the same directory while the first is live fails with a message containing "already in use"; reopening after drop returns all events (durability).
- [ ] **Step 2: Run against `MemoryStore` (fails to compile), implement the trait and `MemoryStore` (reopen and lock assertions are skipped for memory).**
- [ ] **Step 3: Implement `WartLogStore`; run the same contract plus `survives_sigkill` (a child process appends 100 events with `durable` flush and is killed; the parent reopens and sees all acknowledged events).**
- [ ] **Step 4: `cargo test -p loribook-core -p loribook-wartlog`; expect PASS.**
- [ ] **Step 5: Commit** `feat: RunStore trait with WartLog backend`.

---

### Task 6: Runner trait, verdicts and output extraction

**Files:**
- Create: `crates/loribook-core/src/runner.rs`
- Test: `crates/loribook-core/tests/runner.rs`

**Interfaces:**
- Consumes: `Step`, `Expect`, `OutputRule` (Task 2).
- Produces:
```rust
pub struct StepRequest { pub script: String, pub env: BTreeMap<String,String>, pub secret_env: BTreeMap<String,String>,
                         pub network: bool, pub mounts: Vec<Mount>, pub needs: Vec<String>, pub timeout: Duration }
pub struct StepOutcome { pub exit: i32, pub output: String /* already scrubbed and capped */, pub ended: Ended }
pub trait Runner { fn run(&mut self, req: &StepRequest) -> anyhow::Result<StepOutcome>; }
pub const MAX_OUTPUT_BYTES: usize = 1_048_576;
pub fn cap_output(raw: &[u8]) -> (String, bool);                     // lossy UTF-8; appends "\n[output truncated at 1 MiB]" when cut
pub fn verdict(exp: &Expect, o: &StepOutcome) -> bool;               // exit matches AND pattern (regex) found AND every line in `lines` present
pub fn extract(rule: &Extract, output: &str) -> Result<String, ExtractError>;  // exactly one value, else Err(NoMatch | Ambiguous)
pub struct ScriptedRunner { /* test double: maps script text or substring → StepOutcome, records requests */ }
```

- [ ] **Step 1: Write failing tests:** `verdict_defaults_to_exit_zero`; `verdict_requires_pattern_and_lines`; `extract_regex_one_capture`; `extract_zero_matches_is_error_not_blank`; `extract_two_matches_is_ambiguous`; `extract_after_marker`; `extract_json_path` (`a.b[0]`); `cap_output_truncates_and_marks`; `cap_output_replaces_invalid_utf8`; `scripted_runner_records_requests`.
- [ ] **Step 2: Run; expect failure.** **Step 3: Implement; `extract` JSON path via `serde_json` pointer-style parsing.** **Step 4: Run; expect PASS.**
- [ ] **Step 5: Commit** `feat(core): runner trait, verdicts, extraction`.

---

### Task 7: The engine: running a runbook

**Files:**
- Create: `crates/loribook-core/src/engine.rs`
- Test: `crates/loribook-core/tests/engine.rs`

**Interfaces:**
- Consumes: everything above; a `CredentialProvider` (Task 8) is passed in, so write this task with a `NoCredentials` stub from `creds.rs`'s trait definition (Task 8 adds the providers).
- Produces:
```rust
pub trait Operator {
  fn approve(&mut self, step: &Step, plan: &str) -> Decision;          // Yes | No
  fn ask(&mut self, step: &Step, q: &Question) -> Option<String>;      // None = stop the run
  fn attest(&mut self, step: &Step) -> Option<String>;                 // note, None = refuse
  fn retry_choice(&mut self, step: &Step) -> FailChoice;               // Retry | Skip | Abandon
  fn show(&mut self, line: Display);                                   // progress for the UI
}
pub struct RunOptions { pub mode: RunMode, pub from: Option<StepId>, pub who: String }
pub fn run<S: RunStore, R: Runner, O: Operator>(rb: &Runbook, store: &mut S, runner: &mut R,
      creds: &dyn CredentialProvider, op: &mut O, opts: RunOptions) -> anyhow::Result<RunStatus>;
pub fn check_all<S: RunStore, R: Runner>(rb: &Runbook, store: &mut S, runner: &mut R, creds: &dyn CredentialProvider) -> anyhow::Result<Vec<(StepId, bool)>>;
pub fn plan_pull_in(rb: &Runbook, a: &Analysis, state: &RunState, from: &StepId) -> Result<Vec<StepId>, EngineError>; // setters needed with no valid value, in order
pub enum EngineError { UnknownStep(String), Locked, Validation(Vec<Diagnostic>) }
```
Behavior per step kind: Script: run check; passes → `StepVerified`, skip; fails → (approve if `approve`, with the rendered `plan`) → run script → run check again; both the script verdict and the re-check must pass. CheckOnly: run check, record. Question: skip silently when a non-secret value exists (from a previous run or the `default`... a design-time `default` counts as present); else `ask`; a secret question is asked every run. Manual: `attest`. Gate: `approve`, always, even in Auto mode. `RunMode::CheckOnly` runs every check, changes nothing, prompts nothing, and never calls `approve` or `ask`. `from` records every earlier step `StepSkipped`; if the starting step needs a variable with no valid value, show the `plan_pull_in` list and call `approve` once before running the pulled-in steps. A failed step calls `retry_choice`. The run first fails with `Validation` if `analyze` has any error diagnostic. A changed answer appends `StepMarkedStale` for dependents.

- [ ] **Step 1: Write failing tests using `MemoryStore`, `ScriptedRunner` and a `ScriptedOperator`:** `all_checks_pass_runs_no_scripts`; `failing_check_runs_script_then_rechecks`; `recheck_failure_fails_step_and_asks_retry_choice`; `check_only_never_approves_or_asks`; `from_step_records_earlier_as_skipped_not_verified`; `from_unknown_step_is_error`; `from_pulls_in_setter_after_approval` (step 3 uses var from step 1; `from=3` → plan shows step 1, runs it first; declined → run does not start); `answered_question_skipped_on_second_run`; `secret_question_asked_every_run`; `design_time_default_counts_as_value`; `gate_requires_approval_even_in_auto`; `approval_event_records_plan_text`; `changed_answer_marks_dependents_stale`; `hash_recorded_on_run_started`; `validation_errors_block_the_run`; `second_run_in_same_store_while_first_open_is_locked` (uses a store that reports a lock).
- [ ] **Step 2: Run; expect failure.** **Step 3: Implement `run`, `check_all` and `plan_pull_in`; one private function per step kind.** **Step 4: Run; expect PASS.**
- [ ] **Step 5: Commit** `feat(core): run orchestration`.

---

### Task 8: Credentials and scrubbing

**Files:**
- Create: `crates/loribook-core/src/creds.rs`
- Test: `crates/loribook-core/tests/creds.rs`

**Interfaces:**
- Produces:
```rust
pub trait CredentialProvider { fn get(&self, name: &str) -> anyhow::Result<String>; }   // short-lived value, never logged
pub struct EnvProvider { pub prefix: String }                    // LORI_CRED_<NAME>
pub struct CommandProvider { pub commands: BTreeMap<String, Vec<String>> }   // name → argv; stdout trimmed is the value
pub struct PromptProvider<F: Fn(&str) -> Option<String>>(pub F);
pub struct Chain(pub Vec<Box<dyn CredentialProvider>>);          // first provider that answers wins
pub struct NoCredentials;
pub fn scrub(text: &str, secrets: &[&str]) -> String;            // replaces every occurrence with "[redacted:<n>]"; skips values shorter than 4 bytes
```
Wire into the engine: before a step, resolve each name in `step.credentials`, place them in `secret_env` (upper-cased name) and collect them for `scrub`; scrub `output` before any event is built. A secret question's answer goes to `secret_env` too.

- [ ] **Step 1: Write failing tests:** `env_provider_reads_prefixed_var`; `command_provider_trims_stdout`; `chain_falls_through_then_errors_with_name`; `scrub_replaces_all_occurrences`; `scrub_handles_secret_inside_longer_line`; `scrub_longest_first` (one secret contains another); `scrub_ignores_tiny_values`; `engine_never_stores_injected_secret` (a `ScriptedRunner` echoes the secret; after the run, no stored event contains it and `StepRequest.script` does not contain it); `unknown_credential_fails_step_with_name_not_value`.
- [ ] **Step 2: Run; expect failure.** **Step 3: Implement; sort secrets by length descending in `scrub`.** **Step 4: Run; expect PASS.**
- [ ] **Step 5: Commit** `feat(core): credential providers and output scrubbing`.

---

### Task 9: The bubblewrap sandbox runner

**Files:**
- Create: `crates/loribook-sandbox/Cargo.toml`, `src/lib.rs`
- Test: `crates/loribook-sandbox/tests/sandbox.rs`

**Interfaces:**
- Consumes: `Runner`, `StepRequest`, `StepOutcome`, `cap_output` (Task 6).
- Produces:
```rust
pub struct BwrapRunner { pub bwrap: PathBuf, pub shell: PathBuf /* default /bin/bash */ }
impl BwrapRunner { pub fn detect() -> anyhow::Result<BwrapRunner>; }   // Err with install hint when bwrap is missing or user namespaces are blocked
impl Runner for BwrapRunner {}
pub struct HostRunner;   // unsandboxed, used only by tests and `lori --no-sandbox` (prints a warning every run)
```
Command shape: `bwrap --unshare-all [--share-net if req.network] --die-with-parent --new-session --ro-bind /usr /usr --ro-bind /etc /etc (plus /bin /lib /lib64 symlinks) --proc /proc --dev /dev --tmpfs /tmp --chdir /tmp`, one `--bind`/`--ro-bind` per mount, `--clearenv` then `--setenv` for each env entry, then `bash -c <script>`. Preflight each name in `needs` with `which` inside the sandbox view; a missing tool is a failed outcome naming the tool, before the script runs. Timeout: kill the whole process group, `ended = TimedOut`. Output: stdout and stderr merged, capped by `cap_output`.

If `bwrap` is missing in the build environment, run `apt-get install -y bubblewrap`; if user namespaces are blocked there, the sandbox tests are marked `#[ignore]` with a reason and `lori` refuses to run without `--no-sandbox`, which must be stated in the commit message.

- [ ] **Step 1: Write failing tests (skip with a printed reason when `detect()` errs):** `runs_script_and_captures_output`; `exit_code_is_returned`; `no_network_by_default` (a script that tries `/dev/tcp/127.0.0.1/1` fails when `network=false`); `cannot_read_home` (`ls $HOME` shows nothing from the host); `cannot_write_outside_tmp_and_mounts`; `rw_mount_is_writable_ro_mount_is_not`; `env_is_cleared_except_declared` (a host variable is absent; a declared one present; a secret in `secret_env` present); `secret_not_in_process_arguments` (`ps` or `/proc/self/cmdline` inside the sandbox never shows it); `timeout_kills_the_process_group` (`sleep 60 & wait`, timeout 1 s, outcome `TimedOut`, no stray `sleep` afterward); `missing_needs_tool_fails_before_script`; `huge_output_is_capped` (200 MiB of `yes`).
- [ ] **Step 2: Run; expect failure.** **Step 3: Implement with `std::process::Command`, a `setsid` process group, a reader thread, and a deadline loop.** **Step 4: Run; expect PASS (or the documented skips).**
- [ ] **Step 5: Commit** `feat(sandbox): bubblewrap runner`.

---

### Task 10: The `lori` CLI and terminal stepper

**Files:**
- Create: `crates/lori-cli/Cargo.toml`, `src/main.rs`, `src/terminal.rs`
- Test: `crates/lori-cli/tests/cli.rs`

**Interfaces:**
- Consumes: the engine, `WartLogStore`, `BwrapRunner`, providers.
- Produces: the `lori` binary:
  - `lori validate <file.lori>`: diagnostics, exit 1 on any error.
  - `lori check <file.lori>`: no changes, no prompts, exit 1 if any check fails.
  - `lori run <file.lori> [--from <id>] [--auto] [--no-sandbox] [--cred NAME=cmd...]`: runs and prompts.
  - `lori status <file.lori>`: one line per step (`pending`, `verified`, `succeeded`, `answered`, `attested`, `skipped`, `failed`, `stale`) from the latest run.
  - `lori history <file.lori> <step-id>`: every attempt of that step across runs with run id, time, verdict, and a marker where the run used an earlier runbook hash.
  - `lori answer <file.lori> <var> [value]`: records an answer without running; a secret is read from the terminal, never an argument.
- `TerminalOperator` prompts on `/dev/tty`, one `y/N` line per approval, printing the plan sentence.

- [ ] **Step 1: Write failing CLI tests (use `assert_cmd` and `--no-sandbox` with fixture runbooks):** `validate_reports_used_before_set_and_exits_1`; `check_makes_no_history_writes_of_steps` (history gains only a check-run record, no `StepStarted`); `run_then_run_again_runs_no_scripts`; `history_shows_every_attempt_across_runs`; `history_marks_attempts_from_earlier_runbook_version` (edit the file between runs); `secret_answer_not_accepted_as_argument`; `from_unknown_step_exits_2_with_message`; `ctrl_c_midstep_records_interrupted` (send SIGINT to the child; history shows `interrupted`, not an open attempt).
- [ ] **Step 2: Run; expect failure.** **Step 3: Implement with `clap` derive. Install a SIGINT handler that sets a flag the runner's deadline loop polls.** **Step 4: Run; expect PASS.**
- [ ] **Step 5: Commit** `feat(cli): lori command and terminal stepper`.

---

### Task 11: Acceptance test from the worked example

**Files:**
- Create: `crates/lori-cli/tests/fixtures/infra.lori` (synthetic, 12 to 15 steps shaped like the worked example: preflight, create environment, create identity, role assignments, DNS records, variables, pipeline registration, a waiting "has the run finished?" manual step, a tag push, certificate creation, a final by-hand checklist), `crates/lori-cli/tests/fixtures/bin/{az,git,curl,dig}` (stubs that keep their "cloud" in a state directory), `crates/lori-cli/tests/acceptance.rs`
- Create: `docs/superpowers/notes/2026-11-26-acceptance-mapping.md` (how each script idiom maps to the model, and what did not fit)

**Interfaces:** consumes the `lori` binary. No client names, ids or hostnames in any fixture.

- [ ] **Step 1: Write the acceptance tests, one per line of spec section 10:** `rerun_with_all_checks_passing_runs_no_scripts`; `failing_check_runs_step_then_rechecks`; `check_command_changes_nothing_and_asks_nothing`; `start_at_late_step_pulls_in_the_setter` (the fixture has a late step using a variable set by an early question); `changing_a_variable_stales_exactly_its_dependents`; `injected_credential_present_in_env_absent_from_history_and_args` (credential via `--cred TOKEN='echo s3cret-value'`; grep the whole `.lori_history` directory bytes for the value: absent); `undeclared_network_is_denied` (needs the real sandbox; ignored with reason when unavailable); `kill_mid_run_then_resume` (SIGKILL the `lori` process during a step; rerun; history is intact and the run resumes from the first failing check).
- [ ] **Step 2: Build the fixture and stubs until each test fails for the right reason, then until it passes.** Record in the mapping note every idiom that was awkward (shared helper functions, multi-part steps, waiting steps), as input to phase 2.
- [ ] **Step 3: `cargo test --workspace`; expect PASS** (sandbox-dependent tests may be skipped with the printed reason; say which in the note).
- [ ] **Step 4: Commit** `test: acceptance suite from the worked example`.

---

### Task 12: Finish

**Files:**
- Create: `README.md` (what LoriBook is, build, a 20-line example runbook, the commands, the sandbox model and its MVP limits: Linux only, network all-or-nothing, host toolchain, one active run per store)
- Modify: `docs/superpowers/specs/2026-11-26-loribook-design.md` (record where the build differs from the spec, and the spike outcome)

- [ ] **Step 1: Run `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`; all clean.**
- [ ] **Step 2: Write the README and the spec's "built differently" section.**
- [ ] **Step 3: Commit and push to `claude/loribook-6b86s2`.** Do not open a pull request unless Tim asks.

---

## Self-Review

**Spec coverage.** Intent and checks-first (Task 7); start from any step and skipped-by-choice (Task 7); questions and variables with the "skip when present" rule (Tasks 3, 7); derived dependencies, used-before-set, stale (Tasks 3, 4); sandbox default-deny and per-step declarations (Task 9); credential injection and scrubbing (Task 8); append-only WartLog history in `.lori_history`, hash per run, store trait, spike (Tasks 1, 4, 5, 7); phases (Task 10 is phase 1 only); acceptance list (Task 11). Spec gap noted: host allowlist egress is not implementable with bubblewrap alone, so Task 9 makes network all-or-nothing and the Global Constraints record the limit; the spec's wording ("ideally an allowlist") already allows it. Spec's Open items 1 (substitution syntax) and 4 (large output) are settled here as my calls; 2 by Task 1.

**Step scan.** Each step names a test list, a signature, or a command. Bodies are left to the implementer except the bubblewrap command shape, which the signature does not determine.

**Type consistency.** `StepId`, `RunId`, `Event`/`EventKind`, `Runner`/`StepRequest`/`StepOutcome`, `CredentialProvider` are defined once and consumed under the same names. `Ended` is defined in Task 4 and used in Tasks 6 and 9. `RunMode` is defined in Task 4 and used in Task 7. Task 7 depends on the `CredentialProvider` trait from Task 8; Task 8's engine wiring edits `engine.rs` after Task 7, so implement the trait definition at the start of Task 7 (a one-line stub) or do Task 8 first.

**Review Focus.** Each of the five has a test: item 1 in Tasks 2 and 7; 2 in Tasks 6 and 9; 3 in Tasks 9 and 10; 4 in Task 8; 5 in Tasks 7 and 10.

**Proportion.** About 1.6 times the spec, mostly interface blocks and test names, with no function bodies.
