<%*
// 1. Получаем объект активного файла (наш родитель)
const parentFile = app.workspace.getActiveFile();
if (!parentFile) {
    new Notice("Нет активного файла для наследования.", 5000);
    return;
}

// 2. Спрашиваем у пользователя название новой задачи (сохраняем оригинальное название)
const originalTaskName = await tp.system.prompt("Название задачи");
if (!originalTaskName) {
    new Notice("Создание отменено.", 3000);
    return;
}

// 3. Создаем "безопасное" имя для файла
//    - приводим к нижнему регистру
//    - заменяем пробелы и несколько дефисов на один дефис
//    - убираем любые символы, не являющиеся буквами, цифрами или дефисом
const safeTaskName = originalTaskName.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-а-яё]/g, '');
const fileName = `${tp.date.now("YYYYMMDDHHmmss")}-${safeTaskName}`;

// 4. Извлекаем метаданные из родителя
const parentTags = tp.obsidian.getAllTags(parentFile);
const parentArea = parentFile.frontmatter?.area || "";
const parentTitle = parentFile.basename;

// 5. Собираем содержимое для нового файла (используем ОРИГИНАЛЬНОЕ название для заголовка)
const newFileContent = `---
aliases: 
type: 
status: 
project: "[[${parentTitle}]]"
area: ${parentArea}
tags: [${parentTags.join(', ')}]
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

// 7. Вставляем ссылку на новый файл в место курсора в родительском файле
const editor = app.workspace.activeEditor;
if (editor) {
    // Формируем ссылку в виде элемента списка
    const linkToInsert = `- [[${fileName}]]\n`; 
    editor.editor.replaceSelection(linkToInsert);
}

// 8. Открываем новый файл в новой вкладке
app.workspace.openLinkText(newFile.path, '', true);

// 9. Возвращаем пустую строку, чтобы "успокоить" команду вставки 
//return "";
%>
