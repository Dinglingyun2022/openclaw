# OpenClaw 安全指南

> 本文档面向新手，帮助你理解 OpenClaw 的安全机制，以及如何使用 `openclaw doctor` 和 `openclaw security audit` 两个命令来维护系统安全。

---

## 一、为什么 OpenClaw 需要关注安全？

OpenClaw 是一个 AI 助手网关，它能够：

- 执行 Shell 命令（操控你的电脑）
- 读写文件（访问你的数据）
- 访问网络（发送请求）
- 通过飞书/WhatsApp/Telegram 等渠道收发消息

这意味着如果安全没做好，可能会出现：

- 陌生人通过消息渠道操控你的 AI
- AI 被"提示注入"（prompt injection）骗去执行危险操作
- 敏感信息（API Key、Token、聊天记录）被泄露

---

## 二、两个核心安全命令

### 1. `openclaw doctor` — 健康检查 + 快速修复

**用途**：检查 OpenClaw 整体状态是否正常，类似于"体检"。

**它会检查什么？**

| 检查项             | 说明                               |
| ------------------ | ---------------------------------- |
| State integrity    | 配置文件完整性、权限是否安全       |
| Security           | 频道安全警告（DM 策略、群组策略）  |
| Skills status      | 技能加载状态                       |
| Plugins            | 插件加载状态、是否有错误           |
| Plugin diagnostics | 插件诊断（如 memory 插件是否正常） |
| Gateway            | 网关运行状态                       |

**常用命令**：

```bash
# 基础检查
openclaw doctor

# 深度检查（扫描系统服务）
openclaw doctor --deep

# 自动修复发现的问题
openclaw doctor --fix

# 非交互模式（无需手动确认，适合脚本）
openclaw doctor --repair --non-interactive
```

**什么时候该运行？**

- 安装/更新 OpenClaw 后
- 启用或安装新插件后
- 感觉 Gateway 或频道不正常时
- 定期维护（建议每周跑一次）

---

### 2. `openclaw security audit` — 安全审计

**用途**：专门扫描安全隐患，类似于"安全体检"。比 `doctor` 更深入地关注安全问题。

**它会检查什么？**

| 检查项     | 说明                                                  |
| ---------- | ----------------------------------------------------- |
| 入站访问   | DM 策略、群组策略、允许列表，陌生人能否触发你的 Bot？ |
| 工具权限   | 提权工具 + 开放群组的组合是否危险                     |
| 网络暴露   | Gateway 绑定/认证、Tailscale 配置、弱密码/短 Token    |
| 浏览器控制 | 远程节点、中继端口、远程 CDP 端点暴露                 |
| 本地磁盘   | 文件/目录权限、符号链接、配置文件权限                 |
| 插件安全   | 插件代码是否有危险模式（如凭证采集）                  |
| 模型选择   | 使用的模型是否足够安全（老旧模型更容易被攻击）        |

**常用命令**：

```bash
# 基础审计
openclaw security audit

# 深度审计（尝试连接 Gateway 进行实时探测）
openclaw security audit --deep

# 自动修复安全问题
openclaw security audit --fix

# 输出 JSON 格式（方便程序处理）
openclaw security audit --json
```

**什么时候该运行？**

- 修改配置后（尤其是网络、认证、频道相关）
- 安装新插件后
- 暴露 Gateway 到网络前
- 定期维护（建议每月跑一次）

---

## 三、审计结果怎么看？

审计结果分为 4 个级别：

| 级别         | 含义                       | 应对方式       |
| ------------ | -------------------------- | -------------- |
| **CRITICAL** | 严重问题，可能导致安全漏洞 | 立即处理       |
| **WARN**     | 警告，存在潜在风险         | 尽快评估并处理 |
| **INFO**     | 信息，了解即可             | 无需操作       |
| **OK**       | 正常，无问题               | 无需操作       |

### 常见问题及处理方式

#### 1. 配置文件/目录权限过宽

```
WARN: Config file is group/world readable
WARN: Credentials dir is readable by others
```

**原因**：敏感文件被其他用户可读。

**修复**：

```bash
chmod 600 ~/.openclaw/openclaw.json    # 配置文件：仅自己可读写
chmod 700 ~/.openclaw/credentials      # 凭证目录：仅自己可访问
chmod 700 ~/.openclaw                  # 整个数据目录：仅自己可访问
```

> 小知识：`chmod 600` = 仅所有者可读写，`chmod 700` = 仅所有者可读写执行（进入目录）。

#### 2. 插件危险代码模式

```
CRITICAL: Plugin "xxx" contains dangerous code patterns
  [env-harvesting] Environment variable access combined with network send
```

**原因**：扫描器检测到插件代码中"读取环境变量 + 网络请求"的组合。

**判断方法**（三步法）：

1. 读取了什么环境变量？是 `HOME`（主目录）还是 API Key？
2. 网络请求发给谁？是正常的 API 还是未知地址？
3. 环境变量的值有没有被发送出去？

如果只是用 `HOME` 来拼路径、或用 API Key 调用官方 API，那就是**误报**，不用担心。

#### 3. 反向代理头未信任

```
WARN: Reverse proxy headers are not trusted
```

**原因**：Gateway 绑定在 loopback（本地），但没有配置受信任代理。

