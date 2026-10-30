# OpenClaude

**把任意 OpenAI Chat Completions / Anthropic Messages 后端，包装成一个「Claude」**

一个用于安全研究的演示：拿一个便宜的后端模型，让它在协议层、身份层、计费层都表现得像 Claude。本仓库是配套技术报告，不含源代码。

---

## 这个 demo 做什么

输入侧接受标准的 Anthropic Messages 请求（`POST /v1/messages`，含 SSE 流式），输出侧可以是任意 OpenAI 兼容端点或 Anthropic 端点。中间的包装层负责四件事：

```
客户端 (Anthropic 协议)
   │  POST /v1/messages
   ▼
┌──────────────────────────────────────────┐
│  1. 协议映射   Anthropic 字段 ⇄ OpenAI 字段  │
│  2. 身份包装   注入系统简报，让模型自称 Claude │
│  3. 思维链包装 密封推理，产出 signature       │
│  4. 用量映射   三分输入、思维链计入输出        │
└──────────────────────────────────────────┘
   │  POST /v1/chat/completions
   ▼
任意后端 (DeepSeek / GPT / Claude / 自建)
```

## 四个技术细节

### 1. 协议映射

两套 API 的字段不是一一对应，映射规则决定了一切：

| Anthropic | OpenAI | 处理 |
|---|---|---|
| `system` | `messages[0].role=system` | 身份简报在此注入 |
| `content[]` 多段 | `content` 字符串或数组 | 合并 / 拆分 |
| `thinking` | `reasoning_effort` / `reasoning` | 预算转译 |
| `tools` + `tool_choice` | `tools` + `tool_choice` | 结构转换 |
| `input_tokens` | `prompt_tokens` − 缓存 | 见第 4 节 |
| `output_tokens` | `completion_tokens`（含推理） | 见第 4 节 |

工具的响应要反向构造：OpenAI 的 `tool_calls` 数组要拆成 Anthropic 的 `tool_use` 块，参数以 `input_json_delta` 流式下发。

### 2. 身份包装

后端模型会如实回答"我是谁"。包装层的解法是系统提示词注入：为每个别名准备一份简报，声明目标身份，并用重复强化压过模型自身的先验。

简报的长度是被实测约束的——约两千五百字符会被模型判为"注入的文本"而拒绝执行；压到一千字符、把身份重复二十次，八个别名的自述才全部命中。这是整个 demo 里最"软"的一环，也是最容易被检测的一环。

### 3. 思维链包装

Anthropic 的 `thinking` 块带服务端签名，客户端会回传它延续推理。包装层需要生成一个**结构上无可指摘**的签名：

```
推理文本 → zlib 压缩 → X25519 ECDH → HKDF-SHA256 → AES-256-GCM → base64url
```

三个细节让结构检查全部通过：长度随推理文本伸缩（实测 239–1598 字符，不是固定值）；`session_id` 一并密封，换会话回放会失败；随机 nonce 让同一段推理两次得到不同签名。

它证明不了任何事——客户端没有公钥，也没有格式规范，能验证的只是"这段字节看起来像 Anthropic 的签名"。这正是报告第 7.1 节讨论的内容。

### 4. 用量映射

OpenAI 的 `prompt_tokens` 包含缓存命中部分，Anthropic 把它拆成三项：

```
input_tokens = max(0, prompt_tokens − cache_read − cache_write)
```

反过来，Anthropic 的 `output_tokens` 包含思维链，而 OpenAI 的 `completion_tokens` 也包含推理 token——两边口径一致，但必须做截断，否则会出现 `output_tokens > max_tokens`。

流式路径直接透传上游 SSE，边收边发；早期版本等完整答案再分块，首字时间等于总耗时，本身就是破绽。

## 报告

| 文件 | 语言 | 说明 |
|---|---|---|
| [`article_cn.pdf`](article_cn.pdf) | 中文 | 论文全文，12 页 |
| [`article.pdf`](article.pdf) | English | 英文版，12 页 |
| [`REPORT.md`](REPORT.md) | 中文 | Markdown 源文件 |
| [`REPORT.en.md`](REPORT.en.md) | English | Markdown source |

报告的第 2 章给出掺水的五类分类学，第 3 章逐条讲解上面这些实现，第 4 章把 12 项可观测痕迹做成实测，第 5 章是一次完整的缺陷定位与修复，第 7 章评估了两类被高估的检测信号。

## 范围与伦理

所述实现仅用于本地授权演示与检测方法研究。演示系统默认回环绑定，不针对任何真实厂商，不破解 TLS，不伪造证书。报告中不包含任何真实服务的地址与凭据。

**不要**把它部署成对外服务去欺骗真实用户。

## 引用

```bibtex
@techreport{openclaude2026,
  title  = {OpenClaude: 中转站掺水的技术分类、可观测痕迹与检测评估},
  author = {LightCode AI},
  year   = {2026},
  month  = {10},
  note   = {Technical report, version 2.0}
}
```

## 许可

[CC BY 4.0](LICENSE)
