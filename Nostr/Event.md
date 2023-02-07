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
  "sig": <64-bytes signature of the sha256 hash of the serialized event data, which is the same as the "id" field>,
  "ots": <base64-encoded OTS file data>
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

### 事件的种类

#### 0: set_metadata(NIP-01)

`content` 是字符串化的 JSON 对象, 一旦收到一个新的 set_metadata 事件， relay 可能删除过去的 set_metadata 事件(公钥一致的)。

```json
{
  name: <username>,
  about: <string>,
  picture: <url, string>
}
```

##### nip05

格式:

```json
// 大小写不敏感
<local-part>@<domain>
```

解析后

```json
https://<domain>/.well-known/nostr.json?name=<local-part>
```

我的 nip05 [land@snort.social](https://snort.social/.well-known/nostr.json?name=land)

###### 安全约束

`/.well-known/nostr.json` 端不能返回任何 HTTP 重定向。

#### 1: text_note(NIP-01)

`content` 是一条笔记的文本内容（用户想说的任何内容）。 非明文笔记应该使用种类 1000-10000，在 NIP-16 中描述。

#### 2: recommend_server(NIP-01)

`content` 是事件创建者想要他的关注者推荐的 relay 的 URL (例如, wss://somerelay.com).

#### 3: contact list(NIP-02)

联系人列表，被定义为具有 p 标签列表，每个标签对应一个关注的/已知的配置文件。

每个标签条目都应包含配置文件的密钥、可以找到来自该密钥的事件的 relay URL（如果不需要，可以设置为空字符串）以及该配置文件的本地名称（或“petname”）（也可以设置为空字符串或未提供），即 `["p", <32-bytes hex key>, <main relay URL>, <petname>]`。 `content` 可以是任何东西，应该被忽略。

发布的每个新 contact list 都会覆盖过去的 contact list。

#### 4: encrypted direct message(NIP-04)

`content` 必须等于用户想要写入的任何内容的 base64 编码、aes-256-cbc 加密字符串，使用通过将接收者的公钥与发送者的私钥组合生成的共享密码进行加密； 这由 base64 编码的初始化向量附加，就好像它是一个名为“iv”的查询字符串参数。 格式如下：`"content": "<encrypted_text>?iv=<initialization_vector>"`。

`tags` 必须包含一个标识消息接收者的条目（这样 relay 可以自然地将此事件转发给它们），格式为 `["p", "<pubkey, as a hex string>"]`。

`tags` 可以包含一个条目，用于标识对话中的前一条消息或我们明确回复的消息（这样可能会发生上下文相关的、更有组织的对话），形式为 `["e", "<event_id>"]`。

```js
import crypto from "crypto";
import * as secp from "noble-secp256k1";

let sharedPoint = secp.getSharedSecret(ourPrivateKey, "02" + theirPublicKey);
let sharedX = sharedPoint.substr(2, 64);

let iv = crypto.randomFillSync(new Uint8Array(16));
var cipher = crypto.createCipheriv(
  "aes-256-cbc",
  Buffer.from(sharedX, "hex"),
  iv
);
let encryptedMessage = cipher.update(text, "utf8", "base64");
encryptedMessage += cipher.final("base64");
let ivBase64 = Buffer.from(iv.buffer).toString("base64");

let event = {
  pubkey: ourPubKey,
  created_at: Math.floor(Date.now() / 1000),
  kind: 4,
  tags: [["p", theirPublicKey]],
  content: encryptedMessage + "?iv=" + ivBase64,
};
```

[OpenTimestamps](https://en.wikipedia.org/wiki/OpenTimestamps)

#### 5: deletion

`tags` 表明需要删除的事件

`content` 字段可以包含描述删除原因的文本注释。

relays 应该删除或停止发布任何与删除请求具有相同 ID 的引用事件。 客户端应该隐藏或以其他方式指示引用事件的删除状态。

relays 应该无限期地继续发布/共享删除事件，因为客户端可能已经有了要删除的事件。 此外，客户端应该将删除事件广播到其他没有删除事件的中继。

### tags

#### p(pubkey)

“公钥”，它指向事件中引用的某人的公钥

#### e(event)

```json
["e", <event-id>, <relay-url>, <marker>]
```

“事件”，它指向这个事件引用的事件的 ID ，以某种方式回复或提及。

#### r(reference)

与网页相关

#### g(geohash)

与位置相关

#### t(hashtag)

与话题相关

#### nonce

```json
[
  "nonce",
  "1", // 前导零的个数，前导零越多，难度越高
  "20" //
]
```

#### subject

主题标签, 适用于 kind: 1

Proof of Work (PoW) 相关
