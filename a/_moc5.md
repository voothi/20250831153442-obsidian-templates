<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 1.6
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Этот скрипт читает иерархическую структуру из выделенного текста
 или из ТЕЛА текущей заметки (MOC-файла), игнорируя frontmatter,
 и автоматически проставляет `up` property в дочерние заметки.

 Ключевые исправления и особенности:
 1. (ВАЖНОЕ ИСПРАВЛЕНИЕ) Логика для корневых узлов теперь абсолютно точна:
    - Обновляет 'up', только если свойство пустое или отсутствует.
    - НЕ трогает 'up', если оно уже было установлено вручную.
 2. Для дочерних узлов 'up' всегда синхронизируется со структурой в MOC.
 3. Игнорирует YAML frontmatter при чтении MOC.
 4. Выводит список пропущенных "защищенных" корневых узлов.
================================================================================
*/

async function updateHierarchyFromMOC(tp) {
  const activeFile = app.workspace.getActiveFile();
  if (!activeFile) {
      new Notice("❌ Ошибка: Нет активного файла для обработки.", 5000);
      return;
  }

  // 1. Получаем исходный текст (только тело заметки или выделение)
  let mocContent = await tp.file.selection();
  if (!mocContent) {
    new Notice("ℹ️ Выделение не найдено. Обрабатывается тело текущей заметки.", 2000);
    const rawContent = await app.vault.read(activeFile);
    const fileCache = app.metadataCache.getFileCache(activeFile);
    if (fileCache && fileCache.frontmatterPosition) {
        mocContent = rawContent.slice(fileCache.frontmatterPosition.end.offset);
    } else {
        mocContent = rawContent;
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
      const isRootNodeInMOC = rel.parent === `[[${mocFileNameForLink}]]`;

      // <<< КЛЮЧЕВОЙ БЛОК ЛОГИКИ v1.6 >>>
      // ПРАВИЛО ДЛЯ КОРНЕВЫХ УЗЛОВ:
      // Проверяем, является ли узел корневым в этом MOC И есть ли у него УЖЕ непустое 'up' свойство.
      if (isRootNodeInMOC && fm && fm['up']) {
          // Если оба условия истинны - это "защищенный" узел. Пропускаем его.
          skippedCount++;
          skippedFiles.push(rel.child);
          console.log(`Пропущен корневой узел "${rel.child}", т.к. 'up' уже установлен: ${fm['up']}`);
          return; // Выходим, ничего не меняя
      }

      // ПРАВИЛО ДЛЯ ВСЕХ ОСТАЛЬНЫХ СЛУЧАЕВ:
      // (Это либо дочерний узел, либо корневой узел с пустым/отсутствующим 'up')
      // Мы приводим свойство 'up' в соответствие со структурой в MOC.
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