+++
date = '2026-09-22T21:00:00+08:00'
draft = false
title = 'Claude Code 与 Codex 接入 Portcloud 网关教程'
description = '把 Claude Code 和 Codex 指向自建的 LiteLLM 网关，统一调度 Workbuddy 上游模型'
+++

Claude Code 和 Codex 默认分别连接 Anthropic 与 OpenAI 官方 API。但两者都支持通过配置指向自建网关：Claude Code 用 `ANTHROPIC_BASE_URL`，Codex 用 `model_providers`。

本文介绍如何把这两个工具都接到 **Portcloud 网关** —— 一个用 LiteLLM Proxy 搭建的聚合层，背后对接 Workbuddy 上游，对外提供统一的 API。

> 本文面向已获得 Portcloud API Key 的用户。如果你还没有 Key，请联系管理员。

---

## 一、为什么需要这一层

直接对接上游供应商，每个工具都要单独适配协议、单独处理限流和故障。LiteLLM 网关把这些收拢成一层：

- **协议统一**：同时提供 Anthropic `/v1/messages` 和 OpenAI `/v1/responses`、`/v1/chat/completions`，客户端不用改代码
- **多上游调度**：同一个模型名背后可以挂多个上游，主上游故障时自动降级
- **统一观测**：所有请求的用量、耗时、错误集中在一处

对 Claude Code 和 Codex 而言，只是换了个接入地址，其余体验不变。

---

## 二、获取 API Key

向管理员申请后你会拿到一个形如 `sk-xxxxxxxx` 的 Key。这个 Key 决定了你能访问哪些模型。

如果请求了不在权限范围内的模型，网关会返回 `403`：

```
key not allowed to access model. This key can only access [...]
```

> **安全提示**：API Key 等同于账号密码。不要提交到 Git 仓库、不要贴进聊天记录、不要写进公开文档。建议通过环境变量或本地配置文件注入。

---

## 三、可用模型

网关当前提供以下模型：

| 模型名 | 上下文 | 最大输出 | 说明 |
|---|---|---|---|
| `deepseek-v4.1-flash` | 1M | 128K | DeepSeek，支持视觉与工具调用 |
| `hy3` | 192K | 64K | 混元 |
| `hy4-preview` | 1M | 64K | 混元预览版 |
| `hy4-preview-f` | 1M | 64K | 混元预览版 |

查询当前完整列表：

```bash
curl https://litellm.portcloud.online/v1/models \
  -H "x-api-key: sk-你的key"
```

> **关于 `[1m]` 后缀**：上下文为 1M 的模型（`deepseek-v4.1-flash`、`hy4-preview`、`hy4-preview-f`），在 Claude Code 里使用时建议在模型名后加上 `[1m]`，例如 `deepseek-v4.1-flash[1m]`。原因见下方 [4.5 节](#45-关于-1m-后缀)。

---

## 四、接入 Claude Code

### 4.1 环境变量（推荐先这样验证）

```bash
export ANTHROPIC_BASE_URL="https://litellm.portcloud.online"
export ANTHROPIC_AUTH_TOKEN="sk-你的key"
export ANTHROPIC_MODEL="deepseek-v4.1-flash[1m]"

claude
```

三个变量的作用：

| 变量 | 说明 |
|---|---|
| `ANTHROPIC_BASE_URL` | 网关地址。Claude Code 会自动拼接 `/v1/messages` |
| `ANTHROPIC_AUTH_TOKEN` | 你的 API Key |
| `ANTHROPIC_MODEL` | 默认使用的模型名 |

### 4.2 写入配置文件（持久化）

在 `~/.claude/settings.json` 中加入 `env` 段：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://litellm.portcloud.online",
    "ANTHROPIC_AUTH_TOKEN": "sk-你的key",
    "ANTHROPIC_MODEL": "deepseek-v4.1-flash[1m]"
  }
}
```

这样每次启动 `claude` 都会自动生效，不用重复 export。

> 如果该文件已存在，只需把 `env` 段合并进去，不要整个覆盖。

完整的配置文件还可以包含 `modelPicker`，让 `/model` 选择器列出网关的模型，见 [4.6 节](#46-自定义-model-选择器)。

### 4.3 项目级配置

只想在某个项目里用网关，可以在项目根目录建 `.claude/settings.json`，内容同上。项目级配置优先级高于用户级。

> 注意 `modelPicker` 例外——它只认用户级配置，写在项目里不生效。

### 4.4 验证

先做一次最小验证：

```bash
curl https://litellm.portcloud.online/v1/messages \
  -H "x-api-key: sk-你的key" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "deepseek-v4.1-flash[1m]",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "回复：OK"}]
  }'
