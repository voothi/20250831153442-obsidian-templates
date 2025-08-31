<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) { new Notice("Нет активного файла для наследования.", 5000); return; }

// --- НОВОЕ: Получаем путь к папке родительского файла ---
const parentFolderPath = parentFile.parent.path;

// 2. Получаем выделенный текст
const selection = tp.file.selection();
if (!selection) { new Notice("Ничего не выделено. Создание отменено.", 3000); return; }

// 3. ПЫТАЕМСЯ найти wikilinks в выделенном тексте
const linkRegex = /\[\[([^|\]]+)(?:\|([^\]]+))?\]\]/g;
const matches = [...selection.matchAll(linkRegex)];

// --- ОБЩИЙ БЛОК ИЗВЛЕЧЕНИЯ МЕТАДАННЫХ ---
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
// --- КОНЕЦ ОБЩЕГО БЛОКА ---


if (matches.length > 0) {
    // --- РЕЖИМ 1: Найдены ссылки, запускаем массовое создание ---

    new Notice(`Найдено ${matches.length} ссылок. Начинаю обработку...`, 3000);
    let filesCreatedCount = 0;

    for (const match of matches) {
        const fileName = match[1].trim(); 
        const originalTaskName = match[2] ? match[2].trim() : fileName;
        
        // --- НОВОЕ: Формируем ПОЛНЫЙ путь к файлу ---
        const filePath = parentFolderPath === '/' ? `${fileName}.md` : `${parentFolderPath}/${fileName}.md`;

        if (await app.vault.adapter.exists(filePath)) { continue; }

        const newFileContent = `---
aliases: [${originalTaskName}]
type: 
status: 
up: "[[${parentTitle}]]"
down: 
prev: 
next: 
same: 
project: 
area: 
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: "[[${parentTitle}]]"
---

# ${originalTaskName}

## Описание.

## Заметки.
`;

        await app.vault.create(filePath, newFileContent);
        filesCreatedCount++;
    }

    new Notice(`Обработка завершена. Создано новых файлов: ${filesCreatedCount}.`, 5000);
    tR += selection; // Оставляем выделение без изменений

} else {
    // --- РЕЖИМ 2: Ссылок не найдено, работаем с простым текстом ---
    
    const originalTaskName = selection.trim();
    const safeTaskName = originalTaskName.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-а-яё]/g, '');
    const fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;
    
    // --- НОВОЕ: Формируем ПОЛНЫЙ путь к файлу ---
    const filePath = parentFolderPath === '/' ? `${fileName}.md` : `${parentFolderPath}/${fileName}.md`;
    
    if (await app.vault.adapter.exists(filePath)) {
        new Notice(`Файл "${fileName}.md" уже существует.`, 5000);
        return;
    }

    const newFileContent = `---
aliases: [${originalTaskName}]
type: 
status: 
up: "[[${parentTitle}]]"
down: 
prev: 
next: 
same: 
project: 
area: 
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: "[[${parentTitle}]]"
---

# ${originalTaskName}

## Описание.

## Заметки.
`;

    const newFile = await app.vault.create(filePath, newFileContent);
    const editor = app.workspace.getActiveFileView().editor;
    if (editor) {
        const linkToInsert = `[[${fileName}|${originalTaskName}]]`;
        editor.replaceSelection(linkToInsert);
    }
    await app.workspace.openLinkText(newFile.path, '', true);
}
%>