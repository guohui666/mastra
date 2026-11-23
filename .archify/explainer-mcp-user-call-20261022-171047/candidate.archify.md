# 调用天气服务时你传什么 {#user-call}

调用方只传业务参数。这两台服务不检查登录。查天气一次就返回正文。问单位时，旧协议在 SSE 上插入一条请求，新协议则结束本次调用并要求重投。

- 查警报传 `state`。查预报传经纬度。选单位传空对象。
- 登录不放在 JSON 里。这两台服务不读 `Authorization`。
- 旧协议的提问留在未结束的 SSE。调用方用相同 id 回 `result`。
- 新协议的提问是本次结果。调用方用新 id 重投 `inputResponses`。

@answer scope="本地 HTTP。外壳与选单位已核对。天气字段来自工具代码。"
@meta locale=zh-CN
@source auth26 title="MCP 授权：2026-07-28" url=https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
@source basic26 title="MCP 基础：2026-07-28" url=https://modelcontextprotocol.io/specification/2026-07-28/basic
@source http26 title="Streamable HTTP：2026-07-28" url=https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
@source http25 title="Streamable HTTP：2025-11-25" url=https://modelcontextprotocol.io/specification/2025-11-25/basic/transports
@source v2input title="v2 input_required 文档" url=https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/servers/input-required.md
@source v1form title="v1 form elicitation 示例" url=https://github.com/modelcontextprotocol/typescript-sdk/blob/v1.x/src/examples/server/elicitationFormExample.ts

## 调用时传什么、拿回什么？ {#params}

业务参数只放在 `arguments`。给人看的句子在 `content`。程序读取 `structuredContent`。

| 你要做的事 | 传入 | 成功时拿回 |
| --- | --- | --- |
| 查警报 `get-alerts` | `state`，正好 2 个字母 | `text`，加上警报字段 |
| 查预报 `get-forecast` | `latitude` 与 `longitude` | `text`，加上最多 5 段预报 |
| 选单位 `choose-units` | 空对象 `{}` | 先选 `C` 或 `F`，再拿 `Selected unit` |
| 读说明 | uri `weather://protocol` | 纯文本，写明协议修订 |

| 字段 | 含义 | 边界 |
| --- | --- | --- |
| `state` | 美国州代码 | 2 个字母。服务转成大写 |
| `latitude` | 纬度 | -90 到 90 |
| `longitude` | 经度 | -180 到 180 |
| `event` | 警报类型 | 字符串 |
| `area` | 影响区域 | 字符串 |
| `severity` | 严重程度 | 字符串 |
| `description` | 发生了什么 | 字符串 |
| `instructions` | 人们该怎么做 | 字符串 |
| `periods[].name` | 预报时段 | 字符串。最多 5 段 |
| `temperature` | 气温 | 数字，可以是 null |
| `temperature_unit` | 气温单位 | 字符串 |
| `wind_speed` | 风速 | 字符串 |
| `wind_direction` | 风向 | 字符串 |
| `detailed_forecast` | 这段预报 | 字符串 |

空警报仍是成功。`text` 为 `No active alerts for CA` 这种句子。天气接口失败时，这次调用失败。地点限于美国。

旧协议把警报包在 object 里。新协议的警报根是 array。预报两边都是 object。

```json title="旧协议：警报 structuredContent 的形状"
{
  "state": "CA",
  "alerts": [
    {
      "event": "string",
      "area": "string",
      "severity": "string",
      "description": "string",
      "instructions": "string"
    }
  ]
}
```

```json title="新协议：警报 structuredContent 的形状"
[
  {
    "event": "string",
    "area": "string",
    "severity": "string",
    "description": "string",
    "instructions": "string"
  }
]
```

```json title="两边相同：预报 structuredContent 的形状"
{
  "latitude": 38.9,
  "longitude": -77.0,
  "periods": [
    {
      "name": "string",
      "temperature": 70,
      "temperature_unit": "F",
      "wind_speed": "string",
      "wind_direction": "string",
      "detailed_forecast": "string"
    }
  ]
}
```

每次请求还要带外壳。业务字段不放进这些头。

| 外壳 | 旧 `2025-11-25` | 新 `2026-07-28` |
| --- | --- | --- |
| 地址 | `127.0.0.1:3011/mcp` | `127.0.0.1:3012/mcp` |
| 版本 | 会话里一份 | 每个请求的 `_meta` 和头 |
| 会话 | 头 `Mcp-Session-Id` | 没有 |
| 方法头 | 不需要 | `Mcp-Method` 等于 body 的 method |
| 名字头 | 不需要 | `tools/call` 时 `Mcp-Name` 等于 `name` |
| Accept | `application/json` 与 `text/event-stream` 都要 | 一次调用结束 |

```json title="旧协议：查警报的 body"
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get-alerts",
    "arguments": { "state": "CA" }
  }
}
```