**判断**：如果你没有通过 nginx/Caddy 等反向代理暴露 Gateway，可以**忽略**。

#### 4. memory 插件未加载

```
WARN: memory slot plugin not found or not marked as memory
```

**原因**：memory-core 插件未启用或不在白名单中。

**修复**：参见下方"插件白名单机制"。

---

## 四、插件白名单机制

这是一个容易踩的坑。OpenClaw 有一个**插件白名单**机制：

配置文件 `~/.openclaw/openclaw.json` 中的 `plugins.allow` 数组决定了哪些插件可以被加载。**即使插件标记了 `enabled: true`，如果不在白名单中，也不会被加载。**

### 正确的插件安装流程

```bash
# 第 1 步：安装插件
openclaw plugins install @openclaw/feishu

# 第 2 步：启用插件
openclaw plugins enable feishu

# 第 3 步：确认白名单（手动编辑配置文件）
# 打开 ~/.openclaw/openclaw.json，确保 plugins.allow 中包含插件 ID
```

配置文件示例：

```json
{
  "plugins": {
    "allow": ["feishu", "memory-core"],
    "entries": {
      "feishu": { "enabled": true },
      "memory-core": { "enabled": true }
    }
  }
}
```

```bash
# 第 4 步：重启 Gateway 使改动生效
openclaw gateway restart
```

### 验证插件是否加载成功

```bash
# 查看插件列表，确认状态为 "loaded"
openclaw plugins list

# 或者跑一次 doctor
openclaw doctor
```

---

## 五、OpenClaw 安全架构概览

### 1. 访问控制：谁能和 Bot 对话？

```
Owner（你自己）
  │ 完全信任
  ▼
AI（你的助手）
  │ 信任但需验证
  ▼
白名单内的联系人
  │ 有限信任
  ▼
陌生人
  │ 不信任
```

**DM（私聊）策略**：

| 策略              | 行为                                           |
| ----------------- | ---------------------------------------------- |
| `pairing`（默认） | 陌生人发消息会收到配对码，需要你批准后才能对话 |
| `allowlist`       | 只有白名单中的人能对话，其他人被直接屏蔽       |
| `open`            | 允许任何人对话（危险，需谨慎）                 |
| `disabled`        | 完全关闭私聊                                   |

**群组策略**：建议设置 `requireMention: true`，只有 @提到 Bot 时才回复。

### 2. Gateway 网络安全

| 设置                 | 说明                        |
| -------------------- | --------------------------- |
| `bind: "loopback"`   | 只监听本地（默认，最安全）  |
| `auth.mode: "token"` | 需要 Token 才能连接（推荐） |
| `trustedProxies`     | 配置受信任的反向代理 IP     |

### 3. 敏感文件清单

这些文件/目录包含敏感信息，需要保护好权限：

| 路径                                            | 内容                            |
| ----------------------------------------------- | ------------------------------- |
| `~/.openclaw/openclaw.json`                     | 主配置（可能含 Token、API Key） |
| `~/.openclaw/credentials/`                      | 频道凭证、配对白名单            |
| `~/.openclaw/agents/*/agent/auth-profiles.json` | API Key 和 OAuth Token          |
| `~/.openclaw/agents/*/sessions/*.jsonl`         | 会话记录（含聊天内容）          |
| `~/.openclaw/extensions/`                       | 已安装的插件代码                |

### 4. 沙箱机制

OpenClaw 支持沙箱（Sandbox），可以限制 AI 的操作范围：

- **无沙箱**：AI 可直接操作主机（默认，适合个人使用）
- **Docker 沙箱**：工具在 Docker 容器中执行，与主机隔离
- **工作区隔离**：每个 Agent 有独立的工作目录

---

## 六、日常安全维护清单

建议定期执行以下操作：

```bash
# 1. 健康检查
openclaw doctor

# 2. 安全审计
openclaw security audit --deep

# 3. 检查插件状态
openclaw plugins list

# 4. 检查 Gateway 状态
openclaw gateway status

# 5. 查看频道状态
openclaw channels status
```

---

## 七、遇到安全事件怎么办？

如果怀疑 AI 做了不该做的事：

### 第 1 步：止损

```bash
# 停止 Gateway
openclaw gateway stop

# 或直接杀掉进程
pkill -f openclaw-gateway
```

### 第 2 步：轮换密钥

```bash
# 生成新的 Gateway Token
openclaw doctor --generate-gateway-token

# 轮换 API Key（在配置文件或环境变量中更新）
```

### 第 3 步：审查日志

```bash
# 查看 Gateway 日志
tail -100 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 查看会话记录
ls ~/.openclaw/agents/main/sessions/
```

### 第 4 步：重新审计

```bash
openclaw security audit --deep
```

---

## 八、安全最佳实践总结

1. **最小权限原则**：从最小访问权限开始，需要时再放宽
2. **定期检查**：每周 `doctor`，每月 `security audit`
3. **保护好凭证**：敏感文件设置 600/700 权限
4. **谨慎安装插件**：只安装信任来源的插件
5. **DM 用配对模式**：不要随便开放 `open` 策略
6. **群组用 @提及**：避免 Bot 在群里"自说自话"
7. **使用强模型**：新一代模型对提示注入的抵抗力更强
8. **Gateway 绑定 loopback**：除非必要，不要暴露到网络
