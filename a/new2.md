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
const parentTags = tp.obsidian.getAllTags(parentFile);
const parentArea = parentFile.frontmatter?.area || "";
const parentTitle = parentFile.basename;

// 5. Собираем содержимое для нового файла
const newFileContent = `---
aliases: 
type: 
status: 
project: 
area: ${parentArea}
tags: [${parentTags.join(', ')}]
created: ${tp.date.now("YYYY-MM-DD")}
due: 
up: "[[${parentTitle}]]"
down: 
prev: 
next: 
---

# ${taskName}

#### Описание

#### Заметки
`;

// 6. Создаем новый файл
const newFile = await app.vault.create(`${fileName}.md`, newFileContent);

// 7. Вставляем ссылку на новый файл в место курсора в родительском файле
const editor = app.workspace.activeEditor;
if (editor) {
    // Формируем ссылку в виде элемента списка
    const linkToInsert = `- [[${fileName}]]\n`; 
    editor.editor.replaceSelection(linkToInsert);
}

// 8. Открываем новый файл в новой вкладке
app.workspace.openLinkText(newFile.path, '', true);
%>