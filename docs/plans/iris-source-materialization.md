---
created: 2026-05-22
last_modified: 2026-05-22
revisions: 1
doc_type: [PLAN]
---

# IRIS Source Materialization — Making `m-cli` Usable Against the IRIS Routine Store

> **Purpose.** `m-cli`'s source-level tools (`m fmt`, `m lint`, `m lsp`, the
> workspace index, `m test` discovery) assume routine source lives on the
> filesystem as files they can glob and read. That assumption holds for
> YottaDB, which keeps routines as `.m` files on disk. It does **not** hold for
> InterSystems IRIS, which stores routine source *inside the database*
> (`IRIS.DAT`). This plan proposes how to bridge that gap so the existing
> file-based tooling works against IRIS with minimal disturbance to the
> YottaDB path.
>
> **Audience.** `m-cli` maintainers.
>
> **Relationship to the portability plan.** This is a companion to
> [iris-ydb-portability.md](iris-ydb-portability.md). That document plans the
> *runtime* engine adapter (test/coverage dispatch, `^%MONLBL`,
> `$SYSTEM.OBJ.Load`). It explicitly assumes the source files already exist on
> disk (§7.5: source tools "port for free with a glob change from `*.m` to
> `*.{m,mac}`"). **That assumption is the gap this document closes.** The glob
> change is necessary but not sufficient: for IRIS, *something has to put the
> `.mac` files on disk first*. This plan is that "something."

---

## Table of contents

1. [Context: the gap in the portability plan](#1-context-the-gap-in-the-portability-plan)
2. [Problem statement (made precise)](#2-problem-statement-made-precise)
3. [Where the filesystem assumption lives today](#3-where-the-filesystem-assumption-lives-today)
4. [Strategy overview: materialize-to-mirror behind a provider seam](#4-strategy-overview-materialize-to-mirror-behind-a-provider-seam)
5. [Layer 1 — the `SourceProvider` abstraction](#5-layer-1--the-sourceprovider-abstraction)
6. [Layer 2 — the IRIS sync engine](#6-layer-2--the-iris-sync-engine)
7. [End-to-end flow](#7-end-to-end-flow)
8. [Globals: why they are deferred, not ignored](#8-globals-why-they-are-deferred-not-ignored)
9. [Phasing](#9-phasing)
10. [Gotchas and risks to design around](#10-gotchas-and-risks-to-design-around)
11. [Proposals and recommended next steps](#11-proposals-and-recommended-next-steps)

---

## 1. Context: the gap in the portability plan

[iris-ydb-portability.md](iris-ydb-portability.md) §1 correctly identifies the
central architectural difference between the two engines:

| Aspect | YottaDB | IRIS |
|---|---|---|
| Routine source on disk | `.m` files in `$ydb_routines` directories | Conventionally **not** on disk — source lives in the routine database, reached via import/export tools |
| Filesystem workflow | Native (`vim foo.m && ydb -run ^FOO`) | Requires `do $SYSTEM.OBJ.Load(...)` / `Export(...)`; the VS Code ObjectScript extension automates this round-trip |

The portability plan then plans the **runtime** adapter — `ensure_loaded`,
`run_routine`, `^%MONLBL` — and treats the **source-level** tools as nearly
free (§3.1, §7.5, §8): "add `.mac` to the file glob." That is true *only if the
`.mac` files are already sitting on disk*. For YottaDB they always are. For
IRIS they are not, unless a developer has manually exported them (or runs the
VS Code extension's client-side editing).

So today, pointing `m lint` or `m lsp` at a fresh IRIS namespace finds **zero
files** — there is nothing on disk to glob. The portability plan automates the
runtime import; it leaves source *materialization* as an unstated manual
prerequisite. This plan makes materialization a first-class, automated step.

## 2. Problem statement (made precise)

Two clarifications sharpen the design:

**(a) Lint and parse need *routines*, not *globals*.** `m fmt`, `m lint`,
`m lsp`, and the `WorkspaceIndex` operate purely on routine *source text*. They
never read global data. Globals enter the picture only at the **runtime** layer
(`m test`, `m coverage`), and there `m-cli` already reads them *live from the
engine at execution time* — not from disk. (See the portability plan §3.3/§4:
`m-cli` does not read or write global data files today, and shouldn't grow into
a DBA tool.) Therefore the IRIS gap for the named functionality is specifically
**routine source materialization**, not global materialization. Globals are
addressed separately and deferred — see §8.

**(b) The materialization problem is the same one InterSystems already
solved.** The IRIS VS Code ObjectScript extension's "client-side editing" mode
exports DB source to local files, lets a file-based editor work on them, and
imports edits back with a compile. That is exactly the shape `m-cli` needs. The
strategy below is, deliberately, the same proven model — adapted to `m-cli`'s
engine-detection conventions.

## 3. Where the filesystem assumption lives today

The assumption is not centralized; it is four independent globs of `*.m`, each
of which would silently return nothing against an unsynced IRIS namespace:

| Site | Location | What it does |
|---|---|---|
| Runtime staging | `src/m_cli/engine.py:82` `_collect_routines()` | Walks `_ROUTINE_DIRS` (`src`, `routines`, `tests`, …) for `*.m` to stage for the engine |
| Lint file collection | `src/m_cli/lint/cli.py:451` `_collect_files()` | `Path.rglob("*.m")` over passed paths |
| Workspace / LSP index | `src/m_cli/workspace.py:141` `WorkspaceIndex.add_file()` | Indexes labels + refs; routine name derived from the **file stem** |
| Test discovery | `src/m_cli/test/discovery.py:65` | Suites are `.m` files whose stem matches `[A-Z][A-Z0-9]*TST` |

The good news: these are the *only* places source enters the system, and the
existing engine layer (`LocalEngine` / `DockerEngine` / `SSHEngine` +
`detect_engine()` in `src/m_cli/engine.py`) already demonstrates the
transport-abstraction pattern this plan extends.

## 4. Strategy overview: materialize-to-mirror behind a provider seam

Export IRIS routine source into a local **mirror directory**, run all the
existing file-based tooling against the mirror unchanged, and import edits back
to the database with a compile. Wrap the four glob sites behind a single
`SourceProvider` seam so IRIS-awareness enters the source tools in exactly one
place.

```
  IRIS.DAT                m iris sync              .m-cache/USER/*.mac
 ┌─────────┐   Atelier REST / export    ┌──────────────────────────┐
 │ routines│ ──────────────────────────►│  ordinary files on disk  │
 │ (DB)    │                            └──────────────────────────┘
 └─────────┘                                        │
      ▲                                             │  unchanged tooling
      │  m iris push (Load+Compile)                 ▼
      │  + conflict check          m lint / m fmt / m lsp / m test
      └─────────────────────────────────────────────┘
```

Two layers, designed so the YottaDB path is untouched:

- **Layer 1 — `SourceProvider` abstraction** (§5). Centralizes source
  discovery. The YottaDB implementation is today's behavior verbatim; the IRIS
  implementation syncs first, then globs the mirror.
- **Layer 2 — the IRIS sync engine** (§6). The export/import machinery and its
  connection transports, paralleling the existing engine triad.

## 5. Layer 1 — the `SourceProvider` abstraction

Introduce one seam through which all source discovery flows. Today every site
calls `Path.rglob("*.m")` directly; instead they call
`provider.discover_sources(root)`.

```
src/m_cli/source/
├── base.py        # SourceProvider ABC
├── filesystem.py  # FilesystemSourceProvider — today's behavior (glob *.{m,mac})
└── iris.py        # IrisSourceProvider — sync DB → mirror, then glob the mirror
```

```python
class SourceProvider(ABC):
    @abstractmethod
    def discover_sources(self, root: Path) -> list[Path]: ...
    @abstractmethod
    def read(self, path: Path) -> bytes: ...
    @abstractmethod
    def write_back(self, path: Path, data: bytes) -> None: ...   # fmt / quick-fix
```

- `FilesystemSourceProvider` is the YottaDB / default path — a thin wrapper over
  the current glob, extended to `*.{m,mac}`. **Zero behavior change** for
  existing users.
- `IrisSourceProvider.discover_sources()` first triggers a sync into a mirror
  dir (`.m-cache/<instance>/<namespace>/`), then globs that mirror. Downstream,
  `m lint` / `m fmt` / the `WorkspaceIndex` / `m test` discovery all see
  ordinary `.mac` files and need no IRIS knowledge.

This seam is the correct, single home for the portability plan's "glob change"
— and it is valuable on its own merits regardless of IRIS, because it removes
four duplicated globs.

## 6. Layer 2 — the IRIS sync engine

### 6.1 Transports

Mirror the existing `LocalEngine` / `DockerEngine` / `SSHEngine` triad. Three
ways to reach an IRIS instance, in recommended priority:

| Transport | Mechanism | When to use |
|---|---|---|
| **Atelier REST API** (recommended default) | `GET /api/atelier/v1/{ns}/docnames/RTN` to list; `GET .../doc/{name}.mac` to fetch source as line arrays; `PUT .../doc/{name}.mac` to save; `POST .../action/compile`. | Remote / cloud / containerized IRIS, no shell access required. This is the API InterSystems built for exactly this purpose (it backs VS Code client-side editing), so it is version-stable and round-trip-faithful. |
| **`docker exec` / `iris session`** | Pipe a script running `do $SYSTEM.OBJ.Export(...)` (or `^%RO`) into a host-visible directory; import via `do $SYSTEM.OBJ.Load(name,"ck")`. | Local or containerized IRIS with shell access — the direct analogue of the existing `DockerEngine`. Good for CI. |
| **Native API / Embedded Python** | `pip install intersystems-irispython`; `%Library.RoutineMgr` for routines, `$ORDER` walks for globals. | Fine-grained programmatic control; the natural home if/when globals export (§8) becomes real. |

### 6.2 The sync engine itself

A new `m iris sync` (export) / `m iris push` (import) command pair:

- **`m iris sync`** — list routines in the namespace matching the configured
  include filter, download each into the mirror dir, record a per-routine
  **manifest entry** (server timestamp + content hash). Incremental on
  subsequent runs: only fetch what changed.
- **`m iris push`** — for each locally modified mirror file, import +
  compile (`$SYSTEM.OBJ.Load(name,"ck")` or the REST compile action). Surface
  compile errors via `$SYSTEM.OBJ.GetErrorText` / the REST result payload.
  Before overwriting, **conflict-check** against the manifest: if the server
  copy changed since the last sync, refuse unless `--force`.

The manifest is what makes the mirror a *cache* rather than a fork — it is the
basis for both incremental sync and safe write-back.

### 6.3 Configuration

A new `[engine.iris]` block, slotting alongside the existing
`[lint] target_engine` key (`KNOWN_ENGINES` already includes `"iris"` in
`src/m_cli/config.py:61`):

```toml
[engine.iris]
transport   = "rest"                 # rest | session | native
rest_url    = "https://host:52773"
namespace   = "USER"
instance    = "IRIS"                 # for the session transport
creds_env   = "IRIS_PASSWORD"        # name of an env var — never inline secrets
mirror_dir  = ".m-cache"
include     = ["*.mac", "*.int"]     # .int read-only; .cls out of scope
```

Engine selection follows the precedence the portability plan §7.2 already
proposes: `--engine` flag → `[engine] kind` config → `$M_ENGINE` → heuristics
(`$ISC_PACKAGE_INSTALLDIR` / `iris` on `$PATH`) → fall back to YottaDB.

## 7. End-to-end flow

1. User configures `[engine.iris]` and runs `m iris sync` (or any source
   command, which triggers sync via `IrisSourceProvider`).
2. Routine source lands in `.m-cache/USER/*.mac` with a manifest.
3. `m lint` / `m fmt` / `m lsp` / `m test` run against the mirror **unchanged**.
4. `m fmt --write` / LSP quick-fixes mutate mirror files via
   `provider.write_back()`.
5. `m iris push` (or a `m watch` hook) imports + compiles changed files,
   conflict-checking against the manifest first.

`m watch` integration: on a mirror-file save, run `push` (import + compile)
*before* the test path executes, so IRIS never runs stale source — the single
extra step the portability plan §8 flags for `m watch` on IRIS.

## 8. Globals: why they are deferred, not ignored

Per §2(a), none of the named functionality (lint, parse, LSP, fmt) reads
globals, and the runtime tools read them live from the engine. So globals do
**not** need materialization for `m-cli` to be usable on IRIS.

They are deferred to a later, *optional* phase for two narrow future needs:

- **Static test fixtures** — exporting a global subtree so a test seed is
  reproducible without a live DB.
- **Data portability** — a convenient coincidence noted in the portability plan
  §3.3 is that the global extract routines `^%GO` / `^%GI` share names across
  YottaDB and IRIS, so extract files are often interchangeable even though the
  database files are not.

When that need arrives, the Native-API transport (§6.1) is the right vehicle:
an `$ORDER` walk over a global subtree, emitted as a YottaDB-compatible extract
or a `.gof`. Until then, globals stay a pure runtime concern.

## 9. Phasing

Sequenced so each phase is independently shippable and the YottaDB path never
regresses. Complements — does not replace — the portability plan's I-0…I-4.

| Phase | Deliverable | Effort |
|---|:---:|:---:|
| **0** | `SourceProvider` seam: route the four globs (§3) through one `discover_sources()`; add `.mac`/`.int`. Pure refactor, **zero behavior change** for YottaDB. | S |
| **1** | `m iris sync` read-only export via Atelier REST → mirror dir. **lint/fmt/lsp/parse now work on real IRIS code.** The core remedy. | M |
| **2** | Write-back: `m iris push` import + compile, manifest-based conflict detection, `m watch` hook. | M |
| **3** | Runtime adapter (`m test` / `m coverage`) — defers to portability plan I-2/I-3 (`ensure_loaded`, `^%MONLBL`), reusing this plan's connection layer. | M–L |
| **4** | *Optional* globals export (§8) via Native API for static fixtures / data portability. | M |

After **Phase 1** an IRIS developer can already lint, format, navigate, and
edit their code with `m-cli` — the bulk of day-to-day value — before any
runtime adapter exists.

## 10. Gotchas and risks to design around

- **Package-dotted routine names.** IRIS routine names like `Foo.Bar.mac` must
  map to a chosen mirror filename convention (flat `Foo.Bar.mac` vs nested
  dirs). Pin one early, because `WorkspaceIndex` derives the routine name from
  the **file stem** (`src/m_cli/workspace.py`); a mismatch breaks go-to-def.
- **Round-trip fidelity.** Export must preserve exact lines and offsets, or
  `m coverage`'s label-relative line decoding (portability plan §5) misaligns.
  This is a primary reason to favor the Atelier REST API, which returns verbatim
  source lines, over ad-hoc text export.
- **`.int` is generated.** Sync it read-only for navigation/reference; never
  push it. Only `.mac` is authored source.
- **`.cls` is out of scope.** `m-cli` is M-routine-focused, not
  ObjectScript-class-focused (portability plan §10.4). State the limit so users
  don't expect class refactoring.
- **Writes into a live DB are outward-facing and hard to reverse.** `m iris
  push` should confirm before importing unless explicitly authorized, and must
  refuse on manifest conflict without `--force`.
- **Credential hygiene.** Credentials come from an env var / keychain
  (`creds_env`), never inline in `.m-cli.toml`, and are never logged. REST runs
  over HTTPS.
- **Staleness.** Treat the mirror as a cache keyed by the manifest; a bare
  `m iris sync` reconciles. Document that editing mirror files while the DB
  changes underneath is a conflict the manifest is designed to catch, not
  prevent.

## 11. Proposals and recommended next steps

These are proposals for maintainer decision. **No implementation is included
in or implied by this document.**

1. **Adopt the materialize-to-mirror model** as the canonical way `m-cli`
   reaches IRIS source, with the **Atelier REST API as the default transport**
   (fidelity, version stability, no host-shell dependency, remote/cloud
   support) and `docker exec` export as the CI/offline parallel.

2. **Prototype Phase 0 first — the `SourceProvider` seam — as the recommended
   immediate next step.** It is a low-risk, IRIS-independent refactor that:
   - collapses the four duplicated `*.m` globs (§3) into one
     `discover_sources()`;
   - is the correct single home for the portability plan's planned `*.{m,mac}`
     glob change;
   - unblocks every later phase; and
   - delivers value (deduplication, one discovery contract) even if IRIS
     support is never built.

   Suggested prototype scope, kept deliberately minimal: add
   `src/m_cli/source/{base,filesystem}.py`, route the four call sites through
   `FilesystemSourceProvider`, and pin the behavior with tests asserting
   byte-identical discovery results to the current globs. No IRIS code, no
   network, no new dependencies. Following the repo's TDD guardrail
   (`m-cli/CLAUDE.md`), the discovery-equivalence test is written and shown RED
   before the refactor.

3. **Confirm the engine-selection precedence** (§6.3) jointly with the
   portability plan §7.2 so source-materialization and runtime dispatch agree on
   how `ydb` vs `iris` is chosen. They must read from the same `[engine]`
   resolution.

4. **Defer globals (§8)** explicitly until a concrete fixture/portability need
   appears; do not block IRIS source usability on it.

5. **Pin the mirror filename convention** for package-dotted routine names
   (§10) before Phase 1, since the workspace index depends on the file stem.

6. **Decide the IRIS test substrate** (community vs full edition; bundled Docker
   image) in coordination with the portability plan §9 decision points, so both
   plans test against the same instance.

> Recommended sequencing: land Phase 0 (the seam) on its own merits, then
> Phase 1 (read-only sync) to make IRIS code lint/format/navigable, then layer
> in write-back (Phase 2) and the runtime adapter (Phase 3, owned by the
> portability plan).
