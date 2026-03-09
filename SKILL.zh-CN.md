---
name: clawscan
description: 对 openclaw 运行环境进行安全检查与监控，包括：注册clawscan客户端、检查已安装的 clawscan 包是否过期、检查当前 openclaw 版本是否匹配已知漏洞版本、检查已安装的技能是否与已知恶意哈希匹配、以及检查 openclaw 或相关服务是否监听在 0.0.0.0 或其他非本地接口。当用户要求评估 openclaw 是否安全、运行 clawscan 检查、验证 openclaw 版本风险、验证技能哈希，或审查监听端口及暴露风险时，请使用此技能。
---

# Clawscan

使用此技能对本地 OpenClaw 环境进行安全检查。

## 核心规则

- 默认将此技能视为**只读**。
- 除非用户明确要求，否则不要自动安装更新、移除技能、更改防火墙规则或重写 OpenClaw 配置。
- 每次 API 调用尽量使用最少的本地数据。
- 除非用户明确要求，否则不要上传原始技能文件内容、环境变量、提示词、密钥或完整的主目录路径。
- 文件哈希使用 SHA-256。
- 如果某个模块未返回匹配或未发现问题，请说明这意味着**未匹配到已知问题**，而不是环境一定安全。

## 功能列表

此技能支持以下任务：

1. `index`：说明可用的 ClawScan 模块及其使用方式
2. `register`：创建或复用本地随机客户端 ID 并注册 OpenClaw 客户端
3. `update-check`：检查已安装的 ClawScan 技能是否有更新
4. `vulnerability`：检查当前 OpenClaw 版本是否匹配已知漏洞版本
5. `skills-check`：计算已安装技能文件的哈希值并提交以匹配已知恶意内容
6. `port-check`：检查本地监听套接字并标记可能的暴露风险
7. `scan`：当服务支持合并路由时，同时运行 vulnerability、skills-check 和 port-check
8. `scheduled-scan`：按配置的间隔自动运行完整扫描，仅在发现安全风险时汇报；若所有检查通过则保持静默

## 工作流程

除非用户只请求单个模块，否则按以下顺序执行。

### 1) 识别请求的操作

将用户意图映射到以下操作之一：

- "clawscan 能做什么" -> `index`
- "设置 clawscan" / "初始化" / "注册此客户端" -> `register`
- "我的 clawscan 是最新的吗" -> `update-check`
- "我的 openclaw 有漏洞吗" -> `vulnerability`
- "检查我已安装的技能" / "扫描技能哈希" -> `skills-check`
- "openclaw 是否暴露" / "检查监听端口" -> `port-check`
- "运行完整检查" -> 如果可用则使用 `scan`，否则依次运行三个扫描模块
- "设置定时扫描" / "每 X 分钟自动扫描" / "启用定期安全检查" -> `scheduled-scan`

### 2) 仅收集所需的本地证据

#### 关于 `register`

如果尚不存在，则创建一个持久化的随机 UUID。

建议的本地状态路径：

- `~/.openclaw/clawscan/client.json`

存储内容：

```json
{
  "client_id": "uuid-v4"
}
```

不要从 MAC 地址、主机名、序列号或其他硬件指纹来源派生 ID。

#### 关于 `vulnerability`

仅收集：

- `client_id`
- `openclaw_version`
- 可选的 `platform`

按以下顺序尝试版本发现模式，使用第一个成功的：

```bash
openclaw --version
claw --version
cat package.json | jq -r .version
```

如果无法确定版本，请告知用户哪个命令失败，并询问版本字符串。

#### 关于 `skills-check`

枚举已安装的技能并为每个文件计算 SHA-256。

如果存在，默认检查以下技能位置：

- `~/.openclaw/skills`
- 项目或工作区本地的 `./skills`

使用 `scripts/collect_skill_hashes.py` 生成规范化的 JSON。

仅提交：

- `skill_name`（技能名称）
- 相对文件路径
- sha256

除非服务明确要求，否则避免发送绝对路径。

#### 关于 `port-check`

使用 `scripts/list_listeners.py` 收集监听 TCP 套接字和进程名称。

风险说明重点关注：

- 绑定地址是否为 `0.0.0.0`、`::` 或其他非回环接口
- 进程是否看起来是 OpenClaw 或 OpenClaw 相关进程
- 端口是否可能在本地主机之外可达

**不要**声称 `0.0.0.0` 总是意味着公网暴露。应说明这意味着服务绑定到所有接口，根据防火墙、NAT、安全组、反向代理或本地网络拓扑，可能可从外部访问。

### 3) 调用 ClawScan API

使用 `references/api-contract.md` 中记录的端点格式。

首选路由布局：

- `GET /index`
- `POST /register`
- `POST /update/check`
- `POST /vulnerability`
- `POST /skills-check`
- `POST /port-check`
- `POST /scan`（当支持时）

