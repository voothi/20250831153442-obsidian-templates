# Release Notes

## v1.2.0 - 2026-01-05

### Changed
- **Logic Unification**: `vault/templates/_new.md` now shares identical regex and sanitization logic with `ref/obsidian_zid_wikilink.py`.
- **Heading Support**: ZID detection (`zidLineRegex`) now supports Markdown headings (`#` through `######`) as prefixes.
- **Improved Sanitization**: Added support for Eszett (`ẞ`) and better handling of periods in filenames.

## v1.1.0 - 2026-01-04

### Added
- **Functional Example Vault**: The `vault/` directory is now a fully functional Obsidian vault.
  - `vault/templates/`: Master scripts and planner configuration.
  - `vault/voothi/`: Content catalog with live examples of MOCs, ZID Pivots, and Polymorphism.
- **Interactive Documentation**: Technical manuals in `docs/` now feature "Active Links" (dual Wikilink/Markdown) for one-click navigation within Obsidian and on GitHub.
- **Comprehensive RFC Tracking**: Added `docs/rfcs/` for tracking architectural decisions and version staging.
- **ZID Mapping Utility**: Internal tools for re-indexing knowledge bases to new chronographic anchors.

### Changed
- **Directory Consolidation**: Simplified repository root by moving `templater/`, `daily-note-calendar/`, and `assets/` into a centralized `docs/` folder.
- **ZID Baseline**: Re-anchored all example notes to a consistent `20260104` baseline.
- **Terminology Harmonization**: Standardized "Automated Graph" and "MOC Engine" terminology across all manuals.
- **Table of Contents Expansion**: Added H3 granularity to documentation TOCs for better discoverability.

### Fixed
- **Link Integrity Audit**: Resolved 100% of broken links caused by the `docs/` and `vault/` restructuring.
- **Terminology Correction**: Changed "CVS" to "VCS" (Version Control System) in root documentation.
- **Redundancy Pruning**: Removed conflicting legacy MOC entries and duplicate content files.


## v1.0.0 - 2026-01-03

### Added
- **Comprehensive Documentation Overhaul**:
  - New root `README.md` with Table of Contents and project overview.
  - Detailed `templater/README.md` covering `_new.md`, `_update.md`, and `_moc.md`.
  - Detailed `daily-note-calendar/README.md` covering the `day-planner.md` template and plugin setup.
- **Visual Guides**: Integration of asset screenshots as step-by-step visual documentation.
- **Kardenwort Ecosystem**: Added context for the broader learning environment.

### Changed
- **Relative Navigation**: All internal links are now relative for better portability.
- **Plugin Instructions**: Enriched with specific folder paths and hotkey recommendations (`Alt + Q`, `Alt + A`, `Alt + S`).
- **Internal Anchors**: Standardized "Return to Top" links to point to same-file Table of Contents.

### Fixed
- **Plugin Attribution**: Corrected documentation to reflect that Daily Note Calendar (not Periodic Notes) handles the temporal note structure in this setup.
