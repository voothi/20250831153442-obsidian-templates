<%*
// Configuration: Set to true to split the first sentence into the title/link 
// and the rest into the Description section.
const SPLIT_DESCRIPTION = true; 
const SLUG_WORD_COUNT = 4; // Number of words to keep in the filename slug.

// --- HELPER FUNCTIONS ---

// 1. Sanitize File Names
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
    let cleanedForSplitting = processedString.replace(/[^a-zA-Zа-яА-ЯёЁ0-9\s-]/g, '');

    const words = cleanedForSplitting.trim().split(/\s+/);
    const firstWords = words.slice(0, SLUG_WORD_COUNT); 
    let finalName = firstWords.join('-');

    // Clean up trailing hyphens
    finalName = finalName.replace(/-+$/, '');

    return finalName.toLowerCase();
}

// 2. Create Note File (Centralized Logic)
async function createNoteFile(params) {
    const { 
        filename, 
        taskName, 
        description, 
        parentFolderPath, 
        parentTitle, 
        tagsYamlBlock 
    } = params;

    const filePath = parentFolderPath === '/' ? `${filename}.md` : `${parentFolderPath}/${filename}.md`;

    // Check if file exists. If yes, return false (did not create).
    if (await app.vault.adapter.exists(filePath)) { 
        return false; 
    }

    const aliasesYamlBlock = `\n  - ${taskName}`;

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

# ${taskName}

## Description

${description || ""}

## MOC.



## Notes


`;

    await app.vault.create(filePath, newFileContent);
    return true; // Created successfully
}

// --- MAIN SCRIPT ---

const parentFile = app.workspace.getActiveFile();
if (!parentFile) { new Notice("No active file to inherit from.", 5000); return; }

const parentFolderPath = parentFile.parent.path;
const selection = tp.file.selection();
if (!selection) { new Notice("Nothing selected. Creation cancelled.", 3000); return; }

// Metadata & Tags Preparation
const parentMetadata = app.metadataCache.getFileCache(parentFile);
const parentTitle = parentFile.basename;
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

// Regex Definitions
const linkRegex = /\[\[([^|\]]+)(?:\|([^\]]+))?\]\]/g;
const zidLineRegex = /^(\s*(?:(?:[-*+]|\d+\.)(?:\s+\[[ xX]\])?\s+)?)(\d{14})\s+(.*)$/;

const lines = selection.split(/\r?\n/);
const hasZidLines = lines.some(line => zidLineRegex.test(line));
const matches = [...selection.matchAll(linkRegex)];

// --- LOGIC FLOW ---

if (hasZidLines) {
    // --- MODE 3: Mixed Content / Batch Processing ---
    // Handles ZID lines (converts & creates) AND existing inactive links (creates).
    
    new Notice("ZID lines detected. Processing batch...", 2000);
    let processedLines = [];
    let filesCreatedCount = 0;

    for (let line of lines) {
        const zidMatch = line.match(zidLineRegex);
        
        if (zidMatch) {
            // Case A: It is a ZID line. Convert it.
            const prefix = zidMatch[1] || ""; 
            const zid = zidMatch[2];
            let rawText = zidMatch[3];
            let cleanTaskName = rawText;
            let descriptionText = "";

            if (SPLIT_DESCRIPTION) {
                const splitRegex = /^(.*?[.?!])(?:\s+|$)(.*)$/s;
                const splitMatch = rawText.match(splitRegex);
                if (splitMatch) {
                    cleanTaskName = splitMatch[1].trim();
                    descriptionText = splitMatch[2] ? splitMatch[2].trim() : "";
                }
            }

            const safeTaskName = sanitizeName(cleanTaskName);
            const fileName = `${zid}-${safeTaskName}`;
            
            // Create file for the ZID entry
            const created = await createNoteFile({
                filename: fileName,
                taskName: cleanTaskName,
                description: descriptionText,
                parentFolderPath,
                parentTitle,
                tagsYamlBlock
            });
            if (created) filesCreatedCount++;

            // Replace line with link
            processedLines.push(`${prefix}[[${fileName}|${cleanTaskName}]]`);

        } else {
            // Case B: Not a ZID line (Existing link, list item with link, or empty).
            // Check for inactive links within this line and create files if needed.
            const lineLinks = [...line.matchAll(linkRegex)];
            
            for (const match of lineLinks) {
                const originalLinkText = match[1].trim(); 
                const originalTaskName = match[2] ? match[2].trim() : originalLinkText;
                const sanitizedBaseName = sanitizeName(originalLinkText);
                
                // Create file for the existing link (if missing)
                const created = await createNoteFile({
                    filename: sanitizedBaseName,
                    taskName: originalTaskName,
                    description: "", // No description for existing links
                    parentFolderPath,
                    parentTitle,
                    tagsYamlBlock
                });
                if (created) filesCreatedCount++;
            }

            // Preserve the line exactly as is
            processedLines.push(line);
        }
    }

    new Notice(`Batch complete. Created ${filesCreatedCount} files.`, 3000);
    tR += processedLines.join("\n");

} else if (matches.length > 0) {
    // --- MODE 1: Only Links Found (Legacy Batch Creation) ---
    // Creates files for existing [[links]] but changes nothing in text.
    
    new Notice(`Found ${matches.length} links. Creating files...`, 3000);
    let filesCreatedCount = 0;

    for (const match of matches) {
        const originalLinkText = match[1].trim(); 
        const originalTaskName = match[2] ? match[2].trim() : originalLinkText;
        const sanitizedBaseName = sanitizeName(originalLinkText);

        const created = await createNoteFile({
            filename: sanitizedBaseName,
            taskName: originalTaskName,
            description: "",
            parentFolderPath,
            parentTitle,
            tagsYamlBlock
        });
        if (created) filesCreatedCount++;
    }

    new Notice(`Processing complete. New files created: ${filesCreatedCount}.`, 5000);
    tR += selection; 

} else {
    // --- MODE 2: Single Plain Text Entry ---
    
    const originalText = selection.trim();
    let fileName;
    let cleanTaskName = originalText;
    let descriptionText = "";

    const zidMatch = originalText.match(/^(\d{14})\s+(.*)$/s);
    let existingZid = null;

    if (zidMatch) {
        existingZid = zidMatch[1];
        cleanTaskName = zidMatch[2].trim();
    } 

    if (SPLIT_DESCRIPTION) {
        const splitRegex = /^(.*?[.?!])(?:\s+|$)(.*)$/s;
        const splitMatch = cleanTaskName.match(splitRegex);

        if (splitMatch) {
            cleanTaskName = splitMatch[1].trim(); 
            descriptionText = splitMatch[2] ? splitMatch[2].trim() : ""; 
        }
    }

    const safeTaskName = sanitizeName(cleanTaskName);
    
    if (existingZid) {
        fileName = `${existingZid}-${safeTaskName}`;
    } else {
        fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;
    }

    // Note: We don't use the helper here because we need the 'newFile' object to open it.
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

## Description

${descriptionText}

## MOC.



## Notes


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