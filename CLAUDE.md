# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库用途

这是一个 C++ 学习仓库，用于阅读经典书籍/资料、记录笔记、编写练习代码。

## 目录结构

- `references/` — 学习资料（PDF、书籍等）
- `notes/` — 读书笔记，按书籍/资料分子目录（如 `notes/effective_modern_cpp/`）
- `code/` — 练习代码、demo 示例，未来也可能包含经典开源项目（如 leveldb）的本地学习副本

## 笔记规范

- 笔记文件命名格式：`item_XX_简短英文描述.md`（如 `item_01_understand_template_type_deduction.md`）
- 笔记使用中文撰写，代码和术语保留英文
- 笔记结构包含：核心思想、分场景说明（附代码示例）、Things to Remember 总结
- 读取 PDF 时使用 `mcp__pdf-reader__read_pdf` 工具，先从目录页确定内容所在的 PDF 页码

## PDF 页码映射

| 书籍 | PDF 文件 | 目录页(PDF) | 正文起始页(PDF) | 备注 |
|------|----------|-------------|-----------------|------|
| Effective C++ (3rd Edition) | `references/Effective_C++_Scott_Meyers.pdf` | 12 | 22 (= 书本第1页 Introduction) | 共 321 页，前 21 页为推荐语、版权、致谢等 |

## 构建与运行

代码使用 C++17 标准。单文件编译示例：

```bash
g++ -std=c++17 -Wall -Wextra -o output code/some_demo.cpp && ./output
```

如果后续引入 CMake 项目，请遵循项目自带的构建方式。
