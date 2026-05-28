---
created: 2026-05-23
last_modified: 2026-05-23
revisions: 1
doc_type: [PLAN, STRATEGY]
---

# VAEC VistA Development Modernization — Modern M Tooling for VA's IRIS-Hosted VistA

> **Purpose.** Capture the strategy for bringing modern MUMPS development
> tooling — version control, an M-aware IDE, and assertion-based unit testing —
> to VistA development at the US Department of Veterans Affairs (VA), whose
> VistA systems run on **InterSystems IRIS** inside the **VA Enterprise Cloud
> (VAEC)**, a FedRAMP HIGH GovCloud on AWS. It synthesizes the design
> discussion that produced it: the friction points, the architecture options,
> the trade-offs weighed, and the recommended path.
>
> **Audience.** Maintainers and stakeholders scoping a VA-facing,
> IRIS-focused developer-tooling initiative.
>
> **Status.** Seed document for a **new, IRIS-focused repository** (working
> name `m-dev-saas` / `vaec-vista-dev`). It is staged here in `m-cli/docs/plans`
> next to its technical antecedents and is expected to move out into the new
> repo once that repo exists. Nothing here is committed implementation.
>
> **Relationship to existing plans.** Builds directly on
> [iris-source-materialization.md](iris-source-materialization.md) (materialize
> IRIS routine source to the filesystem behind a `SourceProvider` seam) and
> [iris-ydb-portability.md](iris-ydb-portability.md) (the runtime engine
> adapter). This document is the *product/deployment* layer above those two
> *engineering* plans: it explains **why** the VA needs this, **what** to build
> first, and **where** it runs.

---

## Table of contents

