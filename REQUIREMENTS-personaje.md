---
type: requirements
state: draft
updated: 2026-10-19
origin: intrusive-memory/Personaje @ 74721c3 (docs/REQUIREMENTS*.md, RECON_REPORT.md)
sequence: 0 (spec lock) — first across all repositories
---

# SwiftSemblanzas — what Personaje needs

Personaje's character record format is defined in this repo's REQUIREMENTS.md §5
and is moving out: the format itself to a new language-agnostic standard,
`dossier-format`, and its Swift implementation to a new package, SwiftDossier.
SwiftSemblanzas goes back to its original job, training. This file lists the work that
lands here. It is spec work only: SwiftSemblanzas has no `Package.swift` and no
`Sources/` yet.

## Why this is first

Personaje's recon was **BLOCKED** on finding B-10: §5.1–§5.5 (names, discovery,
manifest, tiers, CHARACTER.md) existed only as an uncommitted diff. That text
is now committed on this branch (`739720b`), which settles SM-P1. Every
dossier-format and SwiftDossier sortie is specified by it.

## Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| SM-P1 | *Done (`739720b`).* Commit the working-tree `REQUIREMENTS.md` and `README.md`, as written, before any other edit. | RECON B-10 |
| SM-P2 | *Done 2026-10-19.* §5 (5.1 names, 5.2 discovery, 5.3 manifest, 5.4 tier tables, 5.5 CHARACTER.md) is marked as moving to `dossier-format` (§5.6 the page and §5.8 the library shape go to SwiftDossier). Until that repo exists, §5 stays here as the reference text; afterwards it is replaced with a pointer. | BUNDLE §8, D2 |
| SM-P3 | *Done 2026-10-19.* §4.4 and §5.2 use `<NAME>.dossier` as the character directory. `characters/SHANE/` with no extension is an error. | BUNDLE §3, §8 |
| SM-P4 | *Done 2026-10-19.* The planned `SemblanzasRecord` target is dropped. The package will depend on SwiftDossier (`Dossier`) for the record, and keeps training. | BUNDLE D2, §8 |
| SM-P5 | *Done 2026-10-19 (tables moved to dossier-format `data/tiers.json`).* Tier tables (§5.4): `turnaround` → the view set (`portrait` + `view` front, profile, back); `expressions` grid and `faceSheet` → `expression` (one per emotion) and `face`; `wardrobe` and `palette` are removed and don't count toward completeness. Lead requires happy, sad, frightened, mad; supporting and guest ◐. | BUNDLE §3 |
| SM-P6 | Never write Reparto's `appearance:`. Read `manifest`, `role`, `canonicalName` from Reparto. | Reparto RQ-25, §4 |

## Acceptance

1. `git show HEAD:REQUIREMENTS.md | grep -c manifestChecksum` is greater than 0.
2. No `SemblanzasRecord` target remains in the spec.
3. Every character path in the spec ends in `.dossier`.
4. Re-running Personaje's recon turns B-10 into CONFIRMED.

## Depends on

Nothing.

## Blocks

- **dossier-format** (new repo): §5 is its starting text.
- **SwiftDossier** (new repo): implements it.
- Personaje's recon verdict.

## Not in this mission

Creating the Swift package, LoRA training code, or adopting SwiftDossier in
code. That follows SwiftDossier's first release and isn't on Personaje v1's
path (Personaje REQUIREMENTS §7: no LoRA training UI).