```

正常会返回标准的 Anthropic 格式：

```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "model": "deepseek-v4.1-flash",
  "content": [{"type": "text", "text": "OK"}],
  "stop_reason": "end_turn"
}
```

注意返回体里的 `model` 字段是**不带后缀**的 `deepseek-v4.1-flash` —— 网关会自动归一化，后缀只是给客户端看的。

然后启动 Claude Code 随便问一句，能正常对话就说明接好了。

### 4.5 关于 `[1m]` 后缀

上下文为 **1M** 的模型，在 Claude Code 里建议在模型名后加 `[1m]` 后缀：

```
deepseek-v4.1-flash[1m]
hy4-preview[1m]
hy4-preview-f[1m]
```

**为什么要加**：Claude Code 这类 Agent 需要知道模型的上下文窗口大小，才能决定何时压缩历史、何时截断。它靠内置的模型表来查这个值，而表里只有官方模型。

网关提供的这些模型名不在表内，Claude Code 查不到就会**退回到一个保守的默认值**，可能把上下文窗口估小。结果是：明明模型支持 1M，Agent 却按更小的窗口去压缩对话，长上下文能力用不满。

加上 `[1m]` 后缀等于直接告诉 Agent「这个模型的窗口是 1M」，避免它误判。

**哪些模型需要加**：

| 模型 | 上下文 | 是否加 `[1m]` |
|---|---|---|
| `deepseek-v4.1-flash` | 1M | ✅ 建议加 |
| `hy4-preview` | 1M | ✅ 建议加 |
| `hy4-preview-f` | 1M | ✅ 建议加 |
| `hy3` | 192K | ❌ 不需要 |

`hy3` 的上下文不是 1M，加后缀反而会传递错误信息，保持原样即可。

**注意事项**：

- 后缀只影响客户端对上下文的判断，**不影响实际请求**。网关会自动去掉后缀再转发给上游
- 网关对带后缀和不带后缀的模型名都接受，所以不加也能用，只是可能损失长上下文能力
- 这个后缀是 Claude Code 侧的约定，**Codex 不适用**（Codex 的模型配置方式不同，见第五章）

### 4.6 自定义 `/model` 选择器

Claude Code 的 `/model` 命令默认只列出内置的 Claude 模型（Fable、Opus、Sonnet、Haiku）。这些模型在你的网关上并不存在，列在那里只会造成干扰。

用 `modelPicker` 可以把选择器换成网关实际提供的模型。

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://litellm.portcloud.online",
    "ANTHROPIC_AUTH_TOKEN": "sk-你的key",
    "ANTHROPIC_MODEL": "deepseek-v4.1-flash[1m]"
  },
  "modelPicker": {
    "replaceBuiltInOptions": true,
    "options": [
      { "model": "deepseek-v4.1-flash[1m]", "label": "DeepSeek V4.1 Flash", "description": "1M context · vision & tool calls" },
      { "model": "hy4-preview[1m]", "label": "Hy 4 Preview", "description": "1M context" },
      { "model": "hy4-preview-f[1m]", "label": "Hy 4 Preview F", "description": "1M context" },
      { "model": "hy3", "label": "Hy 3", "description": "192K context" }
    ]
  }
}
```

#### 关键字段

| 字段 | 说明 |
|---|---|
| `replaceBuiltInOptions` | 设为 `true` 时，选择器**只显示 Default 行 + 你列出的模型**，内置的 Fable/Opus/Sonnet/Haiku 全部隐藏。不设或设为 `false` 时，你的模型会**追加**在内置列表之后 |
| `options` | 有序列表，顺序即选择器中的显示顺序 |
| `options[].model` | 模型 ID，**要和网关暴露的名字一致**（1M 模型记得带 `[1m]` 后缀） |
| `options[].label` | 显示名称。省略时显示模型 ID |
| `options[].description` | 副标题说明。省略时显示 `Custom model (<model-id>)` |
| `options[].behavesAs` | 可选。把模型映射到某个已知模型族，让 Claude Code 了解它的能力特征 |

