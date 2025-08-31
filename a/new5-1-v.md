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

// 4. Извлекаем метаданные самым надежным способом
const parentMetadata = app.metadataCache.getFileCache(parentFile);
const parentArea = parentMetadata?.frontmatter?.area || "";
const parentTitle = parentFile.basename;

// --- НАЧАЛО ФИНАЛЬНОГО БЛОКА ТЕГОВ ---
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
// --- КОНЕЦ ФИНАЛЬНОГО БЛОКА ТЕГОВ ---

// 5. Собираем содержимое для нового файла
let tagsYamlBlock = "[]"; 
if (allUniqueTags.length > 0) {
    tagsYamlBlock = "\n  - " + allUniqueTags.join("\n  - ");
}
const newFileContent = `---
aliases: [${originalTaskName}]
type: 
status: 
project: 
area: 
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: "[[${parentTitle}]]"
down: 
prev: 
next: 
---

# ${originalTaskName}

## Описание

## Заметки


`;

// 6. Создаем новый файл с "безопасным" именем
const newFile = await app.vault.create(`${fileName}.md`, newFileContent);

// 7. Вставляем ссылку с алиасом на новый файл в место курсора в родительском файле
const editor = app.workspace.activeEditor;
if (editor) {
    // Формируем ссылку с алиасом в виде элемента списка
    // [[имя-файла|Текст, который будет виден]]
    // const linkToInsert = `- [[${fileName}|${originalTaskName}]]\n`; // <--- ВОТ ИЗМЕНЕНИЕ
    const linkToInsert = `[[${fileName}|${originalTaskName}]]`; // <--- ВОТ ИЗМЕНЕНИЕ
    editor.editor.replaceSelection(linkToInsert);
}

// 8. Открываем новый файл в новой вкладке
app.workspace.openLinkText(newFile.path, '', true);

// 9. Возвращаем пустую строку, чтобы "успокоить" команду вставки 
//return "";
%>