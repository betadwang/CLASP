# Claude 跨设备记忆与会话系统

## 这是什么

本系统利用你的云盘（iCloud / Dropbox / Google Drive / OneDrive）中的 `Claude Memory/` 文件夹作为持久化层，让不同设备上的 Claude 共享记忆和会话上下文。

## 核心约束

> ⚠️ **Claude 只能读写 `Claude Memory/` 文件夹内的文件。严禁读取、修改、删除该文件夹以外的任何云盘文件。**

## 目录结构

```
Claude Memory/
├── SYSTEM.md                    ← 本文件（系统说明书）
├── MEMORY.md                    ← 记忆索引（每行一条，指向独立文件）
├── *.md                         ← 独立知识点记忆文件（由对话自动生成）
│
└── sessions/
    ├── index.json               ← 会话索引
    ├── 2026-05-01-*.md          ← 单次会话记录
    └── ...
```

## 记忆系统（Memory）

### 读取方式
1. 先读 `MEMORY.md` 查看索引（前 200 行）
2. 根据当前对话需求，决定是否读对应的 .md 文件

### 写入方式
对话中发现值得保存的知识点时，自动写入，无需询问用户：

1. 创建 `xxx.md`，包含 frontmatter（name、description、type、provenance、reasoning）
2. 在 `MEMORY.md` 末尾添加一行索引

### 记忆类型
| 类型 | 用途 | 示例 |
|------|------|------|
| `user` | 用户角色、偏好、知识背景 | 用户是数据科学家 |
| `feedback` | 用户对我工作方式的指导 | 用户要求不总结已完成的修改 |
| `project` | 项目上下文、目标、deadline | 合并冻结从 3月5日开始 |
| `reference` | 外部资源位置 | Bug 跟踪在 Linear INGEST 项目 |

### 记忆溯源与回声循环防护（Provenance）

每条记忆写入时必须标记来源（`provenance` 字段），防止 Claude 把自己推理出的内容当成新知识存回去，造成信息在循环中失真（回声循环）。

**来源分级：**

| provenance | 含义 | 保存门槛 |
|---|---|---|
| `user_stated` | 用户直接说的（"我们在做 X 项目"） | **低**，直接存 |
| `agent_found` | Claude 查资料/读文件/搜索得到 | **中**，确认后存 |
| `agent_inferred` | Claude 自己推理/总结出来的 | **高**，需通过回声检查 |

**回声检查（仅对 `agent_inferred` 执行）：**

保存前 Claude 必须自问三个问题并写入 `reasoning` 字段：

1. 这条「新知识」所依据的**原始信息来源**是什么？（必须来自用户或外部，不能来自已有记忆的总结）
2. 有没有**已有的记忆**覆盖了这个话题？（扫描 MEMORY.md 标题与描述）
3. 这条信息如果不存，下次对话会**丢失什么**？（如果丢失的是 Claude 自己的推理而非用户事实 → 不存）

**循环阻断：**

若发现一条记忆的信息源链中出现环——例如 memory-B `derived_from: [memory-A]`，之后 memory-C 又 `derived_from: [memory-B]` 且内容与 memory-A 语义重复——则阻断写入并标记 memory-B 为 `possibly_echo: true`。

## 会话系统（Session）

### 数据结构

`sessions/index.json` 是全部会话的清单：

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

**字段说明：**

| 字段 | 说明 |
|------|------|
| `id` | 会话唯一编号 |
| `date` | 创建日期 |
| `updated_at` | 最后更新时间（ISO 8601 含时区），用于排序和时效判断 |
| `file` | 对应的 .md 文件名 |
| `keywords` | 话题关键词数组，用于自动检索匹配 |
| `has_pending` | 是否有未完成事项（true/false），快速过滤可恢复会话 |
| `access_count` | 累计被加载次数，每次 `/session load N` 自动 +1 |
| `last_accessed` | 最后一次被加载的时间戳（ISO 8601），冷热升降依据 |
| `tier` | `hot` / `warm` / `cold`，决定加载策略 |

每个会话文件的格式：

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
[简要描述，方便另一台设备上的我快速进入状态]
```

### 可用指令

用户可以在对话中使用以下自然语言指令：

| 你说的 | 我做的 |
|--------|--------|
| "整理今天聊了什么" / `/session list` | 读取 `sessions/index.json`，列出所有会话 |
| "把当前会话保存" / `/session save` | 将本次对话结构化写入 `sessions/` 并更新索引 |
| "加载会话 2" / `/session load 2` | 读取对应文件，恢复上下文继续对话 |
| "更新会话 2" / `/session update 2` | 追加新内容到已有会话文件 |
| "删掉会话 2" / `/session rm 2` | 删除文件 + 更新索引 |
| "看一下当前云端有哪些会话" | 同 `/session list` |
| "帮我整理这周/这个月的对话" | 按日期聚合列出 |

## 智能会话检索（自动匹配）

### 触发时机

| 时机 | 行为 |
|------|------|
| **对话启动时** | 快速扫描 index.json，检查是否有 `has_pending: true` 的未完成会话 |
| **用户发送消息后** | 从用户消息中提取关键词，在云端会话中做话题匹配 |

### 检索流程

```
用户发消息 → 提取关键词（名词/专有名词/项目名/技术术语）
     ↓
关键词为空？ → 跳过，不检索
     ↓
扫描 sessions/index.json，在 title + keywords 字段做模糊匹配
     ↓
匹配数 > 0？ → 按 updated_at 降序，取前 N 条
     ↓
读取匹配会话的 frontmatter（仅元数据，不读全文）
     ↓
关键词重叠 ≥ 2 个 或 话题高度相关？
     ↓ 是
提示用户"检测到云端有相关会话，是否加载？"
     ↓
