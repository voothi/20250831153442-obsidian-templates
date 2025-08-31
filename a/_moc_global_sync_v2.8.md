<%*
/*
================================================================================
 Скрипт для Templater: Global MOC Sync
 Версия: 2.8 (Multi-Section Support)
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Скрипт находит MOC-секции, строит из них карту иерархий и синхронизирует 'up'
 property во всех дочерних заметках.

 !!! НОВОЕ в v2.8 !!!
 1. (ПОДДЕРЖКА НЕСКОЛЬКИХ СЕКЦИЙ) Ключевое изменение! Скрипт теперь
    корректно находит и обрабатывает ВСЕ валидные MOC-секции в одном
    файле, а не останавливается после первой.
 2. (ОПТИМИЗАЦИЯ ЛОГИКИ) Ядро скрипта было переработано для более
    надежного поиска и извлечения данных.
================================================================================
*/

async function globalMocSyncV2(tp) {
  // Регулярное выражение теперь имеет флаги 'g' (global) и 'm' (multiline)
  const MOC_HEADER_REGEX_G = /^(#+)\s+(.*\.\s*MOC\b[.\s]*|MOC\b[.\s]*)$/gim;

  new Notice(`🚀 Запуск глобальной MOC-синхронизации v2.8...`, 2000);

  const allFiles = app.vault.getMarkdownFiles();
  const mocFileNames = new Set(); // Используем Set для хранения имен файлов-источников
  const globalParentMap = new Map();

  new Notice(`🔍 Сканирую ${allFiles.length} файлов на наличие MOC-секций...`, 3000);

  for (const file of allFiles) {
    const fileContent = await app.vault.read(file);
    
    // 1. Сначала удаляем многострочные блоки кода из всего файла.
    const contentWithoutCodeBlocks = fileContent.replace(/```[\s\S]*?```/g, '');

    // 2. Находим ВСЕ валидные MOC-заголовки в очищенном контенте.
    const headerMatches = [...contentWithoutCodeBlocks.matchAll(MOC_HEADER_REGEX_G)];
    
    if (headerMatches.length === 0) {
      continue; // В этом файле нет MOC-секций, переходим к следующему.
    }

    // Если нашли хотя бы одну, добавляем имя файла в отчет
    mocFileNames.add(file.basename);

    // 3. Обрабатываем КАЖДУЮ найденную секцию
    for (let i = 0; i < headerMatches.length; i++) {
      const currentMatch = headerMatches[i];
      const headerLevel = currentMatch[1].length;

      // Определяем начало и конец текущей секции
      const sectionStartIndex = currentMatch.index + currentMatch[0].length;
      const nextMatch = headerMatches[i + 1];
      const sectionEndIndex = nextMatch ? nextMatch.index : contentWithoutCodeBlocks.length;
      
      let sectionContent = contentWithoutCodeBlocks.substring(sectionStartIndex, sectionEndIndex);

      // Дополнительно обрезаем секцию до следующего заголовка того же/более высокого уровня
      const nextGenericHeaderRegex = new RegExp(`^#{1,${headerLevel}}\\s+`, "m");
      const nextHeaderMatch = sectionContent.match(nextGenericHeaderRegex);
      if (nextHeaderMatch) {
          sectionContent = sectionContent.substring(0, nextHeaderMatch.index);
      }
      
      // Очищаем от однострочного кода и парсим
      sectionContent = sectionContent.replace(/`[^`]*`/g, '');
      if (sectionContent.trim() === '') continue;

      // --- Начало блока парсинга (применяется к каждой секции) ---
      const lines = sectionContent.split('\n').filter(line => line.trim() !== '');
      const parentStack = [];
      
      for (const line of lines) {
        const indentMatch = line.match(/^(\s*)/);
        const currentIndent = indentMatch[1].length;
        const linkMatches = [...line.matchAll(/\[\[(.*?)(?:\|.*?)?\]\]/g)];
        if (linkMatches.length === 0) continue;

        while (parentStack.length > 0 && parentStack[parentStack.length - 1].indent >= currentIndent) {
          parentStack.pop();
        }
        
        let currentParentLink = parentStack.length > 0 ? `[[${parentStack[parentStack.length - 1].name}]]` : `[[${file.basename}]]`;
        
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
      } // --- Конец блока парсинга ---
    }
  }

  // --- Обновление файлов (без изменений) ---
  if (mocFileNames.size === 0) {
    new Notice(`🟡 Не найдено ни одного файла с валидной MOC-секцией.`, 5000);
    return;
  }
  
  new Notice(`✅ Найдено ${mocFileNames.size} MOC-источников: ${[...mocFileNames].join(', ')}`, 4000);

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

  let summary = `✅ Глобальная синхронизация завершена.\nОбработано MOC-источников: ${mocFileNames.size}.\n`;
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