---
title: "代码审计-PHP"
subtitle: "代码审计-PHP"
summary: "学习笔记"
description: "学习笔记"
image: ""
date: 2025-10-22
lastmod: 2025-10-22
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["CTF"]
tags: ["CTF","网络安全"]
---

参考：[代码审计-PHP篇](https://www.bilibili.com/video/BV12om2YTEgW?spm_id_from=333.788.player.switch&vd_source=f329e64c57c95b7adedc05b814353e29&p=113)

## 原生开发

- 关键词挖掘
- 功能挖掘

### SQL 注入

- 正则搜索 `(update|select|insert|delete|).*?where.*=`
  - 需要搭配经验筛选，模板比较不容易出漏洞
  - 看文件路径，后台漏洞不容易利用
  - 找**可执行变量**、**过滤**，跟踪函数声明、用例
- MySQL Monitor Client 跟踪项目

### 文件安全

- 发现
  - 脚本文件名 `upload del delfile down downfile read readfile`
  - 应用功能点 `下载、上传、读取……` --> 抓包找路径
  - 操作关键字 `$_FILES、move_uploaded_file、unlink`

- 遇见代码加密
  - 需要查找对应解密方法

### 模型开发

- MVC ：Model、Controller、View
- MVC 对审计的影响
  - 文件代码定位：功能被封装，不好搜索
  - 代码过滤分析
  - 前端安全发现

- 版本对比的漏洞发现
  - 新版本修复，反推旧版本漏洞；根据旧版本漏洞看是否修复
  - Beyond、UltraCompare 项目

### 动态调试

- phpstorm + phpstudy + xdebug
- 鉴权漏洞
  - 未引用的鉴权逻辑
    - 错误逻辑：先功能操作后鉴权
  - 脆弱的、不严谨的
    - 没有引用鉴权模块的，可以直接进入