握手之后，这个 body 再加头 `Mcp-Session-Id` 和 `MCP-Protocol-Version: 2025-11-25`。

```json title="新协议：查警报的 body"
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "tools/call",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": { "name": "my-app", "version": "1.0.0" },
      "io.modelcontextprotocol/clientCapabilities": {}
    },
    "name": "get-alerts",
    "arguments": { "state": "CA" }
  }
}
```

头要有 `MCP-Protocol-Version: 2026-07-28`、`Mcp-Method: tools/call`、`Mcp-Name: get-alerts`。查天气可以不声明 `form`。选单位必须声明 `elicitation.form`。[](source:http26)

## 鉴权放在哪？ {#auth}

这两台服务不看 `Authorization`。你现在调用不用带 token。

| 问题 | 现在这两台 | 服务打开鉴权以后 |
| --- | --- | --- |
| 要不要登录 | 不要 {good:不验登录} | 每个 HTTP 请求都要 {caution:按规范} |
| token 放哪 | 不传 | 只放头 `Authorization: Bearer` |
| 能不能放进 JSON | 不传 | 不可以。也不放进 `_meta` {bad:禁止} |
| 没带 token | 不会因此返回 401 | HTTP 401，并带 `WWW-Authenticate` |
| 本地 stdio | 没有 HTTP 头 | 不要照搬这套 HTTP 鉴权 |
| `Mcp-Session-Id` | 只是旧协议的会话号 | 不是密码 |

HTTP 传输应按这套授权来做。stdio 不应该照搬，凭证从环境读取。[](source:basic26)[](source:auth26)

token 必须放在 `Authorization: Bearer` 后面。每个发往该服务的 HTTP 请求都带。不要放进 URL。[](source:auth26)

@callout level=note title="这不是登录"
服务去拉 `api.weather.gov` 时，自己带 `User-Agent: weather-app/1.0`。那是服务端的身份。新服务还会检查 Host。那是挡 DNS 重绑定。

## 收到响应后先看哪一种？ {#handle}

先看 body 属于哪一种，再决定怎么回。不要把提问当成天气正文。

@workflow id=handle-flow title="收到响应后怎么处理"
| got | 收到响应 | external |
| wx | 天气正文? | backend |
| show | 展示文本 | frontend |
| sse | SSE 提问? | backend |
| back | 回这条事件 | frontend |
| need | 还要输入? | backend |
| retry | 重投答案 | frontend |
| bad | 按错误码停 | backend |
| got > wx | |
| wx > show | 是 |
| wx > sse | 否 |
| sse > back | 是 |
| sse > need | 否 |
| need > retry | 是 |
| need > bad | 否 |

@steps id=handle-steps
| h-wx | 天气正文? | `content` 里已有 `text`。把句子给用户。程序再读 `structuredContent`。 | handle-flow | wx |
| h-sse | SSE 提问? | `data` 里有 `method: elicitation/create`。这是旧协议。 | handle-flow | sse |
| h-back | 回这条事件 | 另开一条 POST。body 只有相同 `id` 和 `result`。继续读原来的 SSE。 | handle-flow | back |
| h-need | 还要输入? | `resultType` 为 `input_required`。这是新协议。本次调用已经结束。 | handle-flow | need |
| h-retry | 重投答案 | 用新的 jsonrpc id，再调同一个工具，并带上 `inputResponses`。 | handle-flow | retry |
| h-bad | 按错误码停 | 下面这些码都不是天气结果。 | handle-flow | bad |

| 你看到的失败 | 含义 |
| --- | --- |
| 旧：HTTP 400，`-32000` | 没有会话，或 GET 没有会话 |
| 旧：HTTP 404，`-32001` | 会话已经 `DELETE` |
| 新：HTTP 400，`-32022` | 你发了旧的 `initialize` |
| 新：HTTP 405，`-32000` | 你发了 `GET /mcp` |
| 新：HTTP 400，`-32021` | 选单位时没声明 `elicitation.form` |
| `isError: true` | 用户拒绝了单位。这不是协议错误 |

这两台服务不发进度通知，也不发日志通知。你要回应的只有选单位。

## 旧协议的事件怎么回？ {#old-reply}

SSE 里的 `data` 若是 `result`，那就是结果。若带 `method`，那就是要你回应的事件。回答走另一条 POST。原来的 SSE 要保持打开。[](source:http25)[](source:v1form)

@sequence id=old-reply-flow title="旧协议：事件怎么回"
| actor orc | 调用方 | external |
| actor ors | 旧服务 | backend |
| phase | 握手 |
| orc | ors | POST initialize | emphasis |
| ors | orc | SSE + 会话号 | return |
| orc | ors | 通知已就绪 |
| ors | orc | HTTP 202 | return |
| phase | 查天气 |
| orc | ors | 查警报 |
| ors | orc | 警报正文 | return |
| phase | 提问 |
| orc | ors | choose-units | emphasis |
| ors | orc | 提问事件 | emphasis |
| orc | ors | 回 result | emphasis |
| ors | orc | 单位文本 | return |

