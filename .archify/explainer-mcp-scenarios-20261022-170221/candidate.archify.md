# 按场景看新旧 MCP {#mcp-scenarios}

按场景看这两台服务，成功和失败分开。成功路径得到文本 `Selected unit: C.`。协议失败停在会话、版本或能力。用户拒绝时，结果带 `isError`。

- 旧成功先拿到 `Mcp-Session-Id`，提问留在未结束的 SSE。
- 新成功没有会话。当次结果是 `input_required`，再用新 id 重投。
- 旧失败是没有会话，或会话已经 `DELETE`。
- 新失败是 `-32022`、`405` 和 `-32021`。

@answer scope="本地 HTTP。成功与协议失败已核对。旧的用户拒绝只看了代码。"
@meta locale=zh-CN
@source v1form title="v1 form elicitation 示例" url=https://github.com/modelcontextprotocol/typescript-sdk/blob/v1.x/src/examples/server/elicitationFormExample.ts
@source v2input title="v2 input_required 文档" url=https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/servers/input-required.md
@source v2express title="v2 Express 挂载" url=https://ts.sdk.modelcontextprotocol.io/v2/serving/express

## 有哪些场景？ {#map}

同一件 `choose-units`，先看结果属于哪一类，再看协议。

| 场景 | 旧 2025-11-25 | 新 2026-07-28 |
| --- | --- | --- |
| 成功问到单位 | SSE 上 `accept`，text 为 `Selected unit: C.` {good:成功} | `inputResponses` 里 `accept`，文本相同 {good:成功} |
| 用户拒绝 | 代码返回 `isError` {caution:拒绝} | 已核对 `isError`，text 为 `Unit choice decline.` {caution:拒绝} |
| 没有会话 | HTTP 400，`-32000` {bad:失败} | 没有会话 |
| 旧握手打到新服务 | 这是它自己的成功起点 | HTTP 400，`-32022` {bad:失败} |
| GET | 有会话是 SSE。无会话是 400 | HTTP 405 {bad:失败} |
| 没声明 `form` | 本页不把这列进已核对失败 | HTTP 400，`-32021` {bad:失败} |

旧服务在 `:3011/mcp`。新服务在 `:3012/mcp`。[](source:v1form)[](source:v2express)

## 旧协议怎样算成功？ {#old-success}

先握手，再读取，再提问，最后关掉会话。读 SSE 的连接要一直开着，直到 `Selected unit: C.` 出现。

@sequence id=old-ok title="旧协议成功"
| actor oldok-c | 客户端 | external |
| actor oldok-s | 旧协议 | backend |
| phase | 握手 |
| oldok-c | oldok-s | POST initialize | emphasis |
| oldok-s | oldok-c | SSE + Session-Id | return |
| oldok-c | oldok-s | notifications/initialized |
| phase | 读取 |
| oldok-c | oldok-s | tools/list |
| oldok-s | oldok-c | draft-07 object | return |
| oldok-c | oldok-s | resources/read |
| oldok-s | oldok-c | protocol 文本 | return |
| phase | 提问 |
| oldok-c | oldok-s | choose-units | emphasis |
| oldok-s | oldok-c | elicitation/create | emphasis |
| oldok-c | oldok-s | result accept | emphasis |
| oldok-s | oldok-c | Selected unit: C. | return |
| phase | 关闭 |
| oldok-c | oldok-s | DELETE /mcp |
| oldok-s | oldok-c | HTTP 200 | return |

@steps id=old-ok-steps
| ok-init | POST initialize | `protocolVersion` 为 `2025-11-25`。要声明 `elicitation.form`。 | old-ok | oldok-c |
| ok-session | SSE + Session-Id | 响应头带回 `Mcp-Session-Id`。后面的请求都带上它。 | old-ok | oldok-s |
| ok-elicit | elicitation/create | 同一条 SSE。核对到的 id 为 0。`unit` 用 `oneOf`。 | old-ok | oldok-s |
| ok-accept | result accept | 另一条 POST。没有 `method`。HTTP 202。 | old-ok | oldok-c |
| ok-done | Selected unit: C. | 回到原来的 SSE，id 为 5。流然后关闭。 | old-ok | oldok-s |

```json title="成功：initialize"
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "elicitation": { "form": {} } },
    "clientInfo": { "name": "wire-check", "version": "1.0.0" }
  }
}
```

头要有 `Accept: application/json, text/event-stream` 和 `MCP-Protocol-Version: 2025-11-25`。响应是 SSE。

```json title="成功：SSE 上的提问"
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "elicitation/create",
  "params": {
    "mode": "form",
    "message": "Which temperature unit should forecasts use?",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "unit": {
          "type": "string",
          "title": "Unit",
          "oneOf": [
            { "const": "C", "title": "Celsius" },
            { "const": "F", "title": "Fahrenheit" }
          ]
        }
      },
      "required": ["unit"]
    }
  }
}
```

