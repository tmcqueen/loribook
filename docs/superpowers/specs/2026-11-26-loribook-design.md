# LoriBook Design

Status: approved by Tim, 2026-11-26 (scope and history location confirmed).
Owner: Tim (tmcqueen)
Sources: the LoriBook design doc and the Runbooks spec (both Claude artifacts), a worked example (`dev-admin-setup.sh`, 21 steps, not to be implemented as is), the WartLog repository and its library design (PR 7), and the design conversation of 2026-11-26.

Labels: **Decided** means Tim stated or confirmed it. **My call** means I chose it and Tim has not confirmed it. **Open** means it is waiting on someone.

## 1. Intent

A runbook is an ordered list of steps that are too slow, ambiguous or judgment-heavy for an agent to do alone. LoriBook runs each step, records exactly what happened, and shows a human the results so the human can decide what to do about failures.

The idea came from an agent-generated idempotent bash script of about 20 steps. Running it from the top re-checked steps 1 to 10 and then did step 11. That behavior is the core. LoriBook adds a permanent history of every run of every step, questions that create variables, a graphical view, and sandboxed execution.

LoriBook is a standalone product. The Kaikaku factory spec is a different thing and is not a constraint here (Decided).

## 2. Goals, non-goals, anti-goals

### Goals

- Run a linear list of steps, each with a required check, and record every attempt.
- Rerun from the top safely: checks run first, passing steps are skipped, real work starts at the first failing check.
- Start from any step. Skipped steps are shown as skipped by choice, never as passed.
- Questions create variables. Dependencies are derived from where variables are used, never declared by hand.
- Run every script step in a sandbox by default, or on the local machine when the step says so, with credentials injected at run time.
- Keep a permanent, append-only history that answers "show every run of step 4".
- Grow in three phases, each building on the last: engine with a terminal UI, then a web UI, then a native GUI.

### Non-goals

- Branching or conditionals. A process that needs a real fork is two runbooks.
- Recreating Terraform or Ansible: no resource types, no desired-state reconciliation, no drift detection. A check is a question the human reads. It never converges anything.
- A module system with versioned imports.
- Dependence on Docker or a container runtime (Decided). Docker may be one backend later.

### Anti-goals (a failure even if it technically works)

- A runbook that is harder to write or read than the bash script it replaces.
- A history that can be edited, or that silently drops a run.
- Secrets in the history, in a command line, or in captured output.
- A sandbox so strict that real infrastructure steps cannot run, or so loose that it is decoration.

## 3. Constraints

- Rust, one codebase (Decided). Built natively per platform, not one binary everywhere (Decided).
- Lightweight isolation, with no Docker dependency (Decided). Windows may go through WSL (Decided as acceptable, not as the only route).
- Definitions are plain files. Run state is never saved in git (Decided).
- State goes in an append-only log, and WartLog is the backend (Decided). WartLog is Linux-only today and has one writer per store.
- The MVP targets Linux only (My call, forced by the two points above).
- Secrets are never stored. A secret passes to a step through the environment only, and captured output is scrubbed before it is appended (Decided).

## 4. Approaches considered

### Build shape

| | What | Trade-off |
|---|---|---|
| **A. Engine and terminal UI first (chosen)** | Core library plus a `lori` CLI with a terminal stepper. | Proves sandbox, credentials, storage and dependency tracking. No graphical view until phase 2. |
| B. Engine and web UI together | Core plus a local server and browser UI. | Graphical view sooner, but UI work competes with the hard engine problems. |
| C. UI-first prototype with a fake runner | A throwaway to feel the screens. | Teaches nothing about the engine. Fine as a labeled spike, not as the MVP. |

Tim picked A on 2026-11-26 and confirmed the progression engine and terminal UI, then web UI, then GUI.

### Run history storage

| | What | Trade-off |
|---|---|---|
| **WartLog behind a `RunStore` trait (chosen)** | LoriBook owns an `external` WartLog store through a small trait. | Matches the decided backend. The WartLog library facade (PR 7) is not built yet, so the MVP depends on `wartlog-store` directly at a pinned commit and swaps to the facade when it lands. |
| Wait for the WartLog facade | Build nothing until it ships. | Blocks the MVP on another project. |
| SQLite append-only tables | Mature, indexed. | Not the decided backend. Kept as the fallback behind the same trait if the spike in section 8 fails. |

Chosen because the backend is decided and the trait keeps the choice reversible (My call on the trait and the pinned dependency).

## 5. Architecture

```
  lori CLI / terminal stepper   (phase 1)
  web UI over a local server    (phase 2)   all thin shells over one core
  native GUI                    (phase 3)

  loribook-core   model, YAML parse, validation, variables, dependency and stale logic,
                  step state machine, run orchestration
  loribook-run    Runner trait; Linux sandbox backend; credential provider
  loribook-store  RunStore trait; WartLog implementation
```

