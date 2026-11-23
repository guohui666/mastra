# 新旧天气 MCP 的参数差在哪 {#mcp-params}

旧协议把提问留在未结束的 SSE 上。新协议让当次调用结束，并返回 `input_required`。客户端再用新的请求 id，把回答放进 `inputResponses`。

- 旧服务用 `Mcp-Session-Id` 找回会话里的 transport。
- 新服务设置 `legacy: 'reject'`，每个请求新建一台 `McpServer`。
- 旧的回答是一条没有 `method` 的 JSON-RPC result。
- 新的回答在 `params.inputResponses.unit`。

@answer scope="本地 HTTP 已核对。SDK 1.32.1 与 server 2.3.1。不含鉴权。"
@meta locale=zh-CN
@source v1form title="v1 form elicitation 示例" url=https://github.com/modelcontextprotocol/typescript-sdk/blob/v1.x/src/examples/server/elicitationFormExample.ts
@source v2input title="v2 input_required 文档" url=https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/servers/input-required.md
@source v2express title="v2 Express 挂载" url=https://ts.sdk.modelcontextprotocol.io/v2/serving/express

## 两台服务各自怎么接上？ {#wiring}

客户端打的都是 `POST /mcp`。旧服务先查会话。新服务不保存会话。[](source:v1form)[](source:v2express)

@architecture id=two-stacks title="两台天气服务的接法"
| client | 客户端 | external |
| oldApp | 旧 Express | backend |
| sessions | 会话 Map | backend |
| oldMcp | 旧 McpServer | backend |
| elicit | elicitInput | backend |
| newApp | 新 Express | backend |
| handler | 新 Handler | backend |
| newMcp | 新 McpServer | backend |
| ask | inputRequired | backend |
| client > oldApp | POST /mcp |
| oldApp > sessions | 查会话 |
| sessions > oldMcp | 一会话一台 |
| oldMcp > elicit | 会话内提问 | emphasis |
| client > newApp | POST /mcp |
| newApp > handler | legacy reject |
| handler > newMcp | 一请求一台 |
| newMcp > ask | 结果里提问 | emphasis |

@steps id=wire-steps
| old-post | 旧 Express | 没有会话且 body 是 initialize 时，新建 transport。 | two-stacks | oldApp |
| old-map | 会话 Map | `sessionIdGenerator` 用 `randomUUID`。`onsessioninitialized` 写入 Map。 | two-stacks | sessions |
| wire-elicit | elicitInput | `relatedRequestId` 等于这次 `tools/call` 的 id，问题回到这条 SSE。 | two-stacks | elicit |
| new-handler | 新 Handler | `createMcpHandler(buildServer, { legacy: 'reject' })`。 | two-stacks | handler |
| wire-input | inputRequired | 工具返回 `inputRequests.unit`，不在流上发请求。 | two-stacks | ask |

@callout level=warning title="不带 relatedRequestId 会丢问题"
不带 `relatedRequestId` 时，`elicitation/create` 走 `GET /mcp`。GET 没打开，传输层丢掉这条请求，`tools/call` 会一直等。

## 旧协议一次提问走哪些消息？ {#old-call}

先握手，再在同一次 `tools/call` 上提问。读 SSE 的连接要保持打开，直到最后一条结果到达。

@sequence id=old-ask title="旧协议 2025-11-25"
| actor client | 客户端 | external |
| actor old | 旧协议 | backend |
| phase | 握手 |
| client | old | POST initialize | emphasis |
| old | client | SSE + Session-Id | return |
| client | old | notifications/initialized |
| phase | 提问 |
| client | old | tools/call choose-units | emphasis |
| old | client | elicitation/create | emphasis |
| client | old | result accept | emphasis |
| old | client | Selected unit: C. | return |
| phase | 关闭 |
| client | old | DELETE /mcp |

@steps id=old-steps
| old-hello | POST initialize | `params.protocolVersion` 为 `2025-11-25`。`capabilities.elicitation.form` 必须有。 | old-ask | client |
| old-session | SSE + Session-Id | 响应头 `Mcp-Session-Id`。之后的 POST、GET、DELETE 都带回它。 | old-ask | old |
| old-elicit | elicitation/create | 同一条 SSE。例中 id 为 0。`mode` 为 `form`。`unit` 用 `oneOf`。 | old-ask | old |
| old-answer | result accept | 另一条 POST。没有 `method`。`result.action` 为 `accept`。HTTP 202。 | old-ask | client |