#### 注意事项

- **`replaceBuiltInOptions` 是替换，不是追加**。开启后内置模型和网关自动发现的模型都会被隐藏，所以要把想用的模型全部列进 `options`
- **只认用户级和 managed 配置**。写在项目的 `.claude/settings.json` 里不生效，必须放在 `~/.claude/settings.json`
- **不与其他来源合并**。多个地方定义 `modelPicker` 时，取优先级最高的那个，不会把列表拼起来
- 修改后需要**重启 Claude Code** 才会生效

#### 验证

重启后敲 `/model`，应该能看到 Default 行加上你配置的模型，且没有内置的 Claude 模型。如果选择器里的模型名点进去报错，多半是 `model` 字段与网关实际名字不一致，用下面的命令核对：

```bash
curl https://litellm.portcloud.online/v1/models \
  -H "x-api-key: sk-你的key"
```

### 4.7 子 Agent 报 400 的修复

**现象**：主对话一切正常，但一启动子 Agent（Agent 工具）就失败：

```
Agent "Inspect sidebar visual system" failed: Agent terminated early due to an API error:
API Error: 400 anthropic_messages: Invalid model name passed in model=claude-opus-5-5.
Call `/v1/models` to view available models for your key.
```

报错里的模型名有时是 `claude-opus-5-5`，有时是 `claude-haiku-4-5-20251001` 或 `claude-sonnet-5`。

**原因**：`/model` 选择器只决定**主线程**用哪个模型。子 Agent 走的是另一条路径 —— 它按**档位**取模型，而档位对应的是 Anthropic 官方模型名：

| 触发路径 | 发出的模型名 |
|---|---|
| 子 Agent 指定 `model: opus` | `claude-opus-5-5` |
| 子 Agent 指定 `model: sonnet` | `claude-sonnet-5` |
| 子 Agent 指定 `model: haiku`（Explore 等只读 Agent 的默认档） | `claude-haiku-4-5-20251001` |

这些名字在你的网关上并不存在，会被 LiteLLM 的请求闸门直接拦下并返回 400 —— 请求根本到不了上游。

**修复**：在 `~/.claude/settings.json` 的 `env` 里补四个变量，把三个档位都指向网关的模型：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://litellm.portcloud.online",
    "ANTHROPIC_AUTH_TOKEN": "sk-你的key",
    "ANTHROPIC_MODEL": "deepseek-v4.1-flash[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4.1-flash[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4.1-flash[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4.1-flash[1m]",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4.1-flash[1m]"
  }
}
```

| 变量 | 作用 |
|---|---|
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | 把 opus 档重映射到网关模型 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | 把 sonnet 档重映射到网关模型 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 把 haiku 档重映射到网关模型 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 兜底开关，所有子 Agent 统一用这个模型 |

四个都设最稳妥。`CLAUDE_CODE_SUBAGENT_MODEL` 单独设就能让子 Agent 跑起来；三个档位变量则保证即使显式写了 `model: opus` 这类档位也不会再报错。

映射目标按需选择：想让所有档位都用同一个模型，就都填 `deepseek-v4.1-flash[1m]`；也可以把 haiku 档（后台小任务、只读 Explore）指到更便宜的模型以省额度。

> **关于 `[1m]` 后缀**：档位重映射的目标模型同样建议带上后缀（1M 模型），理由与 [4.5 节](#45-关于-1m-后缀)相同。实测网关对带与不带后缀都接受，会先剥掉后缀再转发。

**验证**：改完重启 Claude Code，让子 Agent 干点活：

```
用 Agent 工具，subagent_type 选 Explore，model 选 haiku，列出 /tmp 下的文件
```

能正常返回就说明修好了。也可以去网关侧确认不再有被拒的请求：

```bash
docker logs litellm 2>&1 | grep "Invalid model name"
```

没有输出即为正常。

> 想让网关一次修好**所有**客户端（包括其他机器、其他工具），也可以在 LiteLLM 配置的 `router_settings` 下加 `model_group_alias`，把这些 Claude 档位名直接映射到网关模型。本文只介绍客户端侧的改法。

---

## 五、接入 Codex

Codex 走的是 OpenAI 的 **Responses API**，配置方式与 Claude Code 不同，需要在 `config.toml` 里声明一个自定义 provider。

### 5.1 修改 config.toml

编辑 `~/.codex/config.toml`：

```toml
model = "deepseek-v4.1-flash"
model_provider = "portcloud"
model_reasoning_effort = "medium"

