<%*
async function globalMocSyncV4_2(tp) {
  const MOC_HEADER_REGEX_G = /^(#+)\s+(.*\.\s*MOC\b[.\s]*|MOC\b[.\s]*)$/gim;
  new Notice(`🚀 Starting global MOC sync v4.2...`, 2000);

  const globalParentMap = new Map();
  const mocFileNames = new Set();
  const allFiles = app.vault.getMarkdownFiles();

  new Notice(`🔍 Scanning ${allFiles.length} files... (Pass 1/2)`, 3000);

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
                
                const childFile = app.metadataCache.getFirstLinkpathDest(childName, file.path);
                
                if (childFile) {
                    if (!globalParentMap.has(childFile.path)) globalParentMap.set(childFile.path, []);
                    const childParents = globalParentMap.get(childFile.path);
                    if (!childParents.includes(currentParentLink)) childParents.push(currentParentLink);
                } else {
                    console.warn(`Link to non-existent file "${childName}" ignored.`);
                }
                currentParentLink = `[[${childName}]]`;
                lastChildNameInLine = childName;
            }
            if (lastChildNameInLine) parentStack.push({ indent: currentIndent, name: lastChildNameInLine });
        }
    }
  }

  new Notice(`📝 Compiling update list... (Pass 2/2)`, 2000);
  const filesWithUp = new Set();
  for (const file of allFiles) {
    const cache = app.metadataCache.getFileCache(file);
    if (cache?.frontmatter?.up) {
      filesWithUp.add(file.path);
    }
  }
  
  const filesToProcess = new Set([...globalParentMap.keys(), ...filesWithUp]);
  if (mocFileNames.size === 0) { new Notice(`🟡 No MOC sections found.`, 5000); return; }
  new Notice(`✅ Found ${mocFileNames.size} MOC sources.`, 4000);
  if (filesToProcess.size === 0) { new Notice("ℹ️ No files found to update.", 3000); return; }

  let updatedCount = 0;
  new Notice(`⏳ Updating ${filesToProcess.size} notes...`, 3000);
  
  for (const childPath of filesToProcess) {
    const childFile = app.vault.getAbstractFileByPath(childPath);
    if (!childFile || !(childFile instanceof tp.obsidian.TFile)) continue;

    const newParents = globalParentMap.get(childPath) || [];
    const uniqueNewParents = [...new Set(newParents)].sort();

    const cache = app.metadataCache.getFileCache(childFile);
    let currentUp = cache?.frontmatter?.up || [];
    if (typeof currentUp === 'string') currentUp = [currentUp];
    const sortedCurrentUp = [...new Set(currentUp)].sort();

    if (JSON.stringify(uniqueNewParents) === JSON.stringify(sortedCurrentUp)) {
      continue;
    }

    await app.vault.process(childFile, (content) => {
      let newUpYaml = '';
      if (uniqueNewParents.length === 1) {
        newUpYaml = `up: ${uniqueNewParents[0]}`;
      } else if (uniqueNewParents.length > 1) {
        newUpYaml = `up:\n${uniqueNewParents.map(p => `  - ${p}`).join('\n')}`;
      }

      const upRegex = /^up:.*(?:(?:\r\n|\n) {2}-.*)*/m;
      const hasUp = upRegex.test(content);
      const hasFrontmatter = content.startsWith('---');

      if (!hasFrontmatter && newUpYaml) {
        return `---\n${newUpYaml}\n---\n\n${content}`;
      }
      
      let newContent = content;

      if (hasUp) {
        if (newUpYaml) {
          newContent = content.replace(upRegex, newUpYaml);
        } else {
          const upWithTrailingNewlineRegex = /^up:.*(?:(?:\r\n|\n) {2}-.*)*\s*(\r\n|\n)?/m;
          newContent = content.replace(upWithTrailingNewlineRegex, '');
        }
      } else if (newUpYaml) {
        newContent = content.replace(/^---\s*(\r\n|\n)/, `---\n${newUpYaml}\n`);
      }
      
      return newContent;
    });

    updatedCount++;
  }

  let summary = `✅ Global sync complete.\nProcessed MOC sources: ${mocFileNames.size}.\n`;
  summary += (updatedCount > 0) ? `Updated/cleared 'up' in ${updatedCount} files.\n` : `All 'up' attributes were already up to date.\n`;
  new Notice(summary, 15000);
}

try {
  await globalMocSyncV4_2(tp);
} catch (e) {
  new Notice("❌ A critical error occurred. See developer console (Ctrl+Shift+I).", 10000);
  console.error("Templater script error:", e);
}
%>