```json title="旧协议 initialize 的 body"
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

头还有 `Accept: application/json, text/event-stream` 和 `MCP-Protocol-Version: 2025-11-25`。响应是 SSE，不是单个 JSON。

```json title="旧协议 SSE 上的 elicitation/create"
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

```json title="客户端对 elicitation/create 的回答"
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": { "action": "accept", "content": { "unit": "C" } }
}
```

这条 POST 得到 HTTP 202，body 为空。原来的 SSE 接着返回 id 5 的结果，`content[0].text` 为 `Selected unit: C.`，然后关闭。

没有 `Mcp-Session-Id` 的 `tools/list` 得到 HTTP 400，code `-32000`，message 为 `No valid session ID provided`。带会话的 `GET /mcp` 得到 HTTP 200 和 `text/event-stream`。`DELETE` 成功是 HTTP 200。再用该会话得到 HTTP 404，code `-32001`。

## 新协议一次提问走哪些消息？ {#new-call}

没有 `initialize`，也没有会话。每个请求自己带版本和能力。[](source:v2input)

@sequence id=new-ask title="新协议 2026-07-28"
| actor client | 客户端 | external |
| actor new | 新协议 | backend |
| phase | 拒绝旧握手 |
| client | new | POST initialize | dashed |
| new | client | 400 / -32022 | return |
| client | new | GET /mcp | dashed |
| new | client | 405 | return |
| phase | 提问 |
| client | new | choose-units 无 form | dashed |
| new | client | 400 / -32021 | return |
| client | new | choose-units | emphasis |
| new | client | input_required | return |
| client | new | 重投 inputResponses | emphasis |
| new | client | Selected unit: C. | return |

@steps id=new-steps
| new-reject | 400 / -32022 | `data.supported` 只有 `2026-07-28`。`data.requested` 为 `2025-11-25`。 | new-ask | new |
| new-cap | 400 / -32021 | `clientCapabilities` 为空时，不能发 `inputRequests.unit`。 | new-ask | new |
| step-input | input_required | 当次 POST 结束。`inputRequests.unit.method` 为 `elicitation/create`。 | new-ask | new |
| new-retry | 重投 inputResponses | 新的 JSON-RPC id。key 名 `unit` 对应 `inputRequests.unit`。 | new-ask | client |

```json title="新协议拒绝 2025-11-25 的 initialize"
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

`GET /mcp` 得到 HTTP 405，code `-32000`，message 为 `Method not allowed.`

```json title="新协议 tools/list 的 params"
{
  "_meta": {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientInfo": { "name": "wire-check", "version": "1.0.0" },
    "io.modelcontextprotocol/clientCapabilities": { "elicitation": { "form": {} } }
  }
}
```

头要和 body 一致：`MCP-Protocol-Version: 2026-07-28`，`Mcp-Method: tools/list`。`tools/call` 还要 `Mcp-Name`，值等于 `params.name`。`resources/read` 的 `Mcp-Name` 等于 `params.uri`。

```json title="choose-units 返回的 input_required"
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

没有 `elicitation.form` 时，同一调用得到 HTTP 400，code `-32021`。`data.requiredCapabilities.elicitation.form` 为 `{}`。

```json title="重投时多出来的 params"
{
  "name": "choose-units",
  "arguments": {},
  "inputResponses": {
    "unit": { "action": "accept", "content": { "unit": "C" } }
  }
}
```

结果 `resultType` 为 `complete`，text 为 `Selected unit: C.`。`action` 为 `decline` 时，`isError` 为 true，text 为 `Unit choice decline.`

## 字段对照 {#fields}

