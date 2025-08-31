<%*
/*
================================================================================
 Скрипт для Templater: Global MOC Sync
 Версия: 2.0 (Section-Based Processing)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт находит ВСЕ заметки, содержащие специальную секцию (например, "## MOC"),
 строит из этих секций единую глобальную карту иерархий и синхронизирует 'up'
 property во всех дочерних заметках.

 !!! НОВОЕ в v2.0 !!!
 1. (ТОЧЕЧНАЯ ОБРАБОТКА) Скрипт больше не использует теги. Вместо этого он
    ищет в файлах заголовок, содержащий слово "MOC" (например, '## MOC'
    или '### MOC.'). Обрабатывается ТОЛЬКО содержимое этой секции.
 2. (НАДЕЖНОСТЬ) Критерием для MOC-файла является само наличие такой секции,
    что исключает ошибки с синхронизацией тегов.
 3. (ОПТИМИЗАЦИЯ) Скрипт быстро проверяет файлы на наличие заголовка
    и выполняет полный парсинг только для релевантных секций.
================================================================================
*/

async function globalMocSyncV2(tp) {
  const MOC_HEADER_REGEX = /^(#+)\s+(MOC\b.*)/im;

  new Notice(`🚀 Запуск глобальной MOC-синхронизации v2.0...`, 2000);

  // --- Функция для извлечения содержимого MOC-секции из файла ---
  function getMocSectionContent(fileContent) {
    const match = fileContent.match(MOC_HEADER_REGEX);
    if (!match) {
      return null; // Секция не найдена
    }

    const headerLevel = match[1].length;
    const contentAfterHeader = fileContent.substring(match.index + match[0].length);

    // Ищем следующий заголовок того же или более высокого уровня
    const nextHeaderRegex = new RegExp(`^#{1,${headerLevel}}\\s+`, "m");
    const nextHeaderMatch = contentAfterHeader.match(nextHeaderRegex);

    if (nextHeaderMatch) {
      return contentAfterHeader.substring(0, nextHeaderMatch.index);
    } else {
      return contentAfterHeader; // Секция идет до конца файла
    }
  }

  // 1. Найти все файлы, содержащие MOC-секцию, и извлечь ее
  const allFiles = app.vault.getMarkdownFiles();
  const mocSources = [];
  
  new Notice(`🔍 Сканирую ${allFiles.length} файлов на наличие MOC-секций...`, 3000);

  for (const file of allFiles) {
    const fileContent = await app.vault.read(file);
    const mocContent = getMocSectionContent(fileContent);

    if (mocContent && mocContent.trim() !== '') {
      mocSources.push({
        fileName: file.basename,
        content: mocContent
      });
    }
  }

  if (mocSources.length === 0) {
    new Notice(`🟡 Не найдено ни одного файла с валидной MOC-секцией.`, 5000);
    return;
  }
  
  const mocFileNames = mocSources.map(s => s.fileName).join(', ');
  new Notice(`✅ Найдено ${mocSources.length} MOC-источников: ${mocFileNames}`, 4000);

  // 2. Собрать одну глобальную карту связей из всех MOC-секций
  const globalParentMap = new Map();

  for (const source of mocSources) {
    const mocFileName = source.fileName;
    
    // --- Начало блока парсинга (идентичен вашему скрипту v2.1, но работает с mocContent) ---
    const lines = source.content.split('\n').filter(line => line.trim() !== '');
    const parentStack = [];
    
    for (const line of lines) {
      const indentMatch = line.match(/^(\s*)/);
      const currentIndent = indentMatch[1].length;
      const linkMatches = [...line.matchAll(/\[\[(.*?)(?:\|.*?)?\]\]/g)];
      if (linkMatches.length === 0) continue;

      while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) {
        parentStack.pop();
      }
      
      let currentParentLink;
      if (parentStack.length > 0) {
        currentParentLink = `[[${parentStack[parentStack.length - 1].name}]]`;
      } else {
        currentParentLink = `[[${mocFileName}]]`;
      }
      
      let lastChildNameInLine = '';
      for (const match of linkMatches) {
        const childName = match[1];

        if (!globalParentMap.has(childName)) {
            globalParentMap.set(childName, []);
        }
        const childParents = globalParentMap.get(childName);
        if (!childParents.includes(currentParentLink)) {
            childParents.push(currentParentLink);
        }

        currentParentLink = `[[${childName}]]`;
        lastChildNameInLine = childName;
      }

      if (lastChildNameInLine) {
        parentStack.push({ indent: currentIndent, name: lastChildNameInLine });
      }
    }
    // --- Конец блока парсинга ---
  }

  // 3. Обновить frontmatter во всех дочерних файлах на основе глобальной карты
  if (globalParentMap.size === 0) {
    new Notice("ℹ️ Не найдено ни одной валидной ссылки в MOC-секциях.", 3000);
    return;
  }

  let updatedCount = 0;
  let notFoundCount = 0;
  const notFoundFiles = [];

  new Notice(`⏳ Обновляю ${globalParentMap.size} заметок...`, 3000);

  for (const [childName, newParents] of globalParentMap.entries()) {
    const childFile = tp.file.find_tfile(childName);

    if (!childFile) {
      console.warn(`Файл не найден для заметки: "${childName}"`);
      notFoundCount++;
      notFoundFiles.push(childName);
      continue;
    }

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      let currentUp = fm.up || [];
      if (typeof currentUp === 'string') {
        currentUp = [currentUp];
      }
      
      const uniqueNewParents = [...new Set(newParents)].sort();
      const sortedCurrentUp = [...new Set(currentUp)].sort();
      
      const needsUpdate = JSON.stringify(uniqueNewParents) !== JSON.stringify(sortedCurrentUp);

      if (needsUpdate) {
        fm.up = uniqueNewParents.length === 1 ? uniqueNewParents[0] : uniqueNewParents;
        updatedCount++;
      }
    });
  }

  // 4. Выводим итоговый результат
  let summary = `✅ Глобальная синхронизация завершена.\nОбработано MOC-источников: ${mocSources.length}.\n`;
  summary += (updatedCount > 0) ? `Обновлено 'up' в ${updatedCount} файлах.\n` : `Все 'up' атрибуты уже были актуальны.\n`;
  
  if (notFoundCount > 0) {
    summary += `\n❌ Не найдено ${notFoundCount} файлов: ${notFoundFiles.join(', ')}`;
    console.error("Не найдены следующие файлы:", notFoundFiles);
  }
  
  new Notice(summary, 20000);
}

// Запускаем основную функцию
try {
  await globalMocSyncV2(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}
%>