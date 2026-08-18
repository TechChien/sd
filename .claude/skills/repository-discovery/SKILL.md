---
name: repository-discovery
description: Use when documenting a repository for OpenSpec plan-with-ai repository discovery, creating or updating repository catalog, local workspace mapping, system map, baseline evidence, or AI-readable repo onboarding docs.
---

# Repository Discovery

## Overview

Create AI-readable repository discovery documents before decomposition, SDD, or implementation planning for the repository in the current workspace. Prefer verified repository facts over inference, and record unknowns as blockers.

This skill satisfies the discovery contract defined in
`openspec/schemas/plan-with-ai/templates/repository-discovery.md`. The runner enforces the
contract deterministically, immediately before every `system_decomposition` (or, in the hotfix
bundle, `sdd_lite`) dispatch attempt — not before `start` (ADR-0025) — and fails with a policy
error when any output file is missing, a repo ID is not traceable to the catalog, a verified repo
lacks a code outline recording the lock commit, `workspace.local.yaml` omits a field the
implementation phase requires, or a recorded commit — the lock's or the workspace's
`observed_commit` — no longer matches the repo's live HEAD. Preparing these documents right after
`init`, ahead of `system_decomposition`, is still recommended even though the runner no longer
requires it before `start`.
Two companion contracts complete the set:
`openspec/schemas/plan-with-ai/templates/cross-repo-map.md` (workspace-level relationships) and
`openspec/schemas/plan-with-ai/templates/code-outline.md` (per-repo public surface and
conventions). None of the three are schema artifacts; use them as the source of truth for
required output files and their contents.

When a repository already has discovery documents from an earlier change, start from those rather
than from scratch: `landscape/code-outline/<repo-id>.md` carries its own Valid At Commit, so diff
that commit against live HEAD and follow "Refreshing After HEAD Moves" below. Those documents are
the only per-repo seed to work from. Do not keep a separate profile or summary of the same facts
elsewhere — a copy without a Valid At Commit has nothing that detects it going stale, and a stale
seed is read as verified evidence.

## Workflow

