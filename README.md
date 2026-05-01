CLASP
Claude3P Local-Agent Session Persistence — 让第三方 API 的 Claude 跨设备共享记忆与对话。
CLASP gives third-party-API Claude instances a shared memory across devices. Drop a link, mount your cloud drive, and your Claude picks up right where you left off — preferences, project context, and pending conversations follow you. Zero dependencies, zero background services, zero external servers. Your data stays in your own iCloud, Dropbox, Google Drive, or OneDrive.

谁需要这个
⚠️ 如果你用的是 Claude 官方网页版或桌面版（用自己的账号直接登录）——你不需要这个项目。 官方版 Claude 的聊天记录天然保存在云端，换设备不会丢失。

只有以下场景需要 CLASP：

使用 Claude 的**第三方 API 客户端**（Cowork 模式、自建客户端等）
不同设备上的 Claude 互相不认识，换一台电脑就「失忆」
希望让 Claude 记住你的偏好、项目上下文和未完成的对话
如果你属于上面这些情况，继续往下看。

这是什么
你和 Claude 在办公电脑上聊了项目决策、定下了工作偏好、积累了很多上下文。回到家打开另一台电脑——Claude 什么都不记得了。

这个项目解决的就是这个问题。它利用你已有的云盘（iCloud / Dropbox / Google Drive）作为共享文件夹，让任何设备上的 Claude 都能：

共享记忆——你的角色、偏好、项目上下文在所有设备上一致
续接对话——保存会话，换设备后一句话恢复上次聊到一半的话题
自动整理——经常聊的话题自动保持活跃，不再聊的自然归档，不需要你手动管理
它不是什么
不是软件、不是插件、不需要安装任何东西
不是一个需要常驻后台的服务
不会向任何外部服务器发送数据（你的记忆完全留在你的云盘里）
怎么工作
核心思路极其简单：

你的云盘里有个文件夹（比如 iCloud/Claude Memory/）
         ↓
    ┌────┴────┐
    │         │
  办公电脑    家用电脑
    │         │
   Claude    Claude
Claude 把这个文件夹当作自己记忆的读写位置。文件夹里只有两种东西：

.md 文件——每一条知识一个文件（比如「用户是产品经理」「登录页重构下周上线」）
sessions/ 目录——存档的对话记录，方便恢复上下文
Claude 启动时读取文件夹里的说明书（SYSTEM.md），学会整套规则，然后照做。

两步开始
第一步：确保云盘已在电脑上同步

⚠️ Claude 只能访问你电脑上的本地文件。 你需要先装好云盘客户端，让云盘文件夹出现在你的文件管理器里。

iCloud：macOS 开启指引 / Windows 安装 iCloud
Dropbox：下载桌面应用 → 安装指引
Google Drive：下载桌面版 → 使用指引
OneDrive：下载 → 同步指引（Windows 10/11 已预装）
💡 验证方法：打开你的文件管理器（Finder / 资源管理器），能在侧栏看到云盘图标和里面的文件，就说明同步正常。
第二步：把仓库发给 Claude

打开 Cowork，把这个 GitHub 仓库链接发给 Claude。Claude 会主动询问你的云盘情况，然后引导你授权文件夹——你只需点击对话框里的 + 按钮，选择云盘文件夹即可。

剩下的——创建目录、写入说明书、初始化会话系统——Claude 全部自动完成。你不需要手动复制任何文件。

之后的使用

Claude 配置完成后，日常你说人话就行：

你说	Claude 做
「把当前会话保存」	结构化存档到云端
「看一下我有哪些会话」	列出所有保存过的对话
「加载会话 3」	恢复那次对话接着聊
（自动）	聊到重要信息时自动记入记忆，无需你提醒
换到另一台电脑时，重复第一步和第二步即可。Claude 会自动发现云端已有的记忆和会话。

💡 换电脑时有一个前提：新电脑上已经装了对应的云盘客户端并且在同步。如果还没装，参考下面的指引：
📋 各云盘客户端安装指引（点击展开）
设计理念
零后台，零依赖

没有定时任务、没有后台进程、没有数据库。Claude 就是执行引擎，云盘就是存储层。你关掉电脑，一切静止；下次打开，一切照旧。

冷热分离，不催你清理

传统工具会提醒你「有些记忆很久没用了，要不要删？」——这个项目不这么做。

