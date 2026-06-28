---
layout: post
title: "Learn Tutorial Generator — AI 驱动的教科书级教程生成器"
date: 2026-06-28 10:00:00 +0800
categories: 技术 教程
tags: [AI, 教程, 开源, Learn-Tutorial-Generator]
---

# Learn Tutorial Generator：AI 驱动的教科书级教程生成器

> 开源地址：[github.com/Yobeeo/Learn-Tutorial-Generator](https://github.com/Yobeeo/Learn-Tutorial-Generator)

---

想学一个新东西——FastAPI、React、LangGraph、Rust——打开搜索引擎，出来几百篇教程。

- 这篇太老了，用的还是 Python 2
- 那篇太浅了，翻完两页还是不知道从哪下手
- 还有一篇上来就讲架构设计，连 `pip install` 都没教

于是收藏了一堆文章，看了半小时，合上电脑……**还是不知道该从哪开始。**

为了解决这个问题，**Learn Tutorial Generator** 诞生了——**输入你想学的东西，AI 自动生成一本教科书级的系统教程。**

不是拼凑的文章合集，不是摘要式的概览，而是一本**从零到精通、有类比有代码有练习、每一章都环环相扣的系统教程。**

开源，免费，无需任何付费 API Key 即可本地使用。

---

## 它是什么

Learn Tutorial Generator 是一个基于 AI 的学习教程生成器。

在 Claude Code 中输入 `/learn`，它通过对话了解使用者的学习需求：

- 想学什么？
- 为什么想学？（工作需要？转行？纯兴趣？）
- 当前水平如何？
- 想学到什么程度？
- 开发环境是什么？
- 每天能花多少时间学？
- 有没有想做的具体项目？

**然后——它开始工作。**

1. **联网搜索**权威资料（官方文档、知名教程、技术博客）
2. **整理知识结构**，列出完整的知识点清单
3. **设计章节大纲**，确认后再开始写
4. **逐章撰写**，每章写完即可查看
5. **自动质量审查**，8 个维度打分，找出问题

整个过程大约 20-30 分钟，生成一本 8-15 章的系统教程。

---

## 和其他教程生成工具的区别

市面上不缺 AI 写教程的工具。但大多数有个共同问题：**写出来的东西看起来不错，但没法真正用来学习。**

Learn Tutorial Generator 试图解决以下几个痛点：

### 1. 先了解人，再写教程

大多数 AI 教程生成器拿到主题就直接开写。"学 Python" → 上来就讲变量和类型。

但它不知道使用者是零基础小白还是十年老手转 Python，不知道是为了面试还是为了搭个人项目，不知道电脑是 Windows 还是 Mac。

Learn Tutorial Generator 先通过 7 个核心问题了解基本情况。而且不是机械地填表——每个问题的措辞会根据之前的回答动态调整。说想转行，就追问目标公司和岗位；说每天只能学 20 分钟，章节就会拆得更短。如果某个回答不够清晰，还会追加追问（比如"学过面向对象吗？""会自己排查报错吗？"）。整个过程大约 10-15 轮对话，直到有足够的了解才开始写教程。

### 2. 教科书级结构，不是博客文章拼凑

每章都遵循教科书的标准结构：

```
前置知识检查 → 本章目标 → 生活类比引入 → 原理讲解
→ 代码实战（逐行解释）→ 试试看（微型练习）
→ 常见问题 → 章节练习（⭐/⭐⭐/⭐⭐⭐）
→ 本章小结 → 关键术语 → 下章预告
```

不是"概念定义 + 一段代码"就结束了。它有**渐进的难度曲线**、**可运行的代码示例**、**分层练习**、**术语表**，甚至有一个**贯穿所有章节的主线项目**——每章都在同一个项目上增量添加功能，学完最后一章手里已经有一个完整的东西。

### 3. 绝不重复

这是最难做到的。大多数 AI 生成教程的通病是：第 3 章又在解释第 1 章讲过的概念，第 5 章的代码和第 2 章几乎一模一样。

Learn Tutorial Generator 有一套**防重复机制**：每章写之前先读前面所有章节的小结和术语表，已定义的术语绝不重新解释，已讲过的概念只说「详见第 X 章」。代码示例也基于前章代码增量编写，不是每次都重写。

### 4. 写完了还要自己查

教程生成完之后，还有一个 `/learn-review` 质量审查技能。它会按 8 个维度打分（总分 /55）：

| 维度 | 权重 | 检查什么 |
|------|------|---------|
| 🔴 代码可运行性 | ×3 | import 是否完整？变量是否已定义？ |
| 🔴 跨章引用正确性 | ×2 | 「详见第 X 章」的 X 真的讲了那个内容吗？ |
| 🟡 难度曲线 | ×1 | 有没有突然的难度跳跃？ |
| 🟡 内容重复 | ×1 | 相邻章节有没有在讲同一个东西？ |
| 🟡 个性化一致性 | ×1 | 教程里的环境指令匹配你的 OS 吗？ |
| 🟢 结构完整性 | ×1 | 每章该有的小节都有吗？ |
| 🟡 练习质量 | ×1 | 练习题是真的在考知识，还是照猫画虎？ |
| 🟡 语言难度 | ×1 | 术语密度匹配你的水平吗？ |

发现问题还会给出具体的修复建议。

### 5. 搜的是中文资料，不是英文文档

Anthropic 自带的搜索工具在美国服务器上运行，在国内环境下很多中文网站根本打不开。

Learn Tutorial Generator 使用的是 **DuckDuckGo** 搜索，免费、无需 API Key、国内直连。搜索策略也是中文优先——先搜 `{主题} 教程 入门`，再搜 `{topic} official documentation`，兼顾中文社区和英文一手资料。

---

## 技术实现

整个项目非常轻量：

- **后端**：Python FastAPI（57 行核心代码）
- **前端**：Jinja2 模板 + 手写 CSS（283 行）
- **存储**：纯文件系统，Markdown + JSON，零数据库依赖
- **LLM**：通过 Claude Code 的 Skill 系统驱动，支持 Anthropic / DeepSeek / OpenAI
- **搜索**：DuckDuckGo，免费无需 Key
- **部署**：`pip install -r requirements.txt && python main.py`，一行命令启动

生成的教程是**纯 Markdown 文件**，不需要任何特殊工具也能读。可以直接 `git push` 到博客仓库，或者用内置的发布工具发到博客园和微信公众号。

---

## 工作流程

整个流程分五个阶段：

```
Phase 1: 采集需求（7 个核心问题 + 条件追问）
    ↓
Phase 2: 搜索资料（DuckDuckGo + 网页抓取）
    ↓
Phase 3: 生成大纲（确认后再动笔）
    ↓
Phase 4: 逐章撰写（每章 ~5-10 分钟）
    ↓
Phase 5: 质量审查（8 维度打分 + 修复建议）
```

**Phase 1 — 需求采集**

不是填问卷，是一场真正的对话。比如回答"想学 React"，不会机械地问下一个问题，而是会追问：

> 「你了解过 Vue 或者 Angular 吗？为什么选了 React？」
> 「看到别人用 React 做了什么让你觉得『我也要做这个』？」

这些追问决定了教程里要不要放对比段落、核心示例围绕什么功能展开。

**Phase 2 — 资料搜索**

多轮搜索，三轮查询策略：

```
第一轮："{主题} official documentation tutorial"
第二轮："{主题} beginner guide best practices 2026"
第三轮："{主题} common pitfalls mistakes beginners"
```

然后抓取最有价值的 5-8 个页面，提取结构化知识清单。

**Phase 3 — 大纲设计**

基于知识清单和用户画像，设计 8-15 章的学习路径。每章标注它覆盖的知识领域和一个「里程碑标记」——学完这一章能跑出一个有成就感的东西。

**Phase 4 — 逐章撰写**

这是最耗时的部分。每章大约 5-10 分钟，包含：

1. 针对本章重新搜索补充资料
2. 读取前面积累的术语表，确保不重复
3. 确认主线项目的增量——本章要在项目上加什么
4. 撰写完整的 Markdown 教程
5. 写入文件，更新进度

写完后可以立刻开始读已经写好的章节，不用等全部完成。

**Phase 5 — 质量审查**

运行 `/learn-review`，逐章逐维度审查，输出问题清单和修复建议。

---

## 举个例子

假设想学 **LangGraph**。

工具会先问：
- 想学 LangGraph 做什么？（Agent 开发？RAG 知识库？）
- 当前水平如何？（用过 LangChain 吗？）
- 有 LLM API Key 吗？
- 想做一个什么项目？

然后根据回答，搜索 LangGraph 官方文档、GitHub 示例、社区教程，整理出完整的知识清单：

```
核心概念：StateGraph、Node、Edge、ConditionalEdge、CompiledGraph
进阶用法：Send API（动态扇出）、Interrupt（人工审核）、Checkpointer（状态持久化）
设计模式：Supervisor 多 Agent、Map-Reduce 并行、Human-in-the-loop
生产实战：SqliteSaver vs PostgresSaver、线程管理、错误处理
```

然后设计一个 10 章的大纲，从"什么是 LangGraph"一路讲到"做一个带人工审核的 AI Agent"。

每章都有代码、有练习、有小结。最后一章手里已经有一个**完整可运行的 AI Agent 项目**。

---

## 适合谁用

- **想系统学习新技能的开发者** — 告别碎片化阅读，获得一本真正从零到一的教程
- **技术团队的内训素材** — 根据团队成员的实际水平定制教程，比丢一篇链接有效得多
- **喜欢做笔记的人** — 生成的教程是 Markdown，天然适合放进笔记系统

---

## 怎么开始

### 前置条件

- Claude Code 账号（或任何支持 Claude API 的客户端）
- Python 3.11+

### 安装

```bash
git clone https://github.com/Yobeeo/Learn-Tutorial-Generator.git
cd lesson
python -m venv .venv
.venv\Scripts\activate    # Windows
source .venv/bin/activate # macOS / Linux
pip install -r requirements.txt
```

### 生成教程

在 Claude Code 中输入：

```
/learn
```

然后回答 7 个核心问题（根据回答会有追问），剩下的交给 AI。

### 本地阅读

```bash
python main.py
# 浏览器打开 http://localhost:8000
```

内置的 Web 阅读器支持目录导航、Markdown 渲染、代码高亮，体验接近正经的在线文档站。

### 发布

教程写完后，可以直接发布到博客园或微信公众号：

```bash
# 博客园（本地直连，无需额外部署）
python tools/platform-publisher/publisher.py langgraph -p cnblogs

# 微信公众号（需要固定 IP，详见 README）
python tools/platform-publisher/publisher.py langgraph -p wechat
```

> **微信公众号注意：** 微信 API 要求请求 IP 在白名单中。本地动态 IP 无法直接发布，需部署代理服务器中转。详见 README。

---

## 开源协议

MIT License，自由使用，保留署名即可。

---

## 写在最后

学习不应该这么难。

我们不需要更多碎片化的文章，不需要更多"5 分钟入门"的标题党。需要的是有人坐下来，系统地、耐心地把一个知识点从头到尾讲清楚——有类比帮你建立直觉，有代码帮你动手实践，有练习帮你巩固理解。

以前这需要一位好老师花几周时间准备。现在，AI 可以做到。

如果你也曾面对一个陌生的技术主题无从下手，试试这个工具吧。输入你想学的东西，剩下的交给 AI。

---

*如果觉得这个项目有用，欢迎 Star 和 Fork。有问题或建议可以在 [GitHub Issues](https://github.com/Yobeeo/Learn-Tutorial-Generator/issues) 里提。*
