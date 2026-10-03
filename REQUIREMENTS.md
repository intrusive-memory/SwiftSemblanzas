---
type: requirements
updated: 2026-09-25
status: draft
---

# SwiftSemblanzas — Requirements

SwiftSemblanzas has two jobs:

1. **The character record (§5)** is the one shared definition of what a
   character *is* on disk. It covers the naming rule, how a character is
   found through `CAST.md`, the per-character `manifest.json`, `CHARACTER.md`,
   the asset kinds each role tier needs, and the `dossier.html` that presents
   it all. Every app that reads or writes character assets does so through
   this library: Personaje, Produciesta, Vinetas and SwiftEchada's tooling.
2. **Character-consistent image generation on Apple Silicon (§3).** It turns
   a small set of source photos (± a prompt) into a *reusable trained
   character identity*, then renders short, coherent storyboard **sequences**
   in which that character stays recognizably the same across panels.

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

- **F1 — Character intake.** Read a character's record (§5): its `CAST.md`
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
  canonical UPPERCASE name (§5.1) and rebuildable from `CAST.md` plus the
  manifests. It holds the training-asset metadata
  (`TrainingImage`, `LoRAAsset`) that has no business in a git-tracked markdown
  file, and nothing else that `CAST.md` already states.
- **Write-back goes through SwiftReparto.** Anything that must land in
  `CAST.md` (the member's `manifest:` pointer, §5.2) is written by handing a
  mutated `CastDocument` back to `CastMarkdownParser`. This package never
  serializes `CAST.md` itself; see SwiftReparto `RQ-INV-2` (one writer).
- **Trained adapters are recorded in the manifest**, not in `CAST.md`: the
  `.safetensors` is a `lora` asset with its trigger word in the asset entry
  (§5.3). This supersedes the earlier plan to write `appearance.lora` /
  `appearance.triggerWord`. **`appearance` is deprecated** (SwiftReparto
  RQ-25, decided 2026-09-25).

The practical consequence: a character's face and their voice cannot disagree
about who they are, because both hang off the same roster entry, and cast
changes stay reviewable in a PR diff.

### 4.4 On-disk layout

The character directory is defined by the character record (§5.2):
`characters/<CHARACTER>/`, where `<CHARACTER>` is the exact UPPERCASE
screenplay name. Training work uses subdirectories inside it:

```
characters/THE PRACTITIONER/
  manifest.json  CHARACTER.md  dossier.html      # §5
  images/        voice/                          # §5.4 asset kinds
  source/        references/   training/   lora/ # this section
```

Heavy bytes live here, pointers live in SwiftData, and every file that
counts is listed in the manifest. This **replaces** the earlier plan to reuse
Vinetas' lowercase `characters/<slug>/` convention. Vinetas' `deriveSlug`
stays internal to Vinetas' own `~/Library` store and never names anything in
a production.

---

## 5. The character record (shared contract)

This section is the contract every app uses to read and write characters. It
has no dependency on MLX or FLUX. It ships as its own Foundation-only library
target (§5.8), so any app can read characters without linking the training
stack.

### 5.1 Names: exactly the screenplay's, in UPPERCASE

A dramatic character has **one name**, the one its screenplay cue uses, in
uppercase. It's written identically everywhere: in the screenplay, as the
`CAST.md` `character:` value, as the directory name, as the manifest's
`character`, as the `CHARACTER.md` `character:` value, in file names that
carry the name (`voice/THE PRACTITIONER.vox`), and in CLI arguments. There
are **no slugs** and no lowercase or derived forms.

Canonicalization is **implemented in SwiftReparto** as `CastMember.canonicalName(_:)`
(SwiftReparto REQUIREMENTS RQ-22), so every consumer shares one implementation,
including SwiftEchada, which doesn't link this package. It's applied to every
name read from a screenplay, typed in, or found in `CAST.md`:

1. Strip cue extensions: `(V.O.)`, `(O.S.)`, `(O.C.)`, `(CONT'D)`, and any
   other trailing parenthetical. `NARRATOR (V.O.)` becomes `NARRATOR`. The
   Fountain `@` force-marker is also removed.
2. Trim, and collapse runs of whitespace to one space.
3. Uppercase, because Fountain allows mixed case after `@`: `@McCLANE`
   becomes `MCCLANE`.
4. Keep everything else: punctuation, digits and accents. `DR. O'NEILL`,
   `GUARD #2` and `JOSÉ` are all valid. Store names in Unicode **NFC**.
5. A name containing `/` or `:`, or starting with `.`, is an **error** to be
   fixed in the screenplay. There is never silent substitution, because a
   substituted name wouldn't be the screenplay's name.

Since only uppercase forms are created, case-insensitive APFS collisions
(`NARRATOR/` vs `narrator/`) can't arise. Writers refuse to create a
directory that differs from an existing one only by case. Names contain
spaces, so every path consumer must quote or percent-encode.

### 5.2 Discovery: CAST.md → manifest

**Discovery always starts at `CAST.md`.** Each member that has assets
carries an explicit `manifest:` path, relative to the directory holding
`CAST.md`:

```yaml
---
type: cast
cast:
  - character: THE PRACTITIONER
    role: supporting
    manifest: "characters/THE PRACTITIONER/manifest.json"
    voicePrompt: "…"
    voices:
      voxalta:
        - "characters/THE PRACTITIONER/voice/THE PRACTITIONER.vox"
  - character: GUARD #2
    role: background            # no assets, so no manifest
---
```

Invariants, enforced on read by `CharacterRecord` and on write by the
manifest writer:

- **The manifest's parent directory name equals `character` exactly.**
  `characters/THE PRACTITIONER/manifest.json` is valid for
  `THE PRACTITIONER`, and `characters/practitioner/…` is an error. The
  grandparent (`characters/`) is a convention, not a rule.
- **The manifest's `character` field equals the CAST.md `character`.** On a
  mismatch the record doesn't load, and the error names both values.
- **A member without `manifest:` has no assets.** That's valid for
  background roles. Nothing is found by scanning directories: a
  `characters/X/` folder with no CAST.md member pointing at it is an
  **orphan**. It's reported, never loaded, and never deleted.
- **Adoption.** `manifest refresh` is the one exception to the scanning rule.
  If an orphan folder's name exactly equals a member that has no `manifest:`
  (e.g. a folder SwiftEchada created for a new voice, per SwiftEchada EC-23
  rule 3), refresh creates the manifest and sets the pointer through
  Reparto. Refresh never adopts a folder that doesn't match a name exactly.
- The pointer is written through SwiftReparto (§4.3). `manifest` and `role`
  become first-class `CastMember` fields in **SwiftReparto 0.2.0** (its
  REQUIREMENTS RQ-19 to RQ-21, drafted). Until that ships they're carried as
  `extraKeys`, which Reparto already round-trips.

### 5.3 The manifest

`manifest.json` inventories the character directory. It **follows
SwiftAcervo's `CDNManifest`**, so integrity rules are identical across the
suite.

```json
{
  "manifestVersion": 1,
  "character": "THE PRACTITIONER",
  "role": "supporting",
  "updatedAt": "2026-09-25T18:04:00Z",
  "generator": { "name": "personaje", "version": "1.0.0" },
  "files": [
    { "path": "CHARACTER.md",                    "sha256": "…", "sizeBytes": 4120 },
    { "path": "images/turnaround-3f2a.png",      "sha256": "…", "sizeBytes": 2210034 },
    { "path": "lora/THE PRACTITIONER-v2.safetensors", "sha256": "…", "sizeBytes": 38011904 },
    { "path": "voice/THE PRACTITIONER.vox",      "sha256": "…", "sizeBytes": 88211 },
    { "path": "voice/sample-neutral.m4a",        "sha256": "…", "sizeBytes": 190442 }
  ],
  "assets": [
    {
      "path": "images/turnaround-3f2a.png",
      "kind": "turnaround",
      "status": "selected",
      "label": "Turnaround, overcoat",
      "createdAt": "2026-09-25T17:40:12Z",
      "provenance": {
        "provider": "openai", "model": "gpt-image-1",
        "prompt": "Character turnaround reference sheet of …",
        "references": ["source/01.jpg"], "seed": null
      },
      "review": { "passed": ["allAnglesPresent", "consistentDesign"], "notes": "" }
    },
    {
      "path": "lora/THE PRACTITIONER-v2.safetensors",
      "kind": "lora",
      "status": "selected",
      "lora": { "triggerWord": "prctnr_v2", "rank": 16, "steps": 1500,
                "defaultScale": 0.8, "baseModel": "klein4B_base_bf16" }
    },
    {
      "path": "voice/sample-neutral.m4a",
      "kind": "voiceSample",
      "status": "selected",
      "variant": "neutral",
      "derivedFrom": "voice/THE PRACTITIONER.vox",
      "sampleText": "…"
    }
  ],
  "manifestChecksum": "…"
}
```

**There are two layers.**

- **`files`** has *exactly* Acervo's `CDNManifestFile` shape: `path`,
  `sha256` and `sizeBytes`, all required, nothing else. Acervo's rules apply
  unchanged:
  - Paths are relative to the manifest's directory. A leading `/` is
    stripped, and empty, `.` and `..` segments are rejected.
  - Files are sorted by `path`.
  - `manifestChecksum` is the SHA-256 of the **sorted `sha256` strings**
    joined together. Follow Acervo's code (`CDNManifest.swift:263`), not its
    prose.
  - Zero-byte files are refused.
  - The scan skips `manifest.json`, hidden files, `.DS_Store`, symlinks and
    non-regular files.
  - Writes are atomic, `.prettyPrinted` + `.sortedKeys`, and are re-read and
    verified after writing.
- **`assets`** is the semantic layer, keyed by `path`. Each entry records
  `kind`, `status`, `variant`, `label`, `provenance`, `review`, and
  kind-specific data (`lora`, `derivedFrom`, `sampleText`). Every asset has
  a `files` entry. A file without an asset entry is **unclassified**, which
  is allowed and gets reported.

| Top-level field | Required | Notes |
|-----------------|:--------:|-------|
| `manifestVersion` | ✅ | Int, currently `1`. Unknown versions are rejected, as in Acervo. |
| `character` | ✅ | The canonical name (§5.1). It must equal both the directory name and the CAST.md member. |
| `role` | — | A copy of the CAST.md tier. CAST.md wins on conflict. |
| `updatedAt` | ✅ | ISO-8601, `.withInternetDateTime`. |
| `generator` | — | Name and version of the writing tool. Acervo has no equivalent. |
| `files` | ✅ | Acervo shape. |
| `assets` | ✅ | May be empty. |
| `manifestChecksum` | ✅ | Acervo algorithm. |

**`status`**: `selected` counts toward completeness, and there is at most one
per `kind` + `variant`. The others are `candidate` and `rejected`. Rejected
files are kept until explicitly pruned.

**Who registers what.** Tools that create files register them through the
manifest writer: Personaje for images and samples, this package for `lora`
and `training`. SwiftEchada is **not** required to link SwiftSemblanzas. A
`.vox` written by `echada` shows up as an unclassified file, and the next
Personaje or `semblanzas manifest refresh` run adopts it as `kind: vox`.

### 5.4 Asset kinds and role tiers

**Where the tiers come from.** SAG-AFTRA contracts formally distinguish
**principal performers** from **background actors**, and have an
**under-five** category (five lines or fewer) and voice-over categories.
Lead, supporting, guest star and co-star are the industry's casting and
billing tiers built on that split. `role:` in CAST.md takes one of:

| `role` | Basis | Meaning |
|--------|-------|---------|
| `lead` | Principal, series regular | Carries the story |
| `supporting` | Principal, series regular / recurring | Significant arc, recurs |
| `guest` | Principal, single-episode featured | Major within one episode |
| `narrator` | Voice-over principal | Voice only |
| `costar` | Principal, day performer | A few scenes, functional role |
| `underfive` | Under-five performer | ≤ 5 lines |
| `background` | Background actor | No lines, or walla only |

**Major** = `lead`, `supporting`, `guest`, `narrator`. This matches the
CAST.md body's planned Major/Minor sections (Reparto RQ-8). A tier can be
*suggested* from screenplay line and episode counts, but it's never applied
without confirmation.

**Required assets.** ✅ required · ◐ recommended · — not needed

| Asset | `kind` | lead | supp. | guest | narr. | costar | u/5 | bkgd |
|-------|--------|:----:|:-----:|:-----:|:-----:|:------:|:---:|:----:|
| CHARACTER.md, full | `description` | ✅ | ✅ | ✅ | ✅ | — | — | — |
| CHARACTER.md, short | `description` | — | — | — | — | ✅ | ✅ | — |
| Portrait / headshot | `portrait` | ✅ | ✅ | ✅ | ◐ | ✅ | ◐ | — |
| Turnaround (front, ¾, side, back) | `turnaround` | ✅ | ✅ | ✅ | — | ◐ | — | — |
| Face sheet (front, ¾, profile) | `faceSheet` | ✅ | ✅ | ◐ | — | — | — | — |
| Expression sheet (≥ 6) | `expressions` | ✅ | ◐ | ◐ | — | — | — | — |
| Wardrobe, one per costume | `wardrobe` | ◐ | ◐ | — | — | — | — | — |
| Color palette | `palette` | ✅ | ✅ | ◐ | — | — | — | — |
| Source photo(s) | `sourcePhoto` | ◐ | ◐ | ◐ | ◐ | ◐ | — | — |
| Character LoRA | `lora` | ◐ | — | — | — | — | — | — |
| Training set | `training` | *(with lora)* | | | | | | |
| Crowd reference | `crowdRef` | — | — | — | — | — | — | ◐ |
| Voice prompt | *(CAST.md)* | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| `.vox`, 1.7B | `vox` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| `.vox`, 0.6B | `vox` | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ | — |
| Voice sample, neutral | `voiceSample` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| Voice samples, range (≥ 3) | `voiceSample` | ✅ | ◐ | ◐ | ◐ | — | — | — |
| Pronunciation guide | *(CHARACTER.md)* | ✅ | ✅ | ◐ | ✅ | — | — | — |
| Dossier | `dossier` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |

- **Voice samples are `.m4a` renders** of fixed sample text, made from the
  `.vox`. The dossier must play something, and browsers can't play `.vox`.
  The default range set is neutral, angry, quiet and joyful.
- **Complete** means every ✅ for the tier exists as a `selected` asset. The
  table is data in the library (`RoleTier.requirements`), not prose that
  apps re-implement.

### 5.5 CHARACTER.md

This is the written half of the record: front matter for facts, and a body
for prose.

```markdown
---
type: character
character: THE PRACTITIONER     # must equal CAST.md exactly
role: supporting
age: "40s"
pronouns: she/her
occupation: …
---
## Logline
## Appearance
## Personality
## Backstory
## Arc
## Relationships
## Voice & Speech
## Pronunciation
## Wardrobe
```

- **Full** (major tiers) has every section. **Short** (`costar`,
  `underfive`) has Logline, Appearance, and Voice & Speech.
- In `## Relationships`, other characters are referred to by their exact
  UPPERCASE name, so the dossier can link to them.
- The CAST.md body keeps its one-paragraph summaries (Reparto D5). CHARACTER.md
  is the long form.
- Unknown front-matter keys are preserved, following SwiftProyecto's
  app-section pattern.

### 5.6 The dossier

`characters/<CHARACTER>/dossier.html` is the presentation of a complete
character: every selected asset on one page, with a playable voice. The
**format and renderer are defined here** (`DossierRenderer`), so every app
produces and reads the same page.

**Properties:**

- **Generated, never hand-edited.** It's rendered from the CAST.md member,
  CHARACTER.md and the manifest. It's registered as `kind: dossier` with
  `derivedFrom` hashes of its inputs, so a stale dossier can be detected.
- **Self-contained and offline.** It's a single file with inline CSS and
  minimal inline JS. There are no network requests, web fonts or CDNs. It
  links to assets by relative, percent-encoded path, so it works over
  `file://` and inside a zip.
- **Audio** uses native `<audio>` pointing at `voice/*.m4a`. Playback is
  click-only, never autoplay.
- **Accessible:** light and dark mode (`prefers-color-scheme`), a print
  stylesheet (players hidden, sample text shown), and usable at phone width.
- **Machine-readable too.** The page embeds the record as
  `<script type="application/json" id="character-record">`, so tools that
  only have the HTML can still read it.

**Layout:**

```
┌──────────────────────────────────────────────────────────────┐
│ [portrait]  THE PRACTITIONER                   Supporting    │
│             "Logline."   Played by: …   ▶ Hear the voice     │
├──────────────────────────────────────────────────────────────┤
│ Facts: age · pronouns · occupation · height · language        │
├──────────────────────────────────────────────────────────────┤
│ LOOK  [ turnaround, full width ]                              │
│       [ face sheet ] [ expressions ]  Wardrobe · Palette ■■■■ │
├──────────────────────────────────────────────────────────────┤
│ VOICE  voicePrompt · ▶ neutral ▶ angry ▶ quiet ▶ joyful       │
│        models: 1.7B ✓ 0.6B ✓ · pronunciation                  │
├──────────────────────────────────────────────────────────────┤
│ STORY  personality · backstory · arc · relationships (links)  │
├──────────────────────────────────────────────────────────────┤
│ COMPLETENESS  supporting: 10/10 required ✓ · suggestions: …   │
├──────────────────────────────────────────────────────────────┤
│ ▸ Provenance: per-asset provider/model/prompt/date · LoRA      │
│   trigger word · manifest checksum · generator version        │
└──────────────────────────────────────────────────────────────┘
```

**Content rules:**

- Only `selected` assets are shown. Candidate and rejected assets appear as
  counts under Provenance.
- Missing ✅ items are listed by name. ◐ items appear as suggestions.
- Relationships link to `../<OTHER CHARACTER>/dossier.html`, but only when
  that member exists in CAST.md and has a dossier.
- Integrity problems (missing files, hash mismatches, stale selections, or a
  name mismatch) appear as a banner at the top.

**Cast index:** `characters/index.html` is generated from CAST.md order. It
shows each member's portrait, name, tier and ▶ voice, grouped Major / Minor,
with links to the dossiers.

### 5.7 Verification

`verify` (library API and `semblanzas verify`) walks CAST.md and reports, per
member:

- a missing or invalid `manifest:`
- a name mismatch (§5.2)
- files that are missing, modified (by hash) or unclassified
- a stale dossier
- completeness against the tier

It also reports orphan directories. It never modifies anything. The
`.acervo-verified.json` marker pattern may be reused to skip re-hashing.

### 5.8 Library shape

| Target | Depends on | Contains |
|--------|------------|----------|
| **`SemblanzasRecord`** | Foundation, CryptoKit, SwiftReparto | `CharacterRecord` (load a member via CAST.md), `CharacterManifest` + writer, `AssetKind`, `RoleTier` + requirements, `CharacterDescription` (CHARACTER.md), `DossierRenderer`, `CastIndexRenderer`, `Verifier` |
| **`SwiftSemblanzas`** | `SemblanzasRecord`, Flux2Core, SwiftAcervo | Training, sequence generation, QA (§3) |
| **`semblanzas`** CLI | both | `verify`, `manifest refresh`, `dossier`, `train`, `sequence` |

Personaje, Produciesta and Vinetas link **`SemblanzasRecord`** only, unless
they train.

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
| **0** | The character record (§5), needed first by Personaje | `SemblanzasRecord` target: naming, CAST.md discovery, manifest read/write/verify, tier requirements, CHARACTER.md, dossier + cast-index renderers; `semblanzas verify` / `manifest refresh` / `dossier` |
| **1** | Single character → 3-panel sequence | SwiftData catalog; un-stub training → `Flux2Core`; sequence render; DINOv2 QA gate; CLI |
| **2** | Multi-character scenes ("Dan and Rick talk") | `Convocatoria` composition type; widen engine API from one-adapter to fuse-a-cast + reload-base; cast-batched generation loop; per-entity identity QA |
| **3** | Props | 3D-model → turntable render → captioned training set (greenfield — nothing in the collection reads 3D today); or prop-as-reference-image path |
| **4** | Locations | Location LoRA *or* reference-conditioning-only (backgrounds are more forgiving than faces) |
| — | GUI | **Personaje** (`apps/Personaje`) is the app layer. This package never ships UI. |

---

## 8. Dependencies (provisional)

- **`SwiftReparto`**: the cast list (`CAST.md`), where discovery starts (§5.2), and the only writer
  of that file (§4.3). `SemblanzasRecord` links it. A leaf library: YAML parsing and nothing else, so it costs this package
  nothing to link. **Firm, not provisional.**
- `CryptoKit`: manifest hashing (§5.3). The record target needs nothing heavier.
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
6. **SwiftReparto fields (§5.2).** *Drafted* in SwiftReparto
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
   it adopt the record (§5), or stay a private cache? Recommendation: in a
   production, Vinetas reads characters via `SemblanzasRecord`, and its
   `~/Library` store becomes a cache only.

---

## 10. Critical build rules (inherited)

- **Never `swift build` / `swift test`.** Use `make` targets, then
  XcodeBuildMCP locally; `xcodebuild` in CI.
- **`ARCHS=arm64 ONLY_ACTIVE_ARCH=YES`** — MLX is arm64-only.
- macOS 26+ / Swift 6.2+ runners; pin iOS Simulator OS exactly.
- Sibling `Package.swift` pattern for `intrusive-memory/*` deps; keep
  `Package.resolved` out of git for a library.
