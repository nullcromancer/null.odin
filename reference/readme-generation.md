# Repository README generation and maintenance

ODIN's primary deliverable is `findings.zip`. This document covers a **second,
optional deliverable**: maintaining the repository's own root `README.md` as a
wiki-style document serving business analysts, developers and executives at once.

Run this when the caller asks for it — "update the README", "document this repo",
"maintain the readme" — or as Phase 9b of a full run when the caller wants both.

The README is a **repository artifact, not packet content**. Two protocol rules are
therefore scoped differently here, and the difference is deliberate:

| Protocol rule | In the packet | In the repository README |
| --- | --- | --- |
| No wall-clock time in deterministic content | binding | **inverted** — a `Last Updated` date is required |
| REPO_ROOT must not be modified | binding | **suspended for `./README.md` only**, and only when the caller asked for README maintenance |

Everything else — the no-invention rule, evidence labelling, secret redaction — applies
unchanged. If the caller did **not** ask for README maintenance, `./README.md` is
evidence like every other file and must not be touched.

## Mode selection (non-negotiable)

Decide before writing anything.

**FULL GENERATION** — when `./README.md` does not exist, or exists but is empty or
whitespace only. Create the file from scratch following the required structure below.
Overwrite entirely.

**INCREMENTAL UPDATE** — when `./README.md` exists and is not empty. Do **not** rewrite
the whole file.

- Update only what changed during the current session.
- Determine "changed" strictly from repository evidence: prefer git evidence
  (`git diff`, recent commits, files changed since session start); otherwise infer from
  file timestamps and metadata.
- Preserve existing wording and sections unless evidence requires a change.
- Patch-style edits only. Keep headings and anchors stable.
- If you cannot prove anything changed this session, **do not modify content** — except
  `Last Updated` / `Last Commit Date` if evidence supports it.

One narrow exception permits a full rewrite of a non-empty README: the existing file
predates this structure and lacks the required sections outright. Say so explicitly in
your response, with the reason, rather than migrating silently.

## Non-negotiable rules

- **Do not ask the caller questions.** Infer everything from repository evidence.
- **Do not hallucinate.** If you cannot prove something from files in the repository,
  write **"Insufficient Evidence"** and list what you searched.
- Prefer evidence-driven statements naming file paths, project names, module names,
  class and function names, and entrypoints.
- Avoid dumping large code blocks. Summarise concisely; include short snippets only
  where directly useful.
- If the repository contains several solutions, projects, services, apps or packages,
  **document each separately**.

### Evidence annotation

Conditional on what your environment can actually support:

- If you can reliably cite line numbers, every major claim is followed by evidence in
  parentheses: `(path/to/file.ext:123)` or `(path/to/file.ext:123-145)` or
  `(path/to/file.ext)`.
- If line numbers are **not** reliable, use file-path evidence only —
  `(path/to/file.ext)` — and **do not invent line references**.

ODIN's own `static-analysis/symbols.jsonl` carries real line numbers for every parsed
symbol, so cite them when a symbol came from a parser. Do not cite line numbers for
claims drawn from prose.

## Dates

- A **Last Updated** section near the top, carrying today's local date from the
  environment.
- **Last Commit Date** when repository metadata provides it —
  `inventory/vcs-state.json` records `head_commit_evidence.committer_date_repository_evidence`
  from a read-only `git log`. If no such evidence exists, write
  **"Insufficient Evidence"**; do not guess and do not fabricate a date.
- In INCREMENTAL UPDATE mode, only move a date when evidence supports it. Do not
  blindly bump.

> **Environment note.** The original form of this specification forbade running `git`
> because its host had no git executable, and required the IDE-injected repository
> context instead. ODIN's rule is the underlying one: **never fail because git is
> absent**. Where git is present, ODIN already invokes it read-only and
> `inventory/vcs-state.json` holds the result. Where it is absent, that file records
> the absence and the README says "Insufficient Evidence".

## Repository-wide analysis — scan everything first

Identify what technologies exist **before** writing. `inventory/roles.json` and
`inventory/files.jsonl` already classify most of this; confirm rather than re-derive.

**Language and build manifests (polyglot):**

