# 信令字段参考 · Signaling Field Reference

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

<a id="zh-field-notation"></a>

### 使用方法

本篇按需查阅，保留 16 类共享结构、14 类消息与确认内容、共 152 条字段记录。复用结构只列一次；外层封装与最终 Signal enc 属性单列。群组、视频等条件字段也保留在相应消息表中。

`@name` 是属性，`$body` 是字节内容，`[]` 是重复节点。消息先引用共享结构，再列专有字段。“解析／发送”表示该处理方向；接收要求与发送条件分别以最后一列为准。

| 必需性 | 解释 |
| --- | --- |
| 必需 | 当前结构适用时必须提供 |
| 条件必需 | 消息种类、父节点、方向或组合条件触发后必须提供 |
| 可选 | 可以省略；提供后仍按所列类型和规则校验 |
| 依分支 | 读取或使用由处理分支决定，具体行为见条件列 |

父节点可选，不代表其中所有属性也可省略。例如 encopt 存在时 keygen 必须可解析；net 存在时 medium 必须可解析。子结构失败与整条消息失败仍需分开：certificate 的某些属性错误只记录并继续外层流程。

常用入口：[音频](#zh-shared-audio) · [线上 enc](#zh-wire-enc) · [relay](#zh-shared-relay) · [offer ACK](#zh-message-offer_ack) · [accept ACK](#zh-message-accept_ack)。

### 外层封装

| 位置 | 字段 | 含义与处理 |
| --- | --- | --- |
| 出站 call | `@to` | 目的用户或设备 JID，由路由模式决定 |
| 入站 call | `@from` | 来源 JID；继续核对账户、设备与当前通话参与关系 |
| call | `@id` | 当前 stanza 的非空标识，用于确认关联 |
| call 的业务子节点 | 标签 | offer、accept、preaccept、reject、terminate、transport 等；先分派再解析 |
| ack | `@class` | 通话确认使用 call；回执确认使用 receipt |
| 通话 ack | `@id`、`@type` | 原信令 ID 与原业务标签；与待确认记录匹配 |
| 入站通话 ack | `@from` | 必须提供可解析的 JID |
| 通话 ack | `@error` | 整数，缺失默认 0；还要继续处理子节点错误 |
| receipt | `@to` / `@from`、`@id` | 接收者或来源，以及被回执的原 stanza ID |
| receipt 子节点 | `offer` / `accept` / `reject` | 对应被回执的信令种类；具体接收分派按支持的种类处理 |

receipt 写入形式不要求外层 `type="call"`。原生通话 receipt 分派中，offer 与 accept 进入各自处理；未知 receipt 子标签可被忽略，因此“可以写出 reject receipt”和“所有接收层均处理 reject receipt”是不同条件。

<a id="zh-wire-enc"></a>

### 线上 Signal enc

| 字段 | 内容 | 行为 |
| --- | --- | --- |
| `enc@v` | 十进制文本 2 | 加密封装版本，与客户端发布版本及 call-id 无关 |
| `enc@type` | msg 或 pkmsg | 由实际 Signal 密文类型决定 |
| `enc@count` | 条件计数 | 非零计数可以写出；具体位置和重试分支决定是否提供 |
| `enc/$body` | Signal 序列化密文 | 真实二进制 body；不是原始 32 字节 callKey，也不是示例标签文本 |

下面的[共享 enc 表](#zh-shared-enc)单列引擎内的材料约束。线上 body 按本节解释；封装流程见[protobuf、填充与 Signal](04-device-routing-and-keys.md#zh-signal-envelope)。

### 共享结构目录

- [共同属性：`common`](#zh-shared-common)
- [音频：`audio`](#zh-shared-audio)
- [候选和所选端点：`te`](#zh-shared-te)
- [密钥生成选项：`encopt`](#zh-shared-encopt)
- [引擎材料边界：`enc`](#zh-shared-enc)
- [能力信息：`capability`](#zh-shared-capability)
- [网络类型：`net`](#zh-shared-net)
- [证书指纹：`certificate`](#zh-shared-certificate)
- [参与者元数据：`metadata`](#zh-shared-metadata)
- [通话配置：`voip_settings`](#zh-shared-voip_settings)
- [中继令牌：`relay_token`](#zh-shared-relay_token)
- [中继信息：`relay`](#zh-shared-relay)
- [中继端点属性：`relay_te`](#zh-shared-relay_te)
- [中继测量端点：`relaylatency_te`](#zh-shared-relaylatency_te)
- [通话摘要：`call_summary`](#zh-shared-call_summary)
- [摘要中的参与者：`summary_participant`](#zh-shared-summary_participant)

### 消息目录

- [`offer`：呼叫提议](#zh-message-offer)
- [`accept`：接受通话](#zh-message-accept)
- [`preaccept`：预接受](#zh-message-preaccept)
- [`reject`：拒绝通话](#zh-message-reject)
- [`terminate`：结束通话](#zh-message-terminate)
- [`transport`：传输更新](#zh-message-transport)
- [`relaylatency`：中继测量](#zh-message-relaylatency)
- [`ready`：就绪通知](#zh-message-ready)
- [`relay_info_update`：中继信息更新](#zh-message-relay_info_update)
- [`offer_ack`：offer 的 ACK](#zh-message-offer_ack)
- [`accept_ack`：accept 的 ACK](#zh-message-accept_ack)
- [`relaylatency_ack`：relaylatency 的 ACK](#zh-message-relaylatency_ack)
- [`offer_receipt`：offer 的接收回执](#zh-message-offer_receipt)
- [`accept_receipt`：accept 的接收回执](#zh-message-accept_receipt)

### 共享结构

<a id="zh-shared-common"></a>

#### 共同属性：`common`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@call-id` | 非空文本；64字节缓冲区；文本属性 | 解析及发送 | 必需 | 普通消息进入分支解析前要求 call-id 非空；offer 也单独检查非空。最多存63字节文本及终止符；内存结构长度不等于网络报文长度。 |
| `@call-creator` | 设备 JID 文本；JID 类型属性 | 解析及发送 | 条件必需 | 发送时使用通话创建者设备；接收时允许省略，存在时必须是可解析的 JID。 |

<a id="zh-shared-audio"></a>

#### 音频：`audio`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@enc` | 信令中的编解码器标签；文本属性 | 解析及发送 | 条件必需 | 发送固定写 opus；accept／preaccept 的音频节点要求此属性存在。offer 的采样率遍历不单独校验 enc。opus 标签不固定实际使用的编解码路径；voip_settings 的编码配置另行控制 MLow 分支。 |
| `@rate` | 采样率，int32；十进制文本 | 解析及发送 | 条件必需 | accept／preaccept 要求采样率可解析；发送值从8000、16000、24000、48000中选择。采样率位为1／2／4／8；offer 可有多个 audio，accept 发送时选最高有效采样率。 |

<a id="zh-shared-te"></a>

#### 候选和所选端点：`te`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `$body` | 端点字节串；原始字节：IPv4+端口6字节；IPv6+端口18字节；也可解析二者拼接的24字节形式 | 解析及发送 | 必需 | 每个 te／te2／rte 端点。地址内容是二进制字节，不是可打印的 IP 文本；端口沿用端点结构中的原始字节。 |
| `@priority` | 存为 uint8；按 int32 解析，默认0；十进制文本 | 解析及发送 | 可选 | 普通候选地址列表。列表最多20项；单个端点转换失败不一定使整个列表失败，列表仍保留声明的项数。 |

<a id="zh-shared-encopt"></a>

#### 密钥生成选项：`encopt`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@keygen` | int32；十进制文本 | 解析及发送 | 条件必需 | encopt 缺失时默认2；存在时 keygen 必须能解析。 |

<a id="zh-shared-enc"></a>

#### 引擎材料边界：`enc`

此处两条记录描述加密前／解密后的引擎结构。`1..32` 不限制线上 Signal 密文长度。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `$body` | 加密前／解密后的密钥材料，1..32字节；原始字节 | 解析及发送 | 依分支 | accept 的密钥材料节点可选；offer 是否读取密钥受其处理分支控制。1..32字节约束用于加密前／解密后的密钥材料；最终 enc 内容经过 Signal 加密，长度会改变。 |
| `@count` | int32；十进制文本 | 发送 | 可选 | offer 直接密钥节点写 count；destination 和 accept 的对应节点不写 count。 |

<a id="zh-shared-capability"></a>

#### 能力信息：`capability`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@ver` | 正 int32；十进制文本 | 解析及发送 | 依分支 | capability 的版本及内容有效时。 |
| `$body` | 1..64字节；原始字节 | 解析及发送 | 可选 | 仅在版本大于0且内容长1..64字节时发送。缺失或不可用的版本／长度将能力值清为 version=−1、len=0；ver 数值解析失败另行处理。 |

<a id="zh-shared-net"></a>

#### 网络类型：`net`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@medium` | int32；十进制文本 | 解析及发送 | 条件必需 | net 缺失时 medium 默认3；节点存在时 medium 必须可解析。 |
| `@protocol` | int32；十进制文本 | 解析及发送 | 可选 | 仅 transport 使用；默认−1，发送省略−1。 |
| `@health_status` | int32；十进制文本 | 解析及发送 | 可选 | 发送仅用于 transport-message-type=11；接收只要 net 存在就尝试读取。 |

<a id="zh-shared-certificate"></a>

#### 证书指纹：`certificate`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@algorithm` | 文本，64字节缓冲区；文本属性 | 解析及发送 | 条件必需 | certificate 节点存在时要求此属性。 |
| `@fingerprint` | 文本，256字节缓冲区；文本属性 | 解析及发送 | 条件必需 | certificate 节点存在时要求此属性。 |
| `@hmac` | 发送方提供32字节的材料；标准 Base64，带 padding | 解析及发送 | 可选 | 仅 accept 的 certificate 使用；发送由 HMAC 存在标志控制。解码缓冲区容量为32字节；该分支检查解码状态并设置存在标志，不额外要求结果恰好32字节。 |

<a id="zh-shared-metadata"></a>

#### 参与者元数据：`metadata`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@peer_abtest_bucket` | 文本，128字节缓冲区；文本属性 | 解析及发送 | 可选 | 对应内容存在标志为真时。 |
| `@peer_abtest_bucket_id_list` | 文本，256字节缓冲区；文本属性 | 解析及发送 | 可选 | 对应内容存在标志为真时。 |
| `@xrtt_ms` | int32，默认0；十进制文本 | 解析 | 可选 | metadata 存在时读取。此 metadata 发送分支不写 xrtt_ms。 |

<a id="zh-shared-voip_settings"></a>

#### 通话配置：`voip_settings`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@uncompressed` | 字面值1；文本属性 | 解析 | 条件必需 | 此路径预期未压缩形式；属性默认0。此配置路径处理未压缩文本；压缩形式进入错误记录和断言分支。 |
| `@jid` | 设备 JID；文本属性 | 解析 | 可选 | 按每份配置绑定对应设备。 |
| `$body` | JSON 文本配置树；节点字节内容按文本解释 | 解析 | 条件必需 | 成功的一对一 offer ACK 要求至少一份 voip_settings。内容按 JSON 配置树解析；树内配置项分别交给媒体和链路策略处理。 |

<a id="zh-shared-relay_token"></a>

#### 中继令牌：`relay_token`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `$body` | 0..512字节；原始字节 | 解析 | 必需 | 列表有1..8个节点；relay 存在时要求 token 列表。 |

<a id="zh-shared-relay"></a>

#### 中继信息：`relay`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `token[]` | 中继令牌列表；节点列表，见中继令牌结构 | 解析 | 必需 | relay 节点存在时。 |
| `auth_token[]` | 中继认证令牌列表；节点列表，限制同中继令牌 | 解析 | 可选 | 存在 auth_token 节点时。 |
| `(te[]\|te2[])` | 中继端点列表；节点列表；两种端点族中恰有一种非空 | 解析 | 必需 | relay 节点存在时。两种端点族同时存在、均缺失、端点内容为空或没有可用地址都会失败。 |
| `key/$body` | 中继控制密钥，1..24字节；原始字节 | 解析 | 可选 | relay 的 key 子节点存在时。该密钥属于中继控制，与端到端通话密钥用途不同。 |
| `hbh_key/$body` | 解码后30字节；标准 Base64 | 解析 | 条件必需 | hbh_key 存在时解码；warp_mi_tag_len 非0时必须提供有效材料。必须解码出恰好30字节；解码或长度失败会清除材料并失败。 |
| `@transaction-id` | int32；十进制文本 | 解析 | 可选 | relay 存在时；默认−1。 |
| `@self_pid` | 按 int32 读取后存为 uint16；十进制文本 | 解析 | 可选 | relay 存在时；默认−1。 |
| `@uuid` | 文本，17字节缓冲区；文本属性 | 解析 | 可选 | relay 存在时；默认空文本。 |
| `@participant_uuid` | 文本，9字节缓冲区；文本属性 | 解析 | 可选 | relay 存在时；默认空文本。 |
| `@warp_mi_tag_len` | int32；十进制文本 | 解析 | 可选 | relay 存在时；默认0。 |
| `@prefer_relay` | int32；非负值转为真；十进制文本 | 解析 | 可选 | relay 存在时；默认−1。 |
| `@peer_dl` | int32；十进制文本 | 解析 | 可选 | relay 存在时；默认0。 |
| `@prefer_ipv6` | int32；非负值转为真；十进制文本 | 解析 | 可选 | relay 存在时；默认−1。 |
| `@smart_transport_signal` | int32；十进制文本 | 解析 | 可选 | relay 存在时；默认0。 |

<a id="zh-shared-relay_te"></a>

#### 中继端点属性：`relay_te`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `$body` | 二进制端点；见端点结构 | 解析 | 必需 | 每个中继 te／te2 端点。 |
| `@relay_id` | uint64；十进制文本 | 解析 | 可选 | 中继端点；默认端点序号。属性按 uint64 解析；加入中继列表时使用其低32位。 |
| `@token_id` | int32；十进制文本 | 解析 | 可选 | 中继端点；普通读取默认0，另以−1区分是否提供此值。 |
| `@xtoken_id` | int32；十进制文本 | 解析 | 可选 | 中继端点；默认−1。 |
| `@auth_token_id` | int32；十进制文本 | 解析 | 可选 | 中继端点；默认−1。 |
| `@xrtt_ms` | int32；十进制文本 | 解析 | 可选 | 中继端点；默认100。 |
| `@protocol` | int32；十进制文本 | 解析 | 可选 | 中继端点；默认0。 |
| `@c2r_rtt` | int32；十进制文本 | 解析 | 可选 | 中继端点；默认−1。 |
| `@max_peer_c2r_rtt` | int32；十进制文本 | 解析 | 可选 | 中继端点；默认−1。 |
| `@relay_name` | 文本；文本属性 | 解析 | 可选 | 中继端点；默认空文本。 |
| `@domain_name` | 文本；文本属性 | 解析 | 可选 | 中继端点；默认空文本。 |

<a id="zh-shared-relaylatency_te"></a>

#### 中继测量端点：`relaylatency_te`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `$body` | 二进制端点；见端点结构 | 解析及发送 | 必需 | relaylatency 内的 te 节点。 |
| `@latency` | int32；十进制文本 | 解析及发送 | 条件必需 | 可用测量项必须能解析 latency 或 xlatency。 |
| `@xlatency` | int32；十进制文本 | 解析及发送 | 条件必需 | 替代延迟形式；latency 缺失或不能解析时读取。 |
| `@ul_bw` | int32；十进制文本 | 解析及发送 | 可选 | 发送仅写非0值；接收默认0。 |
| `@dl_bw` | int32；十进制文本 | 解析及发送 | 可选 | 发送仅写非0值；接收默认0。 |
| `@is_favored` | 布尔值；十进制文本 | 解析及发送 | 可选 | 发送仅写真；接收默认假。 |
| `@relay_name` | 文本，48字节缓冲区；文本属性 | 解析及发送 | 可选 | 发送仅写非空文本；接收默认空文本。 |
| `@conn_id` | 发送方提供8字节材料；标准 Base64，带 padding | 解析及发送 | 可选 | 发送仅写非0连接标识；解码失败将 conn_id 清零。解码容量为8字节；解码成功本身不额外保证结果恰好8字节。 |

<a id="zh-shared-call_summary"></a>

#### 通话摘要：`call_summary`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@call-creator` | 用户 JID；文本属性 | 解析 | 必需 | call_summary 存在时。 |
| `@call-id` | 文本，复制到64字节缓冲区；文本属性 | 解析 | 必需 | call_summary 存在时。 |
| `@media` | 映射为媒体类型的文本；文本属性 | 解析 | 必需 | call_summary 存在时。 |
| `@call_duration` | uint32；十进制文本 | 解析 | 必需 | call_summary 存在时。 |
| `participant[]\|user[]+bot[]` | 参与者摘要列表；节点列表，最多256项 | 解析 | 可选 | participant 兼容形式优先于 user／bot 形式。 |

<a id="zh-shared-summary_participant"></a>

#### 摘要中的参与者：`summary_participant`

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@jid` | 用户 JID；文本属性 | 解析 | 必需 | 每个摘要条目。 |
| `@state` | 参与者状态文本；文本属性 | 解析 | 必需 | 必须能映射为非0参与者状态。 |
| `@user_pn` | 文本；文本属性 | 解析 | 可选 | user_pn／username 由 LID 身份分支控制；其余属性可选。 |
| `@username` | 文本；文本属性 | 解析 | 可选 | user_pn／username 由 LID 身份分支控制；其余属性可选。 |
| `@push_name` | 文本；文本属性 | 解析 | 可选 | user_pn／username 由 LID 身份分支控制；其余属性可选。 |
| `@account_kind` | 文本或由文本映射的枚举；文本属性 | 解析 | 可选 | user_pn／username 由 LID 身份分支控制；其余属性可选。 |
| `@rering_ts` | uint64，默认0；十进制文本 | 解析 | 可选 | user_pn／username 由 LID 身份分支控制；其余属性可选。 |

### 各消息的字段

`offer_ack`、`accept_ack` 等是本表的分类名，分别描述 `<ack class="call" type="offer">` 等节点的内容。`offer_receipt` 表示 `<receipt><offer ... /></receipt>`，这些分类名都不应直接写成线上标签。

<a id="zh-message-offer"></a>

#### `offer`：呼叫提议

共同属性：[common](#zh-shared-common)。

可复用的子结构：[`audio[]`](#zh-shared-audio)、[`te[]`](#zh-shared-te)、[`net`](#zh-shared-net)、[`encopt`](#zh-shared-encopt)、[`enc`](#zh-shared-enc)、[`capability`](#zh-shared-capability)、[`certificate`](#zh-shared-certificate)、[`metadata`](#zh-shared-metadata)、[`voip_settings`](#zh-shared-voip_settings)、[`relay`](#zh-shared-relay)、[`rte`](#zh-shared-te)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@caller_pn` | JID；文本属性 | 解析 | 可选 | LID 主叫身份元数据。 |
| `@username` | 文本；文本属性 | 解析及发送 | 可选 | 对应源文本非空时。 |
| `@group-jid` | 群组 JID；文本属性 | 解析及发送 | 可选 | 群组通话上下文。 |
| `@scheduled-id` | 文本；文本属性 | 解析及发送 | 可选 | 预约通话上下文。 |
| `@lightweight` | 布尔值，真值文本为1；文本属性 | 解析及发送 | 可选 | 轻量通话。 |
| `@lightweight-key` | 文本，11字节缓冲区；文本属性 | 解析及发送 | 可选 | 轻量通话键非空时。 |
| `@is_call_ended` | 布尔值，真值文本为1；文本属性 | 解析及发送 | 可选 | 迟到或已结束通话的 offer。 |
| `@user_redial_count` | int32；十进制文本 | 发送 | 可选 | 源值非0时发送。 |
| `@device_class` | 十进制文本转换后存为 int16；十进制文本 | 解析及发送 | 可选 | 发送仅写非0值；接收默认0。结果存为 int16。一种接收路径直接按十进制转换；另一种先将转换结果限制在有符号 int32 范围，再存入 int16。 |
| `audio[]` | 音频节点列表；见音频结构 | 解析及发送 | 可选 | 按采样率位逐项发送 audio；offer 接收时汇总采样率。 |
| `screenable/@version` | uint32，默认0；十进制文本 | 解析及发送 | 可选 | 支持来电筛选时。 |
| `@screenable` | 布尔值，真值文本为1；文本属性 | 解析 | 可选 | 兼容形式的来电筛选属性。 |
| `self_info` | 容器节点；包含 capability／video／metadata 子节点 | 发送 | 可选 | 启用 self_info 容器时。 |
| `silence/@reason` | 由文本映射的静音原因；文本属性 | 解析及发送 | 条件必需 | 原因可映射且非空；silence 存在而原因空会失败。 |
| `silence/@is_first_wave` | 布尔值，真值文本为1；文本属性 | 解析 | 可选 | 静音原因处于首轮提示对应分支时。 |
| `session/token/$body` | 会话令牌字节串；原始字节 | 发送 | 可选 | 源字节串长度大于0时。 |
| `caller_metadata/dialer` | 标记节点；空子节点 | 发送 | 可选 | dialer 标志为真时。 |
| `terminate_hint/@terminate_reason` | 原因文本；文本属性 | 解析及发送 | 可选 | is_call_ended 为真且提供结束提示时。 |
| `@terminate_reason` | 兼容形式的原因文本；文本属性 | 解析 | 可选 | is_call_ended 为真但没有 terminate_hint 子节点时。 |
| `terminate_hint/@terminate_transaction_id` | int32，默认−1；十进制文本 | 解析 | 可选 | 结束提示存在时。 |
| `terminate_hint/call_summary` | 通话摘要树；见通话摘要结构 | 解析 | 可选 | 结束提示存在时。 |
| `@version` | 字面值2；文本属性 | 发送 | 可选 | 群组 offer 使用信令版本2模式时。 |
| `@action` | upgrade／create_agc／create_lgc／create_vc；文本属性 | 发送 | 可选 | 信令版本2，操作选择值为1..4时。 |
| `destination/to[]/@jid` | 设备 JID；JID 类型属性 | 发送 | 可选 | 目标设备数大于0时。 |
| `destination/to[]/enc` | 按设备选择的密钥字节串；见 enc；最终信令携带端到端加密后的内容 | 发送 | 可选 | 按目标设备分发时。 |
| `group_info` | 群组信息树；条件结构子树 | 解析及发送 | 可选 | 群组通话上下文。该容器承载群组参与者上下文；普通一对一语音路径不要求它。 |
| `invite` | 邀请信息树；条件结构子树 | 发送 | 可选 | 有额外受邀者时。 |
| `bot_info\|bot` | 机器人上下文；条件结构子树 | 发送 | 可选 | 有机器人参与者或选项时。 |
| `uploadfieldstat\|dontuploadfieldstat` | 遥测开关标记；空节点 | 解析 | 可选 | uploadfieldstat 优先；两种标记均缺失时默认允许上传统计。 |
| `userrate/@interval` | int32；十进制文本 | 解析 | 可选 | userrate 缺失时为−1；节点存在而 interval 缺失或不能解析时为86400。 |
| `client/@self_bad_asn` | int32 转为布尔值；十进制文本 | 解析 | 可选 | client 存在时；默认0。 |

<a id="zh-message-accept"></a>

#### `accept`：接受通话

共同属性：[common](#zh-shared-common)。

可复用的子结构：[`audio`](#zh-shared-audio)、[`te[]`](#zh-shared-te)、[`net`](#zh-shared-net)、[`encopt`](#zh-shared-encopt)、[`enc`](#zh-shared-enc)、[`capability`](#zh-shared-capability)、[`certificate`](#zh-shared-certificate)、[`metadata`](#zh-shared-metadata)、[`relay`](#zh-shared-relay)、[`rte`](#zh-shared-te)、[`voip_settings`](#zh-shared-voip_settings)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `audio` | 音频节点；见音频结构 | 解析及发送 | 必需 | 普通 accept。 |
| `@device_class` | 十进制文本转换后存为 int16；十进制文本 | 解析及发送 | 可选 | 发送仅写非0源值；接收默认0。结果存为 int16。一种接收路径直接按十进制转换；另一种先将转换结果限制在有符号 int32 范围，再存入 int16。 |
| `@retry` | 字面值1；文本属性 | 发送 | 可选 | 重试标志为真时。 |
| `ext_creator_identity_key_sig/$body` | 签名字节串；标准 Base64 文本作为节点内容 | 发送 | 可选 | 扩展签名长度大于0时。该签名用于扩展创建者身份分支，与普通一对一通话密钥用途不同。 |
| `relaylatency/te` | 所选中继的测量结果；见中继测量端点结构 | 解析及发送 | 可选 | 上行或下行带宽至少一项非0时发送。 |
| `video` | 视频参数；条件结构子树 | 解析及发送 | 可选 | 有视频节点时。纯语音路径省略此视频子树。 |

<a id="zh-message-preaccept"></a>

#### `preaccept`：预接受

共同属性：[common](#zh-shared-common)。

可复用的子结构：[`audio`](#zh-shared-audio)、[`encopt`](#zh-shared-encopt)、[`capability`](#zh-shared-capability)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `audio` | 音频节点；见音频结构 | 解析及发送 | 必需 | 普通 preaccept。 |
| `screenable/@version` | uint32，默认0；十进制文本 | 解析及发送 | 可选 | 来电筛选版本非0时。 |
| `video` | 视频参数；条件结构子树 | 解析及发送 | 可选 | 有视频节点时。 |

<a id="zh-message-reject"></a>

#### `reject`：拒绝通话

共同属性：[common](#zh-shared-common)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@reason` | 原因文本，128字节缓冲区；文本属性 | 解析及发送 | 可选 | 发送仅写非空；接收复制文本，默认空。 |
| `@count` | uint32；十进制文本 | 解析及发送 | 可选 | 发送始终写入；接收默认0。 |
| `registration/$body` | 恰好4字节；原始字节 | 解析及发送 | 条件必需 | reason 为 enc 时发送；registration 存在时要求恰好4字节。 |

<a id="zh-message-terminate"></a>

#### `terminate`：结束通话

共同属性：[common](#zh-shared-common)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@reason` | 文本，128字节缓冲区；文本属性 | 解析及发送 | 可选 | 发送仅写非空文本。 |
| `@lightweight-key` | 文本；文本属性 | 发送 | 可选 | 发送仅写非空文本。 |
| `@joinable` | 布尔值，真值文本为1；文本属性 | 解析 | 可选 | 接收与字面值1比较。 |
| `@transaction-id` | int32，默认−1；十进制文本 | 解析 | 可选 | 仅接收侧读取。 |
| `call_summary` | 通话摘要树；见通话摘要结构 | 解析 | 可选 | 子节点存在时。 |
| `destination/to[]/@jid` | 设备 JID；JID 类型属性 | 发送 | 可选 | 目标设备数大于0时。 |
| `@duration` | 正 int32 计数；十进制文本 | 发送 | 可选 | 仅在源计数大于0时发送。沿用通话统计源计数，序列化层不转换单位。 |
| `@audio_duration` | 正 int32 计数；十进制文本 | 发送 | 可选 | 仅在源计数大于0时发送。沿用通话统计源计数，序列化层不转换单位。 |
| `@video_duration` | 正 int32 计数；十进制文本 | 发送 | 可选 | 仅在源计数大于0时发送。沿用通话统计源计数，序列化层不转换单位。 |

<a id="zh-message-transport"></a>

#### `transport`：传输更新

共同属性：[common](#zh-shared-common)。

可复用的子结构：[`te[]|te2[]`](#zh-shared-te)、[`net`](#zh-shared-net)、[`rte`](#zh-shared-te)、[`certificate`](#zh-shared-certificate)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@transport-message-type` | int32，默认0；十进制文本 | 解析及发送 | 可选 | 发送始终写入；类型2使用 te2，其余类型使用 te。 |
| `@p2p-cand-round` | int32，默认0；十进制文本 | 解析及发送 | 可选 | 发送仅写非0值。 |
| `@ice-ufrag` | 文本，32字节缓冲区；文本属性 | 解析及发送 | 条件必需 | 类型13时接收要求属性存在；发送仅写非空文本。 |
| `@ice-pwd` | 文本，128字节缓冲区；文本属性 | 解析及发送 | 条件必需 | 类型13时接收要求属性存在；发送仅写非空文本。 |

<a id="zh-message-relaylatency"></a>

#### `relaylatency`：中继测量

共同属性：[common](#zh-shared-common)。

可复用的子结构：[`relaylatency_te[]`](#zh-shared-relaylatency_te)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@transaction-id` | int32，默认−1；十进制文本 | 解析及发送 | 可选 | 发送省略−1。 |
| `te[]` | 最多8项中继测量；见中继测量端点结构 | 解析及发送 | 必需 | 接收至少需要1个可用测量项，最多存8项；未达到8项时，遇到 latency 和 xlatency 均不能解析的条目会使消息失败。 |
| `destination/to[]/@jid` | 设备 JID；JID 类型属性 | 发送 | 可选 | 目标设备数大于0时。 |

<a id="zh-message-ready"></a>

#### `ready`：就绪通知

共同属性：[common](#zh-shared-common)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@type` | int32，默认0；十进制文本 | 解析 | 可选 | 缺失、不能解析或超出 int32 范围时回退0。 |

<a id="zh-message-relay_info_update"></a>

#### `relay_info_update`：中继信息更新

共同属性：[common](#zh-shared-common)。

可复用的子结构：[`relay`](#zh-shared-relay)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `relay` | 中继结构；见中继结构 | 解析 | 可选 | relay 为可选节点；缺失允许通过此解析分支。 |

<a id="zh-message-offer_ack"></a>

#### `offer_ack`：offer 的 ACK

可复用的子结构：[`relay`](#zh-shared-relay)、[`voip_settings`](#zh-shared-voip_settings)、[`rte`](#zh-shared-te)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `(group_info\|relay\|error[0])/@call-id` | 非空文本；文本属性 | 解析 | 必需 | 从 group_info、relay、首个 error 依次选择；所选 call-id 不得为空。 |
| `relay` | 中继结构；见中继结构 | 解析 | 条件必需 | 成功的一对一 ACK；错误和群组分支另行处理。 |
| `voip_settings[]` | 配置树；见通话配置结构 | 解析 | 条件必需 | 成功的一对一 ACK。 |
| `error[]/@jid` | 用户／设备 JID；文本属性 | 解析 | 条件必需 | 每个错误条目。 |
| `error[]/@code` | uint32，默认外层错误码；十进制文本 | 解析 | 可选 | 每个错误条目；列表最多64项。 |
| `group_info` | 群组信息树；条件子树 | 解析 | 可选 | 群组通话 ACK。 |
| `user` | 用户信息树；条件子树 | 解析 | 可选 | 服务端返回参与者上下文时。 |
| `@joinable` | 字面值1；文本属性 | 解析 | 可选 | 被选为 call-id 来源的子节点。 |
| `screenable/@action` | automatic 标记；文本属性 | 解析 | 可选 | screenable 子节点存在时。 |

<a id="zh-message-accept_ack"></a>

#### `accept_ack`：accept 的 ACK

可复用的子结构：[`relay`](#zh-shared-relay)、[`voip_settings`](#zh-shared-voip_settings)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `accept/@call-id` | 非空文本；文本属性 | 解析 | 条件必需 | 成功且没有 waiting_room 时；缺失 accept 将按忽略处理。 |
| `relay` | 中继结构；见中继结构 | 解析 | 条件必需 | 成功且没有 waiting_room 时。 |
| `voip_settings[]` | 配置树；见通话配置结构 | 解析 | 可选 | 存在时解析；accept ACK 允许缺失此配置。 |
| `guest/@is_guest_eligible` | 布尔值；文本属性 | 解析 | 可选 | guest 子节点存在时。 |
| `waiting_room` | 等候室信息树；条件子树 | 解析 | 可选 | 服务端等候室分支。 |
| `error/@call-id` | 非空文本；文本属性 | 解析 | 条件必需 | 外层 ACK 错误码非0时。 |
| `error/@jid` | 设备 JID；文本属性 | 解析 | 条件必需 | 外层 ACK 错误码非0时。 |

<a id="zh-message-relaylatency_ack"></a>

#### `relaylatency_ack`：relaylatency 的 ACK

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `relaylatency/@call-id` | 非空文本；文本属性 | 解析 | 条件必需 | relaylatency 明确携带 call-id 属性时不得为空；缺失节点或属性时支持回退分支。 |
| `$outer.error` | int32；由外层 ACK 元数据提供 | 解析 | 可选 | 保存外层错误状态，不从子节点内容读取。 |

<a id="zh-message-offer_receipt"></a>

#### `offer_receipt`：offer 的接收回执

共同属性：[common](#zh-shared-common)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@call-id` | 非空文本；文本属性 | 解析 | 必需 | receipt 的 call-id 不得为空。offer 与 accept 的 receipt 各自进入独立消息类型。 |
| `@call-creator` | 设备 JID；文本属性 | 解析 | 可选 | 存在时必须能解析。 |

<a id="zh-message-accept_receipt"></a>

#### `accept_receipt`：accept 的接收回执

共同属性：[common](#zh-shared-common)。

| 字段 | 类型与编码 | 方向 | 必需性 | 条件、默认与行为 |
| --- | --- | --- | --- | --- |
| `@call-id` | 非空文本；文本属性 | 解析 | 必需 | receipt 的 call-id 不得为空。offer 与 accept 的 receipt 各自进入独立消息类型。 |
| `@call-creator` | 设备 JID；文本属性 | 解析 | 可选 | 存在时必须能解析。 |

<a id="zh-body-encoding"></a>

### body 与文本编码

Base64 只用于明确以文本携带材料的字段，例如 hbh_key、certificate hmac 和 conn_id。endpoint、Signal enc 与 token 按原始字节处理；日志展示编码不改变线上结构。

[阅读主线：设备路由与通话密钥封装](04-device-routing-and-keys.md#zh)

[English](#en)

<a id="en"></a>

## English

[Home](../README.md#en) · [中文](#zh)

<a id="en-field-notation"></a>

### How to use this reference

This reference retains all 152 field records across 16 shared structures and 14 message or acknowledgment families. Shared structures appear once; outer envelopes and final Signal enc attributes are listed separately. Conditional group, video and other fields remain in their corresponding message tables.

`@name` denotes an attribute, `$body` byte content and `[]` repeated nodes. Messages first reference shared structures, then list their own fields. Parse/send denotes the processing direction; reception requirements and emission conditions follow the final column.

| Requiredness | Meaning |
| --- | --- |
| Required | Must be supplied when the current structure applies |
| Conditionally required | Required when the message kind, parent node, direction or combination triggers the stated condition |
| Optional | May be omitted; supplied values still follow the stated type and validation rules |
| Branch-dependent | Reading or use is determined by the handler branch; see the condition column |

An optional parent does not make every attribute inside it optional. For example, keygen must parse when encopt exists, and medium must parse when net exists. A substructure failure is also distinct from whole-message failure: some certificate attribute errors are logged while outer processing continues.

Common entries: [Audio](#en-shared-audio) · [Wire enc](#en-wire-enc) · [Relay](#en-shared-relay) · [Offer ACK](#en-message-offer_ack) · [Accept ACK](#en-message-accept_ack).

### Outer envelope

| Location | Field | Meaning and handling |
| --- | --- | --- |
| Outgoing call | `@to` | Destination user or device JID, selected by routing mode |
| Incoming call | `@from` | Source JID; also check account, device and call membership |
| call | `@id` | Nonempty identifier of the current stanza, used for acknowledgment correlation |
| Business child of call | Tag | offer, accept, preaccept, reject, terminate, transport and others; dispatch before branch parsing |
| ack | `@class` | call for call acknowledgments; receipt for receipt acknowledgments |
| Call ack | `@id`, `@type` | Original signal ID and business tag, matched to a pending record |
| Incoming call ack | `@from` | A valid JID is required |
| Call ack | `@error` | Integer, default 0; child errors must also be processed |
| receipt | `@to` / `@from`, `@id` | Recipient or source and the original stanza ID being receipted |
| Receipt child | `offer` / `accept` / `reject` | Kind of receipted signal; receiving dispatch follows its supported kinds |

The receipt writer does not require outer type=call. Native call-receipt dispatch handles offer and accept separately and may ignore unknown tags. Being able to write a reject receipt therefore does not establish support in every receiving layer.

<a id="en-wire-enc"></a>

### Wire Signal enc

| Field | Content | Behavior |
| --- | --- | --- |
| `enc@v` | Decimal text 2 | Encryption-envelope version, independent of client releases and call-id |
| `enc@type` | msg or pkmsg | Determined by the actual Signal ciphertext type |
| `enc@count` | Conditional counter | A nonzero count may be emitted; location and retry branch determine presence |
| `enc/$body` | Serialized Signal ciphertext | Real binary body, not raw 32-byte callKey material or placeholder text |

The [shared enc table](#en-shared-enc) separately records engine-material constraints. Interpret the wire body according to this section; the envelope flow is in [protobuf, padding and Signal](04-device-routing-and-keys.md#en-signal-envelope).

### Shared-structure index

- [Common attributes: `common`](#en-shared-common)
- [Audio: `audio`](#en-shared-audio)
- [Candidate and selected endpoints: `te`](#en-shared-te)
- [Key-generation options: `encopt`](#en-shared-encopt)
- [Engine-material boundary: `enc`](#en-shared-enc)
- [Capabilities: `capability`](#en-shared-capability)
- [Network information: `net`](#en-shared-net)
- [Certificate fingerprint: `certificate`](#en-shared-certificate)
- [Participant metadata: `metadata`](#en-shared-metadata)
- [Call configuration: `voip_settings`](#en-shared-voip_settings)
- [Relay token: `relay_token`](#en-shared-relay_token)
- [Relay information: `relay`](#en-shared-relay)
- [Relay endpoint attributes: `relay_te`](#en-shared-relay_te)
- [Relay measurement endpoint: `relaylatency_te`](#en-shared-relaylatency_te)
- [Call summary: `call_summary`](#en-shared-call_summary)
- [Summary participant: `summary_participant`](#en-shared-summary_participant)

### Message index

- [`offer`: Call offer](#en-message-offer)
- [`accept`: Call acceptance](#en-message-accept)
- [`preaccept`: Pre-acceptance](#en-message-preaccept)
- [`reject`: Rejection](#en-message-reject)
- [`terminate`: Termination](#en-message-terminate)
- [`transport`: Transport update](#en-message-transport)
- [`relaylatency`: Relay measurement](#en-message-relaylatency)
- [`ready`: Ready notification](#en-message-ready)
- [`relay_info_update`: Relay information update](#en-message-relay_info_update)
- [`offer_ack`: ACK for offer](#en-message-offer_ack)
- [`accept_ack`: ACK for accept](#en-message-accept_ack)
- [`relaylatency_ack`: ACK for relaylatency](#en-message-relaylatency_ack)
- [`offer_receipt`: Receipt for offer](#en-message-offer_receipt)
- [`accept_receipt`: Receipt for accept](#en-message-accept_receipt)

### Shared structures

<a id="en-shared-common"></a>

#### Common attributes: `common`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@call-id` | nonempty text; 64-byte buffer; text attribute | Parse and send | Required | All ordinary messages reject an empty call-id before branch parsing; offer also checks it separately. At most 63 text bytes plus a terminator are stored. Internal structure size is not wire length. |
| `@call-creator` | device JID text; typed JID attribute | Parse and send | Conditionally required | The sender uses the creator-device slot. Reception permits omission; a present value must be a valid JID. |

<a id="en-shared-audio"></a>

#### Audio: `audio`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@enc` | signaled codec label; text attribute | Parse and send | Conditionally required | The sender writes opus. The accept/preaccept audio parser requires this attribute. The offer rate loop does not independently validate enc. The opus label does not fix the actual codec path; encoding settings in voip_settings control the MLow branch. |
| `@rate` | sample rate, int32; decimal text | Parse and send | Conditionally required | Accept/preaccept requires a parseable rate. The sender selects from 8000, 16000, 24000 and 48000. Rate bits are 1/2/4/8. An offer can emit multiple audio nodes; the accept sender chooses the highest set rate. |

<a id="en-shared-te"></a>

#### Candidate and selected endpoints: `te`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `$body` | endpoint bytes; raw bytes: IPv4+port in 6 bytes, IPv6+port in 18 bytes; reception also accepts a 24-byte concatenation | Parse and send | Required | Each te, te2 or rte endpoint. Address content consists of binary bytes, not printable IP text. Ports retain the bytes held in the endpoint structure. |
| `@priority` | stored uint8; parsed int32, default 0; decimal text | Parse and send | Optional | Ordinary candidate list. A candidate list has at most 20 entries. Invalid endpoint conversion does not necessarily reject the entire list; its declared count is retained. |

<a id="en-shared-encopt"></a>

#### Key-generation options: `encopt`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@keygen` | int32; decimal text | Parse and send | Conditionally required | A missing encopt defaults to 2. When the node exists, keygen must parse. |

<a id="en-shared-enc"></a>

#### Engine-material boundary: `enc`

These two records describe the engine structure before encryption or after decryption. The 1..32 limit does not cap wire Signal ciphertext.

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `$body` | key material before encryption or after decryption, 1..32 bytes; raw bytes | Parse and send | Branch-dependent | The accept key-material node is optional; whether offer key material is read depends on its processing branch. The 1..32-byte constraint applies before encryption or after decryption. Final enc content is Signal-encrypted and changes length. |
| `@count` | int32; decimal text | Send | Optional | Written on the direct offer material node; omitted on destination and accept material nodes. |

<a id="en-shared-capability"></a>

#### Capabilities: `capability`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@ver` | positive int32; decimal text | Parse and send | Branch-dependent | When capability has a usable version and body. |
| `$body` | 1..64 bytes; raw bytes | Parse and send | Optional | Sent only for a version greater than 0 and a body of 1..64 bytes. Missing or unusable version/length clears capability to version=-1 and len=0. A malformed numeric ver is handled separately. |

<a id="en-shared-net"></a>

#### Network information: `net`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@medium` | int32; decimal text | Parse and send | Conditionally required | A missing net defaults medium to 3. A present net requires a parseable medium. |
| `@protocol` | int32; decimal text | Parse and send | Optional | Used for transport only; defaults to -1, and the sender omits -1. |
| `@health_status` | int32; decimal text | Parse and send | Optional | Sent for transport-message-type=11; reception attempts to read it whenever net exists. |

<a id="en-shared-certificate"></a>

#### Certificate fingerprint: `certificate`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@algorithm` | text; 64-byte buffer; text attribute | Parse and send | Conditionally required | Required when certificate exists. |
| `@fingerprint` | text; 256-byte buffer; text attribute | Parse and send | Conditionally required | Required when certificate exists. |
| `@hmac` | producer supplies 32-byte material; standard Base64 with padding | Parse and send | Optional | Used only by the accept certificate; emission follows the HMAC presence flag. The decoder capacity is 32 bytes. This branch checks decoding status and sets presence without independently requiring exactly 32 decoded bytes. |

<a id="en-shared-metadata"></a>

#### Participant metadata: `metadata`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@peer_abtest_bucket` | text; 128-byte buffer; text attribute | Parse and send | Optional | When the corresponding presence flag is set. |
| `@peer_abtest_bucket_id_list` | text; 256-byte buffer; text attribute | Parse and send | Optional | When the corresponding presence flag is set. |
| `@xrtt_ms` | int32, default 0; decimal text | Parse | Optional | Read when metadata exists. This metadata sending branch does not write xrtt_ms. |

<a id="en-shared-voip_settings"></a>

#### Call configuration: `voip_settings`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@uncompressed` | literal 1; text attribute | Parse | Conditionally required | This path expects uncompressed content; the attribute defaults to 0. This path processes uncompressed configuration text. Compressed content enters error logging and assertion handling. |
| `@jid` | device JID; text attribute | Parse | Optional | Selects the device associated with this settings entry. |
| `$body` | JSON configuration text; node bytes interpreted as text | Parse | Conditionally required | A successful one-to-one offer ACK requires at least one voip_settings. Content is parsed as a JSON configuration tree. Its configuration keys are consumed by media and connectivity policies. |

<a id="en-shared-relay_token"></a>

#### Relay token: `relay_token`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `$body` | 0..512 bytes; raw bytes | Parse | Required | The list contains 1..8 nodes and is required when relay is present. |

<a id="en-shared-relay"></a>

#### Relay information: `relay`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `token[]` | relay-token list; node list; see relay_token | Parse | Required | When relay is present. |
| `auth_token[]` | relay authorization-token list; node list; same token limits | Parse | Optional | When auth_token nodes are present. |
| `(te[]\|te2[])` | relay endpoint lists; node lists; exactly one list must be nonempty | Parse | Required | When relay is present. Both endpoint lists present, both absent, an empty endpoint body or no usable addresses causes failure. |
| `key/$body` | relay control key, 1..24 bytes; raw bytes | Parse | Optional | When relay has a key child. This key controls the relay and has a different purpose from the end-to-end call key. |
| `hbh_key/$body` | 30 decoded bytes; standard Base64 | Parse | Conditionally required | Decoded when hbh_key is present; valid material is required when warp_mi_tag_len is not 0. Exactly 30 decoded bytes are required; decoding or length failure clears the material and fails. |
| `@transaction-id` | int32; decimal text | Parse | Optional | When relay is present; defaults to -1. |
| `@self_pid` | parsed int32, stored uint16; decimal text | Parse | Optional | When relay is present; defaults to -1. |
| `@uuid` | text; 17-byte buffer; text attribute | Parse | Optional | When relay is present; defaults to empty text. |
| `@participant_uuid` | text; 9-byte buffer; text attribute | Parse | Optional | When relay is present; defaults to empty text. |
| `@warp_mi_tag_len` | int32; decimal text | Parse | Optional | When relay is present; defaults to 0. |
| `@prefer_relay` | int32; nonnegative value means true; decimal text | Parse | Optional | When relay is present; defaults to -1. |
| `@peer_dl` | int32; decimal text | Parse | Optional | When relay is present; defaults to 0. |
| `@prefer_ipv6` | int32; nonnegative value means true; decimal text | Parse | Optional | When relay is present; defaults to -1. |
| `@smart_transport_signal` | int32; decimal text | Parse | Optional | When relay is present; defaults to 0. |

<a id="en-shared-relay_te"></a>

#### Relay endpoint attributes: `relay_te`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `$body` | binary endpoint; see endpoint structure | Parse | Required | Each relay te or te2. |
| `@relay_id` | uint64; decimal text | Parse | Optional | Relay endpoint; defaults to the endpoint index. The attribute is parsed as uint64; relay-list insertion uses its low 32 bits. |
| `@token_id` | int32; decimal text | Parse | Optional | Relay endpoint; ordinary reading defaults to 0, with a separate -1 probe to detect presence. |
| `@xtoken_id` | int32; decimal text | Parse | Optional | Relay endpoint; defaults to -1. |
| `@auth_token_id` | int32; decimal text | Parse | Optional | Relay endpoint; defaults to -1. |
| `@xrtt_ms` | int32; decimal text | Parse | Optional | Relay endpoint; defaults to 100. |
| `@protocol` | int32; decimal text | Parse | Optional | Relay endpoint; defaults to 0. |
| `@c2r_rtt` | int32; decimal text | Parse | Optional | Relay endpoint; defaults to -1. |
| `@max_peer_c2r_rtt` | int32; decimal text | Parse | Optional | Relay endpoint; defaults to -1. |
| `@relay_name` | text; text attribute | Parse | Optional | Relay endpoint; defaults to empty text. |
| `@domain_name` | text; text attribute | Parse | Optional | Relay endpoint; defaults to empty text. |

<a id="en-shared-relaylatency_te"></a>

#### Relay measurement endpoint: `relaylatency_te`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `$body` | binary endpoint; see endpoint structure | Parse and send | Required | A te entry inside relaylatency. |
| `@latency` | int32; decimal text | Parse and send | Conditionally required | A usable entry requires parseable latency or xlatency. |
| `@xlatency` | int32; decimal text | Parse and send | Conditionally required | Alternative latency form, read when latency is missing or unparseable. |
| `@ul_bw` | int32; decimal text | Parse and send | Optional | Sent only when the value is not 0; reception defaults to 0. |
| `@dl_bw` | int32; decimal text | Parse and send | Optional | Sent only when the value is not 0; reception defaults to 0. |
| `@is_favored` | boolean; decimal text | Parse and send | Optional | Sent only when true; reception defaults to false. |
| `@relay_name` | text; 48-byte buffer; text attribute | Parse and send | Optional | Sent only when nonempty; reception defaults to empty text. |
| `@conn_id` | producer supplies 8-byte material; standard Base64 with padding | Parse and send | Optional | Sent only for a connection identifier other than 0; decoding failure clears conn_id. Decoder capacity is 8 bytes; successful decoding does not itself require exactly 8 bytes. |

<a id="en-shared-call_summary"></a>

#### Call summary: `call_summary`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@call-creator` | user JID; text attribute | Parse | Required | When call_summary is present. |
| `@call-id` | text; copied into a 64-byte buffer; text attribute | Parse | Required | When call_summary is present. |
| `@media` | mapped media-type text; text attribute | Parse | Required | When call_summary is present. |
| `@call_duration` | uint32; decimal text | Parse | Required | When call_summary is present. |
| `participant[]\|user[]+bot[]` | participant-summary list; node list, at most 256 entries | Parse | Optional | The compatibility participant form takes precedence over the user/bot form. |

<a id="en-shared-summary_participant"></a>

#### Summary participant: `summary_participant`

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@jid` | user JID; text attribute | Parse | Required | Each summary entry. |
| `@state` | participant-state text; text attribute | Parse | Required | Must map to a participant-state enum other than 0. |
| `@user_pn` | text; text attribute | Parse | Optional | user_pn and username depend on LID identity; the other attributes are optional. |
| `@username` | text; text attribute | Parse | Optional | user_pn and username depend on LID identity; the other attributes are optional. |
| `@push_name` | text; text attribute | Parse | Optional | user_pn and username depend on LID identity; the other attributes are optional. |
| `@account_kind` | text or mapped enum; text attribute | Parse | Optional | user_pn and username depend on LID identity; the other attributes are optional. |
| `@rering_ts` | uint64, default 0; decimal text | Parse | Optional | user_pn and username depend on LID identity; the other attributes are optional. |

### Fields for each message

offer_ack, accept_ack and similar names classify the content of `<ack class="call" type="offer">` and related nodes. offer_receipt denotes `<receipt><offer ... /></receipt>`. These category names must not be emitted as wire tags.

<a id="en-message-offer"></a>

#### `offer`: Call offer

Common attributes: [common](#en-shared-common).

Reusable children: [`audio[]`](#en-shared-audio), [`te[]`](#en-shared-te), [`net`](#en-shared-net), [`encopt`](#en-shared-encopt), [`enc`](#en-shared-enc), [`capability`](#en-shared-capability), [`certificate`](#en-shared-certificate), [`metadata`](#en-shared-metadata), [`voip_settings`](#en-shared-voip_settings), [`relay`](#en-shared-relay), [`rte`](#en-shared-te).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@caller_pn` | JID; text attribute | Parse | Optional | Caller metadata for LID identity. |
| `@username` | text; text attribute | Parse and send | Optional | Corresponding source text is nonempty. |
| `@group-jid` | group JID; text attribute | Parse and send | Optional | Group-call context. |
| `@scheduled-id` | text; text attribute | Parse and send | Optional | Scheduled-call context. |
| `@lightweight` | boolean, true represented by 1; text attribute | Parse and send | Optional | Lightweight call. |
| `@lightweight-key` | text; 11-byte buffer; text attribute | Parse and send | Optional | When the lightweight key is nonempty. |
| `@is_call_ended` | boolean, true represented by 1; text attribute | Parse and send | Optional | Late offer or an offer for an ended call. |
| `@user_redial_count` | int32; decimal text | Send | Optional | Sent when its input value is not 0. |
| `@device_class` | decimal text converted to int16; decimal text | Parse and send | Optional | Sent only when the value is not 0; reception defaults to 0. The result is stored as int16. One receiving path converts decimal text directly; another first clamps the conversion result to signed int32, then stores int16. |
| `audio[]` | audio-node list; see audio structure | Parse and send | Optional | The sender emits one audio per set rate bit; offer reception aggregates rates. |
| `screenable/@version` | uint32, default 0; decimal text | Parse and send | Optional | When call screening is supported. |
| `@screenable` | boolean, true represented by 1; text attribute | Parse | Optional | Compatibility form of the screening attribute. |
| `self_info` | container node; capability/video/metadata children | Send | Optional | The self_info container is enabled. |
| `silence/@reason` | mapped silence-reason enum; text attribute | Parse and send | Conditionally required | The reason must be nonempty and map to a supported value; an empty reason on a present silence node fails. |
| `silence/@is_first_wave` | boolean, true represented by 1; text attribute | Parse | Optional | When the silence reason selects the initial-wave branch. |
| `session/token/$body` | session-token bytes; raw bytes | Send | Optional | When the input byte-string length is greater than 0. |
| `caller_metadata/dialer` | marker node; empty child | Send | Optional | When the dialer flag is true. |
| `terminate_hint/@terminate_reason` | reason text; text attribute | Parse and send | Optional | When is_call_ended is true and an ending hint is supplied. |
| `@terminate_reason` | compatibility reason text; text attribute | Parse | Optional | When is_call_ended is true and no terminate_hint child exists. |
| `terminate_hint/@terminate_transaction_id` | int32, default -1; decimal text | Parse | Optional | When an ending hint is present. |
| `terminate_hint/call_summary` | summary tree; see call_summary | Parse | Optional | When an ending hint is present. |
| `@version` | literal 2; text attribute | Send | Optional | When a group offer uses signaling version 2. |
| `@action` | upgrade / create_agc / create_lgc / create_vc; text attribute | Send | Optional | For signaling version 2 and an action selection value of 1..4. |
| `destination/to[]/@jid` | device JID; typed JID attribute | Send | Optional | When the destination-device count is greater than 0. |
| `destination/to[]/enc` | per-device key bytes; see enc; the final stanza carries end-to-end encrypted content | Send | Optional | During per-device fanout. |
| `group_info` | group-information tree; conditional structured subtree | Parse and send | Optional | Group-call context. This container carries group participant context. The ordinary one-to-one voice path does not require it. |
| `invite` | invitation tree; conditional structured subtree | Send | Optional | When there are additional invitees. |
| `bot_info\|bot` | bot context; conditional structured subtree | Send | Optional | When a bot participant or bot options are present. |
| `uploadfieldstat\|dontuploadfieldstat` | telemetry markers; empty node | Parse | Optional | uploadfieldstat takes precedence; when both markers are absent, statistics upload defaults to enabled. |
| `userrate/@interval` | int32; decimal text | Parse | Optional | A missing userrate node defaults to -1. When the node exists, a missing or unparseable interval defaults to 86400. |
| `client/@self_bad_asn` | int32 converted to boolean; decimal text | Parse | Optional | When client is present; defaults to 0. |

<a id="en-message-accept"></a>

#### `accept`: Call acceptance

Common attributes: [common](#en-shared-common).

Reusable children: [`audio`](#en-shared-audio), [`te[]`](#en-shared-te), [`net`](#en-shared-net), [`encopt`](#en-shared-encopt), [`enc`](#en-shared-enc), [`capability`](#en-shared-capability), [`certificate`](#en-shared-certificate), [`metadata`](#en-shared-metadata), [`relay`](#en-shared-relay), [`rte`](#en-shared-te), [`voip_settings`](#en-shared-voip_settings).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `audio` | audio node; see audio structure | Parse and send | Required | Normal accept. |
| `@device_class` | decimal text converted to int16; decimal text | Parse and send | Optional | Sent only for an input other than 0; reception defaults to 0. The result is stored as int16. One receiving path converts decimal text directly; another first clamps the conversion result to signed int32, then stores int16. |
| `@retry` | literal 1; text attribute | Send | Optional | The retry flag is set. |
| `ext_creator_identity_key_sig/$body` | signature bytes; standard Base64 text stored as node bytes | Send | Optional | When the extension signature length is greater than 0. The signature belongs to the extension-creator identity branch and serves a different purpose from the ordinary one-to-one call key. |
| `relaylatency/te` | selected relay measurement; see relaylatency_te | Parse and send | Optional | Sent when ul_bw or dl_bw is not 0. |
| `video` | video parameters; conditional structured subtree | Parse and send | Optional | When video is present. The voice-only path omits this video subtree. |

<a id="en-message-preaccept"></a>

#### `preaccept`: Pre-acceptance

Common attributes: [common](#en-shared-common).

Reusable children: [`audio`](#en-shared-audio), [`encopt`](#en-shared-encopt), [`capability`](#en-shared-capability).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `audio` | audio node; see audio structure | Parse and send | Required | Normal preaccept. |
| `screenable/@version` | uint32, default 0; decimal text | Parse and send | Optional | When the screening version is not 0. |
| `video` | video parameters; conditional structured subtree | Parse and send | Optional | When video is present. |

<a id="en-message-reject"></a>

#### `reject`: Rejection

Common attributes: [common](#en-shared-common).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@reason` | reason text; 128-byte buffer; text attribute | Parse and send | Optional | Sent only when nonempty; reception copies text with an empty default. |
| `@count` | uint32; decimal text | Parse and send | Optional | Always written by the sender; reception defaults to 0. |
| `registration/$body` | exactly 4 bytes; raw bytes | Parse and send | Conditionally required | Sent when reason is enc; when registration is present, its length must be exactly 4 bytes. |

<a id="en-message-terminate"></a>

#### `terminate`: Termination

Common attributes: [common](#en-shared-common).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@reason` | text; 128-byte buffer; text attribute | Parse and send | Optional | Sent only when nonempty. |
| `@lightweight-key` | text; text attribute | Send | Optional | Sent only when nonempty. |
| `@joinable` | boolean, true represented by 1; text attribute | Parse | Optional | Reception compares against the literal 1. |
| `@transaction-id` | int32, default -1; decimal text | Parse | Optional | Read on reception only. |
| `call_summary` | summary tree; see call_summary | Parse | Optional | When the child node is present. |
| `destination/to[]/@jid` | device JID; typed JID attribute | Send | Optional | When the destination count is greater than 0. |
| `@duration` | positive int32 counter; decimal text | Send | Optional | Sent only when the input counter is greater than 0. The value retains its source call-statistics unit; serialization does not convert units. |
| `@audio_duration` | positive int32 counter; decimal text | Send | Optional | Sent only when the input counter is greater than 0. The value retains its source call-statistics unit; serialization does not convert units. |
| `@video_duration` | positive int32 counter; decimal text | Send | Optional | Sent only when the input counter is greater than 0. The value retains its source call-statistics unit; serialization does not convert units. |

<a id="en-message-transport"></a>

#### `transport`: Transport update

Common attributes: [common](#en-shared-common).

Reusable children: [`te[]|te2[]`](#en-shared-te), [`net`](#en-shared-net), [`rte`](#en-shared-te), [`certificate`](#en-shared-certificate).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@transport-message-type` | int32, default 0; decimal text | Parse and send | Optional | Always written; type 2 selects te2, while other types select te. |
| `@p2p-cand-round` | int32, default 0; decimal text | Parse and send | Optional | Sent only when the value is not 0. |
| `@ice-ufrag` | text; 32-byte buffer; text attribute | Parse and send | Conditionally required | For type 13, reception requires the attribute; the sender writes only nonempty values. |
| `@ice-pwd` | text; 128-byte buffer; text attribute | Parse and send | Conditionally required | For type 13, reception requires the attribute; the sender writes only nonempty values. |

<a id="en-message-relaylatency"></a>

#### `relaylatency`: Relay measurement

Common attributes: [common](#en-shared-common).

Reusable children: [`relaylatency_te[]`](#en-shared-relaylatency_te).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@transaction-id` | int32, default -1; decimal text | Parse and send | Optional | The sender omits -1. |
| `te[]` | up to 8 relay measurements; see relaylatency_te | Parse and send | Required | Reception requires at least 1 usable measurement and stores at most 8. Before 8 entries are stored, an entry with neither parseable latency nor parseable xlatency fails the message. |
| `destination/to[]/@jid` | device JID; typed JID attribute | Send | Optional | When the destination count is greater than 0. |

<a id="en-message-ready"></a>

#### `ready`: Ready notification

Common attributes: [common](#en-shared-common).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@type` | int32, default 0; decimal text | Parse | Optional | Missing, unparseable or out-of-int32-range values fall back to 0. |

<a id="en-message-relay_info_update"></a>

#### `relay_info_update`: Relay information update

Common attributes: [common](#en-shared-common).

Reusable children: [`relay`](#en-shared-relay).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `relay` | relay structure; see relay | Parse | Optional | The relay child is optional; absence is accepted by this branch. |

<a id="en-message-offer_ack"></a>

#### `offer_ack`: ACK for offer

Reusable children: [`relay`](#en-shared-relay), [`voip_settings`](#en-shared-voip_settings), [`rte`](#en-shared-te).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `(group_info\|relay\|error[0])/@call-id` | nonempty text; text attribute | Parse | Required | Select from group_info, relay and the first error, in that order; the selected call-id must be nonempty. |
| `relay` | relay structure; see relay | Parse | Conditionally required | Successful one-to-one ACK; error and group branches have their own rules. |
| `voip_settings[]` | configuration; see voip_settings | Parse | Conditionally required | Successful one-to-one ACK. |
| `error[]/@jid` | user/device JID; text attribute | Parse | Conditionally required | Each error entry. |
| `error[]/@code` | uint32, default outer error; decimal text | Parse | Optional | Each error entry; at most 64 entries. |
| `group_info` | group tree; conditional subtree | Parse | Optional | Group-call ACK. |
| `user` | user tree; conditional subtree | Parse | Optional | When the service returns participant context. |
| `@joinable` | literal 1; text attribute | Parse | Optional | The child selected as the call-id source. |
| `screenable/@action` | automatic marker; text attribute | Parse | Optional | When the screenable child is present. |

<a id="en-message-accept_ack"></a>

#### `accept_ack`: ACK for accept

Reusable children: [`relay`](#en-shared-relay), [`voip_settings`](#en-shared-voip_settings).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `accept/@call-id` | nonempty text; text attribute | Parse | Conditionally required | Success without waiting_room; a missing accept child is ignored. |
| `relay` | relay structure; see relay | Parse | Conditionally required | Success without waiting_room. |
| `voip_settings[]` | configuration; see voip_settings | Parse | Optional | Parsed if present; unlike offer ACK, accept ACK permits its absence. |
| `guest/@is_guest_eligible` | boolean; text attribute | Parse | Optional | When the guest child is present. |
| `waiting_room` | waiting-room tree; conditional subtree | Parse | Optional | The service waiting-room branch. |
| `error/@call-id` | nonempty text; text attribute | Parse | Conditionally required | When the outer ACK error is not 0. |
| `error/@jid` | device JID; text attribute | Parse | Conditionally required | When the outer ACK error is not 0. |

<a id="en-message-relaylatency_ack"></a>

#### `relaylatency_ack`: ACK for relaylatency

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `relaylatency/@call-id` | nonempty text; text attribute | Parse | Conditionally required | An explicit relaylatency call-id must be nonempty; a missing node or attribute permits fallback. |
| `$outer.error` | int32; provided by outer ACK metadata | Parse | Optional | Retains the outer error status rather than reading it from the child body. |

<a id="en-message-offer_receipt"></a>

#### `offer_receipt`: Receipt for offer

Common attributes: [common](#en-shared-common).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@call-id` | nonempty text; text attribute | Parse | Required | Receipt call-id must be nonempty. Offer and accept receipts enter separate internal message types. |
| `@call-creator` | device JID; text attribute | Parse | Optional | A present value must be a valid JID. |

<a id="en-message-accept_receipt"></a>

#### `accept_receipt`: Receipt for accept

Common attributes: [common](#en-shared-common).

| Field | Type and encoding | Direction | Requiredness | Conditions, defaults and behavior |
| --- | --- | --- | --- | --- |
| `@call-id` | nonempty text; text attribute | Parse | Required | Receipt call-id must be nonempty. Offer and accept receipts enter separate internal message types. |
| `@call-creator` | device JID; text attribute | Parse | Optional | A present value must be a valid JID. |

<a id="en-body-encoding"></a>

### Body and text encoding

Base64 is used only where material is explicitly carried as text, such as hbh_key, certificate hmac and conn_id. Endpoints, Signal enc and tokens use raw bytes; log-display encoding does not change their wire structure.

[Continue the main sequence: Device routing and call-key envelopes](04-device-routing-and-keys.md#en)

[中文](#zh)
