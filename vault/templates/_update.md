<%*
// --------------------------------------------------------------------------
// Configuration
// --------------------------------------------------------------------------
// No specific config needed, works based on current file context.

// --------------------------------------------------------------------------
// Helper Functions
// --------------------------------------------------------------------------

function escapeRegExp(string) {
    return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'); 
}

// --------------------------------------------------------------------------
// Main Logic
// --------------------------------------------------------------------------

const activeFile = app.workspace.getActiveFile();
if (!activeFile) {
    new Notice("No active file detected.");
    return;
}

// 1. Get the NEW title from the # H1 header in the editor
const editor = app.workspace.activeLeaf.view.editor;
const lineCount = editor.lineCount();
let newTitle = "";

for (let i = 0; i < lineCount; i++) {
    const line = editor.getLine(i);
    // Look for the first level 1 header
    if (line.startsWith("# ")) {
        newTitle = line.substring(2).trim();
        break;
    }
}

if (!newTitle) {
    new Notice("No H1 header ('# Title') found in the current file.");
    return;
}

// 2. Get the OLD title from the Frontmatter (Aliases)
// We assume the first alias is the 'primary' title to check against.
const cache = app.metadataCache.getFileCache(activeFile);
const frontmatter = cache?.frontmatter;
let oldTitle = "";

if (frontmatter && frontmatter.aliases) {
    if (Array.isArray(frontmatter.aliases)) {
        // Use the first alias as the previous source of truth
        oldTitle = frontmatter.aliases[0]; 
    } else if (typeof frontmatter.aliases === 'string') {
        oldTitle = frontmatter.aliases;
    }
}

// If we couldn't find an alias, we can't safely determine what to replace in other files.
if (!oldTitle) {
    new Notice("No 'aliases' found in Frontmatter. Cannot determine the old title to replace.");
    return;
}

oldTitle = oldTitle.trim();

if (newTitle === oldTitle) {
    new Notice("The H1 header matches the current alias. No changes needed.");
    return;
}

// 3. Update the Frontmatter of the CURRENT file
// We use processFrontMatter to safely update the YAML without breaking other fields.
try {
    await app.fileManager.processFrontMatter(activeFile, (fm) => {
        if (!fm.aliases) {
            fm.aliases = [newTitle];
        } else if (Array.isArray(fm.aliases)) {
            // Find and replace the specific old title, or prepend if not found
            const index = fm.aliases.indexOf(oldTitle);
            if (index !== -1) {
                fm.aliases[index] = newTitle;
            } else {
                fm.aliases.push(newTitle);
            }
        } else {
            // If it was a string
            if (fm.aliases === oldTitle) {
                fm.aliases = newTitle; // Keep as string or convert to array? Sticking to array is safer usually.
            } else {
                fm.aliases = [fm.aliases, newTitle];
            }
        }
    });
    new Notice(`Updated aliases in current file: "${oldTitle}" -> "${newTitle}"`);
} catch (e) {
    console.error(e);
    new Notice("Error updating frontmatter.");
    return;
}

// 4. Search and Update Backlinks
const backlinks = app.metadataCache.getBacklinksForFile(activeFile);
let filesUpdatedCount = 0;

if (backlinks && backlinks.data) {
    const fileBasename = activeFile.basename;
    const escapedBasename = escapeRegExp(fileBasename);
    
    // Regex to find: [[Filename|Anything]]
    // We catch the 'Anything' to compare it with oldTitle.
    const linkRegex = new RegExp(`\\[\\[${escapedBasename}\\|([^\\]]+)\\]\\]`, 'g');

    for (const [sourcePath, linkItems] of backlinks.data) {
        const sourceFile = app.vault.getAbstractFileByPath(sourcePath);
        
        // Skip if source is the file itself (just in case)
        if (!sourceFile || sourceFile.path === activeFile.path) continue;
        
        // Read file content
        let sourceContent = await app.vault.read(sourceFile);
        let contentModified = false;

        // Replace logic
        sourceContent = sourceContent.replace(linkRegex, (match, linkText) => {
            // CONSTRAINT: Only change if display text matches old value exactly (trimmed)
            if (linkText.trim() === oldTitle) {
                contentModified = true;
                return `[[${fileBasename}|${newTitle}]]`;
            }
            // Otherwise return the original match (don't touch unique context links)
            return match;
        });

        if (contentModified) {
            await app.vault.modify(sourceFile, sourceContent);
            filesUpdatedCount++;
        }
    }
}

new Notice(`Update complete. Fixed links in ${filesUpdatedCount} other files.`);
%>