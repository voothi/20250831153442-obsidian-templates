<%*
/*
================================================================================
 Скрипт для Templater: Global MOC Sync
 Версия: 2.3 (Process Existing Files Only)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт находит MOC-секции, строит из них карту иерархий и синхронизирует 'up'
 property во всех дочерних заметках.

 !!! НОВОЕ в v2.3 !!!
 1. (ОБРАБОТКА ТОЛЬКО СУЩЕСТВУЮЩИХ ФАЙЛОВ) Скрипт теперь игнорирует ссылки
    на еще не созданные файлы (плейсхолдеры). Он больше не будет выдавать
    ошибку "Не найдено N файлов". Поле 'up' будет добавлено в заметку
    автоматически при следующем запуске скрипта ПОСЛЕ ее создания.
 2. Сохранены все предыдущие улучшения (исключение кода, гибкий поиск заголовка).
================================================================================
*/

async function globalMocSyncV2(tp) {
  const MOC_HEADER_REGEX = /^(#+)\s+(.*\bMOC\b.*)/im;

  new Notice(`🚀 Запуск глобальной MOC-синхронизации v2.3...`, 2000);

  function getMocSectionContent(fileContent) {
    const match = fileContent.match(MOC_HEADER_REGEX);
    if (!match) return null;
    const headerLevel = match[1].length;
    const contentAfterHeader = fileContent.substring(match.index + match[0].length);
    const nextHeaderRegex = new RegExp(`^#{1,${headerLevel}}\\s+`, "m");
    const nextHeaderMatch = contentAfterHeader.match(nextHeaderRegex);
    return nextHeaderMatch ? contentAfterHeader.substring(0, nextHeaderMatch.index) : contentAfterHeader;
  }

  const allFiles = app.vault.getMarkdownFiles();
  const mocSources = [];
  
  new Notice(`🔍 Сканирую ${allFiles.length} файлов на наличие MOC-секций...`, 3000);

  for (const file of allFiles) {
    const fileContent = await app.vault.read(file);
    let mocContent = getMocSectionContent(fileContent);
    if (mocContent && mocContent.trim() !== '') {
      // Санитизация контента: удаляем блоки кода
      mocContent = mocContent.replace(/```[\s\S]*?```/g, '').replace(/`[^`]*`/g, '');
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

  // 2. Собрать глобальную карту связей ТОЛЬКО для существующих файлов
  const globalParentMap = new Map();

  for (const source of mocSources) {
    const mocFileName = source.fileName;
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
      
      let currentParentLink = parentStack.length > 0 ? `[[${parentStack[parentStack.length - 1].name}]]` : `[[${mocFileName}]]`;
      
      let lastChildNameInLine = '';
      for (const match of linkMatches) {
        const childName = match[1];

        // >>>>>>>>>> ГЛАВНОЕ ИЗМЕНЕНИЕ ЗДЕСЬ <<<<<<<<<<
        // Проверяем, существует ли файл, ПРЕЖДЕ чем добавить его в карту
        const childFile = tp.file.find_tfile(childName);
        if (!childFile) {
            // Если файл не найден, просто переходим к следующей ссылке
            console.warn(`Ссылка на несуществующий файл "${childName}" проигнорирована.`);
            currentParentLink = `[[${childName}]]`; // Но все равно обновляем родителя для горизонтальной цепочки
            lastChildNameInLine = childName;
            continue;
        }
        // >>>>>>>>>> КОНЕЦ ГЛАВНОГО ИЗМЕНЕНИЯ <<<<<<<<<<

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
  }

  // 3. Обновить frontmatter во всех дочерних файлах (теперь все они гарантированно существуют)
  if (globalParentMap.size === 0) {
    new Notice("ℹ️ Не найдено существующих файлов для обновления в MOC-секциях.", 3000);
    return;
  }

  let updatedCount = 0;
  new Notice(`⏳ Обновляю ${globalParentMap.size} существующих заметок...`, 3000);

  for (const [childName, newParents] of globalParentMap.entries()) {
    const childFile = tp.file.find_tfile(childName);
    // Эта проверка теперь избыточна, но оставим для надежности
    if (!childFile) continue;

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      let currentUp = fm.up || [];
      if (typeof currentUp === 'string') currentUp = [currentUp];
      
      const uniqueNewParents = [...new Set(newParents)].sort();
      const sortedCurrentUp = [...new Set(currentUp)].sort();
      
      if (JSON.stringify(uniqueNewParents) !== JSON.stringify(sortedCurrentUp)) {
        fm.up = uniqueNewParents.length === 1 ? uniqueNewParents[0] : uniqueNewParents;
        updatedCount++;
      }
    });
  }

  // 4. Выводим итоговый результат без блока ошибок "не найдено"
  let summary = `✅ Глобальная синхронизация завершена.\nОбработано MOC-источников: ${mocSources.length}.\n`;
  summary += (updatedCount > 0) ? `Обновлено 'up' в ${updatedCount} файлах.\n` : `Все 'up' атрибуты уже были актуальны.\n`;
  
  new Notice(summary, 15000);
}

// Запускаем основную функцию
try {
  await globalMocSyncV2(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}
%>