- .NET — `.sln`, `**/*.csproj`, `*.fsproj`, `*.vbproj`, `Directory.Build.props`/`.targets`, `global.json`, `NuGet.config`
- Node/JS/TS — `package.json`, `pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`, `tsconfig.json`, `nx.json`, `turbo.json`
- Python — `pyproject.toml`, `requirements*.txt`, `setup.py`, `Pipfile`, `poetry.lock`, conda env files
- Java/Kotlin — `pom.xml`, `build.gradle`, `settings.gradle`, `gradlew`, `gradle.properties`
- Go — `go.mod`, `go.sum`
- Rust — `Cargo.toml`, `Cargo.lock`
- Ruby — `Gemfile`, `Gemfile.lock`
- PHP — `composer.json`, `composer.lock`
- C/C++ — `CMakeLists.txt`, `Makefile`, `*.vcxproj`
- Mobile — `AndroidManifest.xml`, `*.xcodeproj`, `*.xcworkspace`
- Frontend — `next.config.*`, `vite.config.*`, `angular.json`, `vue.config.*`, `svelte.config.*`, `remix.config.*`
- Monorepo tooling — `lerna.json`, workspace configs, bazel, buck, pants

**App configuration and secrets patterns:** `appsettings*.json`, `web.config`,
`*.config`, `.yaml`/`.yml`, `.env`, config folders; references to secret managers
(Vault, AWS/GCP/Azure secrets, SSM, Doppler, 1Password). Identify config **keys and
environment variable names** — never print actual secret values.

**CI/CD:** `.github/workflows/*`, `azure-pipelines.yml`, `Jenkinsfile`, GitLab CI,
buildkite, teamcity; build scripts `*.ps1`, `*.cmd`, `*.sh`, make targets, npm scripts,
gradle tasks.

**Containers, IaC, deploy:** `Dockerfile`, `docker-compose*`, helm charts, k8s
manifests, bicep, terraform, pulumi, cloudformation.

**Source and runtime entrypoints:** all source extensions for detected languages;
entrypoints, route definitions, job schedulers, workers, CLI commands.

**Tests:** `/Tests/`, `**/*.Tests.*`, and framework configs — xUnit, NUnit, MSTest,
Jest, Vitest, Pytest, JUnit.

**Docs:** `/docs`, `/wiki`, ADRs, README fragments, diagrams, architecture notes.

`odin.py readme-scan` performs this sweep mechanically and writes
`readme/component-map.json`. Treat its output as evidence to confirm, not as prose to
paste.

## Technology discovery — do this first

- Identify the primary languages and frameworks, **ranked by presence and importance**.
- Determine whether the repository is: (a) a single app, (b) a multi-component system
  (API + UI + workers + libs), or (c) a monorepo with many packages.
- Build a **Component Map** from repository evidence. Per component: name (from project
  or package metadata), type (web app, API, library, worker, CLI tool, UI,
  infrastructure), language/framework/runtime, path, and evidence-based purpose.

## Framework-specific discovery — apply only what exists

For each detected ecosystem, document the equivalents of:

| | Area |
| --- | --- |
| A | Entrypoints and runtime boot |
| B | Dependency and package map, grouped by purpose |
| C | Composition, DI and wiring, if applicable |
| D | Data layer — ORMs, providers, migration strategy, stored-procedure usage summary |
| E | Integrations |
| F | Security — AuthN/AuthZ, CORS, CSP, headers, cookies, session settings |
| G | Observability — logging, metrics, tracing |
| H | Deep code index, prioritising core domain and service logic |

Skip an area entirely when the repository has no such thing. Do not invent a data layer
for a library that has none.

## Required README structure

Exactly these sections, in this order. The first two are at the very top.

1. **Business Analyst Summary** — audience: BAs, product, stakeholders. 5-10 bullets,
   evidence-based, plain language. Cover what it does, who uses it, key workflows, key
   data concepts, integrations at a high level, operational impact. Avoid internal
   implementation detail unless it affects behaviour or requirements. Unknown →
   "Insufficient Evidence" plus what you searched.
2. **Technical Summary** — audience: developers, DevOps, architects. 8-15 bullets,
   evidence-based. Cover components, runtimes, major frameworks, hosting model, data
   layer approach, auth and security model, CI/CD, observability. Include
   "Where to start" pointers: paths to entrypoints and key folders.
3. **Title** — from discovered repo/project/component metadata. Unclear → say
   Insufficient Evidence and use the repository folder name.
4. **Last Updated** — today's local date plus Last Commit Date (or Insufficient Evidence).
5. **Table of Contents** — clickable anchor links, including anchors for the BA and
   Technical summaries.
