# Obsidian Templates & Workflows

[![Version](https://img.shields.io/badge/version-v1.0.0-blue)](https://github.com/voothi/20250831153442-obsidian-templates)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A curated collection of Obsidian templates and scripts for Templater and Daily Notes, designed for an efficient Zettelkasten-style workflow.

## Table of Contents
- [Description](#description)
- [Templates Overview](#templates-overview)
  - [Templater Scripts](#templater-scripts)
  - [Daily Note Templates](#daily-note-templates)
- [The ZID System: Philosophy & Standards](#the-zid-system-philosophy--standards)
- [Physical Folders: Boundaries of Responsibility](#physical-folders-boundaries-of-responsibility)
- [Workflow & Features](#workflow--features)
- [Installation](#installation)
- [Plugin Setup & Hotkeys](#plugin-setup--hotkeys)
- [Syncing](#syncing)
- [Release Notes](#release-notes)
- [RFCs](#rfcs)
- [Kardenwort Ecosystem](#kardenwort-ecosystem)
- [License](#license)

---

## Description
This repository contains advanced [Templater](https://github.com/SilentVoid13/Templater) scripts and structural templates for Obsidian.md. It focuses on automating note creation, metadata synchronization, and Map of Content (MOC) management using Zettelkasten IDs (ZID).

[Return to Top](#table-of-contents)

## Templates Overview

### Templater Scripts

#### [`_new.md`](templater/_new.md)
The primary script for creating new notes. 
- **Features**: 
  - Converts ZID lines to formatted notes with links.
  - Automatically sanitizes filenames (handling umlauts, special characters).
  - Supports splitting the first sentence into the note title and the rest into the Description section.
  - Batch processes selected text containing multiple ZIDs or links.

> [!TIP]
> **Planned Enhancement**: Future versions will improve handling of nested ZID entries within sentences, automatically splitting them into new lines and processing them recursively.

#### [`_update.md`](templater/_update.md)
Maintains consistency across your vault.
- **Features**:
  - Updates the `aliases` in the current file based on the H1 header.
  - Automatically searches for and updates backlinks in other files to reflect the new title, ensuring display text stays in sync.

#### [`_moc.md`](templater/_moc.md)
The engine for your Map of Content.
- **Features**:
  - Globally scans the vault for `# MOC` sections.
  - Automatically updates the `up` field in frontmatter based on hierarchical links found in MOC sections.
  - Supports nested indentation for parent-child relationship mapping.

### Daily Note Templates

#### [`day-planner.md`](daily-note-calendar/day-planner.md)
A structured template for daily planning.
- **Features**:
  - Hourly breakdown for tasks and blocks.
  - Integrated ZID links for rapid note referencing during the day.
  - Pre-allocated slots for health, work, and personal development.

[Return to Top](#table-of-contents)

---

## The ZID System: Philosophy & Standards

The **Zettelkasten ID (ZID)** is the heartbeat of this ecosystem. It is a strictly formatted 14-digit timestamp (`YYYYMMDDHHMMSS`) used as a permanent, immutable identifier for every atomic unit of knowledge.

### Core Purpose
1. **Permanent Identity**: The ZID decouples the note's identity from its title. Titles "evolve" as understanding deepens, but the ZID remains an anchor that never breaks links.
2. **Collision Prevention**: The per-second granularity ensures that no two notes will ever have the same ID, even in a high-volume research environment.
3. **Semantic Anchoring**: The ZID allows you to see *when* a thought was born relative to others, providing an implicit chronological index to your knowledge base.

### Rules of Engagement
- **Chronological Integrity**: By default, the ZID reflects the moment of file creation.
- **Logical Dating**: We do **not** always go forward. If you are processing notes from the past (e.g., digitizing an old analog journal), you should manually set the ZID to match the actual historical date. This preserves the chronological context of the knowledge.
- **Batching & Sequencing**: When creating multiple related notes at once, it is acceptable to generate sequential ZIDs (manual increment) to cluster them together logically, even if they were technically created in the same minute.
- **ID Overrides**: The `_new.md` script will prioritize an existing ZID if you provide one. If none is found, it generates a "Current-Time" ZID as a fallback.

### Semantic Connection
The ZID is not just a name; it's a key. By keeping the ZID in the filename and YAML, external tools and internal scripts can always find the "truth" of the note, even if you rename `20260103205223-old-theory.md` to `20260103205223-proven-law.md`.

### Technical Advantages: The "Flat Base" Solution
The ZID system specifically solves the **"Dendron Problem"**—where hierarchical knowledge is hard-coded into physical file paths and names. 

- **Breaking the Filesystem Chain**: Unlike hierarchical systems that rigidly nail your structure to the OS filesystem, this system uses a **flat directory structure**. 
- **Escape OS Limitations**: By decoupling hierarchy from paths, you avoid issues with maximum path lengths and invalid filename characters that often break Git synchronization on Android and other mobile platforms.
- **Outliner-style Nesting**: The logical structure is hardwired into **Frontmatter** and **Backlinks**, maintained by the `_moc.md` engine. You can achieve infinite nesting depths (similar to Logseq or Workflowy) without ever deep-nesting a folder on your drive.

### Wikilink Standards
To maintain this portability and flexibility, we follow strict linking rules:
- **No Paths in Links**: Wikilinks must never contain folder paths (e.g., use `[[20260103210512-note]]`, never `[[folder/subfolder/20260103210512-note]]`).
- **Internal Maintenance**: Because links are naked, Obsidian's internal cache and our scripts manage the "truth" of the connection even if you move files between folders.
- **Portability**: This ensures the entire vault can be synced across any OS (Windows, Linux, Android) without breaking links or hitting CVS (Concurrent Versions System) restrictions.

[Return to Top](#table-of-contents)

---

## Physical Folders: Boundaries of Responsibility

In this ecosystem, physical catalogs (folders) serve a specific, non-hierarchical purpose. They are the **watersheds between boundaries of responsibility**.

### Modular Delegation
- **Administrative Logic**: While ZIDs and MOCs handle the *knowledge* hierarchy, folders handle the *administrative* boundaries.
- **Access Control**: Folders act like separate "library wings." This structure allows you to share, unshare, or delegate responsibility for an entire catalog (e.g., via Git submodules or separate vault syncing) without breaking the logical connections maintained by ZIDs.
- **Granularity**: You can provide separate access to specific project folders or "unshare" private research catalogs, ensuring that the "Flat Base" remains structurally simple but administratively robust.

### Future Outlook: Link Auditing & Partial Access
A key challenge of this modular approach is **Link Integrity** when sharing partial units of the knowledge base:
- **The "Out-of-Container" Link Problem**: When sharing a specific folder (container/catalog), we must ensure that the notes within do not critically depend on links pointing to notes *outside* that shared boundary.
- **Link Auditing**: Future enhancements include developing dedicated scripts to audit and refactor links before delegation. These scripts will help identify "external dependencies" and ensure that the shared unit remains fully functional and contextually complete for the recipient.

[Return to Top](#table-of-contents)

---

## Workflow & Features
- **ZID Integration**: Uses 14-digit timestamps (YYYYMMDDHHMMSS) for unique note identification.
- **Hierarchical Navigation**: Uses the `up` field in frontmatter for breadcrumb-style navigation, automated by `_moc.md`.
- **Primary Primary Aliases**: The first entry in the `aliases` YAML field is treated as the primary "human-readable" title.

[Return to Top](#table-of-contents)

## Installation
1. Copy the contents of the `templater/` directory to your Obsidian vault's template folder.
2. Configure **Templater** settings to point to this folder.
3. (Optional) Copy `daily-note-calendar/day-planner.md` to your daily notes template folder.

[Return to Top](#table-of-contents)

## Plugin Setup & Hotkeys

This workflow relies on specific configurations for the following plugins:

### [Templater](https://github.com/SilentVoid13/Templater)
Essential for running the `.md` scripts autonomously.
- **Folder Templates**: Auto-trigger `templates/day-planner.md` when creating new files in `voothi/journal/daily`.
- **Options**: 
  - Enable **Trigger Templater on new file creation**.
  - Enable **Syntax highlighting** (desktop & mobile).
- **Template Hotkeys**:
  - `Alt + Q`: `templates/_new.md`
  - `Alt + A`: `templates/_moc.md`
  - `Alt + S`: `templates/_update.md`

### [Daily Note Calendar](https://github.com/quandv/obsidian-daily-note-calendar)
Provides the calendar interface and manages temporal note structures.
- **General**: Enable "Display notes created on selected date" and "Display an indicator on each date that has a note".
- **Notes Settings**: Display order set to **Ascending**.
- **Daily Notes**:
  - **Format**: `yyyy-MM-dd`
  - **Folder**: `voothi/journal/daily`
  - **Template**: `templates/day-planner.md`
- **Weekly/Monthly/Quarterly/Yearly**: Configured to point to their respective folders (e.g., `Weekly notes`, `Monthly notes`).

### [Obsidian Git](https://github.com/Vinzent03/obsidian-git)
Highly recommended for version control and syncing. ([obsidian://show-plugin?id=obsidian-git](obsidian://show-plugin?id=obsidian-git))

[Return to Top](#table-of-contents)

## Syncing
For mobile synchronization via Git, consider using **[GitSync](https://github.com/ViscousPot/GitSync)**.

[Return to Top](#table-of-contents)

## Release Notes
Detailed changes for each version can be found in [release-notes.md](release-notes.md).

## RFCs
Technical decisions and implementation details are documented in the [docs/rfcs/](docs/rfcs/) directory.

[Return to Top](#table-of-contents)

## Kardenwort Ecosystem
This project is part of the **[Kardenwort](https://github.com/kardenwort)** environment, designed to create a focused and efficient learning ecosystem.

[Return to Top](#table-of-contents)

## License
MIT License. See LICENSE file for details.




