# CODEBUDDY.md
This file provides guidance to CodeBuddy when working with code in this repository.

---

## 1. 常用命令

> 本项目**无构建系统、无依赖、无测试框架**——纯静态资源 + 浏览器原生 API。

| 任务 | 命令 / 操作 |
|------|-------------|
| 预览生成的 HTML | 双击打开 `去做自己的山.project/*.html`，或 `start xxx.html` |
| 本地起一个静态服务器 | `python -m http.server 8000`（在仓库根目录运行），浏览器访问 `http://localhost:8000/去做自己的山.project/` |
| 检查 git 状态 | `git status` |
| 提交并推送 | `git add . && git commit -m "描述" && git push` |
| 推送（首次需代理） | `git config --global http.proxy http://127.0.0.1:7890 && git push -u origin master` |
| 撤销未暂存的修改 | `git restore .` |
| 重新生成单个文件 | 直接在 IDE 中重新调用 Agent 即可，**不要写脚本**——这是 AI 协作生成的内容 |

---

## 2. 仓库结构与架构

### 2.1 顶层布局

```
d:\code_buddy\
├── .git/                          # 本地仓库（已配置代理 127.0.0.1:7890）
├── .gitignore                     # 忽略 .codebuddy/、IDE 配置等
├── README.md                      # 用户面向的项目说明
├── CODEBUDDY.md                   # 本文件，AI Agent 协作指南
└── 去做自己的山.project/          # 第 1 章子项目（按章节独立文件夹）
    ├── Prompt.md                  # 本章节使用的 Prompt（约束 AI 输出）
    ├── 去做自己的山-C01-...-P01.jpg   # 原始照片
    ├── 去做自己的山-C01-...-P02.jpg   # 原始照片
    ├── 去做自己的山-C01-...-S01-...-P01.html   # 主输出（带交互）
    └── 去做自己的山-C01-...-S01-...-P01.md     # 附加输出（纯文本表格）
```

> 命名约定：`<书名>-C<章号>-<章名>-S<节号>-<节名>-P<页号>.<ext>`。

### 2.2 核心架构：AI 协作生成流水线

本仓库**不是传统代码项目**，而是一个**"Prompt + AI 生成产物"**的工作流：

```
用户上传照片
  ↓
[OCR + 翻译 + 词汇提取]  ← 由 Agent 按 Prompt.md 规则完成
  ↓
├─→ *.html （带朗读、选中联动高亮等交互）
└─→ *.md   （纯文本对照表）
```

**关键设计**：
- **Prompt.md 是真正的"源代码"**：所有生成规则（翻译方向、表格结构、词汇标注规范、交互逻辑）都在其中。
- **HTML / MD 是"构建产物"**：它们不直接维护，每次需要修改时应当**先改 Prompt，再重新生成**，避免产物与 Prompt 漂移。
- **同名文件成对出现**：每个 `*.html` 必有同名 `*.md`，扩展名之外完全一致。

### 2.3 HTML 文件的内部结构

每个 `*.html` 自包含所有 CSS/JS，无外部依赖，由三部分构成：

#### ① CSS（`<style>` 内联）
- `.col1 / .col2 / .col3` 三列布局
- `.section-title` 段落分隔标题
- `.vocab-item` 单词卡片（带 `.vocab-word`、`.pos`、`.phonetic`、`.sense-cn`）
- `.phrase-block` 常用短语块
- `mark.hl` 译文中可被高亮的短语；`mark.hl.active` 触发高亮时的样式
- `.translation` 可点击朗读的整句译文

#### ② HTML 表格主体
每行（除段落标题行）以 `data-pair-row` 标记，结构如下：
```html
<tr data-pair-row>
  <td class="col1">
    <div class="source-cell">
      普通文本 <mark class="hl" data-pair="原文短语">原文短语</mark> ...
    </div>
  </td>
  <td class="col2">
    <div class="translation">
      ... <mark class="hl" data-pair="原文短语">译文短语</mark> ...
    </div>
  </td>
  <td class="col3"> ...词汇卡片... </td>
</tr>
```

**关键约定**：`data-pair` 属性值在**同一行**的第一列与第二列中**必须相同**——这是选中联动高亮的匹配键。

#### ③ JavaScript（`<script>` 末尾）
两个独立功能模块，**互不耦合**：

- **朗读模块**：`speak(text, lang)` + 监听 `.translation` 和 `.vocab-word` 的 click
- **选中联动高亮模块**：监听 `selectionchange` → 800ms 防抖 → 校验选区（1~50 字符、语义完整）→ 查找同行的 `mark.hl[data-pair=...]` → 添加 `.active` 类

### 2.4 词汇卡片（第三列）的元数据规范

每个 `<div class="vocab-item">` 必须按以下顺序包含：

1. **单词 / 短语** —— `<span class="vocab-word">`，点击只朗读其本身（只取第一个文本节点）
2. **词性**（仅单词）—— `<span class="pos">` 如 `adj. / v. / n.`
3. **音标**（仅单词）—— `<span class="phonetic">` 包裹 `/.../`
4. **句中释义**（仅单词）—— `<span class="sense-cn">`，红色虚线框
5. **句中成分** —— `<div class="meta-line">` 自然语言描述
6. **常用短语**（仅单词）—— `<div class="phrase-block">` 包含短语 + 例句
7. **短语释义**（仅短语）—— 用 `<div class="meta-line">` 直接写"短语释义：……"

### 2.5 上下文衔接规则

由于书页连续，**P01 最后一句** + **P02 第一句** 在原文中属同一完整句时，必须**合并到一行**，并在第一列前缀标注 `【衔接句】`，在表格上方加引用块说明。

---

## 3. 修改工作流（Agent 必读）

当用户要求修改时，按以下顺序决策：

1. **是否需要先改 Prompt？** 若涉及"输出格式 / 翻译规则 / 词汇标注规范 / 交互逻辑"的变更 → **先改对应章节的 `Prompt.md`**，再用新 Prompt 重新生成 HTML/MD。
2. **纯样式 / 交互微调？** 可直接修改 `*.html` 的 `<style>` 或 `<script>`，不需要动 Prompt。
3. **数据更新（新的书页）？** 保持 Prompt 不变，按相同命名约定新增文件。
4. **修改完成后**：若用户要求同步 GitHub，按 `.gitignore` 规则暂存后 `git add . && git commit -m "..." && git push`（推送需代理 `http://127.0.0.1:7890`）。

---

## 4. 重要约束（来自 Prompt.md 与 README.md）

- 翻译方向：中文 → 英文 / 英文 → 中文，**自动判断**，除非用户指定。
- 表格行长度：每行 = 一个完整句子（衔接句除外）。
- 朗读：第二列点击朗读整句；第三列点击**只朗读单词/短语本身**（不读音标、词性、例句）。
- 选中联动：仅在 800ms 防抖后触发；选区超 50 字符 / 跨段 / 空选区不处理。
- **网络**：GitHub 推送需走代理 `http://127.0.0.1:7890`，**不要尝试更改远程地址为 SSH**，直接保留 HTTPS。
- **不要删除** `.codebuddy/` 文件夹（项目元数据，由系统管理）。
