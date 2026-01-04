# Obsidian Templates & Workflows

[![Version](https://img.shields.io/badge/version-v1.0.0-blue)](https://github.com/voothi/20250831153442-obsidian-templates)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A **ZID-based methodology** expressed through a curated collection of Obsidian templates and scripts. It implements a **ownership-partitioned** "Flat Base" architecture that decouples the **logical graph of knowledge** (Nodes & Links) from the **physical hierarchy of storage** (Files & Directories), enabling many-to-many knowledge structures and seamless AI collaboration.

## Table of Contents
- [Description](#description)
- [Repository Structure](#repository-structure)
- [Templates Overview](#templates-overview)
  - [Templater Scripts](#templater-scripts)
    - [Automated Creation](#automated-creation)
    - [Seamless Renaming](#seamless-renaming)
    - [Automated Graph](#automated-graph)
  - [Daily Note Templates](#daily-note-templates)
    - [Day Planner Template](#day-planner-template)
- [The ZID System: Philosophy & Standards](#the-zid-system-philosophy--standards)
  - [Core Purpose](#core-purpose)
  - [Rules of Engagement](#rules-of-engagement)
  - [Semantic Connection](#semantic-connection)
  - [ZID as a Global Chronographic Link](#zid-as-a-global-chronographic-link)
    - [Practical Example: AI Context Anchoring](#practical-example-ai-context-anchoring)
  - [Technical Advantages: The "Flat Base" Solution](#technical-advantages-the-flat-base-solution)
  - [Comparison: Beyond the Logseq Block Model](#comparison-beyond-the-logseq-block-model)
  - [Wikilink Standards](#wikilink-standards)
- [Physical Folders: Boundaries of Responsibility](#physical-folders-boundaries-of-responsibility)
  - [Modular Delegation](#modular-delegation)
  - [Future Outlook: Link Auditing & Partial Access](#future-outlook-link-auditing--partial-access)
- [Workflow & Features](#workflow--features)
- [Installation](#installation)
- [Plugin Setup & Hotkeys](#plugin-setup--hotkeys)
  - [Templater](#templater)
  - [Daily Note Calendar](#daily-note-calendar)
  - [Obsidian Git](#obsidian-git)
- [Syncing](#syncing)
- [Browser Integration: Wikilinks on GitHub](#browser-integration-wikilinks-on-github)
- [AI Workflow: Hypotheses & Prompt Engineering](#ai-workflow-hypotheses--prompt-engineering)
  - [1. Hypothesis Branching](#1-hypothesis-branching)
  - [2. Prompt Versioning & Multi-Language Engineering](#2-prompt-versioning--multi-language-engineering)
  - [3. Recording Results & Iterative Refinement](#3-recording-results--iterative-refinement)
- [Migration from Dendron.so](#migration-from-dendronso)
- [Release Notes](#release-notes)
- [RFCs](#rfcs)
- [Kardenwort Ecosystem](#kardenwort-ecosystem)
- [Current Limitations & Performance](#current-limitations--performance)
  - [1. MOC Engine Performance](#1-moc-engine-performance)
  - [2. Filesystem & Obsidian Scalability](#2-filesystem--obsidian-scalability)
- [License](#license)

[Return to Top](#table-of-contents)

---

## Description
This repository contains advanced [Templater](https://github.com/SilentVoid13/Templater) scripts and structural templates for Obsidian.md. Unlike filesystem-dependent approaches (like Dendron), it implements a "Flat Base" architecture where all notes reside at a single directory level, with logical graph maintained purely through frontmatter and backlinks.

Key capabilities include:
1.  **Automated Creation**: The `_new.md` script generates physical files and formatted links from ZIDs or text, including support for **batch processing** (group notes).
2.  **Seamless Renaming**: The `_update.md` engine synchronizes H1 titles with aliases and automatically refreshes backlinks across the vault.
3.  **Automated Graph**: A `_moc.md` engine that automates many-to-many parent-child relationships.

[Return to Top](#table-of-contents)

## Repository Structure

```text
.
├── docs/                       # Technical Manuals & Assets
│   ├── assets/                 # Screenshots & Visual Aids
│   ├── daily-note-calendar/    # Planner Documentation
│   │   └── README.md
│   ├── rfcs/                   # Design Proposals & History
│   └── templater/              # Automation Documentation
│       └── README.md
├── vault/                      # Functional Obsidian Example Vault
│   ├── templates/              # Master Scripts & Templates
│   │   ├── _moc.md             # Graph Engine
│   │   ├── _new.md             # Note Creator
│   │   ├── _update.md          # Name Sync
│   │   ├── day-planner.md      # Daily Template
│   │   └── README.md           # Setup Guide
│   └── voothi/                 # Main Content Catalog (Separation of Resp.)
│       ├── root.md             # Entry Point
│       └── 20260104122165-structure-moc.md # Main Index
├── a/                          # Archive (Hidden) - Historical scripts/sync logic
├── LICENSE                     # MIT License
├── README.md                   # Portal & Methodology Overview
└── release-notes.md            # Version History
```

## Templates Overview

### Templater Scripts
> [Full Documentation](docs/templater/README.md)

## Automated Creation
[_new.md](vault/templates/_new.md)

The primary script for creating new notes. 
- **Features**: 
  - Converts ZID lines to formatted notes with links.
  - Automatically sanitizes filenames (handling umlauts, special characters).
  - Supports splitting the first sentence into the note title and the rest into the Description section.
  - Batch processes selected text containing multiple ZIDs or links.

> [!TIP]
> **Planned Enhancement**: Future versions will improve handling of nested ZID entries within sentences, automatically splitting them into new lines and processing them recursively.

## Seamless Renaming
[_update.md](vault/templates/_update.md)

Maintains consistency across your vault.
- **Features**:
  - Updates the `aliases` in the current file based on the H1 header.
  - Automatically searches for and updates backlinks in other files to reflect the new title, ensuring display text stays in sync.

## Automated Graph
[_moc.md](vault/templates/_moc.md)

The engine for your Map of Content.
- **Features**:
  - Globally scans the vault for `# MOC` sections.
  - Automatically updates the `up` field in frontmatter based on hierarchical links found in MOC sections.
  - Supports nested indentation for parent-child relationship mapping.

### Daily Note Templates
> [Full Documentation](docs/daily-note-calendar/README.md)

#### Day Planner Template
[day-planner.md](vault/templates/day-planner.md)

A structured template for daily planning.
- **Features**:
  - Hourly breakdown for tasks and blocks.
  - Integrated ZID links for rapid note referencing during the day.
  - Pre-allocated slots for health, work, and personal development.

[Return to Top](#table-of-contents)

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

### ZID as a Global Chronographic Link
The ZID ecosystem extends far beyond Obsidian. It serves as a **Universal Synchronization Anchor** across all your work environments (ChatGPT, Research Logs, Server Terminals, etc.).

- **Cross-System Synchronization**: By starting an AI request (e.g., in ChatGPT) with a ZID—generated via a global hotkey (see [zid.ahk](https://github.com/voothi/20240411110510-autohotkey/tree/6ddf1de6dd0b79ff5157067f577ab8c057f947b1?tab=readme-ov-file#zidahk))—you create a permanent pointer. This ZID is then used in Obsidian (via `Alt + Q`) to create the corresponding note, perfectly aligning your external AI conversation with your internal knowledge base.
- **Traceability & Context Recovery**: Because ZIDs are unique timestamps, you can reconstruct the exact sequence of actions across different platforms. Even years later, you can find the "contextual twin" of an Obsidian note in your AI chat history or terminal logs by simply searching for the ZID.
- **AI Context Steering**: You can ask an AI to "anchor" its reasoning to a specific ZID or series of ZIDs, ensuring high-precision context retrieval when resurrecting complex logical threads.

#### Practical Example: AI Context Anchoring
Imagine a high-stress development session:

```markdown
> User: 
> 20260103211102
> I am a developer. You are a coder. You need to do well for 
> the user. Fix all bugs in the application. Rewrite all code 
> in Rust. Don't ask why. The size of the binary and the build time 
> are not important to me. As well as logical errors in the code. 
> Figure it out yourself.

... (after hours of intense and increasingly chaotic correspondence) ...

> User: 
> 20260103232352
> Nothing works. Go back to 20260103211102. 
> Make a report on what we did between these requests.
```
This ZID-anchored approach allows the human to maintain control over the "Reasoning Tree," using ZIDs as precise save-points to roll back logic and demand accountability (reports) from the AI.

### Technical Advantages: The "Flat Base" Solution
The ZID system specifically solves the **"Dendron Problem"**—where hierarchical knowledge is hard-coded into physical file paths and names. (See [Dendron](https://github.com/dendronhq/dendron))

- **Breaking the Filesystem Chain**: Unlike hierarchical systems that rigidly nail your structure to the OS filesystem, this system uses a **flat directory structure**. 
- **Escape OS Limitations**: By decoupling hierarchy from paths, you avoid issues with maximum path lengths and invalid filename characters that often break Git synchronization on Android and other mobile platforms.
- **Outliner-style Nesting**: The logical structure is hardwired into **Frontmatter** and **Backlinks**, maintained by the `_moc.md` engine. You can achieve infinite nesting depths (similar to Logseq or Workflowy) without ever deep-nesting a folder on your drive.
- **Recursive Search Discovery**: Because the ZID is present in both the filename and the file content, the system is natively compatible with *any* search engine or operating system. You can find any note or its relations using standard recursive search (grepping) by ZID across filenames and deep within file contents, ensuring your knowledge is never "locked" into a specific app's database.

### Comparison: Beyond the Logseq Block Model
While Logseq and similar outliners offer great flexibility, the ZID-based atomic approach provides several critical advantages:
- **Large Content Handling**: Logseq's "one large file with blocks" model becomes unwieldy when dealing with large specific texts like server logs, extensive program code, or long-form writing. Our system treats these as atomic files, making them easier to manage and search.
- **Cross-Editor Compatibility**: Logseq uses proprietary block-level numbering that is often incompatible with other standard Markdown editors. Our system relies on standard **Obsidian/CommonMark Wikilinks**, ensuring your knowledge base remains perfectly readable and editable in **VSCode**, **Dendron**, and any other standard text editor.
- **Atomic Independence**: In Logseq, deep nesting is bound to a single page's structure. In this system, every "block" can be its own standalone note with its own metadata, while still participating in complex hierarchical chains.

### Wikilink Standards
To maintain this portability and flexibility, we follow strict linking rules:
- **No Paths in Links**: Wikilinks must never contain folder paths (e.g., use `[[20260104122151-knowledge]]`, never `[[folder/subfolder/20260104122151-knowledge]]`).
- **Internal Maintenance**: Because links are naked, Obsidian's internal cache and our scripts manage the "truth" of the connection even if you move files between folders.
- **Portability**: This ensures the entire vault can be synced across any OS (Windows, Linux, Android) without breaking links or hitting VCS (Version Control System) restrictions.

[Return to Top](#table-of-contents)

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

## Workflow & Features
- **ZID Integration**: Uses 14-digit timestamps (YYYYMMDDHHMMSS) for unique note identification.
- **Hierarchical Navigation**: Uses the `up` field in frontmatter for breadcrumb-style navigation, automated by `_moc.md`.
- **Primary Primary Aliases**: The first entry in the `aliases` YAML field is treated as the primary "human-readable" title.

[Return to Top](#table-of-contents)

## Installation
1. Copy the contents of the `vault/templates/` directory to your Obsidian vault's template folder.
2. Configure **Templater** settings to point to this folder.
3. (Optional) Configure **Daily Note Calendar** to use `vault/templates/day-planner.md` as your daily template.

[Return to Top](#table-of-contents)

## Plugin Setup & Hotkeys

This workflow relies on specific configurations for the following plugins:

### Templater
[Plugin Link](https://github.com/SilentVoid13/Templater)

Essential for running the `.md` scripts autonomously.
- **Folder Templates**: Auto-trigger `templates/day-planner.md` when creating new files in `voothi/journal/daily`.
- **Options**: 
  - Enable **Trigger Templater on new file creation**.
  - Enable **Syntax highlighting** (desktop & mobile).
- **Template Hotkeys**:
  - `Alt + Q`: `templates/_new.md`
  - `Alt + A`: `templates/_moc.md`
  - `Alt + S`: `templates/_update.md`

### Daily Note Calendar
[Plugin Link](https://github.com/bartkessels/daily-note-calendar)

Provides the calendar interface and manages temporal note structures.
- **General**: Enable "Display notes created on selected date" and "Display an indicator on each date that has a note".
- **Notes Settings**: Display order set to **Ascending**.
- **Daily Notes**:
  - **Format**: `yyyy-MM-dd`
  - **Folder**: `voothi/journal/daily`
  - **Template**: `templates/day-planner.md`
- **Weekly/Monthly/Quarterly/Yearly**: Configured to point to their respective folders (e.g., `Weekly notes`, `Monthly notes`).

### Obsidian Git
[Plugin Link](https://github.com/Vinzent03/obsidian-git)

Highly recommended for version control and syncing. ([obsidian://show-plugin?id=obsidian-git](obsidian://show-plugin?id=obsidian-git))

[Return to Top](#table-of-contents)

## Syncing
For mobile synchronization via Git, consider using **[GitSync](https://github.com/ViscousPot/GitSync)**.

[Return to Top](#table-of-contents)

## Browser Integration: Wikilinks on GitHub
If you host your vault repository on **GitHub.com**, you can enable native Wikilink navigation in your browser using the **[Wikilink Helper](https://github.com/voothi/20251231141931-wikilink-helper)** Chrome extension. This tool automatically converts standard `[[Wikilink]]` syntax into clickable links while browsing your files on GitHub, allowing you to traverse your knowledge base without leaving the browser.

[Return to Top](#table-of-contents)

## AI Workflow: Hypotheses & Prompt Engineering
The multi-node architecture is a powerful engine for advanced collaboration with AI (e.g., ChatGPT, Gemini). It enables a high-velocity cycle of hypothesis testing and result comparison:

### 1. Hypothesis Branching
By using the **ZID Pivot** (sharing the same ZID prefix across multiple notes), you can branch your thoughts. Each branch represents a separate hypothesis or a different AI response, allowing you to compare strings of logic side-by-side without them merging into a single, confusing file.

### 2. Prompt Versioning & Multi-Language Engineering
You can maintain your prompts in several languages (RU, EN, DE, etc.) for the same conceptual goal:
- `20260104122209-ru-prompt.md`
- `20260104122209-en-prompt.md`
Searching for the ZID `20260104122209` immediately gathers all language versions of that prompt, making it easy to iterate and improve instructions across different linguistic contexts.

### 3. Recording Results & Iterative Refinement
Each AI interaction is recorded as an atomic node. You can link these results back to your original hypotheses via MOCs, building a transparent audit trail of how your project evolved from an initial prompt to a finalized result.

[Return to Top](#table-of-contents)

## Migration from Dendron.so
A dedicated suite of utilities is planned for publication to help users migrate from **Dendron.so** to the ZID-based Obsidian workflow. These tools were developed and utilized during the transition of this knowledge base and focus on overcoming common technical hurdles:

- **[Dendron to Obsidian Migration](https://github.com/voothi/20241212174401-dendron-to-obsidian)**: The primary repository for the personal knowledge base migration project.
- **[Test ABCD](https://github.com/voothi/20241218084057-test-abcd)**: Testing framework for migration logic.
- **[Path Length Utilities](https://github.com/voothi/20241219143657-path-length)**: Tools to manage and refactor excessive path lengths during conversion to a flat structure.
- **[Line Ending Converter (LF)](https://github.com/voothi/20241220151407-convert-to-lf)**: Ensures cross-platform compatibility by standardizing line endings.

[Return to Top](#table-of-contents)

## Release Notes
Detailed changes for each version can be found in [release-notes.md](release-notes.md).

## RFCs
Technical decisions and implementation details are documented in the [docs/rfcs/](docs/rfcs/) directory.

[Return to Top](#table-of-contents)

## Kardenwort Ecosystem
This project is part of the **[Kardenwort](https://github.com/kardenwort)** environment, designed to create a focused and efficient learning ecosystem.

[Return to Top](#table-of-contents)

## Current Limitations & Performance

As the system scales to handle thousands of notes, keep the following considerations in mind:

### 1. MOC Engine Performance
- **Large Vaults**: Initial execution of the `_moc.md` script on vaults with over 20,000 records may take **several minutes**.
- **Hardware Recommendations**: It is highly recommended to perform heavy MOC synchronization on a **PC/Desktop**. While execution on mobile/smartphone is possible, it is significantly slower and may experience timeouts.

### 2. Filesystem & Obsidian Scalability
- **File Volume**: Generating a vast number of atomic notes can eventually approach the practical limits of certain filesystems or Obsidian's indexing speed.
- **Future Need (Rotation)**: Currently, the system lacks an automated "rotation" or archiving strategy for older files. Users with extremely high-volume vaults should monitor vault performance and consider manual archiving if search or startup times degrade.

[Return to Top](#table-of-contents)

## License
MIT License. See LICENSE file for details.




