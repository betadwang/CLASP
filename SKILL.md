# Session Sync — 跨设备记忆与会话同步

## 这是什么

本 skill 通过云盘中的 `Claude Memory/` 文件夹作为持久化层，让不同设备上的 Claude 共享记忆和会话上下文。

## 核心约束

> ⚠️ Claude 只能读写 `Claude Memory/` 文件夹内的文件。严禁读取、修改、删除该文件夹以外的任何云盘文件。

## 路径

云盘记忆文件夹路径（需先通过 request_cowork_directory 挂载）：

- **iCloud macOS**：`~/Library/Mobile Documents/com~apple~CloudDocs/Documents/Claude Memory/`
- **Dropbox**：`~/Dropbox/Claude Memory/`
- **Google Drive**：`~/Google Drive/Claude Memory/`
- **OneDrive**：`~/OneDrive/Claude Memory/`

挂载后使用 Read/Write/Edit/Glob 工具直接操作此路径。

## 目录结构

```
Claude Memory/
├── SYSTEM.md                    ← 系统说明书（v2）
├── MEMORY.md                    ← 记忆索引（每行一条，指向独立文件）
├── *.md                         ← 独立知识点记忆文件
│
└── sessions/
    ├── index.json               ← 会话索引（schema v2，含 tier/access_count/provenance）
    ├── 2026-05-01-*.md          ← 单次会话记录
    └── ...
```

## 记忆系统（Memory）

### 读取方式
1. 先读 `MEMORY.md` 查看索引
2. 根据当前对话需求，决定是否读对应的 .md 文件

### 写入方式
对话中发现值得保存的知识点时自动写入，无需询问用户：
1. 创建 `xxx.md`，包含 frontmatter（name、description、type、provenance、reasoning）
2. 在 `MEMORY.md` 末尾添加一行索引

### 记忆类型
- `user` — 用户角色、偏好、知识背景
- `feedback` — 用户对工作方式的指导
- `project` — 项目上下文、目标、deadline
- `reference` — 外部资源位置

### Provenance 溯源与回声循环防护

每条记忆写入时必须标记来源：

| provenance | 含义 | 保存门槛 |
|---|---|---|
| `user_stated` | 用户直接说的 | 低，直接存 |
| `agent_found` | Claude 查资料/读文件/搜索得到 | 中，确认后存 |
| `agent_inferred` | Claude 自己推理/总结的 | 高，需回声检查 |

**回声检查（仅对 agent_inferred 执行）**——保存前 Claude 必须自问三个问题：
1. 这条「新知识」的原始信息来源是什么？（必须来自用户或外部，不能来自已有记忆的总结）
2. 有没有已有记忆覆盖了这个话题？
3. 不存的话下次对话会丢失什么？（丢失的是 Claude 自己的推理而非用户事实 → 不存）

**循环阻断**：若发现信息源链中出现环 → 阻断写入 + 标记中间记忆为 `possibly_echo: true`。

## 会话系统（Session）

### index.json 格式（schema v2）

```json
{
  "schema_version": 2,
  "sessions": [
    {
      "id": 1,
      "title": "示例：项目架构讨论",
      "date": "2026-05-01",
      "updated_at": "2026-05-01T00:00:00+08:00",
      "file": "2026-05-01-示例-项目架构讨论.md",
      "keywords": ["记忆系统", "跨设备同步"],
      "has_pending": true,
      "access_count": 3,
      "last_accessed": "2026-05-01T12:00:00+08:00",
      "tier": "hot"
    }
  ]
}
```

新增字段：`access_count`（累计加载次数）、`last_accessed`（最后加载时间）、`tier`（hot/warm/cold）。

### 会话文件格式

```markdown
---
id: 1
date: 2026-05-01
updated_at: 2026-05-01T00:00:00+08:00
title: 示例：项目架构讨论
keywords: [记忆系统, 跨设备同步]
---

## 议题
- ...

## 关键决策
- ...

## 创建的文件
- ...

## 未完成事项
- ...

## 可恢复上下文
[简要描述，方便另一台设备快速进入状态]
```

### 可用指令

| 用户说的 | 做什么 |
|--------|--------|
| "整理今天聊了什么" / `/session list` | 读取 index.json，列出所有会话 |
| "把当前会话保存" / `/session save` | 将本次对话结构化写入 sessions/ 并更新索引 |
| "加载会话 N" / `/session load N` | 读取对应文件，恢复上下文，access_count+1 |
| "更新会话 N" / `/session update N` | 追加新内容到已有会话文件 |
| "删掉会话 N" / `/session rm N` | 删除文件 + 更新索引 |
| "锁定会话 N" | 手动将会话设为 hot 并锁定不自动降级 |
| "归档会话 N" | 手动设为 cold |
| "看一下当前云端有哪些会话" | 同 list |
| "帮我整理这周/这个月的对话" | 按日期聚合列出 |

## 冷热三级加载

| 层级 | 条件 | 加载策略 |
|------|------|---------|
| 🔥 hot | access_count ≥ 2 或 7天内访问过 | 启动时全文加载 |
| 🌤 warm | has_pending 或 关键词相关 | 仅加载 title+keywords+未完成摘要，按需展开 |
| ❄️ cold | 其余 | 仅索引行，用户明确要求时才加载 |

**升降规则**：每次 load → access_count+1 → 检查阈值；30天未访问 hot→warm；90天未访问 warm→cold。

**与检索的配合**：hot 直接匹配正文，warm 匹配 title+keywords，cold 跳过。

## 智能会话检索（自动匹配）

### 触发时机
- 对话启动时：快速检查 index.json 中 `has_pending: true` 的会话
- 用户发送消息后：提取关键词做话题匹配

### 检索流程
1. 从用户消息提取关键词（名词/专有名词/项目名/技术术语）
2. 关键词为空则跳过
3. 扫描 sessions/index.json，在 title + keywords 字段做模糊匹配
4. 按 updated_at 降序，取匹配度最高的
5. 读取匹配会话的 frontmatter（仅元数据，不读全文）
6. 关键词重叠 ≥ 2 个或话题高度相关时提示用户

### 检索范围限制
- 单次最多扫描 20 条记录
- 只检索最近 90 天内的会话
- 只匹配 title 和 keywords 字段（不扫描全文）
- 已加载/已拒绝的会话不重复提示

### 提示方式
发现匹配时询问，不直接加载：
> 我注意到云端有会话 [#N] 涉及「话题」，最后更新于 [日期]，是否加载上下文继续？

### 保存/更新时的自动补全
1. 更新 `updated_at` 为当前时间
2. 从议题和关键决策中提取关键词，写入 `keywords`
3. 有未完成事项 → 自动标记 `has_pending: true`
4. 冷热升降检查（更新 access_count、last_accessed、tier）

## 执行检查清单

每次执行保存/加载/同步操作时：
- [ ] 确认 Claude Memory 路径已挂载（Write 可用）
- [ ] 直接写入 `sessions/` 目录
- [ ] 更新 index.json（含 updated_at、access_count、tier）
- [ ] 检查 has_pending 状态
- [ ] 新记忆写入前标记 provenance + 完成回声检查（仅 agent_inferred）
- [ ] 新记忆 reasoning 字段已填写
