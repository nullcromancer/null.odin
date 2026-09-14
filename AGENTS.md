# ODIN

This directory is the **ODIN** skill: a deterministic repository-forensics protocol
that analyzes one repository read-only and produces an evidence packet named exactly
`findings.zip`.

## To run ODIN

1. Read `SKILL.md` — the operating procedure, phases 0–12.
2. Read `PROTOCOL.md` — normative and binding. Where anything else disagrees with it,
   it wins.
3. Load `reference/*.md` as each phase needs it.
4. Use `python scripts/odin.py` for the mechanical phases (`reference/toolkit.md`).

## To work *on* this skill

Files and their jobs:

| Path | Role |
| --- | --- |
| `SKILL.md` | entry point and phase plan; keep it short enough to read every run |
| `PROTOCOL.md` | the normative protocol, content-faithful to the source specification — **do not reword requirements here**; add operational guidance to `reference/` instead |
| `reference/` | per-phase operational detail, loaded on demand |
| `templates/` | document skeletons for the packet |
| `scripts/odin.py` + `scripts/odin_lib/` | the stdlib-only toolkit |
| `prompts/odin.md` | Codex / generic slash-prompt shim |
| `install/` | installers for Claude Code and Codex |

Constraints on the toolkit: Python 3.8+, **standard library only**, no network, never
writes into REPO_ROOT, never executes repository-controlled code, deterministic output
(sorted keys, bytewise path order, LF, no wall-clock time in packet content).

If you change what the packet must contain, update all four of: `PROTOCOL.md`'s
contract section, `reference/packet.md`, `REQUIRED_PACKET_ARTIFACTS` in
`scripts/odin_lib/verify.py`, and the templates.
