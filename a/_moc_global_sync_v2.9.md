<%*
/*
================================================================================
 Скрипт для Templater: Global MOC Sync
 Версия: 2.9 (Orphan Cleanup)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт находит MOC-секции, строит из них карту иерархий и синхронизирует 'up'
 property во всех дочерних заметках.

 !!! НОВОЕ в v2.9 !!!
 1. (ОЧИСТКА "СИРОТ") Ключевое исправление! Скрипт теперь корректно
    обрабатывает удаление ссылок. Если из всех MOC-секций удалить
    последнюю ссылку на файл, скрипт найдет этот файл-"сироту" и
    принудительно очистит его свойство 'up'.
================================================================================
*/

async function globalMocSyncV2(tp) {
  const MOC_HEADER_REGEX_G = /^(#+)\s+(.*\.\s*MOC\b[.\s]*|MOC\b[.\s]*)$/gim;
  new Notice(`🚀 Запуск глобальной MOC-синхронизации v2.9...`, 2000);

  // --- ПРОХОД 1: Построение актуальной карты связей ---
  const globalParentMap = new Map();
  const mocFileNames = new Set();
  const allFiles = app.vault.getMarkdownFiles();

  new Notice(`🔍 Сканирую ${allFiles.length} файлов... (Проход 1/2)`, 3000);

  for (const file of allFiles) {
    const fileContent = await app.vault.read(file);
    const contentWithoutCodeBlocks = fileContent.replace(/```[\s\S]*?```/g, '');
    const headerMatches = [...contentWithoutCodeBlocks.matchAll(MOC_HEADER_REGEX_G)];
    if (headerMatches.length === 0) continue;
    mocFileNames.add(file.basename);

    for (let i = 0; i < headerMatches.length; i++) {
      const currentMatch = headerMatches[i];
      const headerLevel = currentMatch[1].length;
      const sectionStartIndex = currentMatch.index + currentMatch[0].length;
      const nextMatch = headerMatches[i + 1];
      const sectionEndIndex = nextMatch ? nextMatch.index : contentWithoutCodeBlocks.length;
      let sectionContent = contentWithoutCodeBlocks.substring(sectionStartIndex, sectionEndIndex);
      const nextGenericHeaderRegex = new RegExp(`^#{1,${headerLevel}}\\s+`, "m");
      const nextHeaderMatch = sectionContent.match(nextGenericHeaderRegex);
      if (nextHeaderMatch) sectionContent = sectionContent.substring(0, nextHeaderMatch.index);
      sectionContent = sectionContent.replace(/`[^`]*`/g, '');
      if (sectionContent.trim() === '') continue;

      const lines = sectionContent.split('\n').filter(line => line.trim() !== '');
      const parentStack = [];
      for (const line of lines) {
        const indentMatch = line.match(/^(\s*)/);
        const currentIndent = indentMatch[1].length;
        const linkMatches = [...line.matchAll(/\[\[(.*?)(?:\|.*?)?\]\]/g)];
        if (linkMatches.length === 0) continue;
        while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) parentStack.pop();
        let currentParentLink = parentStack.length > 0 ? `[[${parentStack[parentStack.length - 1].name}]]` : `[[${file.basename}]]`;
        let lastChildNameInLine = '';
        for (const match of linkMatches) {
          const childName = match[1];
          const childFile = tp.file.find_tfile(childName);
          if (childFile) {
            if (!globalParentMap.has(childName)) globalParentMap.set(childName, []);
            const childParents = globalParentMap.get(childName);
            if (!childParents.includes(currentParentLink)) childParents.push(currentParentLink);
          } else {
            console.warn(`Ссылка на несуществующий файл "${childName}" проигнорирована.`);
          }
          currentParentLink = `[[${childName}]]`;
          lastChildNameInLine = childName;
        }
        if (lastChildNameInLine) parentStack.push({ indent: currentIndent, name: lastChildNameInLine });
      }
    }
  }

  // --- ПРОХОД 2: Идентификация и обновление всех затрагиваемых файлов ---
  new Notice(`📝 Составляю список для обновления... (Проход 2/2)`, 2000);

  const filesWithUp = new Set();
  for (const file of allFiles) {
    const cache = app.metadataCache.getFileCache(file);
    if (cache?.frontmatter?.up) {
      filesWithUp.add(file.basename);
    }
  }

  // Объединяем файлы, которые должны иметь 'up', и те, которые УЖЕ имеют 'up'
  const filesToProcess = new Set([...globalParentMap.keys(), ...filesWithUp]);

  if (mocFileNames.size === 0) {
    new Notice(`🟡 Не найдено ни одного файла с валидной MOC-секцией.`, 5000);
    return;
  }
  new Notice(`✅ Найдено ${mocFileNames.size} MOC-источников: ${[...mocFileNames].join(', ')}`, 4000);

  if (filesToProcess.size === 0) {
    new Notice("ℹ️ Не найдено файлов для обновления.", 3000);
    return;
  }

  let updatedCount = 0;
  new Notice(`⏳ Обновляю ${filesToProcess.size} заметок...`, 3000);

  for (const childName of filesToProcess) {
    const childFile = tp.file.find_tfile(childName);
    if (!childFile) continue;

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      // Получаем НОВЫХ родителей из карты. Если файла в карте нет - значит, он "сирота".
      const newParents = globalParentMap.get(childName) || [];
      const uniqueNewParents = [...new Set(newParents)].sort();
      
      // Получаем ТЕКУЩИХ родителей из файла
      let currentUp = fm.up || [];
      if (typeof currentUp === 'string') currentUp = [currentUp];
      const sortedCurrentUp = [...new Set(currentUp)].sort();
      
      // Сравниваем и обновляем ТОЛЬКО если есть разница
      if (JSON.stringify(uniqueNewParents) !== JSON.stringify(sortedCurrentUp)) {
        if (uniqueNewParents.length === 0) {
          delete fm.up; // Явно удаляем свойство, если родителей больше нет
        } else {
          fm.up = uniqueNewParents.length === 1 ? uniqueNewParents[0] : uniqueNewParents;
        }
        updatedCount++;
      }
    });
  }

  let summary = `✅ Глобальная синхронизация завершена.\nОбработано MOC-источников: ${mocFileNames.size}.\n`;
  summary += (updatedCount > 0) ? `Обновлено/очищено 'up' в ${updatedCount} файлах.\n` : `Все 'up' атрибуты уже были актуальны.\n`;
  new Notice(summary, 15000);
}

try {
  await globalMocSyncV2(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}
%>