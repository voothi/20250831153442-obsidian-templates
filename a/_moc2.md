<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 1.3
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Этот скрипт читает иерархическую структуру из выделенного текста
 или текущей заметки (MOC-файла) и автоматически проставляет
 `up` property (ссылку на родителя) в frontmatter каждой дочерней заметки.

 Ключевые особенности:
 1. Корневые узлы (без родителя в иерархии) получают ссылку на сам MOC-файл.
 2. (НОВОЕ) Если у корневого узла УЖЕ есть свойство `up`, скрипт
    НЕ БУДЕТ его перезаписывать, уважая установленную вручную связь.
 3. Для всех остальных (дочерних) узлов свойство `up` будет обновляться
    в соответствии со структурой в MOC.
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
      parentLink = `[[${mocFileNameForLink}]]`; // Родитель по умолчанию - сам MOC
    }

    relationships.push({ child: currentName, parent: parentLink });
    parentStack.push({ indent: currentIndent, name: currentName });
  }

  // 3. Обновляем frontmatter в файлах
  if (relationships.length === 0) {
    new Notice("ℹ️ Не найдено ссылок для обработки в тексте.", 3000);
    return;
  }

  let updatedCount = 0;
  let skippedCount = 0;
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
    
    if (childFile.basename === mocFileNameForLink) continue;

    await app.fileManager.processFrontMatter(childFile, (fm) => {
      // Определяем, является ли обрабатываемый узел корневым в рамках этого MOC
      const isRootNodeRelationship = rel.parent === `[[${mocFileNameForLink}]]`;

      // <<< ИЗМЕНЕНИЕ: Основная логика проверки
      // Если это корневой узел И у него уже есть 'up' свойство, пропускаем его.
      if (isRootNodeRelationship && fm && fm['up']) {
        skippedCount++; // Считаем пропущенные
        console.log(`Пропущен корневой узел "${rel.child}", т.к. 'up' уже установлен: ${fm['up']}`);
        return; // Ничего не делаем, выходим из колбэка
      }

      // Для всех остальных случаев (дочерние узлы или корневые без 'up') обновляем
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
    summary += `Пропущено ${skippedCount} корневых узлов с уже установленным 'up'.\n`;
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