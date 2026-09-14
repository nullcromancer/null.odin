# `odin.py` — toolkit reference

Stdlib-only Python 3.8+. No third-party packages, no network, no writes into
REPO_ROOT, no execution of repository-controlled code. Every subcommand prints a JSON
summary on stdout and exits non-zero when its own status is `FAIL`.

```
python <skill>/scripts/odin.py <command> [--artifacts ARTIFACT_ROOT] [options]
```

State lives in `ARTIFACT_ROOT/.odin-state.json`. Commands find it from `--artifacts`,
then `$ODIN_ARTIFACT_ROOT`, then by walking up from the working directory. Export
`ODIN_ARTIFACT_ROOT` once after `init` and the rest is implicit.

---

## `init`

```
odin.py init --repo <REPO_ROOT> [--artifacts <DIR>]
             [--network denied|authorized-advisory-only|authorized]
             [--sandbox none|container|microvm|platform-sandbox|<description>]
```

Resolves REPO_ROOT and ARTIFACT_ROOT and **refuses an ARTIFACT_ROOT inside
REPO_ROOT**. Without `--artifacts`, picks `<system temp>/odin-<repo name>-<12 hex of
the repo path>` — stable across runs for the same repository. Creates the packet
skeleton, `scratch/`, `scratch/home/` and `scratch/tmp/`, applies the determinism
environment (`TZ`, `LC_ALL`, `LANG`, `PYTHONHASHSEED`, `SOURCE_DATE_EPOCH`, umask) and
records anything that could not be applied. Seeds `ACTION_MANIFEST.json`. Probes
filesystem case sensitivity.

Re-running `init` is safe: the packet and the manifest are preserved.

## `env`

Probes ~45 toolchain and analysis binaries for versions, and records a sanitized
environment: build-relevant names always, values only for demonstrably non-sensitive
variables, `<REDACTED>` otherwise. Writes `inventory/environment.json`.

Tool availability decides which analyses are possible. An unavailable tool becomes an
`unavailable` manifest action — never a silent substitution.

## `inventory`

```
odin.py inventory [--tree-limit N]
```

The exhaustive traversal. Filesystem order, not Git's ignore rules, defines the
inventory: dotfiles, ignored files, vendored trees, build output, binaries and `.git`
itself are all included.

Per non-directory entry it records path, entry type, size, mode, `nlink`, SHA-256,
symlink target (raw, resolved, inside/outside REPO_ROOT, dangling), hard-link group,
text/binary/empty classification with evidence, encoding, EOL style, line count, magic
format and MIME guess, extension, `.gitattributes` linguist attributes, Git
tracked/untracked/ignored/submodule state, primary classification with its evidence
rule plus preserved alternatives, role hint, origin hint, sensitivity hint, and the
per-file document path.

Traversal guarantees: bytewise-sorted path order; directories never traversed through
a symlink; no directory alias traversed twice; symlink targets outside REPO_ROOT
recorded but never read.

**Writes:** `inventory/files.jsonl`, `directories.jsonl`, `tree.txt`,
`source-hashes.sha256` (the baseline), `languages.json`, `file-types.json`,
`roles.json`, `vcs-state.json`, `inventory-summary.json`,
`per-file-documents-required.json`, `traversal-errors.json` (when non-empty), and
`files/_PATH_MAP.json`.

`_PATH_MAP.json` records the rare paths whose document name had to be escaped because
the host filesystem cannot represent it verbatim — Windows reserved device names
(`aux.c`), forbidden characters, trailing dots. The mapping stays injective.

## `todos`

Scans inventoried text files for TODO, FIXME, XXX, HACK, BUG, TEMP, WIP, DEPRECATED,
not-implemented markers, panic/todo stubs, skipped and disabled tests, placeholder
values, stub/mock language and incomplete feature flags. Each hit carries file, line,
context, marker, owner tag, issue reference, category and confidence, with vendored,
generated, VCS-metadata, documentation and fixture occurrences separated from
actionable work. Caps: 8 MB per file, 500 matches per file, both recorded.

**Writes:** `static-analysis/unfinished-work.json`.