[model_providers.portcloud]
name = "Portcloud"
base_url = "https://litellm.portcloud.online/v1"
env_key = "PORTCLOUD_API_KEY"
wire_api = "responses"
```

各项含义：

| 配置项 | 说明 |
|---|---|
| `model` | 默认模型名，取值见上方模型列表 |
| `model_provider` | 指向下面定义的 provider 名称 |
| `model_reasoning_effort` | 推理强度：`low` / `medium` / `high` |
| `base_url` | 网关地址，**末尾要带 `/v1`** |
| `env_key` | 存放 API Key 的**环境变量名**（不是 Key 本身） |
| `wire_api` | 固定为 `responses`，Codex 依赖 Responses API |

> **关键点**：`env_key` 填的是环境变量**名字**，Codex 会去读这个环境变量的值。Key 本身不写进配置文件，避免误提交。

### 5.2 设置环境变量

把 Key 写进 shell 配置（`~/.zshrc` 或 `~/.bashrc`）：

```bash
export PORTCLOUD_API_KEY="sk-你的key"
```

然后 `source ~/.zshrc` 或重开终端。

### 5.3 验证

```bash
codex exec --skip-git-repo-check "回复：OK"
```

正常输出类似：

```
OpenAI Codex v0.155.1
--------
workdir: /your/path
model: deepseek-v4.1-flash
provider: portcloud
--------
codex
OK
```

也可以直接启动交互模式：

```bash
codex
```

### 5.4 换模型

改 `config.toml` 里的 `model` 即可，也可以在交互模式里临时切换：

```bash
codex -m hy3
```

---

## 六、注意事项

### 模型元数据警告

首次用 Codex 连接时，可能会看到这样的警告：

```
warning: Model metadata for `deepseek-v4.1-flash` not found.
Defaulting to fallback metadata; this can degrade performance and cause issues.
```

这是因为 Codex 内置的模型表里没有这些模型名。**不影响正常使用**，忽略即可。

### `unrecognized_model` 日志

Claude Code 启动时可能打印一行：

```
[claude-code:unrecognized_model] {"model":"deepseek-v4.1-flash[1m]","query_source":"sdk"}
```

这只是 Claude Code 在说「这个模型名不在我的内置模型表里」，属于提示性质。**它不会拦截请求、也不会改写模型名**，网关侧对应请求仍是 200。使用自建网关时必然出现，忽略即可。

### 子 Agent 与档位重映射

子 Agent 不跟随 `/model` 选择器，而是按档位（opus / sonnet / haiku）取模型，发的是 Anthropic 官方模型名。不重映射就会 400，详见 [4.7 节](#47-子-agent-报-400-的修复)。

### 推理模型的 token 消耗

部分模型会在正式回答前产生大量 reasoning token。如果 `max_tokens` 给得太小，可能全部被推理消耗掉，导致返回内容为空。用 curl 测试时建议 `max_tokens` 不低于 500。

Claude Code 和 Codex 会自动管理 token 预算，正常使用不受影响。

### 429 与冷却

网关对失败的上游会做短时冷却（默认 60 秒）。如果遇到：

```
No deployments available for selected model, Try again in 60s
```

说明该模型的所有上游都在冷却中，等一会儿再试即可。多上游的模型会自动降级到备用上游，一般感知不到。

### Codex 桌面版

Codex 桌面版读取同一个 `~/.codex/config.toml`，配置方式一致。

但要注意：**macOS 上从 Finder 或 Dock 启动的应用不继承 shell 环境变量**，可能导致 `env_key` 读不到。如果 CLI 能用而桌面版报鉴权失败，可以这样处理：

```bash
launchctl setenv PORTCLOUD_API_KEY "sk-你的key"
```

设置后重启桌面版即可。

---

## 七、其他客户端

网关同时兼容 OpenAI 和 Anthropic 两套协议，三类客户端都能接。

### 7.1 OpenAI 兼容客户端

适用于 Chatbox、NextChat、Cherry Studio、Open WebUI 等，以及任何使用 OpenAI SDK 的程序：

```bash
curl https://litellm.portcloud.online/v1/chat/completions \
  -H "Authorization: Bearer sk-你的key" \
  -H "content-type: application/json" \
  -d '{
    "model": "hy3",
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