@steps id=old-reply-steps
| o-init | POST initialize | `protocolVersion` 为 `2025-11-25`。要声明 `elicitation.form`。 | old-reply-flow | orc |
| o-sid | SSE + 会话号 | 响应是 SSE。头里有 `Mcp-Session-Id`。后面都带回。 | old-reply-flow | ors |
| o-note | 通知已就绪 | 方法 `notifications/initialized`。没有 `id`。这是通知，不是请求。 | old-reply-flow | orc |
| o-202 | HTTP 202 | 服务接受通知或你的回答时，body 为空。没有 JSON 结果。 | old-reply-flow | ors |
| o-ask | 提问事件 | 同一条 SSE。核对到的 id 为 0。问题是选 `C` 或 `F`。 | old-reply-flow | ors |
| o-back | 回 result | 另一条 POST。没有 `method`。带同一个会话号。 | old-reply-flow | orc |
| o-text | 单位文本 | 回到原来的 SSE。核对到 `Selected unit: C.`。然后流关闭。 | old-reply-flow | ors |

```json title="旧协议：SSE 上的提问事件"
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

```json title="旧协议：你回给这条事件的 body"
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": { "action": "accept", "content": { "unit": "C" } }
}
```

这条 POST 得到 HTTP 202，body 为空。最终文本在原来的 SSE 上：`Selected unit: C.`

先停掉 SSE 读取，最后一段可能收不到。不带本次调用 id 时，提问会改走 `GET /mcp`。GET 没打开，调用会一直等。

## 新协议为什么不是一条事件？ {#new-reply}

服务不再往流上插入独立请求。要你补充的内容写在本次 `result` 里。你用新 id 重投原调用。[](source:http26)[](source:v2input)

@sequence id=new-reply-flow title="新协议：提问怎么回"
| actor nrc | 调用方 | external |
| actor nrs | 新服务 | backend |
| phase | 查天气 |
| nrc | nrs | 查警报 | emphasis |
| nrs | nrc | 警报正文 | return |
| phase | 还要输入 |
| nrc | nrs | choose-units | emphasis |
| nrs | nrc | input_required | return |
| nrc | nrs | 重投答案 | emphasis |
| nrs | nrc | 单位文本 | return |

@steps id=new-reply-steps
| n-call | 查警报 | 不要先发 `initialize`。每次请求自带 `_meta`。 | new-reply-flow | nrc |
| n-need | input_required | 本次调用结束。核对到的 id 为 13。`unit` 用 `enum`。 | new-reply-flow | nrs |
| n-retry | 重投答案 | 新的 jsonrpc id。`arguments` 仍是 `{}`。答案在 `inputResponses.unit`。 | new-reply-flow | nrc |
| n-text | 单位文本 | `resultType` 为 `complete`。文本仍是 `Selected unit: C.` | new-reply-flow | nrs |

```json title="新协议：本次结果就是提问"
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
            "properties": { "unit": { "type": "string", "enum": ["C", "F"] } },
            "required": ["unit"]
          }
        }
      }
    }
  }
}
```

```json title="新协议：重投时多出来的 params"
{
  "name": "choose-units",
  "arguments": {},
  "inputResponses": {
    "unit": { "action": "accept", "content": { "unit": "C" } }
  }
}
```

重投时，`_meta` 和三个头照常带。`Mcp-Name` 仍是 `choose-units`。没有 `Mcp-Session-Id`。

| 用户动作 | 你放入的回答 | 服务返回 |
| --- | --- | --- |
| 选 C | `action` 为 `accept`，`content.unit` 为 `C` | 文本 `Selected unit: C.` {good:成功} |
| decline | `action` 为 `decline` | 新协议已核对：`isError`，文本 `Unit choice decline.` {caution:拒绝} |
| cancel | `action` 为 `cancel` | 代码同样返回 `isError`。这次没有单独打 HTTP |

旧协议的 decline 和 cancel 也走代码里的 `Unit choice` 加动作名。这次没有单独打 HTTP。

不要对这台服务发 `GET`，也不要发 `initialize`。列表和读取成功时，结果还有 `resultType: complete`、`ttlMs: 0`、`cacheScope: private`。

## 你发出去之前核对这几项 {#check}

- [ ] 查警报只传 2 位 `state`。空列表仍当成功
- [ ] 查预报传 `latitude` 与 `longitude`。地点限美国
- [ ] 不在 JSON、`_meta` 或 URL 里放 token
- [ ] 旧协议先保存 `Mcp-Session-Id`。提问时继续读原来的 SSE
- [ ] 新协议每次带齐 `_meta`、`Mcp-Method` 和 `Mcp-Name`
- [ ] 看到 `input_required` 后，用新 id 重投，不要去回一条 SSE 事件
- [ ] `isError` 表示用户没选成单位。不要把它当成 `Selected unit`