```json title="成功：客户端的 accept"
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": { "action": "accept", "content": { "unit": "C" } }
}
```

读取成功时，结果里还有这些字段：

- `get-alerts` 的根是 object，字段是 `state` 与 `alerts`
- `$schema` 是 draft-07
- `resources/read` 的 uri 为 `weather://protocol`
- 正文写 `revision: 2025-11-25`，没有 `ttlMs`

## 旧协议怎样算失败？ {#old-failure}

这三条都发生在提问之前，或发生在会话关掉之后。它们不是 `choose-units` 的业务拒绝。

@sequence id=old-fail title="旧协议失败"
| actor oldbad-c | 客户端 | external |
| actor oldbad-s | 旧协议 | backend |
| phase | 没有会话 |
| oldbad-c | oldbad-s | tools/list 无会话 | dashed |
| oldbad-s | oldbad-c | 400 无 Session | return |
| phase | GET 没有会话 |
| oldbad-c | oldbad-s | GET 无会话 | dashed |
| oldbad-s | oldbad-c | 400 GET 无会话 | return |
| phase | 会话已删除 |
| oldbad-c | oldbad-s | DELETE 后再 POST | dashed |
| oldbad-s | oldbad-c | 404 会话没有 | return |

@steps id=old-fail-steps
| fail-post | 400 无 Session | code `-32000`。message 为 `No valid session ID provided`。`id` 为 null。 | old-fail | oldbad-s |
| fail-get | 400 GET 无会话 | code `-32000`。message 为 `Missing or unknown session ID`。 | old-fail | oldbad-s |
| fail-gone | 404 会话没有 | `DELETE` 本身是 HTTP 200。再用该 id 得到 `-32001`，`Session not found`。 | old-fail | oldbad-s |

```json title="失败：没有会话的 POST"
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "No valid session ID provided"
  },
  "id": null
}
```

```json title="失败：会话已经删除"
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Session not found"
  },
  "id": null
}
```

会话超过 1000 个时，代码返回 HTTP 503，code `-32000`，message 为 `Too many open sessions`。这条没有单独打过 HTTP。

## 新协议怎样算成功？ {#new-success}

不要先发 `initialize`。每个请求自己带 `_meta`。`choose-units` 会结束，然后用新 id 重投。[](source:v2input)

@sequence id=new-ok title="新协议成功"
| actor newok-c | 客户端 | external |
| actor newok-s | 新协议 | backend |
| phase | 读取 |
| newok-c | newok-s | tools/list | emphasis |
| newok-s | newok-c | resultType complete | return |
| newok-c | newok-s | resources/read |
| newok-s | newok-c | ttlMs 0 | return |
| phase | 提问 |
| newok-c | newok-s | choose-units | emphasis |
| newok-s | newok-c | input_required | return |
| newok-c | newok-s | 重投 inputResponses | emphasis |
| newok-s | newok-c | Selected unit: C. | return |

@steps id=new-ok-steps
| ok-list | tools/list | `_meta` 里要有版本、`clientInfo` 和 `elicitation.form`。 | new-ok | newok-c |
| ok-input | input_required | 当次 POST 结束。`inputRequests.unit.method` 为 `elicitation/create`。 | new-ok | newok-s |
| ok-retry | 重投 inputResponses | 新的 JSON-RPC id。key 名 `unit` 对应 `inputRequests.unit`。 | new-ok | newok-c |
| ok-text | Selected unit: C. | `resultType` 为 `complete`。 | new-ok | newok-s |

```json title="成功：每个请求都带的 _meta"
{
  "_meta": {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientInfo": { "name": "wire-check", "version": "1.0.0" },
    "io.modelcontextprotocol/clientCapabilities": { "elicitation": { "form": {} } }
  }
}
```

头要一致：`MCP-Protocol-Version: 2026-07-28`，`Mcp-Method` 等于 method。`tools/call` 的 `Mcp-Name` 等于 `params.name`。`resources/read` 的 `Mcp-Name` 等于 `params.uri`，这里是 `weather://protocol`。

`tools/list` 与 `resources/read` 的结果带 `resultType=complete`、`ttlMs=0`、`cacheScope=private`。`get-alerts` 的根是顶层 array。`$schema` 是 draft 2020-12。

```json title="成功：当次返回 input_required"
{
  "jsonrpc": "2.0",
  "id": 13,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "unit": {
        "method": "elicitation/create",
        "params": {
          "mode": "form",
          "message": "Which temperature unit should forecasts use?",
          "requestedSchema": {
            "type": "object",
            "properties": {
              "unit": { "type": "string", "enum": ["C", "F"] }
            },
            "required": ["unit"],
            "$schema": "https://json-schema.org/draft/2020-12/schema"
          }
        }
      }
    }
  }
}
```

