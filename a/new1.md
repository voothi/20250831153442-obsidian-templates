<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) {
    new Notice("Нет активного файла для наследования.", 5000);
    return;
}

// 2. Спрашиваем у пользователя название новой задачи
const taskName = await tp.system.prompt("Название задачи");
if (!taskName) {
    new Notice("Создание отменено.", 3000);
    return;
}

// 3. Формируем уникальное имя для нового файла
const fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${taskName}`;

// 4. Извлекаем метаданные из родителя
const parentTags = tp.obsidian.getAllTags(parentFile); // Получаем все теги, включая из текста
const parentArea = parentFile.frontmatter?.area || ""; // Безопасно получаем 'area'
const parentTitle = parentFile.basename;

// 5. Собираем содержимое для нового файла (здесь можно использовать переменные)
const newFileContent = `---
aliases: 
type: задача
status: к_выполнению
project: "[[${parentTitle}]]"
area: ${parentArea}
tags: [${parentTags.join(', ')}]
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: "[[${parentTitle}]]"
---

# ${taskName}

#### Описание

#### Заметки
`;

// 6. Создаем новый файл и открываем его
const newFile = await app.vault.create(`${fileName}.md`, newFileContent);
app.workspace.openLinkText(newFile.path, '', true);
%>

