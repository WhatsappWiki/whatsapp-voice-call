# 设备路由与通话密钥封装 · Device Routing and Call-Key Envelopes

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 用户寻址与设备加密

同一账户可以有多个来电设备，offer 为每个目标保留独立的通话材料。账户与设备身份的区别见[架构篇](01-architecture.md#zh-device-identity)。

主叫为接收方设备准备各自的随机 32 字节通话主秘密，通过设备之间的端到端会话发送；用户实际在哪个设备接听，就使用该设备对应的秘密建立通话。秘密只用于本次通话并保留在客户端内存中。[官方多设备架构说明](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/)

<a id="zh-device-fanout"></a>

### 单设备与 destination 形式

单设备结构可直接放置加密节点：

```xml
<offer call-id="CALL_ID" call-creator="CALLER_DEVICE_JID">
  <audio enc="opus" rate="16000" />
  <encopt keygen="2" />
  <enc v="2" type="msg">SIGNAL_CIPHERTEXT</enc>
</offer>
```

多设备结构为每个目标保留自己的密文：

```xml
<offer call-id="CALL_ID" call-creator="CALLER_DEVICE_JID">
  <audio enc="opus" rate="16000" />
  <encopt keygen="2" />
  <destination>
    <to jid="CALLEE_DEVICE_A_JID">
      <enc v="2" type="pkmsg">DEVICE_A_SIGNAL_CIPHERTEXT</enc>
    </to>
    <to jid="CALLEE_DEVICE_B_JID">
      <enc v="2" type="msg">DEVICE_B_SIGNAL_CIPHERTEXT</enc>
    </to>
  </destination>
</offer>
```

不同设备的 Signal 会话状态可能不同，因此同一次扇出中可以同时出现 `pkmsg` 与 `msg`。不能对一个设备加密后，把同一份密文无条件复制给其他设备。

接收侧根据自身设备身份选取对应 `<to>`。找不到本机目标、重复本机目标或结构出现歧义时需要进入错误处理；不能直接取第一个 enc 作为自己的材料。

<a id="zh-signal-envelope"></a>

### 通话材料如何进入 Signal 密文

通话密钥使用消息 protobuf 封装：外层 Message 的第 10 字段是 Call，Call 的第 1 字段是字节串 callKey。两层都是 length-delimited 字段。

```protobuf
message Call {
  optional bytes callKey = 1;
}

message Message {
  optional Call call = 10;
}
```

当内部仅包含 32 字节 callKey 时，未填充的最小消息为 `52 22 0a 20` 后接 32 字节材料。这里的十六进制前缀描述 protobuf 封装，不是可以直接放入最终 `<enc>` 的完整内容。

进入 Signal 加密前，消息还有业务填充：随机选择 `n=1..16`，在末尾附加 `n` 个值为 `n` 的字节。Signal 组件随后按设备会话执行加密，生成相应类型的序列化密文。

```text
设备对应的 callKey
    → Message.call.callKey 的 protobuf
    → 业务填充
    → 该设备的 Signal 会话加密
    → enc(v=2, type=实际密文类型, body=序列化密文)
```

`type="pkmsg"` 表示 PreKey 消息形式，`type="msg"` 表示普通会话消息形式。选择来自实际 Signal 加密结果，不能仅以“第一次拨打此联系人”推断。

若所用 Signal 适配层已经负责业务填充与去填充，调用者不应再次追加或删除。protobuf、填充和 Signal 消息是三个有序边界。

### 线上 enc 与引擎材料是两个边界

| 边界 | 内容 | 处理方式 |
| --- | --- | --- |
| 引擎发起加密前 | 原始通话材料 | 保留目标设备映射；正常通话主秘密为 32 字节 |
| 最终信令 `<enc>` | Signal 密文 | 带 `v`、`type`，长度随封装与会话类型变化 |
| Signal 解密后 | protobuf 和业务填充边界内的内容 | 按适配层职责去填充并解析 Message.call.callKey |
| 交回引擎 | 解密后的通话材料 | 重新构造引擎接收结构；不把 Signal 密文误当原始 key |

字段参考中 enc body 的 `1..32` 约束属于加密前或解密后的引擎材料读取边界。它不是线上 Signal 密文的最大长度；最终密文可以长于 32 字节。

`count` 与重试场景有关，其写入取决于节点所在分支。直接 offer 的材料节点和 destination 内的材料节点可以使用不同写入规则，不能把某一处 count 的存在推广到全部 enc。

<a id="zh-pending-offer"></a>

### 异步加密与待发送 offer

相关 PreKey 任务未完成时保留 pending offer，完成后按设备加密并替换 enc。回调回到通话队列，重新核对 call-id、目标设备、任务代次和 ending 状态，再提交写入。

等待期间可发生取消、重试或新通话。旧任务的成功结果不能继续发送；Signal 会话自身已提交的状态由其组件管理。

<a id="zh-device-selection"></a>

### 接听设备如何被选择

preaccept 可以暂时更新活动设备信息；正式设备选择由相应 accept、reject 或筛选处理分支决定。主叫选中一个设备后，保留其能力与通话材料，并向其他设备发送带目标列表的结束通知。

常见的其他设备结束原因包括 `accepted_elsewhere`、`rejected_elsewhere` 和 `screened_elsewhere`。这些原因表示同一通话已在别处处理，不应简单计作所有设备各自漏接。

设备选择按通话任务的实际处理顺序推进，不能把它等同于服务端一定采用“第一个网络包获胜”的仲裁。主设备与伴随设备的 reject/terminate 还有状态和原因条件；任意一个伴随设备的拒绝并不必然结束整个呼叫。

选定后的过滤与乱序处理见[状态篇](06-state-and-recovery.md#zh-device-filtering)。

### accept 中的其他安全材料

accept 可携带新 enc、证书指纹及 HMAC，扩展创建者身份路径还可携带签名。它们分别有自己的作用和适用条件。证书指纹不自动等于整个登录连接的服务器证书，身份签名也不意味着每个音频包都使用 Signal 签名。

语音媒体与通话安全架构，见[公开 Wiki 的语音通话目录](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#zh)。

[下一篇：地址协商与中继信息](05-transport-negotiation.md#zh)

[English](#en)

<a id="en"></a>

## English

[Home](../README.md#en) · [中文](#zh)

### User addressing and device encryption

An account can have multiple devices receiving a call, so an offer retains independent call material for each target. The distinction between account and device identity is explained in [architecture](01-architecture.md#en-device-identity).

The caller prepares a random 32-byte call master secret for each recipient device and sends it through device-to-device end-to-end sessions. The device on which the user answers uses its corresponding secret to establish the call. The secret is held in client memory and used only for that call. [Official multi-device architecture](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/)

<a id="en-device-fanout"></a>

### Single-device and destination forms

A single-device structure can carry the encrypted node directly:

```xml
<offer call-id="CALL_ID" call-creator="CALLER_DEVICE_JID">
  <audio enc="opus" rate="16000" />
  <encopt keygen="2" />
  <enc v="2" type="msg">SIGNAL_CIPHERTEXT</enc>
</offer>
```

A multi-device structure retains separate ciphertext for each target:

```xml
<offer call-id="CALL_ID" call-creator="CALLER_DEVICE_JID">
  <audio enc="opus" rate="16000" />
  <encopt keygen="2" />
  <destination>
    <to jid="CALLEE_DEVICE_A_JID">
      <enc v="2" type="pkmsg">DEVICE_A_SIGNAL_CIPHERTEXT</enc>
    </to>
    <to jid="CALLEE_DEVICE_B_JID">
      <enc v="2" type="msg">DEVICE_B_SIGNAL_CIPHERTEXT</enc>
    </to>
  </destination>
</offer>
```

Devices may have different Signal session states, so one fanout can contain both `pkmsg` and `msg`. Ciphertext encrypted for one device cannot be copied indiscriminately to the others.

The receiver selects the matching `<to>` using its own device identity. A missing local target, duplicate local targets or an ambiguous structure enters error handling; it does not simply select the first enc.

<a id="en-signal-envelope"></a>

### How call material becomes Signal ciphertext

The call key is wrapped in a message protobuf: field 10 of Message is Call, and field 1 of Call is the callKey byte string. Both fields are length-delimited.

```protobuf
message Call {
  optional bytes callKey = 1;
}

message Message {
  optional Call call = 10;
}
```

With only a 32-byte callKey inside, the minimal unpadded message is `52 22 0a 20` followed by the 32-byte material. This hexadecimal prefix describes protobuf framing, not the complete content of the final `<enc>`.

Before Signal encryption, application padding randomly chooses `n=1..16` and appends `n` bytes, each valued `n`. Signal then encrypts using the device session and produces serialized ciphertext of the corresponding type.

```text
callKey for the target device
    → Message.call.callKey protobuf
    → Application padding
    → Signal session encryption for that device
    → enc(v=2, type=actual ciphertext type, body=serialized ciphertext)
```

`type="pkmsg"` denotes a PreKey message; `type="msg"` denotes a regular session message. The choice comes from the actual Signal output, rather than whether this is the first call to that contact.

When the Signal adapter already adds and removes application padding, the caller must not repeat it. Protobuf encoding, padding and the Signal message are three ordered boundaries.

### Wire enc and engine material are separate boundaries

| Boundary | Content | Handling |
| --- | --- | --- |
| Before encryption at the engine | Raw call material | Preserve the target-device mapping; the normal call master secret is 32 bytes |
| Final signaling `<enc>` | Signal ciphertext | Carries v and type; length varies with envelope and session type |
| After Signal decryption | Content within the protobuf and application-padding boundaries | Remove padding according to adapter responsibility and parse Message.call.callKey |
| Handoff to the engine | Decrypted call material | Rebuild the receiving structure; do not treat Signal ciphertext as a raw key |

The `1..32` enc-body constraint in the field reference applies to engine material before encryption or after decryption. It does not cap the wire Signal ciphertext, which can exceed 32 bytes.

count relates to retry handling, and its emission depends on the node's branch. A direct offer material node and one inside destination may have different count-writing rules; presence in one location does not require it in every enc.

<a id="en-pending-offer"></a>

### Asynchronous encryption and pending offers

When related PreKey work is unfinished, retain the pending offer. Once ready, encrypt per device and replace enc. The callback returns to the call queue and rechecks call-id, target device, task generation and ending state before enqueueing the write.

Cancellation, retries or a new call can occur while waiting. An old task's successful result must not be sent. Committed Signal session state remains under the session component's control.

<a id="en-device-selection"></a>

### Selecting the answering device

preaccept can temporarily update active-device information. Formal selection follows the applicable accept, reject or screening branch. Once the caller selects a device, it retains that device's capabilities and call material and sends termination notifications with a destination list to the other devices.

Common reasons include accepted_elsewhere, rejected_elsewhere and screened_elsewhere. They indicate that the same call was handled elsewhere and should not automatically be recorded as an independent missed call on every device.

Selection follows actual call-task processing order; this does not imply that server arbitration always selects the first arriving network packet. Primary and companion devices have additional reject/terminate conditions based on state and reason, so a rejection from any companion does not necessarily end the entire call.

See [state handling](06-state-and-recovery.md#en-device-filtering) for filtering and reordering after selection.

### Other security material in accept

accept can carry a new enc, certificate fingerprint and HMAC. An extension-creator identity branch can also carry a signature. Each has its own purpose and conditions. A certificate fingerprint is not automatically the server certificate of the login connection, and an identity signature does not mean every audio packet receives a Signal signature.

For voice media and call security architecture, see the [public Wiki voice-calling directory](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#en).

[Next: Transport negotiation and relay information](05-transport-negotiation.md#en)

[中文](#zh)
