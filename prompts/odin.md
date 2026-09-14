---
description: Run the ODIN repository-forensics protocol and produce findings.zip
argument-hint: [REPO_ROOT] [--artifacts DIR] [--network denied|authorized] [--sandbox none|container|microvm]
---

Run the **ODIN** protocol: a deterministic, read-only forensic analysis of one
repository that produces a self-contained evidence packet named exactly `findings.zip`.

The skill lives at:

    {{ODIN_DIR}}

(If that path was not substituted at install time, locate the skill directory — it
contains `SKILL.md`, `PROTOCOL.md`, `reference/` and `scripts/odin.py` — and use it.)

## Do this now

1. Read `{{ODIN_DIR}}/SKILL.md` in full. It is the operating procedure.
2. Read `{{ODIN_DIR}}/PROTOCOL.md` in full. It is normative and binding; where it and
   anything else disagree, it wins.
3. Resolve inputs from the arguments below, falling back to the protocol's defaults:
   REPO_ROOT defaults to the current working directory; ARTIFACT_ROOT defaults to a
   writable directory **outside** REPO_ROOT; network defaults to `denied`; sandbox
   defaults to `none` unless you can confirm real isolation.
4. Execute Phases 0–12 from `SKILL.md`, loading `reference/*.md` as each phase needs
   it. Use `python {{ODIN_DIR}}/scripts/odin.py` for the mechanical phases.
5. Finish with the Phase 12 completion summary and nothing else.

## Non-negotiable

- REPO_ROOT is evidence: never modify it, never write generated output into it.
- Never execute repository-controlled code without an adequate sandbox; without one,
  do static analysis only and document the skip.
- Never invent a version, test result, coverage number, architecture edge, runtime
  behavior or vulnerability.
- Never print a secret value, and never validate a discovered credential anywhere.
- Fail soft: one failed phase does not end the run. Package a partial packet with
  explicit blocker documentation rather than abandoning the analysis.
- Do not ask the user questions. Resolve unknowns with the protocol's defaults and
  record every assumption.

Arguments: $ARGUMENTS
