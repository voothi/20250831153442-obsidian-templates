# Template & Script Configuration

This folder contains the core logic of your ZID-based vault. To get started, follow the instructions below to configure each component.

---

## Scripts (Templater)

These scripts automate the structure and naming of your notes. 

### 1. [_new.md](_new.md)
**Purpose**: Creates new notes from text or ZIDs.
**Configuration**:
- Open the file and look for `SLUG_WORD_COUNT`.
- **Default**: `4`. 
- **User Action**: Change this number if you want longer or shorter filenames.

### 2. [_update.md](_update.md)
**Purpose**: Synchronizes H1 titles with Aliases and refreshes backlinks.
**User Action**: Run this script (using your Templater hotkey) whenever you rename a note or update its header.

### 3. [_moc.md](_moc.md)
**Purpose**: The "Automated Graph" engine. It scans `# MOC.` sections and updates the `up` field.
**User Action**: Run this periodically to align your logical graph with your Map of Content files.

---

## Templates

### 1. [day-planner.md](day-planner.md)
**Purpose**: A structured daily planning tool.
**User Action**: Configure your **Daily Note Calendar** or **Periodic Notes** plugin to use this file as your daily template.

---

## Technical Manuals
For deep-dives into the logic and advanced usage examples, refer to the root documentation:
- [Templater Automation Guide](../../templater/README.md)
- [Daily Planning Guide](../../daily-note-calendar/README.md)
