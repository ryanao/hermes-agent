---
name: wechat-ft-organizer
description: 自动读取并整理微信"文件传输助手"——通过 wechat-cli 拉取历史消息，按链接/文档/想法/待办/媒体 5 类归档成结构化清单。也支持粘贴降级模式
version: 0.2.0
author: eric
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags: [productivity, personal, wechat, organize]
    related_skills: []
---

# 整理微信文件传输助手

把微信"文件传输助手"那一团乱——链接、文档、语音、随手想法、待办——拉出来，自动分成 5 类，生成结构化报告，可选保存到 `~/notes/wechat-ft/`。

## When to Use

- 用户说"帮我整理一下文件传输助手""帮我理理微信里堆的东西""帮我处理文件传输助手"
- 用户直接输入 `/wechat-ft-organizer`
- 用户想把最近 N 天的文件传输助手内容做日报/周报归档

## Prerequisites

依赖 **[wechat-cli](https://github.com/ryanao/wechat-cli)**（`@canghe_ai/wechat-cli`）——通过读取进程内存提取密钥、解密本地 SQLCipher 数据库来读取聊天记录。默认 JSON 输出，专为 LLM agent tool call 设计。

### 用户还没装？按平台给指令

**macOS**（`WeChat for Mac ≤ 4.1.8.100`，`macOS ≥ 26.3.1`）：

```bash
# 1. 装 Node.js（如果还没装）
brew install node

# 2. 全局安装
npm install -g @canghe_ai/wechat-cli

# 3. 打开微信并保持登录状态

# 4. 初始化（需要读取微信内存取解密 key，所以要 sudo）
sudo wechat-cli init
```

**Windows**（以管理员身份打开 PowerShell 或 CMD）：

```powershell
# 1. 先装 Node.js: https://nodejs.org/
# 2. 全局安装
npm install -g @canghe_ai/wechat-cli

# 3. 打开微信并保持登录状态

# 4. 初始化（需要读取 Weixin.exe 进程内存，必须以【管理员】权限运行）
wechat-cli init
```

**Linux**：

```bash
npm install -g @canghe_ai/wechat-cli
# 确保微信正在运行
sudo wechat-cli init     # 从 /proc/<pid>/mem 读密钥需要 root
```

### 验证安装

```bash
command -v wechat-cli                             # 应该输出一个路径
wechat-cli sessions --limit 3 --format text      # 能看到最近 3 个对话名
```

## Procedure

### 1. 环境检查

用 Bash 工具依次跑：

```bash
command -v wechat-cli
```

- **有输出** → 进入第 2 步
- **无输出** → 按上面 Prerequisites 给出当前平台的安装命令，**暂停**让用户装完再继续

然后用一次轻量调用验证已经 init 过：

```bash
wechat-cli sessions --limit 1 --format json
```

如果报 `keys not initialized` 或类似错误 → 提示用户按平台跑对应 `init` 命令。

### 2. 定位"文件传输助手"会话

不同 wechat-cli 版本对"文件传输助手"的标识名可能是 `文件传输助手` / `filehelper` / `File Transfer`。**先拉 sessions 列表再挑**，别硬猜：

```bash
wechat-cli sessions --limit 30 --format json
```

解析返回的 JSON，在 `name` / `displayName` / `nickname` / `wxid` 等字段里找包含 `文件传输助手`、`filehelper`、`File Transfer` 的条目，**记下它的精确名字**用于下一步。

如果 sessions 里看不到（用户最近没往文件传输助手发过），直接试：

```bash
wechat-cli history "文件传输助手" --limit 1 --format json
```

成功就是对的名字；报错再尝试 `"filehelper"`。

### 3. 跟用户确认参数

除非用户明确说过，否则先简短地问：

- **时间范围**？默认最近 200 条；如果用户说"最近三天""最近一周"，换成 `--start-time`
- **要不要存档**？默认跑完给报告时再问要不要 `write_file` 到 `~/notes/wechat-ft/`
- **要不要顺便留原始备份**？可选额外调用 `wechat-cli export` 存一份原始 markdown

如果用户要求"自动化"（比如 cron 跑），参数全部用默认值，结果强制保存，不再问。

### 4. 拉取消息

```bash
wechat-cli history "<step 2 确认的名字>" \
  --limit <用户指定或 500> \
  --format json \
  [--start-time YYYY-MM-DD] [--end-time YYYY-MM-DD]
```

可选按类型预过滤（效率高一些，但会丢混合消息的上下文，**不推荐**首次运行用）：

```bash
# 例：只拿链接
wechat-cli history "文件传输助手" --type link --format json
```

**默认一次性拿全部，然后在分类阶段判断类型**——这样报告更完整。

### 5. 解析 JSON 并分类

wechat-cli 返回的消息数组里每条大致有 `type`、`content`、`timestamp`（或 `createTime`）、`sender`、附件元信息等字段。按下面规则归到 6 个桶：

| 类别 | 判定标准 |
|------|---------|
| 📎 **待读链接** | `type == "link"` 或 `content` 含 `https?://` |
| 📄 **文档与文件** | `type == "file"` / `"document"`，或附件名含 `.pdf / .docx / .xlsx / .pptx / .zip` 等 |
| 🖼 **图片与媒体** | `type in ("image", "video", "voice")` |
| ✅ **待办事项** | 正文匹配 `记得\|别忘\|必须\|要\|需要\|todo\|deadline\|截止`，或含明确日期/时间词 |
| 💡 **想法碎片** | 其余纯文本，5 字 < 长度 < 200 字 |
| 🗑 **其他** | 以上都不像（极短、纯表情、明显重复粘贴的废稿） |

**时间戳**：优先用 JSON 里的 `timestamp` 字段，转成 `YYYY-MM-DD HH:MM` 显示。

### 6. 输出报告

格式：

```markdown
# 文件传输助手整理结果（共 N 条，YYYY-MM-DD ~ YYYY-MM-DD）

## 📎 待读链接 (X)
- 摘要或标题 — <URL> — *YYYY-MM-DD HH:MM*
- ...

## 📄 文档与文件 (X)
- 文件名（大小如有） — *时间*
- ...

## 💡 想法碎片 (X)
> 原文一句话引用 — *时间*
> ...

## ✅ 待办事项 (X)
- [ ] 待办内容 — 截止/提醒时间（如有） — *时间*
- ...

## 🖼 图片与媒体 (X)
- 图片/语音/视频描述 — *时间*
- ...

## 🗑 其他 (X)
- ...

## 建议下一步
- 待读链接中优先级高的 3 条：...
- 本周应处理的待办：...
- 可以直接删除的：...
```

### 7. 保存（可选）

默认询问：

> 要把这份整理结果保存到 `~/notes/wechat-ft/YYYY-MM-DD.md` 吗？

同意的话：

```bash
mkdir -p ~/notes/wechat-ft
```

用 `write_file` 把分类报告写到 `~/notes/wechat-ft/YYYY-MM-DD.md`。

**同时留原始档**（可选，推荐）：

```bash
wechat-cli export "<step 2 的名字>" \
  --format markdown \
  --output ~/notes/wechat-ft/YYYY-MM-DD-raw.md \
  [--start-time ...] [--end-time ...]
```

这样将来还能追溯原始聊天内容。

## Fallback：粘贴模式（无 wechat-cli 时）

如果用户在不支持的平台（手机、陌生环境），或者装 wechat-cli 失败：

1. 让用户把从微信复制的原文直接粘贴进来
2. **跳过** step 1–4，从 step 5 开始
3. 时间戳只能从粘贴文本里抽取日期字符串，精度降低
4. 媒体只有 `[图片]` / `[语音]` 占位符，保留、不猜内容

## Tips

- **不丢内容**：判断不清一律归"其他"，不删
- **链接保留原始 URL**，不用 `[text](url)` 语法折叠
- **待办保守判断**：模糊语句归"想法"，别把每句话都变成 todo
- **时间戳优先用 JSON 的 `timestamp`**，比从正文里抽日期更可靠
- **超过 500 条**：先给分类计数总览，问用户要不要详细列出
- **Windows 下 `init` 失败** 最常见原因：没用管理员身份打开 PowerShell
- **macOS 不想每次输 sudo 密码**：先跑 `sudo -v` 授权一次，后续命令 5 分钟内不再问

## 常见问题

| 问题 | 解决 |
|------|------|
| `wechat-cli: command not found` | 先装 Node.js，再 `npm install -g @canghe_ai/wechat-cli` |
| `keys not initialized` / `config not found` | 跑对应平台的 `init`：macOS/Linux `sudo wechat-cli init`；Windows 以管理员身份 `wechat-cli init` |
| "文件传输助手"找不到 | 先 `wechat-cli sessions --limit 50 --format json` 列最近会话，找精确 display name |
| 返回消息数量远少于预期 | wechat-cli 读的是**本机**数据库。换设备后的旧记录不在本机，CLI 读不到（微信本身就不完全云同步聊天记录） |
| macOS 报版本不兼容 | 项目要求 `WeChat for Mac ≤ 4.1.8.100`。新版微信改了加密可能不支持；降级或看 wechat-cli 仓库 Issues |
| JSON 解析失败 | 可能 wechat-cli 版本跟 skill 假设的 schema 不同，先 `wechat-cli sessions --format text` 看人类可读版，再 fallback 到粘贴模式 |

## 示例运行

**用户输入**：
> 帮我整理最近三天文件传输助手的东西，存到 notes

**Hermes 执行**：
1. `command -v wechat-cli` → `/usr/local/bin/wechat-cli`
2. `wechat-cli sessions --limit 30 --format json` → 找到 `"name": "文件传输助手"`
3. 今天是 2026-04-22，3 天前 = 2026-04-19
4. `wechat-cli history "文件传输助手" --limit 500 --format json --start-time "2026-04-19"` → 拿到 87 条
5. 分类 → 📎 12 / 📄 3 / 💡 18 / ✅ 4 / 🖼 48 / 🗑 2
6. 生成 markdown 报告
7. 用户说"存到 notes"，直接 `mkdir -p ~/notes/wechat-ft && write_file ~/notes/wechat-ft/2026-04-22.md`
8. 顺带 `wechat-cli export "文件传输助手" --format markdown --output ~/notes/wechat-ft/2026-04-22-raw.md --start-time "2026-04-19"`

**用户输入**：`/wechat-ft-organizer`（无参数）

→ 进入主动询问模式：时间范围？是否保存？是否留原始档？

---

## 如何分享给别人

这个 skill 在 `skills/personal/wechat-ft-organizer/` 下，别人要用：

1. 把这个目录复制到他的 `~/.hermes/skills/wechat-ft-organizer/`，或者他 `git clone` 你的 fork 并把 `external_dirs` 指到 `skills/personal/`
2. 按上面 Prerequisites 装 wechat-cli 并 init
3. 在 Hermes 里 `/wechat-ft-organizer` 直接用
