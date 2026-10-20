---
type: doc
updated: 2026-07-23
status: pre-alpha
---

# SwiftSemblanzas

**Character-consistent storyboards on Apple Silicon.** Turn a handful of source
photos into a reusable, trained character identity — then render short
storyboard **sequences** in which that character stays recognizably the same
from panel to panel.

Part of the [intrusive-memory](https://github.com/intrusive-memory) ecosystem.
Ships as a Swift **library** and a `semblanzas` **CLI**. The GUI is
**Personaje** (`apps/Personaje`), the app layer built on this library.

> **Status: pre-alpha / design.** This repo currently contains the design docs
> ([REQUIREMENTS.md](REQUIREMENTS.md)) only. Implementation begins with Phase 1
> (single character → 3-panel sequence). See the roadmap below.

## Why

Text prompts drift — ask FLUX for "a man" four times and you get four different
men. A storyboard needs the *same* character across frames. SwiftSemblanzas
makes recurring visual entities (characters first; props and locations later)
**first-class, trained, reusable assets**, built on the ecosystem's existing
FLUX.2 / MLX training and inference stack.

## How it works (the short version)

1. **Intake** a character from the production's cast list (`CAST.md`, owned by
   **SwiftReparto**): source photo(s) ± a prompt.
2. **Train** a per-character subject **LoRA** (trigger word + DOP
   regularization) once, against FLUX.2 Base weights, entirely on-device.
   The trained adapter is a durable, reusable asset.
3. **Generate** a sequence — *beginning → mid-action → end-state* — with the
   character's adapter loaded and its trigger word composed into each prompt.
4. **Verify** each panel with a DINOv2 similarity gate; drifted panels are
   regenerated automatically.

Two ideas do the heavy lifting, and they're worth stating plainly:

- **Train once per character, combine at inference.** Each character is trained
  in isolation. Multi-character scenes (a later phase) compose independently
  trained adapters at render time — you never merge training data or train
  per-scene.
- **Identity comes from training; coherence can come from reference images.**
  LoRA locks a face; FLUX.2's training-free multi-reference conditioning keeps
  the overall look coherent. Faces need the former; backgrounds often need only
  the latter.

## Roadmap

| Phase | Deliverable |
|---|---|
| **1** | One character → 3-panel sequence (beginning / mid-action / end-state) |
| **2** | Multi-character scenes ("Dan and Rick talk") |
| **3** | Props (incl. 3D-model → training data) |
| **4** | Locations |
| — | GUI: **Personaje** (`apps/Personaje`) is the app layer |

## Built on

- **[SwiftReparto](https://github.com/intrusive-memory/SwiftReparto)** — the cast
  list (`CAST.md`). Where characters come from, and the only writer of that file.
  Each member's `manifest:` points at `characters/<CHARACTER>.dossier/manifest.json`,
  the character's `.dossier` bundle ([dossier-format](https://github.com/intrusive-memory/dossier-format),
  implemented by SwiftDossier). Its sibling consumer, **SwiftEchada**,
  does the same thing for voice — same roster, different assets, neither aware
  of the other.
- **[flux-2-swift-mlx](https://github.com/intrusive-memory/flux-2-swift-mlx)**
  (`Flux2Core`) — on-device FLUX.2 LoRA training + inference.
- **SwiftAcervo** — shared model discovery / download / cache.
- **SwiftVinetas** — storyboard orchestration + DINOv2 consistency checking.

## Requirements

| | Minimum | Recommended |
|---|---|---|
| **macOS** | 26.0 | 26.0+ |
| **Swift** | 6.2 | 6.2+ |
| **Hardware** | Apple Silicon (M1+) | M3 Pro or later |
| **RAM** | 16 GB (Klein 4B Base) | 32 GB (Klein 9B Base / Dev) |

## Documentation

- [REQUIREMENTS.md](REQUIREMENTS.md) — full requirements, data model, and
  design invariants.
- [CLAUDE.md](CLAUDE.md) — agent/build guidance.

## License

TBD.