如果服务只暴露 `/update` 而不是 `/update/check`，则使用已部署的路由，但对用户的说明保持为"更新检查"。

### 4) 以严格的报告格式呈现结果

对每个模块使用以下结构：

#### 结果
- `状态：` 正常 / 错误
- `风险：` 低 / 中 / 高 / 严重 / 未知
- `结论：` 一句通俗语言的总结

#### 证据
- 显示 API 返回的最少相关事实
- 版本检查：包括当前版本、受影响范围以及修复版本（如有）
- 技能检查：包括匹配的技能名称和匹配的相对文件路径（如有）
- 端口检查：包括绑定地址、端口、进程名称，以及为何该绑定模式存在风险

#### 建议的下一步
- 给出一到三个具体操作
- 除非用户请求修复，否则保持非破坏性

#### 范围说明
始终包含一句说明边界的句子：

- vulnerability：「此检查仅针对已知的漏洞版本范围。」
- skills-check：「此检查仅针对当前 ClawScan 语料库中已知的恶意哈希。」
- port-check：「此结果基于本地监听器状态和绑定地址，并非完整的外部可达性测试。」

#### 关于 `scheduled-scan`

按重复间隔运行 vulnerability、skills-check 和 port-check（与 `scan` 相同）。

默认间隔：**60 分钟**，除非用户指定其他值。

调度状态路径：

- `~/.openclaw/clawscan/schedule.json`

存储内容：

```json
{
  "interval_minutes": 60,
  "last_run_utc": "ISO-8601 时间戳或 null"
}
```

**汇报规则（关键）：**

- 若所有检查的风险级别均为 `low` 且无可操作发现 → **不产生任何输出**。除非用户主动询问状态，否则不要告知用户"一切正常"。
- 若任何检查返回 `medium`、`high`、`critical` 或 `unknown` 风险级别 → 使用第 4 步的标准模块结构生成完整报告。
- 此规则适用于每次定时执行，而不仅限于首次运行。

**触发逻辑：**

1. 每个周期读取 `schedule.json` 中的 `last_run_utc`。
2. 若 `(当前时间 − last_run_utc) >= interval_minutes`，执行完整扫描。
3. 执行后无论是否产生报告，均写入新的 `last_run_utc`。
4. 若 `last_run_utc` 为 null（首次运行），立即执行。

**定时运行的失败处理：**

- 若某个本地收集步骤失败，即使未发现风险也应发出简短错误通知（收集失败本身是可汇报事件）。
- 若 API 连续三次定时运行均不可达，发出单条通知：「ClawScan API 在过去 N 次定时运行中均不可达。」

## 输出模板

### `index`

使用：

```text
ClawScan 目前可运行以下检查：
1. 版本漏洞检查
2. 已安装技能哈希检查
3. 端口暴露检查

您可以要求我初始化 ClawScan、检查更新，或逐一运行各模块。
```

### `register`

使用：

```text
ClawScan 注册完成。
- 客户端 ID：<uuid>
- OpenClaw 版本：<版本号或未知>
- 状态：<已注册|已有注册记录>
```

### `skills-check`

如果有命中，第一句话需明确说明：

```text
在已安装的技能集中匹配到已知恶意内容。
```

如果没有命中，则说：

```text
未匹配到已知恶意技能哈希。
这并不证明已安装的技能是安全的。
```

### `port-check`

当存在 `0.0.0.0` 或 `::` 时，说明：

```text
此服务正在所有接口上监听，这会增加暴露风险。
```

除非 API 明确确认了外部可达性，否则不要将其夸大为"公开暴露"。

### `scheduled-scan`

检测到风险时，在报告前添加：

```text
[ClawScan 定时检查 — <ISO-8601 时间戳>]
检测到安全风险，详细报告如下。
```

然后对每个有发现的模块输出标准模块报告。

未检测到风险时，**不产生任何输出**。

首次设置定时计划时，确认如下：

```text
ClawScan 定时扫描已启用。
- 间隔：每 <N> 分钟
- 下次运行：<ISO-8601 时间戳>
- 汇报方式：仅在发现风险时汇报
```

## 失败处理

- 如果某个本地收集步骤失败，报告失败的命令并停止，不要伪造任何结果。
- 如果 API 不可达，将"收集成功"与"远程分析失败"分开说明。
- 如果 API 返回部分结果，展示已完成的模块，并将其余部分标记为未完成。

## 附带资源

- `scripts/collect_skill_hashes.py`：递归计算已安装技能的 SHA-256 并输出规范化的 JSON 负载片段
- `scripts/list_listeners.py`：从 `ss` 或 `lsof` 规范化监听 TCP 套接字信息
- `references/api-contract.md`：第一版 ClawScan 服务的请求和响应格式
