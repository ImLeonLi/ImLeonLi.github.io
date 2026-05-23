---
layout: post
title: "RST 和 Sphinx 一文通"
date: 2025-01-15 00:00:00 +0800
tags: [RST, Sphinx, Python, 文档]
category: 技术分享
---

# RST 和 Sphinx 一文通

## RST 和 Sphinx 是什么

### .rst文件是什么

Python 社区的相关帮助文件是用 rst 文档编写的。很多人可能都听过 markdown 文档，但是大部分人可能都没听过 rst 文档。RST（reStructuredText）是一种使用简单标记语法编写文档的文本文件格式。它被广泛应用于编写技术文档、软件说明文件、报告等。RST 文件使用一些特定的符号和结构来表示文档的各个部分，比如标题、列表、链接、引用等。RST文件通常以.rst为扩展名。RST 最早由 David Goodger于2002年创造并开发。它最初是为了 Sphinx 文档生成工具而设计的，Sphinx 是一个用于创建文档的工具，常用于 Python 开发者社区，实际上 RST文档比 Markdown 出现的更早。

### Sphinx 是什么

Sphinx 是 Python 文档生成器，它基于 reStructuredText 标记语言，可自动根据项目生成 HTML，PDF 等格式的文档。新版的 Python3 文档就是由 sphinx 生成的，并且它已成为 Python 项目首选的文档工具，同时它对 C/C++ 项目也有很好的支持。

## RST 基本语法

### 标题语法

```rst
==============
一级标题
==============

二级标题
==============

三级标题
--------------

四级标题
^^^^^^^^^^^^^^

五级标题
""""""""""""""

六级标题
**************
```

## 参考资料

- [野火 sphinx文档规范与模版](https://ebf-contribute-guide.readthedocs.io/zh-cn/latest/)
- [一文学会Rst文档撰写（使用VS Code）](https://blog.csdn.net/Whimsically_/article/details/140988075)
