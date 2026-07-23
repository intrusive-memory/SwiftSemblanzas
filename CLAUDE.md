---
type: reference
updated: 2026-07-23
---

# Claude-Specific Agent Instructions — SwiftCharacturas

**Read [REQUIREMENTS.md](REQUIREMENTS.md) first** — it holds the vision, the
non-negotiable design invariants (§2), the Phase-1 scope, and the data model.
This file adds build/tooling guidance on top. An `AGENTS.md` (universal
ecosystem doc) may be added later; until then, treat REQUIREMENTS.md +
this file as authoritative.

---

## 1. What this project is

A Swift **library + `characturas` CLI** that trains reusable per-character
FLUX.2 LoRAs from photos and renders character-consistent storyboard sequences,
on-device via MLX. Library and CLI come first; GUI is deferred. Do not add GUI
code until the library + CLI are proven and tested.

## 2. The design invariants you must not break

From REQUIREMENTS.md §2 — these are load-bearing. If a change violates one,
it's wrong:

1. **Train per-entity, once.** One LoRA per character, reused across all scenes.
   Never one training per scene/pairing.
2. **Never merge training data across entities** — it produces blended,
   unrecognizable output.
3. **Combination happens at inference, not training.**
4. **Trained adapters are durable assets; scene composition is ephemeral.**
5. **Identity needs a LoRA; coherence can use reference images.** There is no
   IP-Adapter/face-embedding path in the stack.
6. **Multi-LoRA inference is fuse-and-reload** — swapping a cast means reloading
   the base transformer, so batch same-cast panels together (Phase 2+).

## 3. Build tools — never `swift build` / `swift test`

The user's global `~/.claude/CLAUDE.md` and this ecosystem forbid `swift build`
and `swift test` (MLX Metal shaders won't compile that way). Order of
preference:

1. **`make` targets** — once a Makefile exists, run `make help`. Prefer these.
2. **XcodeBuildMCP** locally — `swift_package_build`, `swift_package_test`,
   `build_macos`, `test_macos`, `build_sim`, `test_sim`, `list_sims`.
3. **Raw `xcodebuild`** in CI only (XcodeBuildMCP isn't on GitHub runners).

**Always pass `ARCHS=arm64 ONLY_ACTIVE_ARCH=YES`** to `xcodebuild` — MLX is
arm64-only; there is no x86_64 path.

## 4. GitHub Actions

- Runner: `runs-on: macos-26` or later (never `macos-15` or earlier).
- Swift 6.2+.
- iOS Simulator destination pins an exact OS, e.g.
  `'platform=iOS Simulator,name=iPhone 17,OS=26.1'`. `OS=latest` does not work.
- Model-dependent GPU/integration tests are **local-only** and must not gate CI
  unless wired through the cached-model CI pattern (see the
  `acervo-integration-ci` skill).

## 5. Dependencies & packaging

- Primary dependency is **`flux-2-swift-mlx` (`Flux2Core`)** for training +
  inference; also **SwiftAcervo** (model cache) and **SwiftVinetas** (DINOv2
  Understanding / orchestration — see REQUIREMENTS §8 open question on the
  exact relationship).
- Use the ecosystem **sibling `Package.swift` pattern** for `intrusive-memory/*`
  deps (see the `toggle-sibling-libraries` skill).
- Keep **`Package.resolved` out of git** for a library.
- The real LoRA trainer lives in `flux-2-swift-mlx/Sources/Flux2Core/Training/`
  (`TrainingSession`, `LoRATrainingHelper`, `SimpleLoRATrainer`). SwiftVinetas'
  `CharacterTrainer.train()` is a stub that throws — un-stubbing that wiring is
  Phase-1 work and may move here.

## 6. Global rules that apply here

- **Complete candor** — flag ill-advised approaches up front, even unasked.
- **Secrets** — never echo env vars, API keys, or `.env`/`.p8` contents; never
  read `~/.claude/@private` without per-operation consent.
- **Branching** — `main` (protected) and `development`; feature work branches
  off `development`.

---

**Last updated**: 2026-07-23 — initial scaffold (design phase, pre-Phase-1).
