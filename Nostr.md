## Nostr 是什么？

Nostr 是一个协议。服务器和客户端可以使用这个协议来进行通信。它包含一系列的规则。就像 比特币、电子邮件、[Bittorrent](<https://zh.wikipedia.org/wiki/BitTorrent_(%E5%8D%8F%E8%AE%AE)>) 等其他协议。显然，Nostr 不是一个应用，但许多应用可以建立在 Nostr 之上。

在 Nostr 的每个用户拥有一堆密钥:

**公钥**可以视为用户名，你可以将公钥分享给其他用户。

**私钥**应该保存在一个安全的地方，确保它为你私人所有。它可以用来给一个事件签名，证明这个事件是你创建的。

## 什么是事件？

事件是网络(网络由客户端和 relays 组成)上的对象。relays 会公开它的 websocket 端点，客户端可以连接它，从网络上获取新的事件。

```json
{
  "id": <32字节, 采用 sha256 加密的序列化的事件数据>
  "pubkey": <32字节, 采用16进制编码 表明事件创建者的公钥>,
  "created_at": <以秒为单位的 unix 时间戳>,
  "kind": <整数>,
  "tags": [
    ["e", <32-bytes hex of the id of another event>, <推荐的 relay URL>],
    ["p", <32-bytes hex of the key>, <推荐的 relay URL>],
    ... // other kinds of tags may be included later
  ],
  "content": <任意字符>,
  "sig": <64字节, signature of the sha256 hash of the serialized event data, which is the same as the "id" field>
}
```

### 事件的种类

## Clients(客户端)

客户端可以执行的行为:

1. 发布新的事件.
2. 请求或是订阅新的更新.
3. 取消以前的订阅.

## Relays(服务器)

Relays 可以执行的行为

1. 发送时间 Send events.
2. 给客户端发动错误信息或是其他东西.
