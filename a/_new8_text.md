<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) { new Notice("Нет активного файла для наследования.", 5000); return; }

// 2. Получаем выделенный текст
const originalTaskName = tp.file.selection().trim();
if (!originalTaskName) { new Notice("Ничего не выделено. Создание отменено.", 3000); return; }

// 3. Создаем "безопасное" имя для файла
const safeTaskName = originalTaskName.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-а-яё]/g, '');
const fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;

// 4. Извлекаем метаданные родителя
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

// Блок тегов
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

// 5. Собираем содержимое для нового файла
const newFileContent = `---
aliases: [${originalTaskName}]
type: 
status: 
project: ${projectValue}
area: ${parentArea}
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: "[[${parentTitle}]]"
---

# ${originalTaskName}

## Описание

## Заметки
`;

// 6. Создаем новый файл
const newFile = await app.vault.create(`${fileName}.md`, newFileContent);

// 7. ЗАМЕНЯЕМ выделенный текст на ссылку с алиасом
const editor = app.workspace.activeEditor;
if (editor) {
    const linkToInsert = `[[${fileName}|${originalTaskName}]]`;
    editor.editor.replaceSelection(linkToInsert);
}

// 8. Открываем новый файл
await app.workspace.openLinkText(newFile.path, '', true);
%>