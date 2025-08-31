<%*
/*
================================================================================
 Скрипт для Templater: Global MOC Sync
 Версия: 2.6 (Precision Header Rule)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт находит MOC-секции, строит из них карту иерархий и синхронизирует 'up'
 property во всех дочерних заметках.

 !!! НОВОЕ в v2.6 !!!
 1. (МАКСИМАЛЬНО ТОЧНОЕ ПРАВИЛО ДЛЯ ЗАГОЛОВКА) Реализовано строгое правило.
    Скрипт обрабатывает секцию, только если:
    - "MOC" является единственным словом в заголовке (например, "# MOC.").
    - Перед словом "MOC" стоит точка (например, "# Задачи. MOC").
    Все остальные варианты (например, "# Задачи MOC") будут проигнорированы.
================================================================================
*/

async function globalMocSyncV2(tp) {
  // >>>>>>>>>> ГЛАВНОЕ ИЗМЕНЕНИЕ ЗДЕСЬ <<<<<<<<<<
  const MOC_HEADER_REGEX = /^(#+)\s+(.*\.\s*MOC\b[.\s]*|MOC\b[.\s]*)$/im;

  new Notice(`🚀 Запуск глобальной MOC-синхронизации v2.6...`, 2000);

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
        const childFile = tp.file.find_tfile(childName);

        if (childFile) {
            if (!globalParentMap.has(childName)) {
                globalParentMap.set(childName, []);
            }
            const childParents = globalParentMap.get(childName);
            if (!childParents.includes(currentParentLink)) {
                childParents.push(currentParentLink);
            }
        } else {
            console.warn(`Ссылка на несуществующий файл "${childName}" проигнорирована.`);
        }
        
        currentParentLink = `[[${childName}]]`;
        lastChildNameInLine = childName;
      }

      if (lastChildNameInLine) {
        parentStack.push({ indent: currentIndent, name: lastChildNameInLine });
      }
    }
  }

  if (globalParentMap.size === 0) {
    new Notice("ℹ️ Не найдено существующих файлов для обновления в MOC-секциях.", 3000);
    return;
  }

  let updatedCount = 0;
  new Notice(`⏳ Обновляю ${globalParentMap.size} существующих заметок...`, 3000);

  for (const [childName, newParents] of globalParentMap.entries()) {
    const childFile = tp.file.find_tfile(childName);
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

  let summary = `✅ Глобальная синхронизация завершена.\nОбработано MOC-источников: ${mocSources.length}.\n`;
  summary += (updatedCount > 0) ? `Обновлено 'up' в ${updatedCount} файлах.\n` : `Все 'up' атрибуты уже были актуальны.\n`;
  
  new Notice(summary, 15000);
}

try {
  await globalMocSyncV2(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}
%>