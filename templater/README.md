# Templater Automation Scripts

[![Plugin: Templater](https://img.shields.io/badge/Obsidian-Templater-7d3ab2)](https://github.com/SilentVoid13/Templater)

This folder contains the core automation logic for the Zettelkasten workflow. These scripts manage note creation, name sanitization, and hierarchical synchronization.

## Table of Contents
- [`_new.md`: Smart Note Creation](#_newmd-smart-note-creation)
- [`_update.md`: Title & Backlink Sync](#_updatemd-title--backlink-sync)
- [`_moc.md`: Map of Content synchronization](#_mocmd-map-of-content-synchronization)
- [Installation & Hotkeys](#installation--hotkeys)

---

## `_new.md`: Smart Note Creation
Automates the process of turning Zettelkasten IDs (ZIDs) or plain text into formatted notes.

### Features
- **ZID Parsing**: Recognizes 14-digit timestamps and converts them into note titles.
- **Sanitization**: Automatically handles umlauts (`ö` -> `oe`), spaces, and special characters for cross-platform compatibility.
- **Description Split**: If enabled, splits the first sentence into the note title and treats the rest as the body description.
- **Batch Processing**: Select multiple lines of ZIDs or links and process them all at once.

### Visual Guide
````carousel
![Input: ZID lines](../assets/20260103192406.png)
<!-- slide -->
![Result: Linked Note](../assets/20260103192418.png)
<!-- slide -->
![Batch Processing Input](../assets/20260103192425.png)
<!-- slide -->
![Batch Processing Result](../assets/20260103192431.png)
````

---

## `_update.md`: Title & Backlink Sync
Ensures that human-readable titles remain consistent throughout the vault.

### Features
- **Alias Management**: Synchronizes the first `aliases` entry in YAML with the file's H1 header.
- **Backlink Automation**: Searches the entire vault for links pointing to the current note and updates their display text to match the new H1 title.

### Visual Guide
````carousel
![Alias Update](../assets/20260103192516.png)
<!-- slide -->
![Backlink Refresh](../assets/20260103192523.png)
````

---

## `_moc.md`: Map of Content Synchronization
The "Engine" for hierarchical navigation.

### Features
- **Global Scan**: Reads all notes with a `# MOC` header.
- **Heuristic Mapping**: Determines parent-child relationships based on list indentation and links.
- **Frontmatter Automation**: Updates the `up` field in child notes for breadcrumb-style navigation.

### Visual Guide
````carousel
![MOC Scanning](../assets/20260103192534.png)
<!-- slide -->
![Frontmatter Update](../assets/20260103192542.png)
````

---

## Installation & Hotkeys

To use these scripts, copy them to your Templater templates folder and assign the following hotkeys:

| Script | Hotkey | Function |
| :--- | :--- | :--- |
| `_new.md` | `Alt + Q` | Create notes from selection |
| `_moc.md` | `Alt + A` | Synchronize Global MOCs |
| `_update.md` | `Alt + S` | Update titles & backlinks |

[Return to Root README](../README.md)