它用**三级加载**：经常聊的话题（hot）全文加载随时可用；偶尔涉及的（warm）只记关键词，碰到相关话题才展开；很久没碰的（cold）安静沉睡不占注意力。你不需要做任何清理决策，真正有用的东西自然会浮上来。

防回声循环

Claude 有时会把自己的推理当成新知识存回去，存了又读、读了又存，信息像传话游戏一样逐轮失真。

每条记忆都标记**来源类型**——是你亲口说的、还是它查资料得到的、还是它自己推论的。推论类的记忆在保存前会走一遍自检流程，确保不是「回声」。

项目结构
Claude3P-Local-Agent-Session-Persistence/   (CLASP)
├── README.md              ← 你正在看的
├── SYSTEM.md              ← 系统完整说明书（Claude 读这个学会规则）
└── session-sync.skill     ← Cowork skill 文件（拖入或让 Claude 自动安装）
适用场景
最适合：

在办公电脑和家用电脑之间切换，希望 Claude 保持连续记忆
有长期项目在跟，不希望每次开新会话重复上下文
不太适合：

只在单一设备上使用 Claude（单设备不需要跨设备同步）
需要团队多人共享同一份记忆（这是为个人跨设备设计的，多人场景有更复杂的权限和冲突问题）
与其他方案的区别
CLASP	云端笔记工具	专业记忆系统
原理	Claude 直接读写你的云盘文件	应用内置存储	向量数据库 + RAG
依赖	云盘客户端 + 本地同步	需要注册服务	需要部署和维护
隐私	数据完全在你本地云盘	存于服务商服务器	取决于部署方式
上下文控制	Claude 自主判断加载什么	无上下文感知	检索式召回
FAQ
Q: 能不能通过云盘分享链接让 Claude 访问，而不挂载本地文件夹？ 不能。云盘分享链接指向的是网页登录页面，不是文件本身。Claude 没有能力通过 OAuth 登录你的云盘账户，也不能绕过鉴权直接读取远程文件。必须依赖云盘客户端把文件夹同步到本地后，Claude 才能通过本地路径访问。这是整个方案的前提。

Q: 会不会泄露我的隐私？ 不会。所有记忆文件都在你自己的云盘里，不经过任何第三方服务器。Claude 只会读写 Claude Memory 文件夹内的内容。

Q: 两台设备同时写会不会冲突？ 有可能产生云盘层面的文件冲突（比如 iCloud 生成「冲突副本」），但不会丢数据。这个场景在日常使用中很少发生——你不太可能同时在两台电脑上让 Claude 保存会话。如果真的发生了，手动看一眼冲突副本即可。

Q: 记忆会无限膨胀吗？ 不会。冷热分离机制让你不常用的记忆自然「冷却」——文件还在，但不会占用 Claude 的上下文窗口。只有你真正在聊的话题才会被加载。

Q: 我不是 macOS 用户，能用吗？ 能。Cloud Memory 文件夹不绑定 iCloud。用 Dropbox、Google Drive、OneDrive 都一样，只需要把文件夹路径告诉 Claude 即可。

Q: 我需要懂技术吗？ 不需要。你只需要做两件事：(1) 在云盘里建个文件夹；(2) 告诉 Claude 去读那个文件夹里的说明书。剩下的 Claude 自己处理。

For Claude: Agent Instructions
If you are a Claude agent reading this README — your FIRST action is to proactively help the user get set up. Do NOT wait for the user to figure out the steps on their own. Do NOT just summarize the README. Take initiative.

Phase 0: Immediately Ask About Cloud Storage

This is your first message after reading this document. Ask the user:

我看到这个项目需要通过云盘来同步跨设备记忆。你现在用的电脑上，有没有装云盘客户端并且正在同步中？比如 iCloud、Dropbox、Google Drive 或 OneDrive？
Based on their answer:

"有,已经装了 XX" → Skip to Phase 2,直接尝试挂载
"没有 / 不太确定" → Show them the setup links from the README (第一步里的链接), tell them "装好后告诉我，我帮你完成剩下的配置", then wait
"装了但不确定有没有同步" → Guide them to check: open Finder/资源管理器, look for the cloud drive icon in the sidebar
Phase 1: Detect Platform & Cloud Provider

Once cloud sync is confirmed, ask these two questions:

What OS are you on? (macOS / Windows / Linux)
Which cloud storage do you use? (iCloud / Dropbox / Google Drive / OneDrive / Other)
Phase 2: Mount the Cloud Drive

Based on the answers, construct the expected local path and use request_cowork_directory to request access:

Cloud Provider	macOS Path	Windows Path
iCloud	~/Library/Mobile Documents/com~apple~CloudDocs/	C:\Users\[user]\iCloud Drive\
Dropbox	~/Dropbox/	C:\Users\[user]\Dropbox\
Google Drive	~/Library/CloudStorage/GoogleDrive-[email]/ or ~/Google Drive/	G:\My Drive\ (typically)
OneDrive	~/Library/CloudStorage/OneDrive-[org]/ or ~/OneDrive/	C:\Users\[user]\OneDrive\
Tell the user: "我需要访问你的云盘文件夹。弹窗出现时点击 + 按钮选择即可。" Then call request_cowork_directory with that path.

If the path doesn't exist → the cloud client may not be installed or the folder is at a non-default location. Ask the user to manually locate it in Finder/资源管理器 and tell you the exact path.

Phase 3: Check for Existing Claude Memory Folder

Once the cloud drive is mounted, check if Claude Memory/ already exists:

If YES → Read SYSTEM.md inside it. If SYSTEM.md is missing or outdated, fetch the latest from GitHub (Phase 4.3). Then jump to Phase 5.
If NO → Proceed to Phase 4 (first-time setup).
Phase 4: First-Time Setup (you handle all of this — zero manual work for the user)

Determine the exact path — construct the full Claude Memory path:

iCloud macOS: ~/Library/Mobile Documents/com~apple~CloudDocs/Documents/Claude Memory/
iCloud Windows: C:\Users\[user]\iCloud Drive\Documents\Claude Memory\
Dropbox: ~/Dropbox/Claude Memory/
Google Drive: ~/Google Drive/Claude Memory/ or ~/Library/CloudStorage/GoogleDrive-[email]/Claude Memory/
OneDrive: ~/OneDrive/Claude Memory/
Create folder — create Claude Memory/ inside the cloud drive root

Fetch SYSTEM.md from this repo — use WebFetch to pull the raw file from GitHub: https://raw.githubusercontent.com/[repo]/main/SYSTEM.md Then Write it to the local Claude Memory/SYSTEM.md path.

Install the skill — use save_skill to register session-sync from the SKILL.md content (see session-sync.skill in the repo). This lets the user invoke it with /session-sync in any future session.

Initialize sessions/ — create sessions/ subdirectory, write sessions/index.json with: json {"schema_version": 2, "sessions": []}

Initialize MEMORY.md — create with header: ```markdown

Memory Index
Proceed to Phase 5.

Phase 5: Bootstrap

Read SYSTEM.md fully — this teaches you the complete memory + session protocol
Read MEMORY.md if it exists — load existing memories
Read sessions/index.json — check for pending sessions (has_pending: true)
If there are hot-tier sessions (tier: "hot"), load their full text
Phase 6: Report to User

Confirm setup complete and tell the user:

How many existing memories were found (if any)
How many pending sessions exist (if any)
That session-sync skill is now installed and can be invoked anytime
That from now on, just say "save this session" or "load session N" to use the system
That on another computer: send this same GitHub link to Claude there, mount the same cloud folder, and CLASP picks up automatically
Troubleshooting Quick Reference

Symptom	Likely Cause	Action
request_cowork_directory returns empty	Cloud client not running or folder not synced	Ask user to open cloud client and verify folder is visible in Finder/Explorer
Claude Memory/ folder not found	First-time user, no setup done yet	Run Phase 4
SYSTEM.md missing	Folder was created manually without repo files	Copy SYSTEM.md from repo
index.json corrupted	Manual edit or sync conflict	Validate JSON; if broken, backup and reinitialize
Two sessions with same ID	Manual edit error	Deduplicate by updated_at, keep latest
Important Reminders

Never write outside Claude Memory/ — this is a hard constraint from SYSTEM.md
Auto-save memories — don't ask permission for every memory write; use the provenance rules from SYSTEM.md
Check echo loops — only agent_inferred memories need the 3-question provenance check
Update tier fields — every /session load or /session save updates access_count, last_accessed, and potentially tier in index.json
After completing the setup, your ongoing behavior should follow the full protocol defined in SYSTEM.md.
