---
type: requirements
updated: 2026-10-19
status: draft
---

# SwiftSemblanzas — Requirements

SwiftSemblanzas does one job: **character-consistent image generation on
Apple Silicon (§3).** It turns
   a small set of source photos (± a prompt) into a *reusable trained
   character identity*, then renders short, coherent storyboard **sequences**
   in which that character stays recognizably the same across panels.

The **character record** it trains from and writes its adapter into is the
`.dossier` bundle, defined by
[dossier-format](https://github.com/intrusive-memory/dossier-format)
(`docs/DOSSIER-FORMAT.md`) and implemented by **SwiftDossier**. It used to be
specified here as §5; that text moved out on 2026-10-19 (§5 is now a pointer).

**Personaje** (`apps/Personaje`) is the app layer on top of this library: the
macOS app and `personaje` CLI that walk a user through creating a character's
assets and end at the dossier. SwiftSemblanzas has no GUI of its own.

Part of the [intrusive-memory](https://github.com/intrusive-memory) ecosystem.

---

## 1. The problem this solves

A storyboard is only a storyboard if the panels look like *one* story — the same
character, prop, and location recur across frames. Text prompts alone drift:
ask FLUX for "a man" four times and you get four different men. SwiftSemblanzas
makes recurring visual entities **first-class, trained, reusable assets**.

The ecosystem already has the pieces this depends on:

- **flux-2-swift-mlx / `Flux2Core`** — a mature on-device LoRA trainer
  (flow-matching, DOP regularization, trigger words, checkpoint/resume) *and*
  the FLUX.2 inference pipeline with native multi-reference conditioning.
- **SwiftVinetas** — storyboard/panel orchestration, a `CharacterManager`
  on-disk layout, and a DINOv2 "Understanding" module (embeddings +
  `cosineSimilarity`) usable as a consistency QA gate.
- **SwiftAcervo** — shared model discovery/download/cache (App Group
  `group.intrusive-memory.models`).

SwiftSemblanzas is the dedicated home for the **character-identity lifecycle**
that Vinetas started but left stubbed (`CharacterTrainer.train()` currently
throws `"LoRA training not yet available"`).

---

## 2. Core mental model (non-negotiable design invariants)

These are the load-bearing truths the whole design rests on. Violating them
produces blended, unrecognizable output.

1. **Train per-entity, once.** Each character gets exactly **one** LoRA,
   trained in isolation with its own trigger word. Three characters = three
   trainings, reused across every scene pairing — *never* one training per
   scene or per pairing.
2. **Never merge training data across entities.** Merging Dan's and Rick's
   photos teaches FLUX one blended person and destroys both identities.
3. **Combination happens at inference, not in training.** "Dan and Rick talk"
   is produced by composing two independently-trained adapters at generation
   time, not by a joint model.
4. **The trained adapter is a durable asset; the scene composition is
   ephemeral.** Adapters (+ their catalog records and source photos) persist
   for the life of the character. The per-scene composition ("this panel needs
   Dan @0.75 + Rick @0.75 + this prompt") is assembled at render time and torn
   down after.
5. **Identity requires training; coherence can come from reference images.**
   FLUX.2's multi-reference conditioning (training-free) gives *same-look*
   coherence but **not** identity lock — there is no IP-Adapter/face-embedding
   path. Faces need a LoRA; backgrounds/props often need only reference images.
6. **Multi-LoRA at inference is a fuse-and-reload operation.** `Flux2Core`
   stacks adapters by *additively fusing* them into the transformer weights,
   then clears the adapter matrices. Consequence: you cannot un-fuse a single
   character — swapping the cast means **reloading the base transformer**.
   Therefore panels sharing a cast **must be generated in a batch** before any
   cast change. (This constrains Phase 2+, not Phase 1.)

---

## 3. Phase 1 — the first deliverable (single character, 3-panel sequence)

> **Scope:** one character, pulled from a SwiftData store as `(photo[s] ±
> prompt)`, producing **three sequential storyboard panels**: *beginning →
> mid-action → end-state*, with one or more actions occurring. Prove it end to
> end, tested, before touching multi-character.

### 3.1 Functional requirements

- **F1 — Character intake.** Read a character's record (dossier-format): its `CAST.md`
  entry, manifest, `sourcePhoto` assets and `CHARACTER.md` description. The
  SwiftData catalog (§4.1) is populated from the record, never the other way
  round.
- **F2 — Training-data prep.** From 1+ source photos, produce a captioned,
  VAE-dimension-valid (÷16) training set. When only a single photo is
  available, synthesize additional views (img2img turnaround, à la Vinetas'
  `ReferenceSheetGenerator`) — flagged clearly as *synthetic* data, since it
  can bake in its own errors.
- **F3 — Train a subject LoRA.** Wire to `Flux2Core` (`TrainingSession` /
  `LoRATrainingHelper` / `SimpleLoRATrainer`). Trigger word + DOP
  regularization. Trained against the **non-distilled Base** weights
  (`klein4B_base_bf16` / `klein9B_base_bf16`) or Dev bf16 — the registry blocks
  training on distilled/quantized variants. Rank modest (16–32) so it learns
  the *face*, not the background.
- **F4 — Persist the adapter** as a durable asset (`.safetensors` on disk +
  catalog record in SwiftData; see §4).
- **F5 — Sequence generation.** Given the trained character + a beat
  description per panel, generate three panels (*beginning / mid-action /
  end-state*) with the character's trigger word auto-composed into each prompt
  and the adapter loaded. Panels share seed/style discipline for continuity.
- **F6 — Consistency QA gate.** Embed each generated panel (DINOv2) and compare
  against the character's source; below-threshold panels are flagged and
  regenerated automatically (bounded retries; log when the cap is hit — no
  silent truncation).
- **F7 — CLI surface.** Every Phase-1 capability reachable from a
  `semblanzas` CLI (`train`, `sequence`, at minimum) with no GUI dependency.

### 3.2 Explicit non-goals for Phase 1

- No multi-character composition (Phase 2).
- No props (Phase 2) or locations (Phase 3).
- No GUI.
- No regional/spatial LoRA (does not exist in the stack; may never be needed).

### 3.3 Phase-1 acceptance criteria

- Train a character from photos and generate a **solo** panel whose DINOv2
  cosine similarity to the source exceeds a defined threshold. *(single-entity
  identity proof)*
- Produce a **3-panel beginning/mid/end sequence** of that character where all
  three panels pass the QA gate and read as the same person performing a
  progression of actions.
- The full flow (SwiftData → train → sequence → QA) runs from the CLI on
  Apple Silicon and is covered by tests that gate CI where they can
  (model-dependent GPU tests are local-only, per ecosystem convention).

---

## 4. Data model

Two planes, deliberately separated.

### 4.1 Durable catalog — SwiftData (metadata only)

SwiftData holds the **index**, never the heavy bytes. Storing multi-MB image
blobs or `.safetensors` inline in SwiftData is a performance anti-pattern;
those live on disk (or `.externalStorage`) referenced by path.

Provisional entities:

- **`Character`** — `name`, `triggerWord`, `description`, base-model/engine
  compatibility, `createdAt`.
- **`TrainingImage`** — `caption`, `view`/angle, `isSynthetic`, file path;
  belongs to a `Character`.
- **`LoRAAsset`** — adapter file path, `rank`, `steps`, checksum,
  `defaultScale`, `version`, `compatibleEngines`, `trainedAt`; belongs to a
  `Character`.

### 4.2 Ephemeral composition — the call sheet (Phase 2+)

The per-scene mux ("for this panel: these entities at these scales + this
prompt + these reference images") is a **value type assembled at render time**,
not a persisted training artifact. Working name: **`Convocatoria`** (Spanish
"call sheet") — it names who is needed for this panel and at what weight, which
is exactly what a call sheet does. Deferred to Phase 2; noted here so Phase-1
data shapes don't preclude it.

> **Renamed 2026-08-01.** This type was previously called `Reparto`. That name
> now belongs to [SwiftReparto](https://github.com/intrusive-memory/SwiftReparto),
> the ecosystem's cast-list library and this package's source of character
> records — see §4.3. Two different things one layer apart could not share a
> name.

### 4.3 Where characters come from — SwiftReparto

The durable catalog in §4.1 is **not** the source of truth for *who exists*.
That is `CAST.md`, owned by **SwiftReparto**, the leaf library both this package
and SwiftEchada consume.

- **`CAST.md` is canonical.** A character exists because it is in the cast list.
- **The SwiftData catalog is a derived index**, keyed by the character's
  canonical UPPERCASE name (dossier-format) and rebuildable from `CAST.md` plus the
  manifests. It holds the training-asset metadata
  (`TrainingImage`, `LoRAAsset`) that has no business in a git-tracked markdown
  file, and nothing else that `CAST.md` already states.
- **Write-back goes through SwiftReparto.** Anything that must land in
  `CAST.md` (the member's `manifest:` pointer, dossier-format §3) is written by handing a
  mutated `CastDocument` back to `CastMarkdownParser`. This package never
  serializes `CAST.md` itself; see SwiftReparto `RQ-INV-2` (one writer).
- **Trained adapters are recorded in the manifest**, not in `CAST.md`: the
  `.safetensors` is a `lora` asset with its trigger word in the asset entry
  (dossier-format). This supersedes the earlier plan to write `appearance.lora` /
  `appearance.triggerWord`. **`appearance` is deprecated** (SwiftReparto
  RQ-25, decided 2026-09-25).

The practical consequence: a character's face and their voice cannot disagree
about who they are, because both hang off the same roster entry, and cast
changes stay reviewable in a PR diff.

### 4.4 On-disk layout

The character directory is the `.dossier` bundle (dossier-format §3, §4):
`characters/<CHARACTER>.dossier/`, where `<CHARACTER>` is the exact UPPERCASE
screenplay name. `characters/<CHARACTER>/` with no extension is an error.
Training work uses subdirectories inside it:

```
characters/THE PRACTITIONER.dossier/
  manifest.json  CHARACTER.md  THE PRACTITIONER-dossier.html   # dossier-format
  images/        voice/                                        # dossier-format §8
  source/        references/   training/   lora/               # this section
```

The adapter is `lora/<CHARACTER>-v<N>.safetensors`, registered as a `lora`
asset with `lora.triggerWord` (dossier-format §8.4); training images are
`training` assets with `{caption, view, isSynthetic}`. SwiftSemblanzas writes
them through SwiftDossier's `Dossier` API, never by editing `manifest.json`.

Heavy bytes live here, pointers live in SwiftData, and every file that
counts is listed in the manifest. This **replaces** the earlier plan to reuse
Vinetas' lowercase `characters/<slug>/` convention. Vinetas' `deriveSlug`
stays internal to Vinetas' own `~/Library` store and never names anything in
a production.

---

## 5. The character record (moved)

Moved to [dossier-format](https://github.com/intrusive-memory/dossier-format)
on 2026-10-19 (SM-P2): names, discovery, the manifest, asset kinds and tiers,
CHARACTER.md, the dossier page and validation are `docs/DOSSIER-FORMAT.md`
there, with JSON Schemas and `data/tiers.json`. The Swift reader, writer and
renderer are **SwiftDossier**. The text that stood here is version 1 of the
format (dossier-format Appendix A) and is in this file's history at
`739720b`.

What SwiftSemblanzas needs from the record:

- **Reads** (through SwiftDossier `DossierCore`): the member via `CAST.md`,
  `sourcePhoto` assets, the selected `portrait`, `face` and `view` assets as
  synthetic-view candidates, `prompts.identity`, and CHARACTER.md Appearance.
- **Writes** (through SwiftDossier `Dossier`): `training` and `lora` assets.
- **Never writes** `CAST.md` (SwiftReparto's) or Reparto's deprecated
  `appearance:` block (RQ-25).

---

## 6. Training approach (the "what kind of training" decision)

- **Subject LoRA** with trigger word + **DOP** (Differential Output
  Preservation). This *is* the modern equivalent of DreamBooth for FLUX — DOP
  supplies the prior-preservation regularization. Full-weight DreamBooth and
  textual inversion are deliberately **not** used (lower fidelity / heavier /
  no in-stack support).
- **Base weights required** for training (distilled Klein and quantized Dev are
  registry-blocked). Klein-4B-Base is the default target for RAM economy;
  Klein-9B-Base / Dev for quality.
- **LoRA targets** attention Q/K/V/O (+ context projections, optionally FFN).
  Text encoder stays frozen (Mistral) — training it caused collapse upstream.
- **Rank 16–32**, modest, to encode identity without memorizing scene context.

---

## 7. Roadmap

| Phase | Deliverable | Key new work |
|---|---|---|
| **0** | ~~The character record~~ | *Moved 2026-10-19:* the format is dossier-format, the library is SwiftDossier. No `SemblanzasRecord` target. |
| **1** | Single character → 3-panel sequence | SwiftData catalog; un-stub training → `Flux2Core`; sequence render; DINOv2 QA gate; CLI |
| **2** | Multi-character scenes ("Dan and Rick talk") | `Convocatoria` composition type; widen engine API from one-adapter to fuse-a-cast + reload-base; cast-batched generation loop; per-entity identity QA |
| **3** | Props | 3D-model → turntable render → captioned training set (greenfield — nothing in the collection reads 3D today); or prop-as-reference-image path |
| **4** | Locations | Location LoRA *or* reference-conditioning-only (backgrounds are more forgiving than faces) |
| — | GUI | **Personaje** (`apps/Personaje`) is the app layer. This package never ships UI. |

---

## 8. Dependencies (provisional)

- **`SwiftDossier`** (`Dossier`): the character record: read the bundle,
  register `training` and `lora` assets. **Firm.**
- **`SwiftReparto`**: the cast list (`CAST.md`), where discovery starts, and the only writer
  of that file (§4.3). Linked through SwiftDossier. A leaf library: YAML parsing and nothing else, so it costs this package
  nothing to link. **Firm, not provisional.**
- `flux-2-swift-mlx` (`Flux2Core`): training and inference. **Primary** for the `SwiftSemblanzas` target.
- `SwiftAcervo` — model discovery/download/cache.
- `SwiftVinetas` — reuse Understanding (DINOv2) + storyboard orchestration, or
  lift the relevant pieces. **Open question (§9).**
- `SwiftCompartido` / `glosa-av` — screenplay/`<shot>` parsing, if sequences are
  driven from scripts later.

---

## 9. Open questions / decisions to make before coding

1. **Vinetas relationship.** Does SwiftSemblanzas *depend on* SwiftVinetas
   (reusing `CharacterManager` + Understanding), *replace* its character layer,
   or depend directly on `Flux2Core` and lift only DINOv2? Recommendation:
   depend on `Flux2Core` directly for train/infer; reuse Vinetas' Understanding
   module for QA; migrate Vinetas' character layer here over time.
2. **Single-photo intake quality.** How much do we lean on synthetic
   turnarounds vs. requiring N real photos? Affects F2 and identity quality.
3. **Sequence continuity mechanism.** Shared seed + style prompt is the
   Phase-1 default; do we also feed panel N−1 as a reference image into panel N
   for pose continuity? (Reference path is orthogonal to LoRA and available.)
4. **QA threshold + retry budget.** Concrete DINOv2 cosine threshold and max
   regeneration attempts per panel.
5. **Base-model default.** Klein-4B-Base (16 GB friendly) vs 9B/Dev quality
   for the reference training config.
6. **SwiftReparto fields (dossier-format).** *Drafted* in SwiftReparto
   `REQUIREMENTS.md` (target 0.2.0): `role` (RQ-19), `manifest` (RQ-20), the
   string-only manifest/name check (RQ-21), and canonical names (RQ-22).
   Phase 0 depends on it.
7. **`appearance:` is deprecated.** *Decided 2026-09-25* (SwiftReparto RQ-25).
   Visual assets live only in the manifest.
8. **Voices move into the character directory.** *Drafted* in SwiftEchada
   `REQUIREMENTS.md` (target 1.1.0):
   - EC-23: output follows the `manifest:` pointer
   - EC-24: exact-name files
   - EC-26: `echada migrate voices`
   - EC-28: `echada sample` for dossier `.m4a` samples

   `voices.voxalta` stays in CAST.md as SwiftEchada's contract, and the
   manifest lists the same files. That duplication is deliberate until
   SwiftEchada reads manifests (SwiftEchada EC-Q2).
9. **Vinetas' character store.** Vinetas keeps characters in
   `~/Library/SwiftVinetas/characters/<slug>/` with `character.yaml`. Should
   it adopt the record, or stay a private cache? *Decided:* in a production,
   Vinetas reads characters through SwiftDossier `DossierCore`, and its
   `~/Library` store becomes a cache only; a dossier's `lora.triggerWord`
   overrides the store's slug default (dossier-format D7).

---

## 10. Critical build rules (inherited)

- **Never `swift build` / `swift test`.** Use `make` targets, then
  XcodeBuildMCP locally; `xcodebuild` in CI.
- **`ARCHS=arm64 ONLY_ACTIVE_ARCH=YES`** — MLX is arm64-only.
- macOS 26+ / Swift 6.2+ runners; pin iOS Simulator OS exactly.
- Sibling `Package.swift` pattern for `intrusive-memory/*` deps; keep
  `Package.resolved` out of git for a library.
