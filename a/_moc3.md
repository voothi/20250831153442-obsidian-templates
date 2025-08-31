<%*
/*
================================================================================
 Скрипт для Templater: Update Hierarchy from MOC
 Версия: 1.4
 Автор: Gemini AI & User Collaboration
--------------------------------------------------------------------------------
 Назначение:
 Этот скрипт читает иерархическую структуру из выделенного текста
 или текущей заметки (MOC-файла) и автоматически проставляет
 `up` property в frontmatter каждой дочерней заметки.

 Ключевые особенности:
 1. Корневые узлы получают ссылку на сам MOC-файл.
 2. Если у корневого узла УЖЕ есть свойство `up`, скрипт НЕ БУДЕТ его
    перезаписывать, уважая установленную вручную связь.
 3. (НОВОЕ) Пропущенные корневые узлы выводятся списком в итоговом
    уведомлении для полной прозрачности.
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
      parentLink = `[[${mocFileNameForLink}]]`;
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
  const skippedFiles = []; // <<< ИЗМЕНЕНИЕ: Массив для имен пропущенных файлов

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
        skippedFiles.push(rel.child); // <<< ИЗМЕНЕНИЕ: Добавляем имя файла в список
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
  
  // <<< ИЗМЕНЕНИЕ: Формируем красивый список пропущенных файлов
  if (skippedCount > 0) {
    summary += `\nПропущено ${skippedCount} корневых узлов (т.к. 'up' уже был установлен):\n- ${skippedFiles.join('\n- ')}`;
  }

  if (notFoundCount > 0) {
    summary += `\n\n❌ Не найдено ${notFoundCount} файлов: ${notFoundFiles.join(', ')}`;
    console.error("Не найдены следующие файлы:", notFoundFiles);
  }
  
  // Увеличиваем время отображения, если много текста
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