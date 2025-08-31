<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) {
    new Notice("Нет активного файла для наследования.", 5000);
    return;
}

// 2. Спрашиваем у пользователя название новой задачи
const originalTaskName = await tp.system.prompt("Название задачи");
if (!originalTaskName) {
    new Notice("Создание отменено.", 3000);
    return;
}

// 3. Создаем "безопасное" имя для файла
const safeTaskName = originalTaskName.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-а-яё]/g, '');
const fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;

// 4. Извлекаем метаданные
const parentMetadata = app.metadataCache.getFileCache(parentFile);
const parentTitle = parentFile.basename;
const parentArea = parentMetadata?.frontmatter?.area || "";

// --- ФИНАЛЬНАЯ ЛОГИКА ДЛЯ 'project' ---
let projectValue;
const parentProjectRaw = parentMetadata?.frontmatter?.project;

if (parentProjectRaw) {
    // Если у родителя ЕСТЬ поле project, применяем "брутальную" очистку.
    // 1. Превращаем в строку.
    // 2. Удаляем ВСЕ скобки и кавычки.
    const cleanFileName = String(parentProjectRaw)
        .replace(/\[/g, '')
        .replace(/\]/g, '')
        .replace(/"/g, '');
    
    // 3. Оборачиваем очищенное имя файла в правильную ссылку.
    projectValue = `[[${cleanFileName}]]`;

} else {
    // Если у родителя НЕТ поля project, он сам становится проектом.
    projectValue = `[[${parentTitle}]]`;
}


// --- БЛОК ТЕГОВ (без изменений) ---
let frontmatterTags = [];
const tagsFromFM = parentMetadata?.frontmatter?.tags;
if (tagsFromFM) {
    if (Array.isArray(tagsFromFM)) {
        frontmatterTags = tagsFromFM;
    } else {
        frontmatterTags = String(tagsFromFM).split(',').map(t => t.trim());
    }
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
type: задача
status: к_выполнению
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

// 7. Вставляем ссылку в родительский файл
const editor = app.workspace.activeEditor;
if (editor) {
    const linkToInsert = `- [[${fileName}]]\n`; 
    editor.editor.replaceSelection(linkToInsert);
}

// 8. Открываем новый файл
app.workspace.openLinkText(newFile.path, '', true);
%>