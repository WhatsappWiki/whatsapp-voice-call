# 通信架构与标识 · Architecture and Identifiers

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 通信层次

通话控制使用 call、offer、ack 和 receipt 等节点，具有 XMPP 风格。WhatsApp 的业务节点、二进制编码和确认规则按自身协议解释；标准 XMPP Core 用于理解 XML 流与 stanza 等基础概念。[XMPP Core](https://www.rfc-editor.org/rfc/rfc6120.html)

```text
用户动作 / 引擎事件 → 通话控制器
    → 设备寻址与必要的 Signal 加密
    → 节点写入队列 → 二进制协议树
    → Noise 加密通道 → 长度分帧 → 网络连接
```

接收时按相反方向解帧、处理 Noise、解码协议树，再分派通话节点。通话处理器检查身份和业务字段，完成必要的 Signal 解密后交给引擎。Noise 保护客户端到服务的连接；Signal 会话用于设备间端到端传递通话材料。

<a id="zh-identifiers"></a>

### 外层、内层与标识

```xml
<call to="PEER_JID" id="STANZA_ID">
  <offer call-id="CALL_ID" call-creator="CALLER_DEVICE_JID">
    <audio enc="opus" rate="16000" />
    <encopt keygen="2" />
    <enc v="2" type="pkmsg">SIGNAL_CIPHERTEXT</enc>
  </offer>
</call>
```

外层描述路由与当前信令，内层描述通话及其动作。`audio@enc` 是编解码标签，enc 子节点承载加密材料，具体封装见[设备与密钥](04-device-routing-and-keys.md#zh-signal-envelope)。

| 标识 | 作用域 | 用途 |
| --- | --- | --- |
| 外层 id | 单条信令 | 关联该条发送、ACK 与回执 |
| call-id | 一场通话 | 贯穿 offer、accept、transport、terminate |
| call-creator | 通话创建者设备 | 在发送方向改变后仍保持创建身份 |
| from / to | 当前路由 | 入站来源或出站目的，结合账户与设备核对 |
| 设备 JID | 账户的某台设备 | 密钥槽、扇出、接听设备选择和消息过滤 |
| transaction-id | 业务子流程 | 测量、更新或结束事务 |
| 本地任务代次 | 本地异步任务 | 阻止旧回调作用于新通话；不写入信令 |

call-id 为非空文本，接收结构最多保存 63 字节及终止符。通话 ID 与 stanza ID 分别生成，生成器不预设为 UUID。确认关联的例子见[确认语义](02-call-lifecycle.md#zh-confirmations)。

<a id="zh-device-identity"></a>

### PN、LID 与设备身份

PN 和 LID 可以是同一账户的不同寻址形式。可信映射先确定账户，再用设备 JID 确定实际收发设备，最后核对它是否属于本场通话及是否已被选为活动设备。

只比较用户部分会混合多个设备；只比较字符串又会遗漏合法别名。设备扇出与选择规则见[设备路由](04-device-routing-and-keys.md#zh-device-selection)。

<a id="zh-connection-lifecycle"></a>

### 连接与通话生命周期

登录连接可同时承载消息与通话控制。正常挂断释放本场通话的事务、计时器和媒体资源，消息连接继续使用。

连接恢复与通话恢复分别处理：重新建立 Noise 通道后，仍需确定旧通话是否有效，媒体状态也由对应通话重新协调。

[下一篇：一次呼叫的信令过程](02-call-lifecycle.md#zh)

[English](#en)

<a id="en"></a>

## English

[Home](../README.md#en) · [中文](#zh)

### Communication layers

Call control uses nodes such as call, offer, ack and receipt in an XMPP-style structure. WhatsApp business nodes, binary encoding and acknowledgment rules follow their own protocol. XMPP Core provides background on XML streams and stanzas. [XMPP Core](https://www.rfc-editor.org/rfc/rfc6120.html)

```text
User action / engine event → Call controller
    → Device addressing and Signal encryption where needed
    → Node write queue → Binary protocol tree
    → Noise channel → Length framing → Network connection
```

Reception reverses the framing, Noise and protocol-tree steps before dispatching call nodes. The call handler checks identity and business fields, performs Signal decryption where needed, and hands the result to the engine. Noise protects the client-to-service connection; Signal sessions deliver call material end to end between devices.

<a id="en-identifiers"></a>

### Outer envelope, inner node and identifiers

```xml
<call to="PEER_JID" id="STANZA_ID">
  <offer call-id="CALL_ID" call-creator="CALLER_DEVICE_JID">
    <audio enc="opus" rate="16000" />
    <encopt keygen="2" />
    <enc v="2" type="pkmsg">SIGNAL_CIPHERTEXT</enc>
  </offer>
</call>
```

The outer envelope describes routing and the current signaling transaction; the inner node describes the call and its action. `audio@enc` is a codec label, while the enc child carries encrypted material. See [devices and keys](04-device-routing-and-keys.md#en-signal-envelope) for its envelope.

| Identifier | Scope | Purpose |
| --- | --- | --- |
| Outer id | One stanza | Associates transmission, ACK and receipt for that stanza |
| call-id | One call | Continues across offer, accept, transport and terminate |
| call-creator | Call-creating device | Retains the creator identity when message direction changes |
| from / to | Current route | Incoming source or outgoing destination, checked against account and device identity |
| Device JID | One device of an account | Key slots, fanout, answering-device selection and message filtering |
| transaction-id | A business subprocedure | Measurement, update or termination transaction |
| Local task generation | A local asynchronous task | Prevents old callbacks from affecting a new call; never written to signaling |

call-id is nonempty text; the receiving structure stores at most 63 bytes plus a terminator. Call IDs and stanza IDs are generated separately, without assuming UUID generators. See [acknowledgment semantics](02-call-lifecycle.md#en-confirmations) for correlation examples.

<a id="en-device-identity"></a>

### PN, LID and device identity

PN and LID can be different addressing forms for one account. A trusted mapping first identifies the account; a device JID identifies the actual sending or receiving device. Processing then checks whether that device belongs to the call and whether it is the selected active device.

Comparing only the user portion merges multiple devices, while comparing only literal strings misses valid aliases. See [device routing](04-device-routing-and-keys.md#en-device-selection) for fanout and selection rules.

<a id="en-connection-lifecycle"></a>

### Connection and call lifetimes

A logged-in connection can carry both messaging and call control. Normal hangup releases the call's transactions, timers and media resources while messaging continues on the connection.

Connection recovery and call recovery are separate: after a Noise connection is re-established, the old call's validity must still be determined, and its media state must be coordinated by the corresponding call.

[Next: Call signaling lifecycle](02-call-lifecycle.md#en)

[中文](#zh)
