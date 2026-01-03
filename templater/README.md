# Templater Automation Scripts

[![Plugin: Templater](https://img.shields.io/badge/Obsidian-Templater-7d3ab2)](https://github.com/SilentVoid13/Templater)

This folder contains the core automation logic for the Zettelkasten workflow. These scripts manage note creation, name sanitization, and hierarchical synchronization.

## Table of Contents
- [`_new.md`: Smart Note Creation](#_newmd-smart-note-creation)
- [`_update.md`: Title & Backlink Sync](#_updatemd-title--backlink-sync)
- [`_moc.md`: Map of Content synchronization](#_mocmd-map-of-content-synchronization)
- [Usage Examples: `_new.md`](#usage-examples-_newmd)
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
| Input ZID Lines | Resulting linked Note |
| :--- | :--- |
| ![Input: ZID lines](../assets/20260103192406.png) | ![Result: Linked Note](../assets/20260103192418.png) |

| Batch Input | Batch Result |
| :--- | :--- |
| ![Batch Processing Input](../assets/20260103192425.png) | ![Batch Processing Result](../assets/20260103192431.png) |

[Return to Top](#table-of-contents)

---

## `_update.md`: Title & Backlink Sync
Ensures that human-readable titles remain consistent throughout the vault.

### Features
- **Alias Management**: Synchronizes the first `aliases` entry in YAML with the file's H1 header.
- **Backlink Automation**: Searches the entire vault for links pointing to the current note and updates their display text to match the new H1 title.

### Visual Guide
| Alias Update | Backlink Refresh |
| :--- | :--- |
| ![Alias Update](../assets/20260103192516.png) | ![Backlink Refresh](../assets/20260103192523.png) |

[Return to Top](#table-of-contents)

---

## `_moc.md`: Map of Content Synchronization
The "Engine" for hierarchical navigation.

### Features
- **Global Scan**: Reads all notes with a `# MOC` header.
- **Heuristic Mapping**: Determines parent-child relationships based on list indentation and links.
- **Frontmatter Automation**: Updates the `up` field in child notes for breadcrumb-style navigation.

### Visual Guide
| MOC Scanning | Frontmatter Update |
| :--- | :--- |
| ![MOC Scanning](../assets/20260103192534.png) | ![Frontmatter Update](../assets/20260103192542.png) |

[Return to Top](#table-of-contents)

---

[Return to Top](#table-of-contents)

---

## Usage Examples: `_new.md`

### Case 1: Convert Raw ZID Lines to Linked Notes
Turn a selection of lines starting with 14-digit timestamps (ZIDs) into clean Wikilinks.

**Input (Selection):**
```text
20251226135208 This is the first sentence of the first entry. This is the second sentence which moves inside.

- [ ] 20251226135257 This is the first sentence of the second entry.
```

**Output (Updated Selection):**
```text
[[20251226135208-this-is-the-first-sentence-of-the-first-entry|This is the first sentence of the first entry.]]

- [ ] [[20251226135257-this-is-the-first-sentence-of-the-second-entry|This is the first sentence of the second entry.]]
```

**What happens:**
- **Sanitization**: Filenames/Wikilinks are automatically sanitized (handling spaces, special characters, and umlauts) while preserving the **original input language**.
- **Title Extraction**: The first sentence (up to the first `.`) becomes the Wikilink label.
- **Content Migration**: The second sentence ("This is the second sentence...") is moved to the new note's description.
- **Batching**: Processes all selected ZID lines simultaneously while preserving list markers.

---

### Case 2: In-line Selection (New Note from Text)
Highlight a phrase within an existing sentence to extract it into a new note.

**Action:**
1. You have a sentence like: "We should discuss the **Kardenwort Ecosystem** later today."
2. Highlight **Kardenwort Ecosystem**.
3. Run `Alt + Q`.

**Result:**
- The text is replaced with: `[[20260103204319-kardenwort-ecosystem|Kardenwort Ecosystem]]`
- A new file is created with that ZID and title.

---

### Case 3: Selecting Multiple Sentences
What happens if your selection contains more than one sentence?

**Input (Selection):**
```text
The Architecture of Knowledge. It focuses on modularity and ZIDs.
```

**Action:**
Highlight both sentences and run `Alt + Q`.

**Result:**
- **Link Title**: `The Architecture of Knowledge.` (The first sentence).
- **Slug**: `YYYYMMDDHHMMSS-the-architecture-of-knowledge`
- **New Note Content**: The second sentence ("It focuses on modularity and ZIDs.") is placed inside the note's **Description** section.

> [!NOTE]
> The script uses simple punctuation detection (`.`, `?`, `!`) to perform this split, ensuring your main note stays concise while the details are offloaded to the atomic note.

[Return to Top](#table-of-contents)

---

## Installation & Hotkeys

To use these scripts, copy them to your Templater templates folder and assign the following hotkeys:

| Script | Hotkey | Function |
| :--- | :--- | :--- |
| `_new.md` | `Alt + Q` | Create notes from selection |
| `_moc.md` | `Alt + A` | Synchronize Global MOCs |
| `_update.md` | `Alt + S` | Update titles & backlinks |

[Return to Top](#table-of-contents)