| 项目 | 旧 2025-11-25 | 新 2026-07-28 |
| --- | --- | --- |
| 地址 | `127.0.0.1:3011/mcp` | `127.0.0.1:3012/mcp` |
| 版本放在哪 | `params.protocolVersion`，一会话一份 | 每个请求的 `_meta` 和 `MCP-Protocol-Version` |
| 找回连接 | 响应头 `Mcp-Session-Id` {caution:要会话} | 没有会话 {good:一次一请求} |
| 问用户的位置 | SSE 上的 `elicitation/create` {caution:调用不结束} | `result.resultType=input_required` {good:调用结束} |
| 温度字段 | `requestedSchema.properties.unit.oneOf` | `requestedSchema.properties.unit.enum` |
| 回答放在 | 另一次 POST 的 `result.content.unit` | 下一次 `tools/call` 的 `inputResponses.unit` |
| 回答的 id | 等于服务端请求 id，例中为 0 | 新的 JSON-RPC id，不能复用上一次 |
| 成功文本 | `Selected unit: C.` | `Selected unit: C.` |
| 拒绝 | `isError=true`，text 以 `Unit choice` 开头 | 同样，decline 的 text 为 `Unit choice decline.` |
| alerts 根 | object，字段 `state` 与 `alerts` | 顶层 array |
| JSON Schema | `draft-07` | `2020-12` |
| 列表结果 | 没有 `resultType` | `resultType`、`ttlMs=0`、`cacheScope=private` |
| GET | 带会话则是 SSE | HTTP 405 |
| 幂等提示 | `choose-units` 的 `idempotentHint=false` | `choose-units` 的 `idempotentHint=true` |

`annotations` 两边都有。它不是这次分叉的点。

## 代码里提问是哪几行？ {#code}

旧工具在 `explorations/official-mcp-examples/old-protocol/src/weather.ts`。它调用 `server.server.elicitInput`，并把 `relatedRequestId` 设为 `extra.requestId`。

```ts title="旧协议 choose-units 的核心调用"
const result = await server.server.elicitInput(
  {
    mode: 'form',
    message: 'Which temperature unit should forecasts use?',
    requestedSchema: {
      type: 'object',
      properties: {
        unit: {
          type: 'string',
          oneOf: [
            { const: 'C', title: 'Celsius' },
            { const: 'F', title: 'Fahrenheit' }
          ]
        }
      },
      required: ['unit']
    }
  },
  { relatedRequestId: extra.requestId }
);
```

会话表在 `old-protocol/src/http.ts`。`sessions` 是 `Map`。超过 1000 个会话时返回 HTTP 503，code `-32000`。

新工具在 `new-protocol/src/weather.ts`。它不调用 `ctx.mcpReq.elicitInput`。[](source:v2input)

```ts title="新协议 choose-units 的核心返回"
return inputRequired({
  inputRequests: {
    unit: inputRequired.elicit({
      message: 'Which temperature unit should forecasts use?',
      requestedSchema: unitSchema
    })
  }
});
```

`unitSchema` 是 `z.object({ unit: z.enum(['C', 'F']) })`。挂载在 `new-protocol/src/http.ts`：`createMcpHandler(buildServer, { legacy: 'reject' })`，再交给 `toNodeHandler`。

这一轮只有一个问题，所以没有 `requestState`。`requestState` 用来串多轮，并且要当不可信输入来校验。

## 本地怎么核对？ {#check}

两个目录都不在 pnpm workspace 里。在各自目录执行 `npm install`，再 `npm start`。不要在仓库根目录执行。

- [ ] 旧 `initialize` 的响应头有 `Mcp-Session-Id`，`protocolVersion` 为 `2025-11-25`
- [ ] 不带会话的 `tools/list` 为 HTTP 400，code `-32000`
- [ ] `choose-units` 的 SSE 里出现 `elicitation/create`，`oneOf` 含 `C` 与 `F`
- [ ] 回答后，原 SSE 的 text 为 `Selected unit: C.`
- [ ] 同一条 `initialize` 打到新服务，得到 HTTP 400，code `-32022`
- [ ] 新服务 `GET /mcp` 为 HTTP 405
- [ ] `clientCapabilities` 为空时，`choose-units` 为 HTTP 400，code `-32021`
- [ ] 声明 `elicitation.form` 后，结果 `resultType` 为 `input_required`
- [ ] 带 `inputResponses.unit` 重投后，text 为 `Selected unit: C.`

@callout level=note title="图里故意不展开的部分"
`get-alerts` 与 `get-forecast` 都请求 `https://api.weather.gov`。两边的 NWS 调用相同。stdio 入口还在。鉴权没有接上。Host 检查只防 DNS rebinding，不是登录。上面的 JSON 来自本地 HTTP 核对，不是猜的字段。