```json title="成功：重投时多出来的 params"
{
  "name": "choose-units",
  "arguments": {},
  "inputResponses": {
    "unit": { "action": "accept", "content": { "unit": "C" } }
  }
}
```

## 新协议怎样算失败？ {#new-failure}

这三条都到不了“问到单位”。请求在版本、方法或能力上就被拒绝。

@sequence id=new-fail title="新协议失败"
| actor newbad-c | 客户端 | external |
| actor newbad-s | 新协议 | backend |
| phase | 旧握手 |
| newbad-c | newbad-s | POST initialize | dashed |
| newbad-s | newbad-c | 400 / -32022 | return |
| phase | 没有 GET 流 |
| newbad-c | newbad-s | GET /mcp | dashed |
| newbad-s | newbad-c | 405 | return |
| phase | 没声明 form |
| newbad-c | newbad-s | choose-units 无 form | dashed |
| newbad-s | newbad-c | 400 / -32021 | return |

@steps id=new-fail-steps
| bad-ver | 400 / -32022 | `data.supported` 只有 `2026-07-28`。`data.requested` 为 `2025-11-25`。 | new-fail | newbad-s |
| bad-get | 405 | code `-32000`。message 为 `Method not allowed.` | new-fail | newbad-s |
| bad-cap | 400 / -32021 | `clientCapabilities` 为 `{}`。`requiredCapabilities.elicitation.form` 为 `{}`。 | new-fail | newbad-s |

```json title="失败：2025-11-25 的 initialize"
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32022,
    "message": "Unsupported protocol version: 2025-11-25",
    "data": {
      "supported": ["2026-07-28"],
      "requested": "2025-11-25"
    }
  }
}
```

```json title="失败：没有 elicitation.form"
{
  "jsonrpc": "2.0",
  "id": 12,
  "error": {
    "code": -32021,
    "message": "Cannot request input 'unit' (elicitation/create): the request's client capabilities do not declare the required capability",
    "data": {
      "requiredCapabilities": { "elicitation": { "form": {} } }
    }
  }
}
```

`-32021` 的 HTTP 状态是 400。`GET /mcp` 的 HTTP 状态是 405，code 仍是 `-32000`，`id` 为 null。

## 用户拒绝时返回什么？ {#decline}

用户拒绝不是协议失败。请求到达了工具。工具选择不接受这个单位。

| 场景 | 旧 2025-11-25 | 新 2026-07-28 |
| --- | --- | --- |
| 回答动作 | `result.action` 不是 `accept` | `inputResponses.unit.action` 为 `decline` |
| 工具结果 | `isError=true` | `isError=true` {caution:拒绝} |
| 文本 | 代码写成 `Unit choice ` 加 action | 已核对：`Unit choice decline.` |
| 结果类型 | 没有 `resultType` | `resultType=complete` |

```json title="新协议已核对的 decline"
{
  "result": {
    "content": [{ "type": "text", "text": "Unit choice decline." }],
    "isError": true,
    "resultType": "complete"
  }
}
```

旧服务的 `decline` 与 `cancel` 走同一段代码，text 为 `Unit choice ` 加上 action。这两条没有单独打过 HTTP。新服务的 `cancel` 也走同一判断，同样没有单独打过。

成功时的温度字段也不一样。旧的 schema 用 `oneOf` 加 `const`。新的 schema 用 `enum: ["C","F"]`。

## 按场景核对 {#check}

在各自目录执行 `npm install`，再 `npm start`。不要在仓库根目录执行。

- [ ] 成功：旧 `initialize` 响应头有 `Mcp-Session-Id`
- [ ] 成功：旧 SSE 上出现 `elicitation/create`，然后 text 为 `Selected unit: C.`
- [ ] 成功：新 `choose-units` 先返回 `input_required`，重投后 text 相同
- [ ] 失败：旧请求不带会话，得到 `-32000`
- [ ] 失败：旧会话 `DELETE` 后再用，得到 `-32001`
- [ ] 失败：新服务收到 `2025-11-25` 的 `initialize`，得到 `-32022`
- [ ] 失败：新服务 `GET /mcp` 为 405
- [ ] 失败：新请求没有 `elicitation.form`，得到 `-32021`
- [ ] 拒绝：新请求带 `action: decline`，`isError` 为 true

@callout level=note title="没有放进这些场景的部分"
`get-alerts` 与 `get-forecast` 都请求 `https://api.weather.gov`。两边的 NWS 调用相同。stdio 与鉴权没有画。Host 检查只防 DNS rebinding。
