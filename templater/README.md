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
- **ZID Parsing**: Recognizes 14-digit timestamps ([ZIDs](../README.md#the-zid-system-philosophy--standards)) and converts them into note titles.
- **Sanitization**: Automatically handles umlauts (`ö` -> `oe`), spaces, and special characters for cross-platform compatibility.
- **Description Split**: If enabled, splits the first sentence into the note title and treats the rest as the body description.
- **Enforced Slug Length**: Automatically limits the filename slug to the **first 4 words** of the title to keep the vault clean and OS-compatible.
- **Batch Processing**: Select multiple lines of ZIDs or links and process them all at once.

### Visual Guide
| Input ZID Lines | Resulting linked Note |
| :--- | :--- |
| ![Input: ZID lines](../assets/20260103192406.png) | ![Result: Linked Note](../assets/20260103192418.png) |

| Batch Input | Batch Result |
| :--- | :--- |
| ![Batch Processing Input](../assets/20260103192425.png) | ![Batch Processing Result](../assets/20260103192431.png) |

[Return to Top](#table-of-contents)

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

## `_moc.md`: Map of Content
#### [`_moc.md`](_moc.md)
The structural engine of the vault. It scans for specific headers to build and maintain the logical hierarchy of your notes.

### Features
- **Key Syntax**: Matches any header containing `MOC.` (e.g., `## MOC.` or `## Project Alpha. MOC`).
- **Automation**: Automatically populates the `up` field in the frontmatter of linked notes based on their position under an MOC header.
- **Hierarchical Depth**: Supports nested list indentation to map complex parent-child relationships.
- **Frontmatter Automation**: Updates the `up` field in child notes for breadcrumb-style navigation.

### Visual Guide
| MOC Scanning | Frontmatter Update |
| :--- | :--- |
| ![MOC Scanning](../assets/20260103192534.png) | ![Frontmatter Update](../assets/20260103192542.png) |

[Return to Top](#table-of-contents)

## The MOC Engine: `_moc.md`

### 1. What is the `# MOC.` Header?
The script triggers on headers that follow a specific pattern: `# MOC.` or `## [Topic]. MOC`.

> [!IMPORTANT]
> **The Dot Matters**: The script's logic is strictly keyed to identify the string `MOC.` (with the period). This prevents it from accidentally processing random text while allowing for descriptive headers like `## Biology Research. MOC`.

### 2. Logical Multi-Node Chains (Many-to-Many)
This system provides extreme **flexibility**. Unlike a rigid folder tree, a single note can belong to multiple branches of your knowledge base.

- **How it works**: If you list `[[ZID-Atomic-Note]]` under three different MOC sections in three separate files, the `_moc.md` script will aggregate all three parents into the `up` field of that note's frontmatter.
- **The Result**: A truly networked knowledge base where nodes participate in several logical chains simultaneously.

### 3. Usage & Syntax
To map your structure, use indented lists under an MOC header:

```markdown
## Projects. MOC
- [[ZID-Project-A]]
  - [[ZID-Task-1]]
  - [[ZID-Task-2]]
- [[ZID-Project-B]]
  - [[ZID-Task-1]] (Note: Task 1 participates in both Projects!)
```

### 4. Integration with `_new.md`
`_moc.md` and `_new.md` work in a continuous feedback loop:
1. **Creation**: You use `_new.md` to create an atomic note. It initially points back to the note where it was created.
2. **Structuring**: You then place that note's link into one or more MOC sections.
3. **Synchronization**: Running `_moc.md` (via `Alt + A`) globally updates the `up` field, finalizing the note's position in your structural hierarchy.

[Return to Top](#table-of-contents)

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

### Case 2: In-line Selection (New Note from Text)
Highlight a phrase within an existing sentence to extract it into a new note.

**Action:**
1. You have a sentence like: "We should discuss the **Kardenwort Ecosystem** later today."
2. Highlight **Kardenwort Ecosystem**.
3. Run `Alt + Q`.

**Result:**
- The text is replaced with: `[[20260103204319-kardenwort-ecosystem|Kardenwort Ecosystem]]`
- A new file is created with that ZID and title.

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
- **Slug**: `YYYYMMDDHHMMSS-the-architecture-of-knowledge` (First 4 words preserved).
- **New Note Content**: The second sentence ("It focuses on modularity and ZIDs.") is placed inside the note's **Description** section.

> [!NOTE]
> The script uses simple punctuation detection (`.`, `?`, `!`) to perform this split, ensuring your main note stays concise while the details are offloaded to the atomic note.

### Case 4: Process Existing Wikilinks
If your selection contains Wikilinks that point to non-existent files (e.g., `[[Future Concept]]`), the script will generate the `.md` files for them without changing your original text. This is perfect for "filling in" the red links in a Map of Content.

### Case 5: Multi-Language Knowledge Base (ZID Pivot)
How do you maintain the same thought across several languages (Russian, English, German)? You anchor them with the same **ZID**, using the slug to define the language.

**Input (Four translations, same ZID):**
```text
20260103220434 Создать кнопку "Сделай всё хорошо" для пользователя системы.
20260103220434 Create a “Do everything well” button for the system user.
20260103220434 Erstellen Sie einen „Alles gut machen“-Button für den Systembenutzer.
```

**Output after `Alt + Q` (Four unique files, one indexable ID):**
```text
[[20260103220434-создать-кнопку-сделай-всё|Создать кнопку "Сделай всё хорошо" для пользователя системы.]]
[[20260103220434-create-a-do-everything|Create a “Do everything well” button for the system user.]]
[[20260103220434-erstellen-sie-einen-alles|Erstellen Sie einen „Alles gut machen“-Button für den Systembenutzer.]]
```

**Why this works:**
- **ZID as Anchor**: Searching for `20260103220434` will immediately list all available translations in your vault.
- **Independence**: Each file is its own atomic unit with its own descriptive slug, yet they are logically tied together forever.

---

## Behind the Scenes: Metadata & Safety

### 1. Metadata Inheritance
Every note created via `_new.md` is "born" with context:
- **`up` field**: Automatically points back to the note where you ran the script (parent-child relationship).
- **`tags`**: Inherits all tags from the parent note (both from YAML and the body).
- **`aliases`**: The primary title is automatically added to the aliases list.

### 2. Safety First
- **No Overwrites**: If a file with the same ZID-slug already exists, the script will **skip** creation to prevent data loss.
- **Sanitized Filenames**: Ensures all filenames are lowercase and cross-platform compatible (removing invalid characters like `:`, `_`, or trailing dots).

[Return to Top](#table-of-contents)

## Installation & Hotkeys

To use these scripts, copy them to your Templater templates folder and assign the following hotkeys:

| Script | Hotkey | Function |
| :--- | :--- | :--- |
| `_new.md` | `Alt + Q` | Create notes from selection |
| `_moc.md` | `Alt + A` | Synchronize Global MOCs |
| `_update.md` | `Alt + S` | Update titles & backlinks |

[Return to Top](#table-of-contents)