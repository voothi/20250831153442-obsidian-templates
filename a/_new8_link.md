<%*
// 1. Получаем имя новой заметки из самой ссылки
const originalTaskName = tp.config.target_file.basename;

// 2. Идентифицируем родительский файл, из которого кликнули
const parentFile = app.workspace.getActiveFile();

// 3. Инициализируем переменные
let parentTitle = "", parentArea = "", projectValue = "", tagsYamlBlock = "[]";

// 4. Проверяем, что родитель определился и это не тот же файл, что мы создаем
if (parentFile && parentFile.basename !== originalTaskName) {
    // --- БЛОК НАСЛЕДОВАНИЯ (ТОЧНО ТАКОЙ ЖЕ, КАК В ПЕРВОМ ШАБЛОНЕ) ---
    const parentMetadata = app.metadataCache.getFileCache(parentFile);
    parentTitle = parentFile.basename;
    parentArea = parentMetadata?.frontmatter?.area || "";
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
    if (allUniqueTags.length > 0) {
        tagsYamlBlock = "\n  - " + allUniqueTags.join("\n  - ");
    }
    // --- КОНЕЦ БЛОКА НАСЛЕДОВАНИЯ ---
}

// 5. Формируем и ВОЗВРАЩАЕМ содержимое для нового файла.
tR += `---
aliases: [${originalTaskName}]
type: задача
status: к_выполнению
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
%>