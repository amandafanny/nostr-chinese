## 消息是什么?

Nostr 的 客户端和 Relays 用消息来进行通行。

### Clients(客户端)发送给 Relays 的消息

客户端可以执行的行为:

1. 发布新的事件.

```json
["EVENT", <event JSON>]
```

`<event JSON>` [查看](Event.md)

2. 请求或是订阅新的更新.

```json
["REQ", <subscription_id>, <filters JSON>...]
```

`<subscription_id>` 是用来表明订阅的随机字符。
`<filters>` 是一个 JSON 对象。用来决定哪些事件会被发送给这个订阅。

filters 的格式

```json
{
  "ids": <a list of event ids or prefixes>,
  "authors": <a list of pubkeys or prefixes, the pubkey of an event must be one of these>,
  "kinds": <a list of a kind numbers>,
  "#e": <a list of event ids that are referenced in an "e" tag>,
  "#p": <a list of pubkeys that are referenced in a "p" tag>,
  "since": <an integer unix timestamp, events must be newer than this to pass>,
  "until": <an integer unix timestamp, events must be older than this to pass>,
  "limit": <maximum number of events to be returned in the initial query>,
  "search": <string>, // NIP-50
  "include": "spam", // NIP-50
}
```

在接收到 REQ 消息后，Relays 应该查询其内部数据库并返回与过滤器匹配的事件，然后存储该过滤器并将它接收到的所有未来事件再次发送到同一个 websocket 直到 websocket 关闭。 使用相同的 `<subscription_id>` 接收 CLOSE 事件或使用相同的 `<subscription_id>` 发送新的 REQ，在这种情况下，它应该覆盖以前的订阅。

包含列表（例如 id、种类或#e）的过滤器属性是具有一个或多个值的 JSON 数组。 数组的至少一个值必须与事件中的相关字段相匹配，条件才会被视为匹配。 对于 kind 等标量事件属性(只有一个值)，事件的属性必须包含在过滤器列表中。 对于 tags 属性，如 #e，其中一个事件可能有多个值，事件和过滤条件值必须至少有一个共同点。

ids 和 authors 列表包含小写的十六进制字符串，可以是 64 个字符的精确匹配，也可以是事件值的前缀(事件值的精确字符串前几位)。 使用前缀允许在查询大量值时使用更紧凑的过滤器，并且可以为不想透露他们正在搜索的确切作者或事件的客户提供一些隐私保护。

一个 REQ 消息可能包含多个过滤器。 在这种情况下，将返回与任何过滤器匹配的事件。

过滤器的 limit 属性仅对初始查询有效，之后可以忽略。 当 limit: n 存在时，假定初始查询中返回的事件将是最新的 n 个事件。 可以少于限制指定的事件，但 Relays 不会返回比请求多于 n 的事件，这样客户端就不会不必要地被数据淹没。

3. 取消以前的订阅.

```json
["CLOSE", <subscription_id>]
```

4. 授权 relay

```json
["AUTH", <signed-event-json>]
```

### Relays 发送给 Clients(客户端) 的消息

1. 发送给客户端请求的事件.

```json
["EVENT", <subscription_id>, <event JSON>]
```

EVENT 消息必须仅与与客户端先前发起的订阅相关的订阅 ID 一起发送（使用上面的 REQ 消息）。

2. 给客户端发动错误信息或是其他东西.

```json
["NOTICE", <message>], used to send human-readable error messages or other things to clients.
```

3. 通知客户端所有的存储的事件已经发送了。

```json
["EOSE", <subscription_id>]
```

表示在这条消息之后发生的所有事件都是新发布的。

4. 通知客户端一个事件是否成功了

```json
["OK", <event_id>, <true|false>, <message>]
```

返回值:

- `true` 事件已经保存在数据库中了,后续的 message 格式
  - `duplicate:`: 事件重复
- `false` 事件还没有保存到数据库, 后续的 message 格式
  - `blocked:`: 公钥或网络地址已被阻止、禁止或不在白名单中
  - `invalid:`: 事件不是合法的,非格式错误,如果是格式错误，应该使用 NOTICE 消息
  - `pow:`: 事件不满足工作量证明(proof-of-work)的难度
  - `rate-limited:`: relay 有速率限制
  - `error:`: 由于服务器错误事件无法保存

5. 请求客户端授权访问受限资源

```json
["AUTH", <challenge-string>]
```
