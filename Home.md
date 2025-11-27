****# 📟 ROOT CONSOLE

> [!quote] 📅 日志
> ```dataview
> LIST FROM "日记"
> SORT file.name DESC
> LIMIT 3
> ```

---

> [!col]
> >> [!todo] 🚧 活跃项目 (Active Processes)
> > 
> > ```dataview
> > TABLE WITHOUT ID file.link AS "Project", status AS "Status", file.mtime AS "Last Mod"
> > FROM "项目"
> > WHERE file.name != this.file.name
> > SORT file.mtime DESC
> > LIMIT 5
> > ```

---

> [!example] 🧠 近期知识录入 (Latest in Kernel)
> 这里的逻辑是：读取「知识库」，展示最近 7 天内创建或修改的原子笔记。
> ```dataview
> TABLE WITHOUT ID file.link AS "Note", file.folder AS "Category", file.ctime AS "Created"
> FROM "知识库"
> SORT file.ctime DESC
> LIMIT 5
> ```

---

