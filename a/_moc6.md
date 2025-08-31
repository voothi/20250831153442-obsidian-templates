<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 2.0 (Major Update)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт читает иерархическую структуру из MOC-файла и автоматически
 синхронизирует 'up' property в дочерних заметках.

 !!! НОВАЯ АРХИТЕКТУРА !!!
 1. (ПОДДЕРЖКА МНОЖЕСТВЕННЫХ РОДИТЕЛЕЙ) Если заметка встречается в MOC
    несколько раз под разными родителями, свойство 'up' будет содержать
    СПИСОК (YAML array) всех родительских ссылок.
    up:
      - "[[Родитель 1]]"
      - "[[Родитель 2]]"
 2. (ЕДИНЫЙ ИСТОЧНИК ПРАВДЫ) Скрипт теперь полностью переписывает 'up'
    свойство, чтобы оно на 100% соответствовало структуре в MOC.
    Правило "не трогать корневые узлы" упразднено в пользу этой логики.
 3. Игнорирует YAML frontmatter при чтении MOC.
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

  // 2. Агрегируем всех родителей для каждого дочернего узла
  // Используем Map, где ключ - имя дочерней заметки, значение - массив родительских ссылок.
  const parentMap = new Map();
  const lines = mocContent.split('\n').filter(line => line.trim() !== '');
  const parentStack = [];

  for (const line of lines) {
    const indentMatch = line.match(/^(\s*)/);
    const linkMatch = line.match(/\[\[(.*?)(?:\|.*?)?\]\]/);
    if (!linkMatch) continue;

    const currentIndent = indentMatch[1].length;
    const childName = linkMatch[1];

    while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) {
      parentStack.pop();
    }
    
    let parentLink;
    if (parentStack.length > 0) {
      parentLink = `[[${parentStack[parentStack.length - 1].name}]]`;
    } else {
      parentLink = `[[${mocFileNameForLink}]]`;
    }
    
    // Добавляем родителя в список для данного дочернего элемента
    if (!parentMap.has(childName)) {
      parentMap.set(childName, []);
    }
    parentMap.get(childName).push(parentLink);

    parentStack.push({ indent: currentIndent, name: childName });
  }

  // 3. Обновляем frontmatter в файлах
  if (parentMap.size === 0) {
    new Notice("ℹ️ Не найдено ссылок для обработки в теле заметки/выделении.", 3000);
    return;
  }

  let updatedCount = 0;
  let notFoundCount = 0;
  const notFoundFiles = [];

  new Notice(`⏳ Начинаю обновление... Найдено уникальных узлов: ${parentMap.size}`, 3000);

  // Итерируемся по собранной карте связей
  for (const [childName, newParents] of parentMap.entries()) {
    const childFile = tp.file.find_tfile(childName);

    if (!childFile) {
      console.warn(`Файл не найден для заметки: "${childName}"`);
      notFoundCount++;
      notFoundFiles.push(childName);
      continue;
    }
    
    if (childFile.basename === mocFileNameForLink) continue;

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      // Получаем текущее 'up' и приводим его к формату массива для сравнения
      let currentUp = fm.up || [];
      if (typeof currentUp === 'string') {
        currentUp = [currentUp];
      }
      
      // Убираем дубликаты и сортируем для корректного сравнения
      const uniqueNewParents = [...new Set(newParents)].sort();
      const sortedCurrentUp = [...new Set(currentUp)].sort();
      
      // Сравниваем массивы. Обновляем только если есть реальные изменения.
      const needsUpdate = JSON.stringify(uniqueNewParents) !== JSON.stringify(sortedCurrentUp);

      if (needsUpdate) {
        // Если родитель один - сохраняем как строку. Если несколько - как массив.
        if (uniqueNewParents.length === 1) {
          fm.up = uniqueNewParents[0];
        } else {
          fm.up = uniqueNewParents;
        }
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
  
  if (notFoundCount > 0) {
    summary += `\n❌ Не найдено ${notFoundCount} файлов: ${notFoundFiles.join(', ')}`;
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