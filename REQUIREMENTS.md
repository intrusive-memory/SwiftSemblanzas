---
type: requirements
updated: 2026-07-23
status: draft
---

# SwiftSemblanzas — Requirements

**Character-consistent storyboard generation on Apple Silicon.** SwiftSemblanzas
turns a small set of source photos (± a prompt) into a *reusable trained
character identity*, then renders short, coherent storyboard **sequences** in
which that character stays recognizably the same across panels.

Part of the [intrusive-memory](https://github.com/intrusive-memory) ecosystem.
Library + CLI first; GUI later.

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

- **F1 — Character intake.** Read a `Character` (+ its source photos and
  optional base prompt/description) from a SwiftData store.
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

### 4.2 Ephemeral composition — the "cast" (Phase 2+)

The per-scene mux ("for this panel: these entities at these scales + this
prompt + these reference images") is a **value type assembled at render time**,
not a persisted training artifact. Working name: **`Reparto`** (Spanish
"cast"). Deferred to Phase 2; noted here so Phase-1 data shapes don't preclude
it.

### 4.3 On-disk layout

Reuse the Vinetas `CharacterManager` convention where practical:
`characters/<slug>/{source, references, training, lora}/`. Heavy bytes here;
pointers in SwiftData.

---

## 5. Training approach (the "what kind of training" decision)

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

## 6. Roadmap

| Phase | Deliverable | Key new work |
|---|---|---|
| **1** | Single character → 3-panel sequence (this doc's focus) | SwiftData catalog; un-stub training → `Flux2Core`; sequence render; DINOv2 QA gate; CLI |
| **2** | Multi-character scenes ("Dan and Rick talk") | `Reparto` composition type; widen engine API from one-adapter to fuse-a-cast + reload-base; cast-batched generation loop; per-entity identity QA |
| **3** | Props | 3D-model → turntable render → captioned training set (greenfield — nothing in the collection reads 3D today); or prop-as-reference-image path |
| **4** | Locations | Location LoRA *or* reference-conditioning-only (backgrounds are more forgiving than faces) |
| **later** | GUI | Only after library + CLI are proven |

---

## 7. Dependencies (provisional)

- `flux-2-swift-mlx` (`Flux2Core`) — training + inference. **Primary.**
- `SwiftAcervo` — model discovery/download/cache.
- `SwiftVinetas` — reuse Understanding (DINOv2) + storyboard orchestration, or
  lift the relevant pieces. **Open question (§8).**
- `SwiftCompartido` / `glosa-av` — screenplay/`<shot>` parsing, if sequences are
  driven from scripts later.

---

## 8. Open questions / decisions to make before coding

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

---

## 9. Critical build rules (inherited)

- **Never `swift build` / `swift test`.** Use `make` targets, then
  XcodeBuildMCP locally; `xcodebuild` in CI.
- **`ARCHS=arm64 ONLY_ACTIVE_ARCH=YES`** — MLX is arm64-only.
- macOS 26+ / Swift 6.2+ runners; pin iOS Simulator OS exactly.
- Sibling `Package.swift` pattern for `intrusive-memory/*` deps; keep
  `Package.resolved` out of git for a library.