如果用 OpenAI SDK，只需替换 `base_url`：

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://litellm.portcloud.online/v1",
    api_key="sk-你的key",
)

resp = client.chat.completions.create(
    model="deepseek-v4.1-flash",
    messages=[{"role": "user", "content": "你好"}],
)
print(resp.choices[0].message.content)
```

### 7.2 Anthropic 兼容客户端

适用于 Claude Code、Anthropic SDK，以及其他使用 Messages API 的程序：

```bash
curl https://litellm.portcloud.online/v1/messages \
  -H "x-api-key: sk-你的key" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "deepseek-v4.1-flash",
    "max_tokens": 200,
    "system": "你是一个简洁的助手。",
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

用 Anthropic SDK：

```python
from anthropic import Anthropic

client = Anthropic(
    base_url="https://litellm.portcloud.online",
    api_key="sk-你的key",
)

resp = client.messages.create(
    model="deepseek-v4.1-flash",
    max_tokens=200,
    messages=[{"role": "user", "content": "你好"}],
)
print(resp.content[0].text)
```

Anthropic 协议的特性都已支持，包括 `system` 参数、多轮对话、工具调用（`tools`）、流式输出（`stream: true`）。

另有一个辅助接口用于估算 token 数，Claude Code 等客户端会用到：

```bash
curl https://litellm.portcloud.online/v1/messages/count_tokens \
  -H "x-api-key: sk-你的key" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model": "hy3", "messages": [{"role": "user", "content": "你好"}]}'
```

返回：

```json
{"input_tokens": 8}
```

### 7.3 端点对照

| 端点 | 协议 | 适用 |
|---|---|---|
| `/v1/messages` | Anthropic Messages | Claude Code、Anthropic SDK |
| `/v1/messages/count_tokens` | Anthropic Messages | token 估算 |
| `/v1/responses` | OpenAI Responses | Codex、新式 OpenAI SDK |
| `/v1/chat/completions` | OpenAI Chat | 大多数第三方客户端 |

认证头 `x-api-key` 和 `Authorization: Bearer` 都支持，两套协议通用。

---

## 八、管理台

网关自带 Web 管理台，可以查看用量、查看模型：

```
https://litellm.portcloud.online/ui
```

登录凭据由管理员提供。

---

## 小结

两个工具的核心配置：

**Claude Code** —— 环境变量（1M 模型记得加 `[1m]` 后缀，子 Agent 需要重映射档位）：

```
ANTHROPIC_BASE_URL=https://litellm.portcloud.online
ANTHROPIC_AUTH_TOKEN=sk-你的key
ANTHROPIC_MODEL=deepseek-v4.1-flash[1m]
ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4.1-flash[1m]
ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4.1-flash[1m]
ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4.1-flash[1m]
CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4.1-flash[1m]
```

**Codex** —— `~/.codex/config.toml` 里声明 provider，Key 走环境变量：

```toml
model = "deepseek-v4.1-flash"
model_provider = "portcloud"

[model_providers.portcloud]
name = "Portcloud"
base_url = "https://litellm.portcloud.online/v1"
env_key = "PORTCLOUD_API_KEY"
wire_api = "responses"
```

区别在于 Claude Code 走 Anthropic Messages 协议，Codex 走 OpenAI Responses 协议。

网关同时兼容这两套协议，所以无论是 OpenAI 系还是 Anthropic 系的客户端，都能接进来：

| 协议 | 端点 | 典型客户端 |
|---|---|---|
| Anthropic Messages | `/v1/messages` | Claude Code、Anthropic SDK |
| OpenAI Responses | `/v1/responses` | Codex、新式 OpenAI SDK |
| OpenAI Chat | `/v1/chat/completions` | Chatbox、Cherry Studio 等 |

接入后的体验与直连官方 API 无异。
