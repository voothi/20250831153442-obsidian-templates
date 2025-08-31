<%*
// --- ШАГ 1: УНИВЕРСАЛЬНОЕ ОПРЕДЕЛЕНИЕ КОНТЕКСТА ---

const selection = tp.file.selection().trim();
const targetFile = tp.config.target_file;
let originalTaskName;
let parentFile;
let isFromLinkClick = false;

if (targetFile && !await app.vault.adapter.exists(targetFile.path)) {
    // РЕЖИМ 1: Создание по клику на несуществующую ссылку
    isFromLinkClick = true;
    originalTaskName = targetFile.basename; // Имя берем из ссылки
    parentFile = app.workspace.getActiveFile(); // Родитель - активный файл
} else if (selection) {
    // РЕЖИМ 2: Создание из выделения
    originalTaskName = selection;
    parentFile = app.workspace.getActiveFile(); // Родитель - активный файл
} else {
    // РЕЖИМ 3: Создание с нуля
    parentFile = app.workspace.getActiveFile(); // Родитель все равно активный файл
    originalTaskName = await tp.system.prompt("Название задачи");
    if (!originalTaskName) {
        new Notice("Создание отменено.", 3000);
        return;
    }
}

// --- ШАГ 2: ПОДГОТОВКА ДАННЫХ (имя файла, метаданные) ---

// Создаем "безопасное" имя файла, если оно еще не задано кликом по ссылке
const safeTaskName = originalTaskName.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-а-яё]/g, '');
const fileName = isFromLinkClick ? originalTaskName : `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;

// Извлекаем метаданные родителя (если он есть)
let parentTitle = "", parentArea = "", projectValue = "", tagsYamlBlock = "[]";
if (parentFile) {
    parentTitle = parentFile.basename;
    const parentMetadata = app.metadataCache.getFileCache(parentFile);

    if (parentMetadata) {
        parentArea = parentMetadata.frontmatter?.area || "";
        // Логика наследования проекта
        const parentProjectRaw = parentMetadata.frontmatter?.project;
        if (parentProjectRaw) {
            const cleanFileName = String(parentProjectRaw).replace(/\[|\]|"/g, '');
            projectValue = `[[${cleanFileName}]]`;
        } else {
            projectValue = `[[${parentTitle}]]`;
        }
        // Логика наследования тегов
        let frontmatterTags = [];
        const tagsFromFM = parentMetadata.frontmatter?.tags;
        if (tagsFromFM) {
            if (Array.isArray(tagsFromFM)) { frontmatterTags = tagsFromFM; } 
            else { frontmatterTags = String(tagsFromFM).split(',').map(t => t.trim()); }
        }
        frontmatterTags = frontmatterTags.filter(Boolean);
        const bodyTags = parentMetadata.tags?.map(t => t.tag.substring(1)) || [];
        const allUniqueTags = [...new Set([...frontmatterTags, ...bodyTags])];
        if (allUniqueTags.length > 0) {
            tagsYamlBlock = "\n  - " + allUniqueTags.join("\n  - ");
        }
    }
}

// --- ШАГ 3: ФОРМИРОВАНИЕ СОДЕРЖИМОГО И ДЕЙСТВИЯ ---

const newFileContent = `---
aliases: [${originalTaskName}]
type: 
status: 
project: ${projectValue}
area: ${parentArea}
tags: ${tagsYamlBlock}
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: ${parentFile ? `[[${parentFile.basename}]]` : ""}
---

# ${originalTaskName}

## Описание

## Заметки
`;

// Если мы создаем по клику, Templater сам вставит содержимое.
if (isFromLinkClick) {
    tR += newFileContent;
} else {
    // Иначе, мы все делаем сами: создаем файл и заменяем ссылку
    const newFile = await app.vault.create(`${fileName}.md`, newFileContent);
    const editor = app.workspace.activeEditor;
    if (editor) {
        const linkToInsert = `[[${fileName}|${originalTaskName}]]`;
        editor.editor.replaceSelection(linkToInsert);
    }
    await app.workspace.openLinkText(newFile.path, '', true);
}
%>