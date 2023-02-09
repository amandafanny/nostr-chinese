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

`content` 是事件创建者想要向他的关注者推荐的 relay 的 URL (例如, wss://somerelay.com).

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

```json
[
  "delegation",
  <pubkey of the delegator>,
  <conditions query string>,
  <64-byte Schnorr signature of the sha256 hash of the delegation token>
]
```

`tags` 表明需要删除的事件

`content` 字段可以包含描述删除原因的文本注释。

relays 应该删除或停止发布任何与删除请求具有相同 ID 的引用事件。 客户端应该隐藏或以其他方式指示引用事件的删除状态。

relays 应该无限期地继续发布/共享删除事件，因为客户端可能已经有了要删除的事件。 此外，客户端应该将删除事件广播到其他没有删除事件的中继。

##### delegation token

```json
nostr:delegation:<pubkey of publisher (delegatee)>:<conditions query string>
```

#### 7: reaction

表达对其他 note 的反应，是否喜欢

`content`值:

- `+` 表示 喜欢;
- `-`表示不喜欢；
- 也可以是 emoji

#### 40: channel create

```json
{
    "content": "{\"name\": \"Demo Channel\", \"about\": \"A test channel.\", \"picture\": \"https://placekitten.com/200/200\"}",
    ...
}
```

#### 41: channel metadata

```json
{
    "content": "{\"name\": \"Updated Demo Channel\", \"about\": \"Updating a test channel.\", \"picture\": \"https://placekitten.com/201/201\"}",
    "tags": [["e", <channel_create_event_id>, <relay-url>]],
    ...
}
```

#### 42: channel message

```json
// Root message
{
    "content": <string>,
    "tags": [["e", <kind_40_event_id>, <relay-url>, "root"]],
    ...
}

// Reply to another message
{
    "content": <string>,
    "tags": [
        ["e", <kind_42_event_id>, <relay-url>, "reply"],
        ["p", <pubkey>, <relay-url>],
        ...
    ],
    ...
}
```

#### 43: hide message

```json
{
    "content": "{\"reason\": \"Dick pic\"}",
    "tags": [["e", <kind_42_event_id>]],
    ...
}
```

#### 44: mute user

拉黑其他用户

```json
{
    "content": "{\"reason\": \"Posting dick pics\"}",
    "tags": [["p", <pubkey>]],
    ...
}
```

#### 1984: report

report type:

- nudity - 对裸体、色情等的描绘
- profanity - 亵渎、仇恨言论等
- illegal - 在某些司法管辖区可能是非法的
- spam - 垃圾
- impersonation - 某人假装成其他人

##### Example events

```json
{
  "kind": 1984,
  "tags": [
    [ "p", <pubkey>, "nudity"]
  ],
  "content": "",
  ...
}

{
  "kind": 1984,
  "tags": [
    [ "e", <eventId>, "illegal"],
    [ "p", <pubkey>]
  ],
  "content": "He's insulting the king!",
  ...
}

{
  "kind": 1984,
  "tags": [
    [ "p", <impersonator pubkey>, "impersonation"],
    [ "p", <victim pubkey>]
  ],
  "content": "Profile is imitating #[1]",
  ...
}
```

#### 10002: Relay List Metadata

#### 22242: signed

```json
{
  "id": "...",
  "pubkey": "...",
  "created_at": 1669695536,
  "kind": 22242,
  "tags": [
    ["relay", "wss://relay.example.com/"],
    ["challenge", "challengestringhere"]
  ],
  "content": "",
  "sig": "..."
}
```

这个事件不应该被广播给其他客户端

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

#### d

[参考](https://github.com/nostr-protocol/nips/blob/master/33.md)

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

#### delegation

```json
[
  "delegation",
  <pubkey of the delegator>,
  <conditions query string>,
  <64-byte Schnorr signature of the sha256 hash of the delegation token>
]
```

##### 例子

```json
# Delegator: 委托人
privkey: ee35e8bb71131c02c1d7e73231daa48e9953d329a4b701f7133c8f46dd21139c
pubkey:  8e0d3d3eb2881ec137a11debe736a9086715a8c8beeeda615780064d68bc25dd

# Delegatee: 受托人
privkey: 777e4f60b4aa87937e13acc84f7abcc3c93cc035cb4c1e9f7a9086dd78fffce1
pubkey:  477318cfb5427b9cfc66a9fa376150c1ddbc62115ae27cef72417eb959691396
```

delegation string

```json
nostr:delegation:477318cfb5427b9cfc66a9fa376150c1ddbc62115ae27cef72417eb959691396:kind=1&created_at>1674834236&created_at<1677426236
```

delegation token

```json
6f44d7fe4f1c09f3954640fb58bd12bae8bb8ff4120853c4693106c82e920e2b898f1f9ba9bd65449a987c39c0423426ab7b53910c0c6abfb41b30bc16e5f524
```

委托事件

```json
{
  "id": "e93c6095c3db1c31d15ac771f8fc5fb672f6e52cd25505099f62cd055523224f",
  "pubkey": "477318cfb5427b9cfc66a9fa376150c1ddbc62115ae27cef72417eb959691396",
  "created_at": 1677426298,
  "kind": 1,
  "tags": [
    [
      "delegation",
      "8e0d3d3eb2881ec137a11debe736a9086715a8c8beeeda615780064d68bc25dd", // 委托人的公钥
      "kind=1&created_at>1674834236&created_at<1677426236", // 条件
      "6f44d7fe4f1c09f3954640fb58bd12bae8bb8ff4120853c4693106c82e920e2b898f1f9ba9bd65449a987c39c0423426ab7b53910c0c6abfb41b30bc16e5f524" // delegation token
    ]
  ],
  "content": "Hello, world!",
  "sig": "633db60e2e7082c13a47a6b19d663d45b2a2ebdeaf0b4c35ef83be2738030c54fc7fd56d139652937cdca875ee61b51904a1d0d0588a6acd6168d7be2909d693"
}
```

#### content-warning

```json
["content-warning", "reason"] /* reason is optional */
```

使用户能够指定事件的内容是否需要读者批准才能显示。客户端可以隐藏内容，直到用户对其进行操作。

#### expiration

```json
["expiration", "1600000000"]
```
