<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 1.1
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Этот скрипт читает иерархическую структуру из выделенного текста
 или текущей заметки (MOC-файла) и автоматически проставляет
 `up` property (ссылку на родителя) в frontmatter каждой дочерней заметки.

 Как использовать:
 1. В MOC-файле создайте иерархию с помощью отступов (пробелы или табы).
    - [[Корень]]
        - [[Родитель 1]]
            - [[Дочерний элемент 1.1]]
            - [[Дочерний элемент 1.2]]
        - [[Родитель 2]]
            - [[Дочерний элемент 2.1]]
 2. Выделите этот текст.
 3. Вызовите палитру команд (Ctrl/Cmd + P).
 4. Найдите и выберите команду "Templater: Open Insert Template modal".
 5. Выберите этот шаблон ("Update Hierarchy from MOC").
 6. Скрипт обработает выделение и обновит файлы.
    Если текст не выделен, скрипт обработает всю текущую заметку.
================================================================================
*/

async function updateHierarchyFromMOC(tp) {
  // 1. Получаем исходный текст (либо выделение, либо вся заметка)
  let mocContent = await tp.file.selection();
  const activeFile = app.workspace.getActiveFile();

  if (!mocContent && activeFile) {
    new Notice("ℹ️ Выделение не найдено. Обрабатывается вся заметка.", 2000);
    mocContent = await app.vault.read(activeFile);
  } else if (!mocContent) {
    new Notice("❌ Ошибка: Нет активного файла или выделенного текста.", 5000);
    return;
  }

  // 2. Парсим иерархию
  const lines = mocContent.split('\n').filter(line => line.trim() !== '');
  const parentStack = []; // Стек для отслеживания родителей на разных уровнях
  const relationships = []; // Массив для хранения пар { child, parent }

  for (const line of lines) {
    // Ищем отступ и ссылку в строке
    const indentMatch = line.match(/^(\s*)/);
    const linkMatch = line.match(/\[\[(.*?)(?:\|.*?)?\]\]/); // Правильно обрабатывает алиасы [[Ссылка|Алиас]]

    if (!linkMatch) {
      continue; // Пропускаем строки без ссылок
    }

    const currentIndent = indentMatch[1].length;
    const currentName = linkMatch[1]; // Имя файла без [[ ]]

    // Поднимаемся по стеку, пока не найдем родителя с меньшим отступом
    while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) {
      parentStack.pop();
    }

    // Если в стеке кто-то остался - это наш родитель
    if (parentStack.length > 0) {
      const parentName = parentStack[parentStack.length - 1].name;
      relationships.push({
        child: currentName,
        parent: `[[${parentName}]]`
      });
    }

    // Добавляем текущий элемент в стек как потенциального родителя для следующих
    parentStack.push({ indent: currentIndent, name: currentName });
  }

  // 3. Обновляем frontmatter в файлах
  if (relationships.length === 0) {
    new Notice("ℹ️ Не найдено дочерних элементов для обновления.", 3000);
    return;
  }

  let updatedCount = 0;
  let notFoundCount = 0;
  const notFoundFiles = [];

  new Notice(`⏳ Начинаю обновление... Найдено связей: ${relationships.length}`, 3000);

  for (const rel of relationships) {
    const childFile = tp.file.find_tfile(rel.child);

    if (!childFile) {
      console.warn(`Файл не найден для заметки: "${rel.child}"`);
      notFoundCount++;
      notFoundFiles.push(rel.child);
      continue;
    }

    // Безопасное изменение frontmatter
    await app.fileManager.processFrontMatter(childFile, (fm) => {
      // Если значение уже такое же, ничего не делаем, чтобы не менять дату модификации файла
      if (fm['up'] !== rel.parent) {
         fm['up'] = rel.parent;
         updatedCount++;
      }
    });
  }

  // 4. Выводим результат
  let summary = `✅ Обновление завершено.\n`;
  if(updatedCount > 0) {
      summary += `Обновлено 'up' в ${updatedCount} файлах.\n`;
  } else {
      summary += `Все 'up' атрибуты уже были актуальны.\n`;
  }

  if (notFoundCount > 0) {
    summary += `❌ Не найдено ${notFoundCount} файлов: ${notFoundFiles.join(', ')}`;
    console.error("Не найдены следующие файлы:", notFoundFiles);
  }
  new Notice(summary, 10000);
}

// Запускаем основную функцию
try {
  await updateHierarchyFromMOC(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}

%>