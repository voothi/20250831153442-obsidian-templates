<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 1.2
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Этот скрипт читает иерархическую структуру из выделенного текста
 или текущей заметки (MOC-файла) и автоматически проставляет
 `up` property (ссылку на родителя) в frontmatter каждой дочерней заметки.
 Корневые узлы (без родителя в иерархии) получают ссылку на сам MOC-файл.

 Как использовать:
 1. В MOC-файле создайте иерархию с помощью отступов.
 2. Выделите текст с иерархией (или просто находитесь в заметке).
 3. Запустите скрипт через горячую клавишу или палитру команд.
================================================================================
*/

async function updateHierarchyFromMOC(tp) {
  // 1. Получаем исходный текст и активный файл
  let mocContent = await tp.file.selection();
  const activeFile = app.workspace.getActiveFile();

  if (!activeFile) {
      new Notice("❌ Ошибка: Нет активного файла для обработки.", 5000);
      return;
  }

  if (!mocContent) {
    new Notice("ℹ️ Выделение не найдено. Обрабатывается вся заметка.", 2000);
    mocContent = await app.vault.read(activeFile);
  }

  // <<< ИЗМЕНЕНИЕ 1: Получаем имя MOC-файла для ссылки на корень
  const mocFileNameForLink = activeFile.basename; // Имя файла без .md

  // 2. Парсим иерархию
  const lines = mocContent.split('\n').filter(line => line.trim() !== '');
  const parentStack = []; 
  const relationships = []; 

  for (const line of lines) {
    const indentMatch = line.match(/^(\s*)/);
    const linkMatch = line.match(/\[\[(.*?)(?:\|.*?)?\]\]/);

    if (!linkMatch) {
      continue;
    }

    const currentIndent = indentMatch[1].length;
    const currentName = linkMatch[1]; 

    while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) {
      parentStack.pop();
    }

    // <<< ИЗМЕНЕНИЕ 2: Основная логика для определения родителя
    if (parentStack.length > 0) {
      // Это дочерний узел, у него есть родитель в стеке
      const parentName = parentStack[parentStack.length - 1].name;
      relationships.push({
        child: currentName,
        parent: `[[${parentName}]]`
      });
    } else {
      // Это корневой узел (нет родителя в стеке)
      // Устанавливаем его родителем сам MOC-файл
      relationships.push({
        child: currentName,
        parent: `[[${mocFileNameForLink}]]`
      });
    }

    parentStack.push({ indent: currentIndent, name: currentName });
  }

  // 3. Обновляем frontmatter в файлах
  if (relationships.length === 0) {
    new Notice("ℹ️ Не найдено ссылок для обработки в тексте.", 3000);
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
    
    // Проверяем, не является ли дочерний файл самим MOC-файлом, чтобы избежать цикла
    if (childFile.basename === mocFileNameForLink) {
        continue;
    }

    await app.fileManager.processFrontMatter(childFile, (fm) => {
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