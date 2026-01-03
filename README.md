# Obsidian Templates & Workflows

[![Version](https://img.shields.io/badge/version-v1.0.0-blue)](https://github.com/voothi/20250831153442-obsidian-templates)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A curated collection of Obsidian templates and scripts for Templater and Daily Notes, designed for an efficient Zettelkasten-style workflow.

## Table of Contents
- [Description](#description)
- [Templates Overview](#templates-overview)
  - [Templater Scripts](#templater-scripts)
  - [Daily Note Templates](#daily-note-templates)
- [Workflow & Features](#workflow--features)
- [Installation](#installation)
- [Plugin Setup & Hotkeys](#plugin-setup--hotkeys)
- [Syncing](#syncing)
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

## Kardenwort Ecosystem
This project is part of the **[Kardenwort](https://github.com/kardenwort)** environment, designed to create a focused and efficient learning ecosystem.

[Return to Top](#table-of-contents)

## License
MIT License. See LICENSE file for details.




