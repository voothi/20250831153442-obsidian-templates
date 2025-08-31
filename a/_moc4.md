<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 1.5
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Этот скрипт читает иерархическую структуру из выделенного текста
 или из ТЕЛА текущей заметки (MOC-файла), игнорируя frontmatter,
 и автоматически проставляет `up` property в дочерние заметки.

 Ключевые исправления и особенности:
 1. (ВАЖНОЕ ИСПРАВЛЕНИЕ) Скрипт теперь корректно обрабатывает только
    содержимое заметки, полностью игнорируя YAML frontmatter.
 2. Корневые узлы получают ссылку на сам MOC-файл.
 3. Если у корневого узла УЖЕ есть свойство `up`, скрипт его не трогает.
 4. Пропущенные корневые узлы выводятся списком в итоговом уведомлении.
================================================================================
*/

async function updateHierarchyFromMOC(tp) {
  const activeFile = app.workspace.getActiveFile();
  if (!activeFile) {
      new Notice("❌ Ошибка: Нет активного файла для обработки.", 5000);
      return;
  }

  // 1. Получаем исходный текст
  // Сначала проверяем выделение. Если его нет - получаем ТОЛЬКО ТЕЛО заметки.
  let mocContent = await tp.file.selection();
  
  // <<< КЛЮЧЕВОЕ ИСПРАВЛЕНИЕ: Логика получения тела заметки, а не всего файла
  if (!mocContent) {
    new Notice("ℹ️ Выделение не найдено. Обрабатывается тело текущей заметки.", 2000);
    const rawContent = await app.vault.read(activeFile);
    const fileCache = app.metadataCache.getFileCache(activeFile);
    
    // Проверяем, есть ли frontmatter, и если да, отрезаем его
    if (fileCache && fileCache.frontmatterPosition) {
        mocContent = rawContent.slice(fileCache.frontmatterPosition.end.offset);
    } else {
        mocContent = rawContent; // Если frontmatter нет, берем все содержимое
    }
  }

  const mocFileNameForLink = activeFile.basename;

  // 2. Парсим иерархию
  const lines = mocContent.split('\n').filter(line => line.trim() !== '');
  const parentStack = []; 
  const relationships = []; 

  for (const line of lines) {
    const indentMatch = line.match(/^(\s*)/);
    const linkMatch = line.match(/\[\[(.*?)(?:\|.*?)?\]\]/);

    if (!linkMatch) continue;

    const currentIndent = indentMatch[1].length;
    const currentName = linkMatch[1]; 

    while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) {
      parentStack.pop();
    }
    
    let parentLink;
    if (parentStack.length > 0) {
      parentLink = `[[${parentStack[parentStack.length - 1].name}]]`;
    } else {
      parentLink = `[[${mocFileNameForLink}]]`;
    }

    relationships.push({ child: currentName, parent: parentLink });
    parentStack.push({ indent: currentIndent, name: currentName });
  }

  // 3. Обновляем frontmatter в файлах
  if (relationships.length === 0) {
    new Notice("ℹ️ Не найдено ссылок для обработки в теле заметки/выделении.", 3000);
    return;
  }

  let updatedCount = 0;
  let skippedCount = 0;
  let notFoundCount = 0;
  const notFoundFiles = [];
  const skippedFiles = []; 

  new Notice(`⏳ Начинаю обновление... Найдено связей: ${relationships.length}`, 3000);

  for (const rel of relationships) {
    const childFile = tp.file.find_tfile(rel.child);

    if (!childFile) {
      console.warn(`Файл не найден для заметки: "${rel.child}"`);
      notFoundCount++;
      notFoundFiles.push(rel.child);
      continue;
    }
    
    if (childFile.basename === mocFileNameForLink) continue;

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      const isRootNodeRelationship = rel.parent === `[[${mocFileNameForLink}]]`;

      if (isRootNodeRelationship && fm && fm['up']) {
        skippedCount++;
        skippedFiles.push(rel.child);
        console.log(`Пропущен корневой узел "${rel.child}", т.к. 'up' уже установлен: ${fm['up']}`);
        return;
      }

      if (fm['up'] !== rel.parent) {
         fm['up'] = rel.parent;
         updatedCount++;
      }
    });
  }

  // 4. Выводим результат
  let summary = `✅ Обновление завершено.\n`;
  if (updatedCount > 0) {
    summary += `Обновлено 'up' в ${updatedCount} файлах.\n`;
  } else {
    summary += `Все 'up' атрибуты уже были актуальны.\n`;
  }
  
  if (skippedCount > 0) {
    summary += `\nПропущено ${skippedCount} корневых узлов (т.к. 'up' уже был установлен):\n- ${skippedFiles.join('\n- ')}`;
  }

  if (notFoundCount > 0) {
    summary += `\n\n❌ Не найдено ${notFoundCount} файлов: ${notFoundFiles.join(', ')}`;
    console.error("Не найдены следующие файлы:", notFoundFiles);
  }
  
  const noticeDuration = summary.length > 200 ? 20000 : 10000;
  new Notice(summary, noticeDuration);
}

// Запускаем основную функцию
try {
  await updateHierarchyFromMOC(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}

%>