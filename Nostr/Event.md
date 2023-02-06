## 事件是什么

事件是网络(网络由客户端和 relays 组成)上的对象。relays 会公开它的 websocket 端点，客户端可以连接它，从网络上获取新的事件。

## 事件的格式

```json
{
  "id": <32-bytes lowercase hex-encoded sha256 of the the serialized event data>
  "pubkey": <32-bytes lowercase hex-encoded public key of the event creator>,
  "created_at": <unix timestamp in seconds>,
  "kind": <integer>,
  "tags": [
    ["e", <32-bytes hex of the id of another event>, <recommended relay URL>],
    ["p", <32-bytes hex of the key>, <recommended relay URL>],
    ... // other kinds of tags may be included later
  ],
  "content": <arbitrary string>,
  "sig": <64-bytes signature of the sha256 hash of the serialized event data, which is the same as the "id" field>
}
```

### event.id 如何生成

我们对序列化的事件进行 sha256 加密。

序列化的事件的结构（UTF-8 JSON-serialized， 没有空格和换行符）：

```json
[
  0,
  <pubkey, as a (lowercase) hex string>,
  <created_at, as a number>,
  <kind, as a number>,
  <tags, as an array of arrays of non-null strings>,
  <content, as a string>
]
```

## 事件的种类

## tags

[参考链接](https://github.com/nostr-protocol/nips)

后续更新（TODO）
