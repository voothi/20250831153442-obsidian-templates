---
type: project
status: active # active, on-hold, completed, archived
area: 
goal: 
started: <% tp.file.creation_date("YYYY-MM-DD") %>
deadline: 
tags: [project]
---
# Проект: <% tp.file.title %>

> **Цель:** <% locals.goal %>
> **Область:** <% locals.area %>
> **Статус:** <% locals.status %>

---

## 🚀 Задачи по проекту

```dataview
TABLE status as "Статус", due as "Срок"
FROM #task 
WHERE project = this.file.link
SORT status, due
```

## 📚 Ресурсы и материалы

```dataview
LIST
FROM #resource 
WHERE contains(project, this.file.link)
```

## 📝 Заметки и мысли

    -  