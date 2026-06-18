# CLAUDE.md — macOS Dev Issues 仓库操作指南

本文件供 Claude Code 在操作此仓库时参考。每次提交/新增 Issue 均需遵循以下规范。

---

## 仓库目的

记录 macOS 开发与运维中遇到的各类问题，每份文档包含：问题现象 → 排查过程 → 根因分析 → 解决方案。

仓库地址: `https://github.com/victcity/macos-dev-issues`

---

## 目录结构

```
macos-dev-issues/
├── CLAUDE.md                    # 本文件 (Claude Code 操作指南)
├── README.md                    # 中英双语总览 + 索引表格
├── LICENSE                      # CC0 1.0
├── issues/
│   └── YYYY-MM-DD-short-slug/
│       ├── README.zh.md         # 简体中文
│       ├── README.en.md         # English (与中文内容一致)
│       └── assets/              # 截图、日志等附件
└── .claude/                     # Claude Code 配置 (自动生成)
```

### 命名规则

- 目录名: `YYYY-MM-DD-` + 短横线连接的英文关键词，如 `2026-06-18-talkie-metal-gpu`
- `short-slug` 应简洁描述问题，不超过 5 个单词
- 日期取问题首次记录的日期

---

## 新增 Issue 工作流

### 步骤 1: 创建目录

```bash
mkdir -p issues/YYYY-MM-DD-short-slug/assets
touch issues/YYYY-MM-DD-short-slug/assets/.gitkeep
```

### 步骤 2: 撰写中文文档 (README.zh.md)

必须包含以下章节，按顺序：

1. **标题** — `# 问题简述`
2. **概述** — 2-3 句话说明问题和结论
3. **环境** — 设备型号 / OS 版本 / 软件版本 / 关键配置
4. **背景**（可选）— 相关架构、依赖关系等前置知识
5. **复现步骤** — 精确到命令行的步骤，确保他人可重现
6. **排查过程** — 按时间线记录尝试了什么、发现了什么
7. **根因分析** — 对根本原因的判断（已确认 / 推测）
8. **解决方案** — 明确的修复步骤或 workaround
9. **相关链接** — 上游仓库 / Issue / 文档

### 步骤 3: 撰写英文文档 (README.en.md)

- 内容与中文版一致，章节结构相同
- 确保技术术语翻译准确
- 命令、代码块、日志输出无需翻译
- 表格保持一致格式

### 步骤 4: 更新 README.md 索引表

在 `## 目录 / Index` 下的表格中新增一行：

```markdown
| YYYY-MM-DD | [中文标题](issues/slug/README.zh.md) | 领域标签 | [English](issues/slug/README.en.md) |
```

领域标签示例: `LLM / llama.cpp / Metal`、`Homebrew / CLI`、`Python / pip`、`Docker / macOS`

### 步骤 5: 撰寫提交信息

格式: `docs: <英文简述> (zh/en)`

示例:
```
docs: add Talkie 1930 Metal GPU garbage output issue (zh/en)
docs: add Homebrew permission fix issue (zh/en)
```

### 步骤 6: 推送

```bash
git add -A
git commit -m "docs: <message>"
git push origin main
```

推送前确认 `git status` 无意外文件。

---

## 文档撰写规范

### 章节完整性

每份 Issue 文档必须包含的章节：

| 章节 | README.zh.md | README.en.md | 必需 |
|------|-------------|-------------|------|
| 标题 | ✅ | ✅ | 是 |
| 概述 | ✅ | ✅ | 是 |
| 环境 | ✅ | ✅ | 是 |
| 复现步骤 | ✅ | ✅ | 是 |
| 排查过程 | ✅ | ✅ | 是 |
| 根因分析 | ✅ | ✅ | 是 |
| 解决方案 | ✅ | ✅ | 是 |
| 相关链接 | ✅ | ✅ | 否 |

### 语言要求

- **所有文档必须有中英两个版本**
- 中文版本使用简体中文，书面语风格
- 英文版本使用自然、流畅的技术英语
- 代码块、命令、日志输出、表格数据保持原样（不翻译）
- 两个版本内容必须对应，不可一个详细一个简略

### 格式要求

- 使用 Markdown 格式
- 代码块需标注语言: ````bash`、````python`、````json` 等
- 数据对比使用表格
- 路径、文件名使用反引号 `` ` `` 包裹
- 链接使用标准 Markdown 格式 `[text](url)`

---

## README.md 索引表维护

索引表按日期倒序排列（最新在上），格式：

```
| 日期 / Date | 问题 / Issue | 领域 / Domain | English |
|---|---|---|---|
| 2026-06-18 | [标题](issues/slug/README.zh.md) | tag1 / tag2 | [English](issues/slug/README.en.md) |
```

- 日期列使用中文日期链接到对应中文文档 README.zh.md
- 领域列用 `/` 分隔标签，标签越具体越好
- English 列链接到对应的英文 README

---

## 常见操作

### 新增 Issue（完整流程）

```bash
SLUG="YYYY-MM-DD-short-description"
mkdir -p issues/$SLUG/assets
touch issues/$SLUG/assets/.gitkeep

# 撰写 issues/$SLUG/README.zh.md
# 撰写 issues/$SLUG/README.en.md
# 更新 README.md 表格

git add -A
git commit -m "docs: add <issue title> (zh/en)"
git push origin main
```

### 更新已有 Issue

修改对应文件后：
```bash
git add issues/YYYY-MM-DD-slug/
git commit -m "docs: update <issue title>"
git push origin main
```

### 拉取最新

```bash
git pull origin main
```

---

## 注意事项

1. **不要修改 LICENSE 文件**（除非用户明确要求）
2. **不要删除已有的 Issue 目录**（除非用户明确要求）
3. **assets 目录不下大文件**：截图优先使用 PNG（<1MB），日志用 .log 或 .txt
4. **提交前检查**: 确认 `git status` 中没有 `CLAUDE.md` 的临时编辑或无关文件
5. **推送失败时**: 检查网络连接，重试 `git push`；若冲突则先 `git pull --rebase`