1. [Context: the VA development reality](#1-context-the-va-development-reality)
2. [The four friction points](#2-the-four-friction-points)
3. [The reframe: engine-neutral source vs. engine-bound execution](#3-the-reframe-engine-neutral-source-vs-engine-bound-execution)
4. [Version control is the foundation, not a parallel problem](#4-version-control-is-the-foundation-not-a-parallel-problem)
5. [Architecture: three interface planes](#5-architecture-three-interface-planes)
6. [The IRIS interface: Atelier and the filesystem question](#6-the-iris-interface-atelier-and-the-filesystem-question)
7. [SaaS ↔ IRIS interface options](#7-saas--iris-interface-options)
8. [Interleaving with the InterSystems ObjectScript extension](#8-interleaving-with-the-intersystems-objectscript-extension)
9. [The one engine decision: where unit tests execute](#9-the-one-engine-decision-where-unit-tests-execute)
10. [Git as source of truth: source vs. state vs. data](#10-git-as-source-of-truth-source-vs-state-vs-data)
11. [The biggest risk is process, not technology](#11-the-biggest-risk-is-process-not-technology)
12. [Staged environments → git promotion + KIDS](#12-staged-environments--git-promotion--kids)
13. [VAEC / FedRAMP HIGH constraints](#13-vaec--fedramp-high-constraints)
14. [Phased roadmap](#14-phased-roadmap)
15. [Open decisions](#15-open-decisions)

---

## 1. Context: the VA development reality

VA VistA functionality is developed through a staged promotion pipeline:

```
  dev VistA  →  test VistA  →  pre-prod VistA  →  prod VistA
```

Salient facts about this environment that shape every design choice below:

- **Engine.** All VistA instances run on **InterSystems IRIS**, not YottaDB.
- **Hosting.** All instances live in the **VA Enterprise Cloud (VAEC)** — the
  VA's implementation of a **FedRAMP HIGH** certified AWS GovCloud. Any tooling
  must operate inside that authorization boundary.
- **Testing is manual.** There is little to no automated unit testing of MUMPS
  code. The only test tool in use is **M-UNIT** (Joel Ivey) — a 20+-year-old,
  hobbyist-maintained runner that is *not* a modern assertion-based framework.
- **No M IDE.** The IRIS VS Code experience provides **ObjectScript** tooling
  and linting only. There is no M-aware linting, formatting, navigation, or
  language server.
- **No version control of source.** Routine source is **trapped inside the IRIS
  engine** (`IRIS.DAT`), not on the filesystem, so VistA source is not in git.

## 2. The four friction points

| # | Friction point | What VA has today | What we offer |
|---|---|---|---|
| 1 | **No version control** | Routines live inside IRIS; no git history, no PRs, no diffs | Materialize routines to files → VA GitHub Enterprise |
| 2 | **No modern unit testing** | M-UNIT (not assertion-based); mostly manual testing | `m test` + `^STDASSERT` — modern assertion-based suites |
| 3 | **No M IDE/tooling** | ObjectScript-only VS Code; nothing for M | `m lint` / `m fmt` / `m lsp` |
| 4 | **(coverage / profiling)** | None | `m coverage` (same engine dependency as #2) |

The headline value proposition is **#2** — a modern, assertion-based unit
testing tool for MUMPS, which VA simply does not have — closely followed by
**#1** (version control) and **#3** (the M IDE).

## 3. The reframe: engine-neutral source vs. engine-bound execution

The pivotal insight: group the friction points by whether the capability
*executes M code*.

| Capability | Executes M code? | Engine-dependent? |
|---|:---:|:---:|
| Version control (liberate routines from IRIS) | No | **No** — pure file movement |
| Linting / formatting / IDE (LSP) | No | **No** — operates on source text |
| Unit testing | **Yes** | **Yes** |
| Coverage / profiling | **Yes** | **Yes** |

**Three of the four friction points are pure-source operations.** They read and
rewrite `.mac` text and never run it, so they are inherently engine-neutral —
they work identically on IRIS-origin routines whether or not a YottaDB ever
exists in the picture. The YDB-vs-IRIS engine question, which earlier framing
treated as governing the whole product, in fact bites on **only one capability:
execution** (testing and coverage).

Consequence: **most of the value can ship without resolving the engine
question at all.** Version control, the M IDE, lint, and fmt are deliverable on
day one against IRIS-origin source. Only the test/coverage *runner* needs an
execution engine — see §9.

## 4. Version control is the foundation, not a parallel problem

Version control was described as "the huge one." It is — but not as a third
independent track. It is the **enabling first step** on which the other two
depend:

- Linting needs files on disk.
- Git needs files on disk.
- Version-controlled M tests need files on disk.

Until routines are extracted from IRIS onto the filesystem, *none* of the other
tooling can run. The dependency order is therefore fixed:

```
  1. LIBERATE: IRIS routine store ──Atelier REST──► .mac files ──► VA GitHub Enterprise
                                                          │
  2. (now everything else is possible)                   ▼
        ┌──────────────┬─────────────────┬───────────────────────┐
        ▼              ▼                 ▼                       ▼
   m lint / m fmt   m lsp (IDE)    git history / PRs / CI    m test (needs engine — §9)
```

**Concrete first step.** Bulk-export each namespace
(`$SYSTEM.OBJ.ExportAllMatching` or the Atelier `docnames`+`doc` endpoints) into
plain `.mac` files; commit to a **VA GitHub Enterprise** repository
(in-boundary — not public github.com); install an IRIS source-control hook so
subsequent edits flow to git. The moment that lands, VistA source is
version-controlled for the first time and friction points #1–#3 become
addressable with the existing engine-neutral source tools.

> **Format note.** Export must produce *plain, line-faithful* `.mac` source
> (the Atelier / `ExportUDL` representation), **not** the legacy
> `$SYSTEM.OBJ.Export` XML wrapper. XML-wrapped routines are not git-diffable
> and not parseable by `tree-sitter-m`. Verbatim line fidelity also matters for
> coverage's label-relative line decoding.

## 5. Architecture: three interface planes

The deployment splits into three distinct interfaces that are easy to conflate:

```
 VA developer (PIV)                 ── VAEC authorization boundary ──
        │ browser / VS Code Remote                       (Plane 1)
        ▼
 ┌──────────────────────── m-dev-saas (containers) ────────────────────────┐
 │  code-server / Remote-SSH → m lsp · m lint · m fmt · m test · m coverage │
 │   working tree (.mac/.m) — the git checkout                              │
 └───────────────┬──────────────────────────────────────┬──────────────────┘
                 │ git push/pull (Plane 2: M source       │ test execution
                 │ is the contract)                       │ (Plane 3, §9)
                 ▼                                         ▼
        ┌──────────────────┐   CI runner          ┌────────────────────┐
        │ VA GitHub Ent.   │ ───────────────────► │ IRIS test namespace │
        │ repo             │ ◄─────────────────── │ (or embedded YDB)   │
        └──────────────────┘   import + compile   └────────────────────┘
                 ▲                + run tests
        ┌────────┴─────────┐
        │ IRIS VistA dev   │  ← authoritative staged runtimes
        │ test/preprod/prod│
        └──────────────────┘
```

- **Plane 1 — developer ↔ SaaS.** How humans use the tooling: code-server
  (browser VS Code), VS Code Remote-SSH, or Dev Containers hosted in VAEC,
  behind PIV/SAML SSO, with `m lsp` running behind the editor. The easy part.
- **Plane 2 — SaaS ↔ IRIS (source interchange).** How routine source moves
  between the git world and the IRIS routine store. The subject of §6–§8.
- **Plane 3 — SaaS ↔ execution engine.** Where tests actually run. The subject
  of §9.

**Two facts that shrink the problem:**

1. **There is no "JDBC for M routines."** No common runtime RPC exists between
   IRIS and YottaDB. The only neutral interchange is **M source text** — as
   files, in git, or via each vendor's source API (Atelier on the IRIS side).
   Every option is fundamentally a *source-code interchange* design.
2. **Code moves; data does not.** The "no PII" constraint maps onto the
   "defer globals" decision from the materialization plan: the bridge carries
   **routine source only** — never globals. Routine source has no PII; patient
   data does. (Implications in §10.)

## 6. The IRIS interface: Atelier and the filesystem question

IRIS already solved "edit M on the filesystem" — via the **Atelier REST API**
(`/api/atelier/v1/`) and the VS Code ObjectScript extension's two modes:

| | **Client-side editing** | **isfs (server-side)** |
|---|---|---|
| Where source lives | Real `.mac` files in the workspace | Virtual `isfs://` URIs |
| Visible to git / CLI / filesystem tools? | **Yes** | **No** — virtual; only VS Code's in-process provider sees them |
| How edits reach IRIS | export → edit → import + compile | edits go straight to the DB on save |

**Only client-side editing produces real files.** isfs *looks* like files in
the explorer but the bytes are not on disk, so git, ripgrep, pre-commit, and
m-cli see nothing. Any solution that wants version control and filesystem
tooling **must** use the client-side model (export to files), never isfs.

Three caveats define the contract:

- The files are a **projection of the DB, not the runtime** — every edit needs
  an import+compile to take effect; the DB stays source-of-truth for execution.
- **Format matters** — plain UDL/Atelier `.mac`, never XML export (§4).
- **`.mac` is authored source** (round-trippable); **`.int` is generated**
  (read-only, navigation only, never push); **`.cls` is out of scope**
  (ObjectScript classes — VistA is routine-based, not class-based).

## 7. SaaS ↔ IRIS interface options

The menu for Plane 2, ranked for the VA context:

| Option | Mechanism | Best for | Cost / risk |
|---|---|---|---|
| **1. Git as the integration bus** *(recommended)* | Both sides are git clients of a VAEC repo; IRIS side uses a source-control hook / CI to import-on-pull and export-on-change | Decoupling, audit trail, ATO reuse, neutral format | Eventual-consistency, not live; needs the IRIS-side hook wired once |
| **2. Atelier REST direct client** | SaaS speaks `/api/atelier/v1/...` to list/fetch/save/compile against IRIS over the internal network (mTLS) | Low-latency interactive sync | Tight coupling to one instance + IRIS version |
| **3. Shared volume + `$SYSTEM.OBJ`** | IRIS exports UDL to EFS/S3; SaaS reads/writes; `LoadDir`/`ImportDir` re-imports | Bulk / nightly CI loads | Coarse-grained; conflict handling is on you |
| **4. Native API / Embedded Python** | `intersystems-irispython` drives `%RoutineMgr` / IRIS-side harnesses | Driving IRIS-side execution from the SaaS | Most fragile across IRIS versions; deepest coupling |

**Recommendation:** Option 1 for promotion/integration (decoupled, audit-able,
rides already-authorized git infrastructure), complemented by Option 2 for the
interactive inner-loop sync. Neither engine talks directly to the other — both
talk to the repo.

## 8. Interleaving with the InterSystems ObjectScript extension

If both the InterSystems ObjectScript extension *and* the new tooling act on the
same files, the only real hazard is the DB write. The governing principle:

> **There can be only one writer to the DB.**

The subtle failure if two writers exist: the extension's `importOnSave` mutates
the DB out-of-band, bumping the server timestamp; the SaaS's manifest is then
stale and **manufactures phantom conflicts** indistinguishable from real ones —
training users to `--force`, which is exactly when a real concurrent edit gets
clobbered. Timestamp-based conflict detection is unsound the moment a second
writer exists.

Safe configurations assign the DB write to exactly one actor per workspace:

| Model | DB writer | The other tool's role |
|---|---|---|
| **m-cli owns writes** | `m iris push` (manifest-gated) | extension is editor/IntelliSense only (`importOnSave: false`) |
| **extension owns writes** | extension `importOnSave` | m-cli runs pure-source on the exported `src/`; never pushes |
| **dual (read-only mirror)** | extension | m-cli keeps a strictly read-only mirror for analysis |

The recommended desktop posture is **detect-and-defer**: when the extension is
configured for the workspace, m-cli reads `objectscript.conn` for host/namespace
(secret still from its own keychain), points its source provider at the
extension's export folder, and refuses `m iris push` with a clear message.
Two cross-cutting reconciliations regardless of model: **export-layout
alignment** (flat vs. nested package dirs must match the file-stem the workspace
index keys on) and **connection-config drift** (one canonical source for
server/namespace).

## 9. The one engine decision: where unit tests execute

This is the only place the engine question bites. The FedRAMP context flips the
earlier "go YDB-only to simplify" instinct:

| | **Embedded YDB VistA in the SaaS** | **Thin IRIS test-execution adapter** *(recommended for VA)* |
|---|---|---|
| Tests run on | a parallel YDB VistA (synthetic/FOIA, no PII) | VA's actual IRIS engine (a dedicated test namespace) |
| Fidelity to shipping code | gap (Kernel / `$Z*` / error-trap differences) | **exact** — same engine the code ships on |
| New FedRAMP-authorized component? | **Yes — YDB is a new ATO lift in VAEC** | No — IRIS is already authorized |
| Tooling purity | fully YDB-only | a small IRIS *invocation shim* |
| Maintenance | maintain a parallel YDB VistA build | none beyond the shim |

**Key insight:** introducing YottaDB into a FedRAMP HIGH boundary is itself an
authorization lift, while IRIS is already ATO'd. "Go YDB-only" simplifies the
codebase but *increases* deployment friction in this specific regulated
environment — the opposite of the goal. And because the headline value is
"test the code that actually ships," running tests on the real IRIS engine
eliminates the fidelity gap for free.

**The test framework is mostly engine-neutral.** The valuable part of `m test` —
the `^STDASSERT` assertion library, `*TST` discovery conventions, TAP/JUnit/JSON
reporting, `--changed`, isolation, snapshots — is engine-neutral M plus Python
orchestration. Only the "invoke a label, collect STDASSERT output" shim is
engine-specific (the IRIS equivalent of today's `ydb -run ^SUITE` / `%XCMD`
path: an `iris session` pipe or an Atelier run action). That shim is small and
contained — a far smaller commitment than full cross-engine portability.

**Filling a vacuum, not migrating.** Because VA has essentially no existing test
suite (M-UNIT usage is minimal), there is no legacy-conversion burden. Start
clean on the modern assertion-based framework; M-UNIT can coexist untouched
during transition.

**Recommended posture:** keep all source tools engine-neutral (they already
are); for test *execution*, build a contained IRIS adapter that reuses
already-authorized IRIS rather than dragging YottaDB into VAEC.

## 10. Git as source of truth: source vs. state vs. data

Git is the source of truth **for routine source** — both engines consume it;
neither syncs with the other. But a *runnable* VistA is more than its source.
Three distinct kinds of state, with three different owners:

| Kind | Examples | Owner | In git? |
|---|---|---|---|
| **Routine source** | `.mac` / `.m` routines | git | **Yes** |
| **System/config state** | FileMan data dictionary (`^DD`, `^DIC`), KIDS install state, options/protocols | git **via build artifacts** | **Yes, as KIDS builds / DD exports** (PII-free, exportable) |
| **Data** | patient data, test data | a no-PII baseline image | **No** — never |

The clean way to keep git genuinely authoritative is to widen what is versioned
from *just routines* to **routines + KIDS builds + DD/config exports** — all
PII-free, all exportable, all git-friendly. Then a runnable instance is a pure
function of `git + no-PII data baseline`, both engines rebuild from the same
contract, and there is never a stateful long-lived instance to "keep in sync"
with anything. The only thing never in git is patient/test *data*, which is the
baseline's job.

This dissolves an earlier ambiguity: **nothing "syncs with IRIS."** A VistA
instance is either *stateless* (reproducible from git + baseline — the goal) or
*stateful* (accumulates KIDS installs / `^DD` edits / data that live nowhere
else — the anti-pattern to avoid).

## 11. The biggest risk is process, not technology

Everything above is buildable. Adoption hinges on a **culture/process change** —
the single-writer principle (§8) at organizational scale:

- **Git as source of truth means developers stop editing live in the IRIS
  namespace.** Today: edit in the namespace, promote KIDS through stages.
  Tomorrow: edit in git, deploy into the namespace. If developers keep editing
  directly in IRIS (Studio/terminal), git silently drifts and the value
  proposition unravels.
- Needs both a **policy** and a **mechanism**: an IRIS source-control hook that
  **captures or blocks** direct namespace edits so git stays authoritative.
- Manual stage testing remains, but it is now backstopped by automated unit
  tests at the CI gate, and every change is attributable and revertible.

## 12. Staged environments → git promotion + KIDS

The existing `dev → test → pre-prod → prod` pipeline maps cleanly onto a
git-centered flow:

```
  feature branch / PR
        │  CI: m lint + m fmt --check + m test (IRIS test namespace)  ← first automated gate VA has had
        ▼
      main ──tag/release──► KIDS build ──► dev → test → pre-prod → prod IRIS VistA
```

- Branches/PRs gate on `m test` (the first automated gate VA gains).
- Promotion across stages uses **KIDS builds generated from git tags** — the
  formal VA mechanism, now driven from version-controlled source.

## 13. VAEC / FedRAMP HIGH constraints

Constraints that shape (not merely decorate) the design:

- **Stay in-boundary.** All components — container images, the editor host,
  m-cli, any engine — deploy inside VAEC. Use **VA GitHub Enterprise** (not
  public github.com) and a VA-internal container registry (ECR in VAEC). No
  runtime egress to public package registries; vendor dependencies.
- **Reuse existing ATOs.** The marginal authorization cost of a *new* component
  (e.g., YottaDB) can exceed the cost of a small adapter over an
  already-authorized one (IRIS). This is the §9 argument.
- **Auth & audit.** PIV/CAC via VA's IdP (SAML/OIDC); full audit logging.
- **Data posture.** Even "no PII," VistA is healthcare; use synthetic / FOIA
  VistA baselines for any execution engine. Bridge code, never globals (§10).

## 14. Phased roadmap

Each phase is independently valuable and shippable:

| Phase | Deliverable | Unlocks |
|:---:|---|---|
| **1. Liberation + version control** | Atelier bulk export → VA GitHub Enterprise; source-control hook; single-writer policy | The foundation; friction point #1 |
| **2. Source tooling in the SaaS** | code-server / Remote-SSH in VAEC behind PIV SSO, serving `m lsp` / `m lint` / `m fmt` on the git checkout | Friction point #3 — the M IDE IRIS lacks |
| **3. Unit testing** | `^STDASSERT` + `m test` with the thin IRIS execution adapter against a dedicated test namespace; wire as CI gate | Friction point #2 — the headline value |
| **4. Promotion automation** | git tag → KIDS build → staged deploy | Closes the loop into VA's existing pipeline |
| **5. Coverage / profiling** | `m coverage` over the same execution adapter | Friction point #4 |

After Phase 1, VistA source is version-controlled. After Phase 2, VA M
developers have a real IDE. After Phase 3, VA has its first automated,
assertion-based MUMPS testing — on the real engine its code ships on.

## 15. Open decisions

1. **New repo name & scope.** Confirm the new IRIS-focused repository (working
   name `m-dev-saas` / `vaec-vista-dev`) and migrate this document into it.
2. **Test-execution engine (§9).** Confirm the thin IRIS adapter over embedded
   YDB, given the FedRAMP authorization-lift argument. This is the load-bearing
   technical decision.
3. **Single-writer enforcement (§11).** Decide policy + hook mechanism for
   capturing vs. blocking direct IRIS namespace edits.
4. **What is versioned (§10).** Confirm git holds routines + KIDS builds +
   DD/config exports (stateless-instance model), and define the no-PII data
   baseline source.
5. **Editor delivery (Plane 1).** code-server vs. VS Code Remote-SSH vs. Dev
   Containers within VAEC.
6. **Coexistence with the ObjectScript extension (§8).** Detect-and-defer vs.
   own-the-writes, per workspace.

> **Note.** This document is the strategy seed. The engineering substrate it
> relies on is specified in
> [iris-source-materialization.md](iris-source-materialization.md) (the
> `SourceProvider` seam + IRIS sync engine) and
> [iris-ydb-portability.md](iris-ydb-portability.md) (the runtime engine
> adapter). Those remain the implementation references; this document carries
> the VA-specific *why*, *what-first*, and *where-it-runs*.
