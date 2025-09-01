<%*
// Helper function to sanitize file names, inspired by the python script.
// Replaces special characters (like umlauts) and formats the name.
function sanitizeName(inputString) {
    const replacements = {
        'ä': 'ae', 'ö': 'oe', 'ü': 'ue', 'ß': 'ss',
        'Ä': 'ae', 'Ö': 'oe', 'Ü': 'ue', 
        '_': '-', ':': '-', '. ': '-', '.': '-'
    };

    let processedString = inputString;
    for (const char in replacements) {
        processedString = processedString.split(char).join(replacements[char]);
    }
    
    // Remove characters that are not letters, numbers, spaces, or hyphens.
    // Keeps latin and cyrillic characters.
    let cleanedForSplitting = processedString.replace(/[^a-zA-Zа-яА-ЯёЁ0-9\s-]/g, '');

    const words = cleanedForSplitting.trim().split(/\s+/);
    const firstWords = words.slice(0, 6); // Use up to 6 words
    let finalName = firstWords.join('-');

    // NEW: Clean up any trailing hyphens that might result from punctuation at the end.
    finalName = finalName.replace(/-+$/, '');

    return finalName.toLowerCase();
}

// 1. Get the active file object (our parent)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) { new Notice("No active file to inherit from.", 5000); return; }

const parentFolderPath = parentFile.parent.path;
const selection = tp.file.selection();
if (!selection) { new Notice("Nothing selected. Creation cancelled.", 3000); return; }

const linkRegex = /\[\[([^|\]]+)(?:\|([^\]]+))?\]\]/g;
const matches = [...selection.matchAll(linkRegex)];

const parentMetadata = app.metadataCache.getFileCache(parentFile);
const parentTitle = parentFile.basename;
const parentArea = parentMetadata?.frontmatter?.area || "";
let projectValue;
const parentProjectRaw = parentMetadata?.frontmatter?.project;
if (parentProjectRaw) {
    const cleanFileName = String(parentProjectRaw).replace(/\[|\]|"/g, '');
    projectValue = `[[${cleanFileName}]]`;
} else {
    projectValue = `[[${parentTitle}]]`;
}

let frontmatterTags = [];
const tagsFromFM = parentMetadata?.frontmatter?.tags;
if (tagsFromFM) {
    if (Array.isArray(tagsFromFM)) { frontmatterTags = tagsFromFM; } 
    else { frontmatterTags = String(tagsFromFM).split(',').map(t => t.trim()); }
}
frontmatterTags = frontmatterTags.filter(Boolean);
const bodyTags = parentMetadata?.tags?.map(t => t.tag.substring(1)) || [];
const allUniqueTags = [...new Set([...frontmatterTags, ...bodyTags])];
let tagsYamlBlock = "[]"; 
if (allUniqueTags.length > 0) {
    tagsYamlBlock = "\n  - " + allUniqueTags.join("\n  - ");
}


if (matches.length > 0) {
    // --- MODE 1: Links found, start batch creation ---
    new Notice(`Found ${matches.length} links. Processing...`, 3000);
    let filesCreatedCount = 0;

    for (const match of matches) {
        const originalLinkText = match[1].trim(); 
        const originalTaskName = match[2] ? match[2].trim() : originalLinkText;
        
        // Sanitize the file name using the new function
        const sanitizedBaseName = sanitizeName(originalLinkText);
        const filePath = parentFolderPath === '/' ? `${sanitizedBaseName}.md` : `${parentFolderPath}/${sanitizedBaseName}.md`;

        if (await app.vault.adapter.exists(filePath)) { continue; }
        
        const aliasesYamlBlock = `\n  - ${originalTaskName}`;

        const newFileContent = `---
aliases: ${aliasesYamlBlock}
up: "[[${parentTitle}]]"
type: 
status: 
down: 
prev: 
next: 
same: 
project: 
area: 
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
---

# ${originalTaskName}

## Description.

## MOC.

## Notes.
`;

        await app.vault.create(filePath, newFileContent);
        filesCreatedCount++;
    }

    new Notice(`Processing complete. New files created: ${filesCreatedCount}.`, 5000);
    tR += selection; 

} else {
    // --- MODE 2: No links found, process plain text ---
    
    const originalTaskName = selection.trim();
    let fileName;
    let cleanTaskName = originalTaskName;

    const zidRegex = /^(\d{14})\s+(.*)$/s;
    const zidMatch = originalTaskName.match(zidRegex);

    if (zidMatch) {
        // CASE A: ZID found.
        new Notice("ZID detected. Using it.", 2000);
        const existingZid = zidMatch[1];
        const textAfterZid = zidMatch[2].trim();
        cleanTaskName = textAfterZid;
        
        const safeTaskName = sanitizeName(textAfterZid);
        fileName = `${existingZid}-${safeTaskName}`;

    } else {
        // CASE B: ZID not found.
        const safeTaskName = sanitizeName(originalTaskName);
        fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;
    }

    const filePath = parentFolderPath === '/' ? `${fileName}.md` : `${parentFolderPath}/${fileName}.md`;
    
    if (await app.vault.adapter.exists(filePath)) {
        new Notice(`File "${fileName}.md" already exists.`, 5000);
        return;
    }
    
    const aliasesYamlBlock = `\n  - ${cleanTaskName}`;

    const newFileContent = `---
aliases: ${aliasesYamlBlock}
up: "[[${parentTitle}]]"
type: 
status: 
down: 
prev: 
next: 
same: 
project: 
area: 
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
---

# ${cleanTaskName}

## Description.

## MOC.

## Notes.
`;

    const newFile = await app.vault.create(filePath, newFileContent);
    const editor = app.workspace.getActiveFileView().editor;
    if (editor) {
        const linkToInsert = `[[${fileName}|${cleanTaskName}]]`;
        editor.replaceSelection(linkToInsert);
    }
    await app.workspace.openLinkText(newFile.path, '', true);
}
%>