<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) {
    new Notice("Нет активного файла для наследования.", 5000);
    return;
}

// 2. Получаем выделенный текст. Это будет наше название и алиас.
// tp.file.selection() - получает выделенный текст.
// .trim() - убирает случайные пробелы в начале и в конце.
const originalTaskName = tp.file.selection().trim();
if (!originalTaskName) {
    new Notice("Ничего не выделено. Создание отменено.", 3000);
    return;
}

// 3. Создаем "безопасное" имя для файла из выделенного текста
const safeTaskName = originalTaskName.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-а-яё]/g, '');
const fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;

// 4. Извлекаем метаданные родителя (включая project)
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

// --- БЛОК ТЕГОВ (без изменений) ---
let frontmatterTags = [];
const tagsFromFM = parentMetadata?.frontmatter?.tags;
if (tagsFromFM) {
    if (Array.isArray(tagsFromFM)) { frontmatterTags = tagsFromFM; } 
    else { frontmatterTags = String(tagsFromFM).split(',').map(t => t.trim()); }
}
frontmatterTags = frontmatterTags.filter(Boolean);
const bodyTags = parentMetadata?.tags?.map(t => t.tag.substring(1)) || [];
const allUniqueTags = [...new Set([...frontmatterTags, ...bodyTags])];

// 5. Собираем содержимое для нового файла
let tagsYamlBlock = "[]"; 
if (allUniqueTags.length > 0) {
    tagsYamlBlock = "\n  - " + allUniqueTags.join("\n  - ");
}
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

// 6. Создаем новый файл с "безопасным" именем
const newFile = await app.vault.create(`${fileName}.md`, newFileContent);

// 7. ЗАМЕНЯЕМ выделенный текст на ссылку с алиасом
const editor = app.workspace.activeEditor;
if (editor) {
    // Формируем ссылку с алиасом [[имя-файла|Текст, который был выделен]]
    const linkToInsert = `[[${fileName}|${originalTaskName}]]`;
    // Эта команда заменит выделенный текст на нашу новую ссылку
    editor.editor.replaceSelection(linkToInsert);
}

// 8. Открываем новый файл в новой вкладке
app.workspace.openLinkText(newFile.path, '', true);
%>