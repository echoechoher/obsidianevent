# 花9分钟用 Obsidian+OpenClaw+Codex 彻底重构你的 AI 知识管理系统

> **来源**：X @aiwarts（卡尔的AI沃茨）  
> **发布时间**：2026-03-03  
> **原文链接**：https://x.com/aiwarts/status/2028656504438698417  
> **整理时间**：2026-03-03

---

## 背景与痛点

作者三周前完全迁移到 Claude Code + Obsidian，迁移前的工作流是：**微信 → 滴答清单**（兼容度高，但有根本缺陷）。

**旧工作流的问题：**
- 只是把知识从一片海转到一个湖，还是要二次整理到飞书
- 文本去重、视频文案提取、内容排期……排不开

**迁移 Obsidian 要解决的两个核心问题：**
1. 怎么让信息流动到 Obsidian 里？
2. 怎么让 Obsidian 里存的信息有合理结构，让 AI 和人都看懂？

---

## Obsidian 的本质

> Obsidian 本质上是一个**大型 Markdown 文件阅读器**，有超丰富的插件体系，可以内置 Claude Code（Claudian）。

作者后来把 Claudian 换成了 **Codex App**，原因：
- Claudian 限制最多 3 个对话
- 没有可视化定时任务管理

从 Claude Code 迁移记忆到 Codex App 很简单：把本地 `Claude.md` 复制一份命名为 `Agent.md` 放同目录即可。

---

## 安装必要插件

刚装的 Obsidian 是纯素阅读器，需要安装社区插件（可直接让 Claude Code / Codex App 安装，不用手动）：

| 插件 | 用途 |
|------|------|
| Claudian | 把 Claude Code 内置到侧边栏 |
| 笔记同步助手 | 让 Obsidian 同步微信消息 |
| Image Auto Upload | 图片通过 PicGo 上传到 GitHub，降低存储压力 |
| ObShare | 文件同步到飞书，方便分享团队 |

**文件同步方案**：直接把 Obsidian 文件目录放在 iCloud，免去手动同步。

---

## 文件结构选择

> 不需要过度担心 AI 记不住——只要批量操作时让模型录入 memory，或把重要路径写到 `Claude.md`，Claude Code 新开对话时会默认读。

作者的文件目录融合了：
- GitHub: `MarsWang42/OrbitOS`（基础结构）
- GitHub: `heyitsnoah/claudesidian` 的 metadata 目录（存提示语和工作流模版）

**关键经验：** 限制 AI 写文件的目录深度（作者设定为 3 层子目录），AI 忘记文件时把路径和用途重新写入记忆文件即可。

---

## 信息收录到 Obsidian 的三大类方式

### 1. 插件
- **Obsidian Web Clipper** + **HoverNotes**：针对网页端，连公众号和看视频时的笔记都可以剪藏

### 2. 微信
- 笔记同步助手（支持 OneNote、Obsidian、Notion 同步，可把小红书视频做成图文笔记）

### 3. OpenClaw
- 专门处理难解析的链接和视频
- 抓 X 时还可以把评论区一起抓下来

**收录工作流：**
- zip、图像、链接、PDF 先放到收件箱，让模型出整理计划
- GPT 导出到 Obsidian：让 Claude Code 快速生成 `USER.md` 来记住用户偏好

---

## OpenClaw 配置清单

需要给 OpenClaw 安装的扩展：

```
# 联网搜索和链接解析
x-reader：https://github.com/runesleo/x-reader
Agent Reach：https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md
browserwing：https://raw.githubusercontent.com/browserwing/browserwing/main/INSTALL.md

# Obsidian 打通（让 OpenClaw 能写入 Obsidian）
npx clawhub@latest install obsidian

# find-skills（主动找 Skill 解决问题）
npx clawhub@latest install find-skills

# proactive-agent（自我迭代的主动 Agent）
npx clawhub install proactive-agent-1-2-4
```

**高级技巧**：创建软链接，让 OpenClaw 核心配置文件出现在 Obsidian 本地目录里，这样可以直接在 Obsidian 里编写 `SOUL.md`，OpenClaw 立刻生效。

---

## 核心感悟

> 折腾信息管理最大的一点就是**不要心疼信息损耗**。
> 能存到本地的做个备份，图床会失效，链接会过期，记录的方式尽可能接地气，越简单越好。

OpenClaw 会把同一份数据划分总结，沉淀出合适的知识点分开两个地方储存——来源是给 AI 回忆用的，不是给人看的。

> 在一次次对话的过程中，我在编写它的技能和记忆，它也在主动记录我的喜好，我们都在无限进步。
