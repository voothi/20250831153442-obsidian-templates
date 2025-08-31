<%*
/*
================================================================================
 Скрипт для Templater: Global MOC Sync
 Версия: 4.0 (The Path-Centric Engine)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт находит MOC-секции, строит из них карту иерархий и синхронизирует 'up'
 property во всех дочерних заметках.

 !!! НОВОЕ в v4.0 !!!
 1. (СМЕНА АРХИТЕКТУРЫ) Критическое изменение! Скрипт больше не полагается на
    имена файлов. Вся логика перестроена на использование УНИКАЛЬНЫХ ПУТЕЙ
    к файлам (file.path). Это полностью решает проблемы с неоднозначностью,
    повышает надежность и гарантирует идемпотентность.
 2. (ЗОЛОТОЙ СТАНДАРТ) Для разрешения ссылок теперь используется функция
    `app.metadataCache.getFirstLinkpathDest`, которая работает идентично
    механизму самого Obsidian.
================================================================================
*/

async function globalMocSyncV4(tp) {
  const MOC_HEADER_REGEX_G = /^(#+)\s+(.*\.\s*MOC\b[.\s]*|MOC\b[.\s]*)$/gim;
  new Notice(`🚀 Запуск глобальной MOC-синхронизации v4.0...`, 2000);

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

    for (const match of headerMatches) {
        const headerLevel = match[1].length;
        const sectionStartIndex = match.index + match[0].length;
        const contentAfterSectionStart = contentWithoutCodeBlocks.substring(sectionStartIndex);
        const nextHeaderRegex = new RegExp(`\n^#{1,${headerLevel}}\\s+`, "m");
        const nextHeaderMatch = contentAfterSectionStart.match(nextHeaderRegex);
        const sectionEndIndex = nextHeaderMatch ? nextHeaderMatch.index : contentAfterSectionStart.length;
        let sectionContent = contentAfterSectionStart.substring(0, sectionEndIndex);
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
            for (const linkMatch of linkMatches) {
                const childName = linkMatch[1];
                if (childName === file.basename) continue;
                
                // >>>>>>>>>> НОВАЯ НАДЕЖНАЯ ЛОГИКА <<<<<<<<<<
                const childFile = app.metadataCache.getFirstLinkpathDest(childName, file.path);
                
                if (childFile) {
                    // Используем УНИКАЛЬНЫЙ ПУТЬ как ключ
                    if (!globalParentMap.has(childFile.path)) globalParentMap.set(childFile.path, []);
                    const childParents = globalParentMap.get(childFile.path);
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

  new Notice(`📝 Составляю список для обновления... (Проход 2/2)`, 2000);
  const filesWithUp = new Set();
  for (const file of allFiles) {
    const cache = app.metadataCache.getFileCache(file);
    if (cache?.frontmatter?.up) {
      // Сохраняем УНИКАЛЬНЫЙ ПУТЬ
      filesWithUp.add(file.path);
    }
  }
  
  const filesToProcess = new Set([...globalParentMap.keys(), ...filesWithUp]);
  if (mocFileNames.size === 0) { new Notice(`🟡 Не найдено MOC-секций.`, 5000); return; }
  new Notice(`✅ Найдено ${mocFileNames.size} MOC-источников: ${[...mocFileNames].join(', ')}`, 4000);
  if (filesToProcess.size === 0) { new Notice("ℹ️ Не найдено файлов для обновления.", 3000); return; }

  let updatedCount = 0;
  new Notice(`⏳ Обновляю ${filesToProcess.size} заметок...`, 3000);
  
  for (const childPath of filesToProcess) {
    const childFile = app.vault.getAbstractFileByPath(childPath);
    if (!childFile || !(childFile instanceof tp.obsidian.TFile)) continue;

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      const newParents = globalParentMap.get(childPath) || [];
      const uniqueNewParents = [...new Set(newParents)].sort();
      let currentUp = fm.up || [];
      if (typeof currentUp === 'string') currentUp = [currentUp];
      const sortedCurrentUp = [...new Set(currentUp)].sort();
      if (JSON.stringify(uniqueNewParents) !== JSON.stringify(sortedCurrentUp)) {
        fm.up = uniqueNewParents.length === 0 ? [] : (uniqueNewParents.length === 1 ? uniqueNewParents[0] : uniqueNewParents);
        updatedCount++;
      }
    });
  }

  let summary = `✅ Глобальная синхронизация завершена.\nОбработано MOC-источников: ${mocFileNames.size}.\n`;
  summary += (updatedCount > 0) ? `Обновлено/очищено 'up' в ${updatedCount} файлах.\n` : `Все 'up' атрибуты уже были актуальны.\n`;
  new Notice(summary, 15000);
}

try {
  await globalMocSyncV4(tp);
} catch (e) {
  new Notice("❌ Произошла критическая ошибка. См. консоль разработчика (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}
%>