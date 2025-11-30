> [!quote] 📅 日志
> ```dataview
> LIST FROM "日记"
> SORT file.name DESC
> LIMIT 3
> ```

待办

- [ ] 12.10 信安数基 14:00 2h
- [ ] 12.19 计组 14:00 2h
- [x] 12.21 C++ 实验报告
- [ ] 12.25 C++ 14:00 2h
- [ ] 12.26 密码学 14:00 2h
- [ ] 12.31 数据结构 14:00 2h
- [ ] 1.5 概率论 14:00 2h
- [ ] 1.13 大物 9:30 2h
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

