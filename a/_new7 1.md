<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) { new Notice("Нет активного файла для наследования.", 5000); return; }

// 2. Получаем выделенный текст
const selection = tp.file.selection();
if (!selection) { new Notice("Ничего не выделено. Создание отменено.", 3000); return; }

// 3. Находим ВСЕ wikilinks в выделенном тексте
const linkRegex = /\[\[([^|\]]+)(?:\|([^\]]+))?\]\]/g;
const matches = [...selection.matchAll(linkRegex)];

if (matches.length === 0) {
    new Notice("В выделенном тексте не найдено ссылок для создания.", 3000);
    return;
}

new Notice(`Найдено ${matches.length} ссылок. Начинаю обработку...`, 3000);

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

// --- НОВОЕ: Инициализируем счетчик созданных файлов ---
let filesCreatedCount = 0;

// 5. Запускаем ЦИКЛ для каждой найденной ссылки
for (const match of matches) {
    const fileName = match[1].trim(); 
    const originalTaskName = match[2] ? match[2].trim() : fileName;
    const filePath = `${fileName}.md`;

    if (await app.vault.adapter.exists(filePath)) {
        console.log(`Файл "${fileName}.md" уже существует, пропускаю.`);
        continue; 
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

    await app.vault.create(filePath, newFileContent);
    // --- НОВОЕ: Увеличиваем счетчик после успешного создания ---
    filesCreatedCount++;
}

// 6. Сообщаем о завершении с корректным числом
new Notice(`Обработка завершена. Создано новых файлов: ${filesCreatedCount}.`, 5000);

// 7. Говорим Templater, что нужно оставить выделение как есть
tR += selection;

%>