6. **Repository Overview**
7. **Components / Projects / Packages** — table: Component | Type | Language/Framework |
   Runtime/Target | Path | Purpose
8. **Architecture Overview** — Mermaid only if evidenced; otherwise ASCII
9. **Tech Stack and Dependencies**
10. **Project Layout**
11. **Getting Started (Local Development)**
12. **Configuration**
13. **Running the System**
14. **Deployment and CI/CD**
15. **Deep Code Reference (Wiki Section)** — include a **Cross-Reference Index** table:
    Component/Module/Class/Function | File Path | Key Methods | Notes. If a web or API
    surface exists, include **API Surface**: Route/Controller/Endpoint | Auth |
    Request/Response types | Implementation location
16. **Data and Integrations**
17. **Security Notes**
18. **Observability and Monitoring**
19. **Common Tasks and Troubleshooting**
20. **Contributing / Coding Standards**
21. **License** — only if a LICENSE file exists; otherwise Insufficient Evidence

## Readability

- Use tables and bullets heavily.
- Use collapsible sections (`<details>`) for large inventories, such as class lists per
  module.
- Keep the top of the README fast: quick-start links and where-to-begin pointers.
- Keep sections skimmable. Avoid walls of text.

## Formatting constraints (non-negotiable)

These exist because they break real renderers. `odin.py readme-lint` enforces every one
of them mechanically.

**No emoji or icons in Markdown headings.** Any line beginning with one or more `#`
must be plain ASCII. Emoji in headings get corrupted to literal `?` when written to
disk on some systems, breaking both rendering and anchor links.

```
WRONG:  ## :rocket: Getting Started
WRONG:  ## 🚀 Getting Started
RIGHT:  ## Getting Started
```

Emoji may be used sparingly in body text — paragraphs, table cells, bullets — but never
in a heading line.

**No ampersands in headings.** GitHub's anchor generation for headings containing `&`
is unreliable across clients. Use "and".

```
WRONG:  ## Tech Stack & Dependencies
RIGHT:  ## Tech Stack and Dependencies
```

**Table-of-contents anchors.** With plain ASCII headings, anchors are simple
lowercase-hyphenated slugs. No emoji shortcodes, colons or special characters in anchor
hrefs.

```
## Getting Started (Local Development)  -->  #getting-started-local-development
## Tech Stack and Dependencies          -->  #tech-stack-and-dependencies
```

**Forbidden Unicode ranges — headings and body alike.** These render as `?` in the
GitHub web UI:

| Range | Block |
| --- | --- |
| U+2500..U+257F | Box Drawing |
| U+2190..U+21FF | Arrows |
| U+25A0..U+25FF | Geometric Shapes |
| U+2300..U+23FF | Miscellaneous Technical |

Safe replacements, and only these:

| For | Use |
| --- | --- |
| Directory and file trees | indented plain text, spaces and hyphens: `  - SubFolder/`, `    - file.cs` |
| Flow arrows | `-->` or `->` |
| Vertical connections | `\|` (ASCII pipe, U+007C) only |
| Horizontal lines | `---` or `===` |
| Boxes and borders | ASCII `+`, `-`, `\|` |
| Bullets | `-` or `*` |

**Architecture diagrams — preferred approach.** Use a ```` ```mermaid ```` flowchart
block rather than ASCII art whenever showing architecture, data flows or component
relationships. Mermaid renders natively on GitHub and sidesteps every encoding problem.
Fall back to ASCII `+`/`-`/`|` boxes only when the diagram cannot be expressed in
Mermaid.

## Process

1. Determine MODE from README existence and emptiness.
2. Scan the repository; identify languages, frameworks and tooling
   (`odin.py readme-scan`).
3. Enumerate all components, projects and packages, and their relationships.
4. Identify entrypoints, config, build, test and deploy artifacts per component.
5. Produce the README strictly from evidence — full overwrite in FULL GENERATION mode,
   affected sections only in INCREMENTAL UPDATE mode.
6. Run `odin.py readme-lint` and fix every reported violation.
7. At the very end of your response — **after** the README, in the terminal reply, not
   in the file — list **15 to 40 "Evidence Files Referenced"** as bullet paths.

## Relationship to the packet

When both deliverables are produced in one run, the README is a summary for humans
browsing the repository and the packet is the evidence base. Keep them consistent:
every figure in the README should be traceable to a packet artifact. Do not put a
number in the README that the packet contradicts.