1. Inspect only repository metadata and public docs needed for discovery, when present:
   - OpenSpec config and repository-discovery templates
   - package/build manifests such as `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, or equivalent
   - `README.md`
   - agent instructions such as `AGENTS.md`, `CLAUDE.md`, or equivalent
   - environment examples such as `.env.example`
   - `.git/config`, `.git/HEAD`, and refs when Git commands are blocked.
2. Create or update every file in the Output Files list of
   `openspec/schemas/plan-with-ai/templates/repository-discovery.md`. Read that list rather than
   working from memory of it — it is the contract `strict discovery` enforces.
3. Everything except `workspace.local.yaml` is commit-safe. Keep `workspace.local.yaml` local-only
   because it contains workstation paths.
4. Do not read `.env` values. Use `.env.example` only to list required environment keys.
5. Run baseline verification only when it is local and safe. Choose the repo's documented build,
   lint, or test command from verified metadata. Safe to run: file existence checks, package
   metadata reads, the declared build or compile step, and Git metadata when accessible. Document
   rather than run, unless the user asks: runtime entry points, and anything needing a database,
   credentials, an external API, the network, or a migration against a live database.
6. If Git commands fail due dubious ownership or access, record the failure as discovery evidence instead of changing global Git config.
7. Derive edges only from verified evidence: manifest dependencies, client/config files
   naming another repo's endpoints, shared schema/migration files, or explicit human input recorded
   as such. Anything else is an Unknown in `cross-repo-map.md`, never an edge. Edges to external
   systems (third-party APIs, shared databases, brokers) belong in the same table — a single-repo
   change's `cross-repo-map.md` is those edges, not a one-line note that there are no cross-repo
   ones.
8. Record `generated_at` on `cross-repo-map.md`, and `generated_at` plus the Valid At Commit on each
   `code-outline/<repo-id>.md`. When the repo's HEAD no longer matches the commit recorded for it in
   `landscape/repository_lock.yaml`, follow "Refreshing After HEAD Moves" below.
9. Before writing any repository catalog `commands:` block, present one summary row per in-scope
   repository with: detected language/runtime and package manager; derived build, whole-suite test,
   and single-test-file template; and the manifest/file evidence for each command. Show
   "not derived" rather than guessing when evidence is absent. A build or test command that is not
   derived is simply omitted. A `test_file` template that is not derived — a repository with no test
   framework — is recorded as `status: planned` with the `bootstrap_task` that will establish it,
   never as a confirmed command that does not exist. When a confirmed block already exists, show the
   prior values beside the newly derived values so changes are visible.
10. Ask the operator to confirm the command summary explicitly. Only after confirmation, write the
    `commands:` block and record the confirmer in `confirmed_by` and the confirmation time as an
    ISO-8601 `confirmed_at`. Never silently overwrite a prior confirmed set.
11. Verify the written documents with the runner rather than by inspection. The runner is the
    operator-installed `plan-runner` CLI; never build it from source or invoke it by file path.

    a. Confirm it is on `PATH`: `command -v plan-runner` (POSIX) or `Get-Command plan-runner`
       (Windows PowerShell).
    b. If it does not resolve, **stop the session there**. Do not fall back to a source checkout,
       and do not hand the documents over as done. Report to the operator:

       > `plan-runner` is not installed, so repository discovery cannot be verified. Install it
       > globally (`npm install -g plan-runner`, or the tarball provided for this project), then
       > run `plan-runner strict discovery --change <change-id>`.

    c. If it resolves, run from the control repository root:

       ```sh
       plan-runner strict discovery --change <change-id>
       ```

       Fix every reported violation and re-run until it reports the contract passed; this is the
       same check the runner runs immediately before `system_decomposition`/`sdd_lite` dispatches, so a
       failure here is a failure there. When a violation names a field
       you believe you already wrote, the key name is wrong, not the value — the policy reads exact
       keys and silently ignores every other spelling, so copy the key from the YAML block in the
       matching template instead of renaming it by guess.

## Output Contract

**The templates are the contract. This skill does not restate it** — a second copy is how the two
drifted apart before, with this file calling `observed_commit` optional while the runner required
it (ADR-0020). Read the required files, sections, and exact YAML keys from:

- `openspec/schemas/plan-with-ai/templates/repository-discovery.md` — the Output Files list (which
  files are required at all), the `workspace.local.yaml` keys, and the `commands:` block
- `openspec/schemas/plan-with-ai/templates/cross-repo-map.md`
- `openspec/schemas/plan-with-ai/templates/code-outline.md`

Copy every key name from the template's own YAML block rather than typing it from memory: the
policy reads exact keys and silently ignores every other spelling, so a plausible synonym reads as
a missing field.

## Refreshing After HEAD Moves

A repository's HEAD moving does not require redoing discovery from scratch. The commit is recorded
in three places — `landscape/repository_lock.yaml`, `workspace.local.yaml`'s `observed_commit`, and
each `code-outline/<repo-id>.md` Valid At Commit — and all three must equal the live HEAD, so all
three are re-stamped together. What varies is how much else has to change:

1. Take `git diff <recorded commit>..HEAD --name-only` in that repository.
2. **Code outline**: if the diff touches nothing the outline describes (its public surface, key
   types, persistence surface, test layout, or the files cited for conventions), re-stamp the Valid
   At Commit only — the outline is still accurate, and that is exactly what the field asserts.
   Otherwise regenerate the affected sections.
3. **Confirmed command set**: if the diff touches no manifest, lockfile, or build configuration for
   that repository, keep the existing `commands:` block with its original `confirmed_by` and
   `confirmed_at` — nothing that decides those commands changed, and re-asking the operator on every
   unrelated commit trains them to confirm without reading. If any of those files did change, run
   the summary-and-confirmation flow in steps 9-10 again.
4. Re-run `strict discovery` (step 11). It is what proves the re-stamp is complete.

## Common Mistakes

- Do not infer extra repositories, owners, or services.
- Do not map BDD scenarios or define implementation tasks in discovery docs.
- Do not hide dirty working tree, inaccessible remotes, missing tests, or Git ownership problems.
- Do not copy secrets from `.env`; document required keys only.
- Do not record a cross-repo edge without citable evidence; unverified interactions are Unknowns.
- Do not assert a coding convention in the code outline without citing the file(s) that show it.
- Do not infer build or test commands from language convention alone. A command needs evidence in
  the repository's own manifest, wrapper, build file, or documentation; otherwise report it as not
  derived and ask the operator.
- Do not record a command that does not exist yet as confirmed. `confirmed_at` dates a command that
  ran; a command a later setup task will create is `status: planned` with a `bootstrap_task`. A
  catalog that lists `test: ["npm", "test"]` while its own `known_blockers` say no test script is
  declared is the defect this rule exists to prevent.
- Do not treat a single-repo change as a reason to leave `cross-repo-map.md` near-empty. Its
  external-system edges are what `create_contract` reads and what decides whether a machine contract
  is mandatory.
- Do not write a field from this skill's description of it. Every key name and required section
  comes from the template; this file deliberately no longer carries a second copy to work from.
- Do not hand the documents over unverified. Discovery is done when `strict discovery` passes, not
  when the files exist; ending the session without a passing run only moves the failure to the
  operator. The single exception is `plan-runner` not being installed (step 11b) — then stopping is
  correct, but say so explicitly instead of reporting discovery as done.