## `secrets`

Pattern and entropy scan over inventoried text files: cloud keys, VCS and registry
tokens, webhooks, private key blocks, JWTs, credentials in URLs and connection
strings, secret-shaped environment and code assignments, and unattributed high-entropy
strings.

**No secret value is ever written.** Each finding carries rule ID, category, path,
line, `"value": "<REDACTED>"`, value length, charset shape, entropy, a truncated
salted SHA-256 fingerprint for deduplication, an activity assessment
(`example-or-placeholder`, `example-or-test-only`, `active-looking`, `unknown`) with
its reason, and confidence. Nothing is validated against any service. Binary files
with sensitive names are listed separately for manual review.

**Writes:** `security/secrets-redacted.json`.

Its limits are stated in the output: pattern-based, text files only, within the size
cap, and no VCS history. Cover the gaps with a dedicated scanner when one is available
and record it as a separate action.

## `docstub`

```
odin.py docstub [--overwrite]
```

Creates one document per physical regular file at the inventory's `doc_path`, with the
mechanical fields pre-filled from inventory evidence and the analytic fields marked
`PENDING` under a `<!-- odin:pending -->` marker. Existing documents are kept unless
`--overwrite` is passed.

Stubs are a starting point, never a deliverable: `validate` fails while any marker
remains.

## `log`

```
odin.py log --phase P --action-id A --status S --tool T --command "..." [...]
odin.py log --from-json actions.json
```

Appends to `ACTION_MANIFEST.json` with a monotonic sequence number — never a
wall-clock time. Options mirror the protocol's required fields: `--tool-version`,
`--runtime-version`, `--cwd`, `--input-scope`, `--output` (repeatable), `--sandbox`,
`--network`, `--repo-code-executed`, `--exit-code`, `--termination`, `--log-path`
(its SHA-256 is computed automatically), `--reason`, `--evidence` (repeatable),
`--assumption` (repeatable).

`--status` is one of `planned`, `executed`, `succeeded`, `failed`, `timeout`,
`skipped`, `unavailable`, `blocked_by_policy`.

`--from-json` takes one action object or a list, for logging a batch at once.

Redact the sensitive portion of any command string before logging it, and say that
redaction occurred.

## `verify`

Re-walks REPO_ROOT, recomputes every SHA-256, and compares against the Phase 0
baseline and the recorded VCS state. Reports added, removed, content-modified,
size-changed and newly unreadable files, plus VCS differences.

Anything other than `PASS` is a **critical process failure**. Report it prominently and
determine the cause.

**Writes:** `inventory/integrity-verification.json`.

## `validate`

Pre-packaging gate. Checks: required top-level documents; required packet artifacts;
JSON/JSONL/SARIF parse; Mermaid diagram type and bracket balance; internal Markdown
links; one complete document per physical regular file with no stub markers left;
module registry versus module documents; the coverage claim (`measured` must exist,
and a false `measured` may not coexist with numeric coverage keys); action-manifest
structure, required fields and monotonic sequence numbers; and a re-scan of the packet
itself for unredacted secret values.

**Writes:** `VALIDATION.json`. Exits 1 on `FAIL`.

## `manifest`

Generates `MANIFEST.sha256` over every packet file except itself, sorted by normalized
path, `<digest><two spaces><path>`, then verifies it against the packet.

## `package`

```
odin.py package [--skip-manifest] [--compresslevel N]
```

Regenerates the manifest (unless skipped) and writes `ARTIFACT_ROOT/findings.zip`:
sorted member order, fixed 1980-01-01T00:00:00Z timestamps, 0644/0755 permissions with
Unix `create_system`, fixed deflate level. Then reopens the archive, runs
`testzip()`, checks every required top-level document is present, and verifies the
archived bytes against `MANIFEST.sha256`.

Repackaging unchanged content produces a byte-identical archive.

## `status`

Prints run progress: roots, policies, inventory counts, per-file documents written
versus required, module documents, actions logged, which top-level documents exist,
and whether `findings.zip` exists with its SHA-256. Useful for pacing a long
documentation phase.