用户确认 → 执行 /session load N
用户拒绝 → 记录本次不提醒（同会话不重复提示）
```

### 检索范围限制（防超时）

- 单次检索最多扫描 **20 条**会话记录
- 只检索 `updated_at` 在 **最近 90 天内**的会话（可调整）
- 只匹配 `title` 和 `keywords` 字段（不扫描全文，避免读大量文件）
- 已加载过的会话不重复提示
- 若 index.json 超过 50 条，自动截取最近 90 天 + 所有 `has_pending: true` 的条目

### 提示方式

发现匹配时不直接加载，而是询问用户，格式统一：

> 我注意到云端有会话 [#N] 涉及「话题」，最后更新于 [日期]，是否加载上下文继续？

用户回答"好/加载/是的" → 执行 `/session load N`
用户回答"不用/不了/跳过" → 记录跳过，本次不重复问

### 会话保存时的自动补全

执行 `/session save` 或 `/session update` 时，自动：

1. 更新 `updated_at` 为当前时间
2. 从议题和关键决策中提取关键词，写入 `keywords`
3. 如果有未完成事项 → 自动标记 `has_pending: true`
4. 更新 index.json 中的对应条目

## 冷热三级加载（避免上下文膨胀）

随着会话数量增长，全部加载会耗尽上下文窗口。本系统采用**被动冷却策略**——不删数据，不提醒用户手动清理，而是根据实际使用频率自动控制加载深度。

**三级定义：**

| 层级 | 条件 | 加载策略 |
|------|------|---------|
| 🔥 hot | `access_count ≥ 2` 或 `last_accessed` 在 7 天内 | 启动时**全文加载** |
| 🌤 warm | `has_pending: true` 或 关键词与当前话题相关 | 仅加载 `title + keywords + 未完成事项摘要`，用户消息匹配关键词后再读正文 |
| ❄️ cold | 其余全部 | **仅索引行**（存在于 index.json，不读文件），用户明确说 `加载会话 N` 时才展开 |

**升降规则：**

- **升温**：每次 `/session load N` → `access_count +1` → 自动检查升降阈值
- **降温**：超过 30 天未访问的 hot → warm；超过 90 天未访问的 warm → cold
- **手动升降**：用户可以说「锁定会话 N 为 hot」或「归档会话 N」

**与智能检索的配合：**

检索时 hot 会话直接匹配正文内容（已在上下文中），warm 会话匹配 title + keywords，cold 会话被跳过（除非用户明确要求列出全部）。

**对比传统过期机制的优势：**

- 不丢数据 —— 冷数据只是不加载，不是删除
- 无需用户决策 —— 无需提醒"要不要清理记忆"
- 自适应 —— 真正在用的话题自动升到热层，不用的话题自然冷却

## 跨设备工作流

### 设备 A（当前设备）
1. 对话中自动保存记忆 → `Claude Memory/*.md`
2. 用户说 `把当前会话保存` → 写入 `sessions/` 目录
3. 云盘自动同步到云端

### 设备 B（另一设备）
1. 用户打开 Claude，发送 CLASP 仓库链接，挂载 `Claude Memory` 文件夹
2. 我读取 `SYSTEM.md`（本文件）→ 学习全部规则
3. 读取 `MEMORY.md` → 加载已有记忆
4. 用户说 `加载会话 1` → 读取对应文件，恢复上下文
5. 继续对话，新内容同样可以同步回云端

## Skill 安装（推荐）

CLASP 仓库中包含 `session-sync.skill` 文件，可拖入 Cowork 聊天框安装。Skill 加载后，Claude 在任何会话中都能快速进入跨设备记忆同步模式。

---

## 常见陷阱与效率指南（经验总结）

### 陷阱 1：两条路径混淆

系统存在**两条路径**，容易搞混：

| 路径 | 说明 | 可写？ |
|------|------|--------|
| `auto-memory 路径`（系统内置记忆目录） | 仅供 Read | **Write 不可达** |
| `Claude Memory/` 路径（云盘同步目录） | 真实同步目录 | **Read + Write 均可**（需挂载） |

**教训**：不要试图往 auto-memory 路径写入。发现 Read 能读但 Write 报错时，立即切换策略。

### 陷阱 2：先读 SYSTEM.md，不要零散试错

接手同步操作时，第一件事就是**完整读取本文件**。

### 陷阱 3：优先写入已连接的文件夹

会话读写应遵循以下优先级：

```
Claude Memory/ 已挂载？  ──是──→ 直接读写 Claude Memory/sessions/
        ↓ 否
请求挂载 Claude Memory/  ──是──→ 同上
        ↓ 否
回退：写入 outputs 目录 + 告知用户手动处理
```

### 陷阱 4：bash 不可用时不重试

当 Linux 环境反复返回 "starting" 或 "unavailable" 时：
- 重试 1-2 次即可，不要持续重试
- 立即切换到文件工具路径（Read / Write / Edit）

### 执行检查清单

每次执行 `/session save` / `/session load` / 同步操作时：

- [ ] 已读取 SYSTEM.md 确认最新规则
- [ ] 确认 Claude Memory 路径已挂载（Write 可用）
- [ ] 直接写入 `sessions/` 目录，不绕道其他路径
- [ ] 更新 index.json
- [ ] 更新 updated_at 时间戳
- [ ] 更新 access_count、last_accessed、tier（冷热升降）
- [ ] 检查 has_pending 状态是否需更新
- [ ] 新记忆写入前标记 provenance 并完成回声检查（仅 agent_inferred）
- [ ] 新记忆 reasoning 字段已填写

---

*CLASP — Claude3P Local-Agent Session Persistence — v2*
*最后更新: 2026-05-01 (新增冷热分层 + provenance 回声防护)*