The core has no I/O of its own and no UI. The shells and the Runner and RunStore implementations plug into it. Run orchestration is plain code with no model in it.

## 6. The runbook model

The runbook model comes from the Runbooks spec and applies here. This section states what LoriBook builds and the choices this design makes on top of it.

### Steps

A step has a stable id (display numbers are derived), a label, a description, a `plan` sentence shown before it changes anything, a required `check`, an optional `script` (any language the sandbox's runtime provides), an expected output (a reference for the human), and a verdict. The verdict defaults to the exit code and can add a pattern or required lines (My call, from the Runbooks spec).

A step kind is one of: script step, question step, check-only step (no script), manual step (a human attests it is done), or approval gate.

- **Approval is a property of a step as well as a standalone gate** (My call, from the worked example, where nearly every mutating step asks first with a plan sentence). `approve: true` stops the run before the script and records who approved, when, and the exact plan text.
- **Each step is one (check, action) pair.** A multi-part step is written as several steps (My call). Finer steps give finer history and stale tracking.
- **A precondition is its own check-only step,** for example "no release is running" (My call).

### Variables

- A question step creates a variable with a name, type, placeholder and the step that set it. It waits for input only when no value exists. It is skipped silently when a value is already present from a previous run or was supplied at design time (Decided). A design-time default therefore needs no special case: it is a value that is already present.
- A step can produce an output variable from its command output with an explicit rule: a regular expression with one capture group, a line after a marker, or a path into JSON. The step fails if the rule does not find exactly one value (My call, from the Runbooks spec).
- **An output can be marked secret** (Decided, Tim 2026-11-26). The motivating case is a step that runs `az account get-access-token` and exposes the token to later steps as a book-level secret. A secret output is held in memory for the rest of the run only. It is passed to later steps through the environment only, is never stored, and is scrubbed from every captured output, including the output of the step that produced it (the value is extracted first, then the output is scrubbed, then it is appended).
- **A step with a secret output runs its script on every run** in which that step or a later one uses the variable, because the value cannot be kept between runs. Its `check` is the verdict after the script, not a reason to skip it (My call). A changed secret never marks steps stale, since no earlier value is kept to compare against (My call).
- Scripts are templates. `${foo}` passes the value as an environment variable. `${{foo}}` pastes it into the script text. Both count for dependency tracking. A secret variable allows only the environment form, and the runner refuses to paste it. The exact syntax and brace escaping is Open (section 9).
- A variable used before it is set is an error at save and at every rearrangement. A variable that is set and never used is a warning (Decided, from the Runbooks spec).
- A changed answer marks only the steps that use it, and everything downstream of them, as stale. Re-asking with the same answer changes nothing.

### Running

- Start at step 1 or any step. A run from the top runs checks first.
- Starting past a step records it as skipped by choice. The runner can run the earlier checks read-only and warn without blocking.
- If a later step needs a variable that no valid value exists for, the run pulls in the step that sets it, after showing its plan and getting a yes.
- A failed step stops the run. The operator chooses retry, skip explicitly, or abandon. There is no automatic branching.
- Step mode pauses after each step. Automatic mode runs until something stops it. Starting a run never approves an approval step.
- A check-only run (`lori check`) runs every check with no changes and no prompts, like the worked example's `--check`.

## 7. Targets, sandboxing and credentials

- **A step runs on a target** (Decided, Tim 2026-11-26). The MVP has two: `sandbox` (the default) and `local`. A runbook sets a default target and any step can override it, so one runbook can mix both. A step's check runs on the same target as its script.
- **`local` runs on this machine as the operator,** inheriting the operator's environment, which is what makes an existing `az login` session work. It has no isolation, so it is never the default and a step must say `target: local` (My call). The `network` and `mounts` fields mean nothing on `local`, so declaring them there is a validation error (My call).
- **`lori run --sandbox-only` refuses any runbook with a non-sandbox target** (My call), for runbooks the operator did not write.
- The history records the target of every attempt, and the approval prompt shows it (My call).
- An SSH target is deferred (Tim, 2026-11-26). The target concept is in the MVP so history already records where a step ran.

- **One sandbox backend per platform behind the Runner trait** (Decided). The MVP implements Linux only, with bubblewrap and no daemon (My call: bubblewrap, chosen over Landlock alone because it gives mount and network namespaces together).
- **Default deny** (My call). A step starts with a read-only view of the toolchain, a private temporary directory, and no network. A step declares what more it needs:
  - network access, all or nothing in the MVP (bubblewrap cannot filter by host, so a host allowlist is deferred)
  - mounts, read-only or read-write, for example a repository for a `git push`
  - credentials by name
- **Toolchain in the MVP is the host's, read-only** (My call). There are no runtime images yet, so a step's `needs` list (for example `az`, `jq`) is checked by a preflight that fails early and clearly. Runtime images are Open for later.
- **Credentials are injected at run time** (Decided). A credential provider produces short-lived values on demand. MVP providers are an environment variable, a prompt, and a command (for example `az account get-access-token`) (My call). A step names the credentials it needs, receives them through the environment only, and never sees any others.
- Output is scrubbed for every injected value before it is appended. No credential appears in a command line.
- The sandbox protects the host. The cloud-side blast radius is whatever the injected token can do, so tokens should be scoped per step. LoriBook can only help by making the declaration explicit.

## 8. Storage

- A runbook is a YAML file named `<name>.lori` on the file system (the `.lori` extension is Tim's, the YAML content is my call). Run state is never written back to it (Decided).
- The history is an append-only WartLog store, created as `external` and owned by LoriBook alone. One process writes it (My call: one active run per store in the MVP).
- Store location (Decided): for the MVP, a `.lori_history` directory next to the runbook file, so a runbook and its history travel together and the directory stays out of git. Later the location becomes a property inside the runbook file itself (`<filename>.lori`), with `.lori_history` as the default.
- A run is modeled as a trace and a step attempt as a span, with output as log records carrying `run`, `step`, `attempt` and `kind` attributes (My call, **unverified**). The first piece of work is a short spike that confirms the query planner answers "every attempt of step X in runbook Y" without scanning the store. If it does not, the fallback is the trait with a different backend, and the format of the events does not change.
- Current state of a run is derived from its events and cached in memory. It is not read back from the log on every screen refresh.
- Event kinds: run started, step started, check result, script output, output value extracted, answer recorded, approval recorded, attestation recorded, step ended with verdict, run ended, stale marked. Nothing is ever edited. A step that goes stale is a new event.
- A run records the version of the runbook file it used (a content hash) so history stays honest after edits.

## 9. Phases

| Phase | Deliverable |
|---|---|
| 1 (MVP) | Core, Linux sandbox runner, credential providers, WartLog `RunStore`, `lori` CLI: `run`, `check`, `list`, `history`, `status`, `answer`, `validate`. A terminal stepper that shows the state of every step, takes approvals, and starts from a step. |
| 2 | Local server and web UI: run overview, stepper, per-step history, the builder with live validation, notifications. |
| 3 | Native GUI over the same server API. |
| Later | An SSH target, step catalog and templates, runtime images, waiting steps with polling and callbacks, cross-runbook outputs, macOS and Windows runners, Docker backend, a host allowlist for network access. |

The step catalog, the builder and templates are outside the MVP (My call: they matter but none of them proves the engine).

## 10. Acceptance test

The worked example is converted into a LoriBook runbook and run end to end against stub `az`, `git`, `curl` and `dig` commands. Nothing touches a real system. The test must show:

- a run from the top with every check passing runs no scripts
- a failing check runs its step, then re-checks
- `lori check` makes no changes and asks nothing
- start at step 11, then step 8 is pulled in because step 11 uses a variable it sets
- changing one variable marks exactly its dependents and what follows them as stale
- an injected credential is present in the step's environment and absent from the stored history and from every command line
- a step that does not declare network access cannot reach the network
- a token produced by a step through a secret output reaches a later step's environment, and appears nowhere in the stored history, including the producing step's own output
- a `local` step sees the operator's environment, and `--sandbox-only` refuses it
- killing the process mid-run and starting again resumes correctly from the log

## 11. Left to the builder

File layout and crate names beyond section 5, the event encoding, the YAML field spelling beyond the sketch in the conversation, the terminal UI library, error message wording, and how the stepper renders each state.

## 12. Open

1. **Substitution syntax.** The spelling of `${foo}` and `${{foo}}`, and how a literal brace is escaped.
2. **WartLog mapping.** Confirmed or refuted by the spike in section 8.
3. **Runtime images.** How a step names a toolchain that the host does not have, once the host toolchain stops being enough.
4. **Large output.** Whether a very large script output is truncated with a pointer or stored whole.
5. **History retention.** How long it is kept, and whether a run can be deleted.
6. **One setter per variable.** Whether a later step may reassign a variable.
7. **Windows route.** WSL only, or a native backend.
8. **Licence and repository.** The repository is Apache-2.0 today.
9. **Check mode and secret producers.** `lori check` makes no changes, but a later check may need a token, so it runs the script of a step with a secret output. That is a script running in check mode. Is that acceptable, or should such a step declare itself read-only?
