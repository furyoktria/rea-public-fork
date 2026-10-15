# REA deep dive — architecture, contracts, and internals

Personal study notes for this fork.

- **Upstream:** [morluto/rea](https://github.com/morluto/rea) · [rea.tools](https://rea.tools/)
- **Revision studied:** `337d2e3` (2026-10-15), package `rea-agents@6.5.0`
- **License:** MIT (see [`LICENSE`](LICENSE), [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md))
- **This fork** is a study copy, not an official distribution. Get official builds
  from [rea.tools](https://rea.tools/) or [morluto/rea](https://github.com/morluto/rea).

REA is a local-first reverse-engineering toolkit that gives a coding agent (or a
terminal operator) tools to inspect software it has no source for — native
binaries, JavaScript/Electron apps, .NET assemblies, Android packages, firmware,
websites, captures — and to return findings **with their evidence**.

---

## 1. The one-paragraph summary

Software has two faces: source and artifact. REA attacks the artifact face. It
packages 125 MCP tools and a matching CLI behind one story: *ask a question about
a feature, get code, references, and runtime observations back, with provenance
attached*. It does this by (a) owning the messy parts of running analysis engines
(Hopper, Ghidra, IDA, decompilers, browser DevTools, JADX, binwalk, servers),
(b) normalizing everything into one Evidence record format, and (c) refusing to
pretend it knows things it does not — unknowns stay unknown rather than silently
degrading to "absent".

## 2. The thesis behind the project

Most "AI + reverse engineering" glue fails in one of two ways:

1. **It hides uncertainty.** The agent asks for a decompilation, gets noisy
   pseudo-C, and narrates it as fact. REA's answer is a first-class `unknown`
   concept, coverage declarations (`complete` vs `unknown`), and limitations
   attached to every result.
2. **It lies about authority.** Running a program, injecting, mutating, or
   launching a GUI are different powers. REA separates observations,
   derivations, and inferences in the contract, and each tool declares its
   process/filesystem/network/UI/mutation effects truthfully
   (`docs/tool-design.md`).

The design rule that follows is `AGENTS.md`'s **fail-closed integration**:
don't integrate a capability while scope, authority, side effects, or required
verification are unresolved. The roadmap repeats it for new providers
(`docs/roadmap.md`).

## 3. Two front doors, one workflow core

```
                 ┌────────────────────────┐
   terminal ────▶│  rea (CLI)             │──┐
                 │  scripts/rea.mjs       │  │
                 └────────────────────────┘  │      ┌──────────────────────┐
                                             ├─────▶│ application workflows│──▶ providers
                 ┌────────────────────────┐  │      │ src/application/     │
  coding agent ─▶│  MCP stdio server      │──┘      └──────────────────────┘
                 │  src/main.ts + src/    │
                 │  server/               │
                 └────────────────────────┘
```

- `scripts/rea.mjs` is the executable dispatcher: it routes `mcp` / `--mcp` to
  the stdio server and everything else to the one-shot CLI.
- `src/cli.ts` + `src/cli/**` (≈50 command modules) and `src/main.ts` +
  `src/server/**` (51 `register*.ts` modules, 125 `server.registerTool` calls)
  are **adapters**. Both call the same `src/application/**` workflows, which
  call the same providers. The CLI and MCP are not two implementations; they are
  two bindings to one workflow core.

This is why `docs/cli.md` can say "the CLI uses the same application workflows
and evidence contracts as REA's MCP server" without it being marketing: the
structural guarantee is enforced by module boundaries
(`npm run verify:module-boundaries`).

Setup (the part users actually touch) is agent registration plus a packaged
skill. `npx rea-agents setup` proposes changes, asks approval, writes config with
backups, and supports **24 clients** — Claude Code, Claude Desktop, Codex,
Cursor, Gemini CLI, Windsurf, Devin, OpenCode, Antigravity, GitHub Copilot CLI,
Command Code, Qwen Code, Kimi Code, VS Code, Grok Build, OMP, Pi, Hermes, Qoder,
Factory Droid, Trae, WorkBuddy, ZCode, DeepSeekHarness (`docs/installation.md`).

## 4. Layered architecture

From `docs/architecture.mermaid`, the dependency rule is inward:

```
domain  ◀──  contracts  ◀──  provider adapters  ◀──  application workflows  ◀──  CLI / MCP adapters
(pure)       (schemas)       (Hopper, Ghidra,        (compose + evidence)        (transport)
                             IDA, browser, …)
```

| Layer | Location | What lives there |
| --- | --- | --- |
| Domain | `src/domain/` | Pure, side-effect-free modules: `evidence.ts`, `analysisSnapshot.ts`, comparisons, application graphs, residual unknowns |
| Contracts | `src/contracts/` | Caller-visible Zod schemas for every tool surface, plus the shared error algebra (`errorSchemas.ts`) |
| Providers | `src/hopper/`, `src/ghidra/`, `src/ida/`, `src/browser/`, `src/android/`, `src/dotnet/`, `src/firmware/`, `src/wasm/`, `src/evm/`, `src/rizin/`, `src/gdb/`, `src/objdump/`, `src/jeb/`, `src/kuna/`, … | Engine protocols, parsing, lifecycle. Engine-specific code stays here |
| Application | `src/application/` | Shared CLI/MCP workflows: sessions, snapshots, investigation records, JavaScript reconstruction |
| Adapters | `src/cli/`, `src/server/`, `src/main.ts` | Transport, registration, result encoding |
| Composition | `src/composition/` | Fresh typed provider/session factories; one-shot binding; optional provider loading |
| Foundation | `src/process/`, `src/windows/`, `native/windows/` | Owned process groups, private roots, deadlines, cleanup; Windows job objects via a Node-API addon |

The **process foundation** is the most load-bearing part people skip. A PID and
an executable path do not prove ownership after exit or reuse, so REA has
`src/process/` supervision primitives and reuses them everywhere (Ghidra, JADX,
firmware tools, browser, source-map workers). `AGENTS.md`: "Retain ownership of
resources whose cleanup failed." That is why the project can claim reliability
under cancellation, not just on the happy path.

## 5. The concepts worth stealing

### 5.1 Evidence

Evidence is the canonical record produced by every Evidence-producing tool. Per
`docs/mcp-contracts.md`:

- Results carry the complete canonical Evidence record in **both** text and
  structured content.
- `structuredContent.normalized_result` is the operation result;
  `structuredContent.evidence_id` is its identity; `raw_result` holds a
  provider representation (or `null`).
- `analysis_profile` is always present (object or explicit `null`) and
  participates in semantic identity — records omitting it are rejected.
- Evidence produced by one tool can be passed directly to a compatible
  comparison tool (`analyze_function` → `compare_functions`,
  `inspect_artifact` → `compare_artifacts`) — the schema (`src/domain/evidence.ts`)
  carries subject identity (`name`, `sha256`, `local_path`, `format`,
  `architecture`), provider identity (`id`, `name`, `version`), and coverage.

Coverage is producer-owned and explicit: `coverage.scope` (`root` vs
`container-entries`) and `coverage.status` (`complete` vs `unknown`). Diagnostics
may explain; they may not infer coverage.

### 5.2 Unknowns as a first-class value

`src/domain/residualUnknown.ts`, `reconstructionUnknowns.ts`,
`unknownRegistryError.ts`. An unknown is not an error and not an absence. The
distinction is enforced across boundaries: "Unresolved identity or applicability
must remain unknown rather than be treated as absent, foreign, or aligned"
(`AGENTS.md`). There is even an MCP prompt, `audit_residual_unknowns`, to review
the open unknowns at the end of an investigation.

### 5.3 Sessions, targets, snapshots

`SessionProviderRouter` + `BinarySession` hold **one immutable deep binding per
target**, with disjoint auxiliary routing. `AnalysisProviderRegistry` returns
sorted deep candidates and selection is **deterministic, with no fallback** — if
it's ambiguous, the caller chooses (`capability_unavailable` with
`selection_reason: ambiguous` and `candidate_ids`). `AnalysisSnapshotCache` +
`BinarySessionRecords` keep binary target/profile snapshots with a compatibility
facade and post-commit notifications.

The practical consequence: a session on one MCP connection stays coherent
(evidence for target A can't be silently compared against target B), and
provider cold-start pain is survivable — Ghidra first-query deadlines are
documented and tunable (`REA_GHIDRA_STARTUP_TIMEOUT_MS`).

### 5.4 Providers are bring-your-own

REA does not ship or crack commercial tools. Hopper, Ghidra, and IDA are
detected or user-installed; setup may offer Hopper with explicit approval and
verification (macOS: verified vendor DMG; Linux: verified vendor package via
apt/dnf/pacman). IDA integrates through upstream `ida-pro-mcp` (GUI proxy or
owned headless database, read-only — never saves the GUI database). Other
providers: **Kuna** (decompiler), **JEB** (caller-started MCP via
`REA_JEB_MCP_URL`), plus offline `objdump`, `rizin`, `pwntools`/`pwndbg`, WABT
(WASM), EVMole (EVM), JADX (Android), binwalk/unblob (firmware), tshark
(pcap/packets), sigrok-cli (signals), adb, Apktool.

Bridges adapt engines rather than leaking them into the domain:
`bridge/hopper_bridge.py` runs inside Hopper's Python API;
`bridge/ghidra/ReaGhidraBridge.java` is a packaged headless Ghidra script.
Third-party engines are pinned as submodules: `jadx-headless-mcp`, `binwalk`,
`unblob`, `ghidra-nativeaot`, `wakaru`.

### 5.5 Configuration must be additive and idempotent

"Configuration changes must be additive and idempotent, with backups.
Installers must not install or upgrade unrelated software, including Homebrew,
Node.js, npm, Java, or Ghidra." (`AGENTS.md`). Setup shows a plan and requires
approval before writing. `src/config/` parses env with Zod. This is the opposite
of the usual "curl | bash and hope" installer culture.

### 5.6 Reconstruction, formalized

Two documents worth reading in full:

- `docs/reconstruction-obligation-ledgers.md` — turns authenticated Evidence
  into a **deterministic list of reconstruction claims**. A claim closes only
  when a manifest binding supplies a unique owner, required types, every case
  fixture, and a passing verifier whose authority is comparable to the original
  observation. Contradictions or duplicate owners keep it open.
- `docs/reconstruction-readiness.md` — `npm run verify:readiness` runs a
  nine-stage conformance verifier from a clean checkout, including stale/tampered
  replay rejection.

This is the most ambitious idea in the repo: reverse engineering output as
something you can *close*, not merely describe.

## 6. Tool and prompt surface

- **125 MCP tools** registered across **51** `src/server/register*.ts` modules
  (`server.registerTool` count). Patterns: `open_binary`, `analyze_function`,
  `compare_functions`, `trace_crash`, `inspect_artifact`,
  `analyze_javascript_application`, `inspect_analysis_view`,
  `build_reconstruction_obligation_ledger`, browser/Electron observation tools,
  pcap/signal/SQLite/WASM/EVM tools, managed (.NET) tools, ADB/APK/firmware
  tools, session lifecycle tools, Evidence tools.
- **6 MCP prompts** (`docs/mcp-prompts.md`): `investigate_feature`,
  `compare_application_versions`, `verify_reconstruction`, `trace_crash`,
  `audit_residual_unknowns`, `prepare_bounded_process_capture`. Prompts suggest
  and teach; they do not add or invoke tools, and their arguments are rendered
  as untrusted selection data.
- **Tool shapes** (`docs/tool-design.md`): `inspect`, `search`/`list`, `trace`,
  `compare`, `workflow`, `observe`/`capture`. Shapes are task categories, not
  name prefixes. Primitives are preferred; workflows are added only after
  repeated use demonstrates a multi-source outcome.
- Limits are only allowed for real format/protocol/authority/resource
  constraints — "An assumed agent budget is insufficient justification for a
  limit." If a pagination/summary exists, complete Evidence stays accessible.

## 7. Capability matrix (condensed from README)

| Target | Returns | Needs |
| --- | --- | --- |
| Native binaries | Pseudocode, assembly, strings, symbols, calls, references | Hopper / Ghidra / IDA |
| Go binaries | Compiler version, modules, replacements, build settings | Nothing (static ELF/PE/thin Mach-O) |
| Rust binaries | cargo-auditable crates, versions, sources, dep edges | Nothing |
| Offline ELF layout | Sections, segments, symbols/relocations, mitigation candidates | pwntools (Linux x64) |
| WASM | Validation, sections, WAT imports/exports, decoded WAT | WABT 1.0.42 |
| EVM bytecode | Dispatch selectors, offsets, inferred args, mutability | Local Linux x64 worker |
| SQLite | Tables, columns, indexes, views, triggers, selected rows | Node.js 24.15+/26+ native SQLite |
| Recorded crashes | Per-thread registers/signals, optional mappings | pwntools; optional GDB/pwndbg |
| JavaScript / Electron | Modules, imports, source maps, routes, IPC, native add-ons | Node.js/npm (no native engine) |
| Websites | Page structure, scripts, network observations, screenshots | Chrome-family browser |
| Network captures | Requests, responses, payloads, source locations | HAR / SAZ / mitmdump (Linux) |
| .NET assemblies | Metadata, CIL, native deps, build comparisons | Nothing (static) |
| Android APKs | Manifest, classes, decompiled methods, references | Headless JADX + full JDK |
| Android devices | Devices, packages, processes, logs, dumps, screenshots | adb |
| Android resources | Decoded manifest, string tables, locales, versions | Apktool |
| Flutter apps | Dart snapshot hash, per-ABI digests, engine provenance | Nothing (static APK parsing) |
| Firmware | Regions, extraction, native handoffs | Binwalk / Unblob (Linux) |
| Packet captures | Fields, repeated values, filters, decode-as rules | TShark 4.x (Linux) |
| Recorded signals | Decoder annotations, ranges, diagnostics | sigrok-cli (Linux) |
| Packages/resources | Inventories, digests, plists, Apple bundle anatomy | built-in readers |
| Process behavior | Terminal output, interactions, exit/fs observations | Linux/macOS native PTY |
| JEB projects | Engine-backed units, artifact digests, pseudo-code | caller-started JEB MCP |

Static JS and .NET analysis never execute the target. Runtime capture uses the
caller's permissions, and each runtime guide states its effects.

## 8. Repository anatomy (measured)

| Area | Files | Approx. lines |
| --- | --- | --- |
| `src/` | 1,928 | ~444,000 |
| `tests/` | 1,042 (847 `*.test.ts`) | ~253,000 |
| `docs/` | 92 | ~22,500 |
| `scripts/` | 325 | ~70,500 |
| `website/` | 51 | ~17,800 |
| `bridge/` | 28 | ~10,900 |
| `experiments/` | 57 | ~12,800 |

Also: 3,906 tracked files, 206 npm scripts, `CHANGELOG.md` at ~366 KB, 5 pinned
submodules, `native/windows/` Node-API addon (NTFS admission, protected DACLs,
atomic Job Object assignment).

## 9. Selected internals worth studying

- **Source maps** — `src/javascript/sourceMaps/` pins an upstream map codec and
  runs it in an owned, heap-limited Node process, returning complete point
  evidence. A good example of "don't trust a library inside your main process".
- **Browser/CDP** — `src/browser/` does CDP/Playwright capture, script capture,
  native module resolution, and `execution/` coverage, listener and source
  attribution. `src/inspector/` adds passive V8 Inspector discovery sharing the
  same CDP helpers.
- **JavaScript reconstruction** — `src/application/javascript/` +
  `src/artifacts/javascript/` read a directory/ASAR and do AST-only bundle
  projection. `docs/javascript-artifact-reconstruction.md` is the guide.
- **Windows boundary** — `native/windows/src/` (addon.cc, authority.hpp,
  filesystem.cc, process.cc) plus `src/windows/` implement opaque handles, owned
  jobs, and private runtimes; the Node-API layer is where the OS-specific
  authority actually lives.
- **Analysis views** — huge Evidence (a 334 MB Obsidian analysis is cited in
  `docs/cli.md`) is projected by `inspect_analysis_view` into a ~10 KB summary or
  a paged module list, without losing access to the full record. This is the
  context-budget discipline in action.

## 10. Engineering toolchain

- **Runtime:** Node.js `^22.19.0 || ^24.11.0 || >=26.0.0` (`.nvmrc`,
  `package.json#engines`), ESM, TypeScript.
- **Task running:** Turborepo (`turbo.json`) with a pinned version and managed
  agent guidance — including the nice touch that `AGENTS.md` tells agents to
  read the installed `turbo` docs rather than trust training data.
- **Tests:** Vitest with named projects (`acceptance`, `boundary`,
  `cli-boundary`, `process-boundary`, `mcp-boundary`, …), plus
  `test:focused`, `test:changed`, shard runs.
- **Quality:** oxlint + oxfmt, knip (dead code), jscpd (duplication),
  `verify:module-boundaries` (architecture guard), Stryker mutation testing
  (including a semantics-oriented config), `docs:check`, `evidence:check`,
  `licenses:check`, `mcp-catalog:check`.
- **Release:** release-please, Conventional Commits, PR title checks, a PR
  readiness workflow, and a documented human-ownership rule: AI-assisted PRs are
  welcome, but a human must review and own the diff.
- **Verification lanes:** `real-*.yml` workflows for real toolchains
  (Hopper, Ghidra on Windows, browser, Go, EVM, packet capture, crashes, signals,
  source maps, JavaScript recovery, Termux). `docs/testing.md` ranks evidence:
  real public CLI/MCP workflows > production-boundary integration > producer
  goldens > module tests.

## 11. Running it

```bash
# As an agent integration (or: npx rea-agents@latest setup)
npm install --global rea-agents
rea --help

# Static JavaScript application analysis, no native engine required
npx -y rea-agents@latest analyze-javascript-application /path/to/app --json

# Keep evidence in a file, then project a small view
rea analyze-javascript-application /path/to/app --json > app-evidence.json
jq -c '{source:{kind:"inline",evidence:.},view:{kind:"summary"}}' app-evidence.json > app-view.json
rea inspect-analysis-view app-view.json

# Health
rea doctor --json
```

Deep native analysis needs one of Hopper, Ghidra, or IDA configured as a
provider. On Linux, a procps-compatible `ps` must be on `PATH` for the ownership
checks before any supervised child starts.

## 12. What stands out (lessons for my own tooling)

1. **Evidence is the interface.** The same record serves the agent, the CLI
   user, and downstream comparison tools. Provenance is not a side channel.
2. **Unknown is a value, not a failure.** Most agent tooling collapses
   "I couldn't read it" into "it's not there". REA refuses.
3. **One workflow core, two adapters.** CLI/MCP parity is structural, not
   aspirational.
4. **Authority is declared per tool.** Launch/attach/mutate powers are named, and
   installers must not silently upgrade the world.
5. **Deterministic provider selection, no fallback.** Ambiguity is surfaced to
   the caller with candidates instead of guessed away.
6. **Ownership survives cancellation.** Process supervision is a foundation, not
   a helper. Cleanup failures retain ownership.
7. **Reconstruction has a closing condition.** Obligation ledgers and a
   conformance verifier turn "I think this is the algorithm" into something
   falsifiable.

## 13. Suggested reading order

1. `README.md` — capability surface and provider choice.
2. `docs/architecture.mermaid` — the whole map in one file.
3. `docs/tool-design.md` — how tools are justified, shaped, and bounded.
4. `docs/mcp-contracts.md` — result, Evidence, session, cancellation semantics.
5. `docs/cli.md` — commands, provider selection, snapshots, exit statuses.
6. `docs/installation.md` — setup, registration, doctor, prerequisites.
7. `docs/reconstruction-obligation-ledgers.md` + `docs/reconstruction-readiness.md`.
8. `AGENTS.md` — the engineering constitution (authority, performance order,
   cleanup, test policy).
9. `CONTRIBUTING.md` + `docs/testing.md` — how changes are verified.
10. `docs/roadmap.md` — where it goes next.

## 14. Attribution

REA is MIT-licensed work by the [morluto/rea](https://github.com/morluto/rea)
maintainers and contributors (notably N0zoM1z0, credited in `website/README.md`).
This fork preserves upstream history, license, third-party notices, and
attribution. These notes are my own study summary of the revision above and may
drift from later upstream releases — always check upstream `main` before relying
on a detail.
