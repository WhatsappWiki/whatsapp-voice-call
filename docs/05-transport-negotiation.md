# 地址协商与中继信息 · Transport Negotiation and Relay Information

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 交换信息与验证路径

offer、accept 和 transport 交换候选与网络信息，ACK 提供中继授权和配置。传输层执行探测、认证及路径选择；信令层分别记录“得到候选、得到授权、探测成功、选中路径”。接听与媒体状态见[确认语义](02-call-lifecycle.md#zh-confirmations)。

<a id="zh-endpoints"></a>

### te、te2 与 rte

| 节点 | 用途 |
| --- | --- |
| te / te2 | 两种端点列表表示；二者的区别不等于 IPv4／IPv6 的区别 |
| rte | 当前选择的端点 |

端点 body 为二进制地址与端口，可用 6 字节 IPv4、18 字节 IPv6，接收还支持 24 字节双族形式。组包使用端点编码器。普通候选上限、priority 与无效项规则见[端点表](03-signaling-fields.md#zh-shared-te)。

<a id="zh-transport"></a>

### transport 更新

```xml
<call to="PEER_DEVICE_JID" id="TRANSPORT_STANZA_ID">
  <transport call-id="CALL_ID" call-creator="CALLER_DEVICE_JID"
             transport-message-type="1" p2p-cand-round="1">
    <te priority="1">ENDPOINT_BYTES</te>
    <net medium="NETWORK_MEDIUM" protocol="NETWORK_PROTOCOL" />
  </transport>
</call>
```

类型与轮次为示例值。特殊分支如下，其余字段见[transport 表](03-signaling-fields.md#zh-message-transport)。

| transport-message-type | 处理 |
| --- | --- |
| 缺失 | 按 0 处理 |
| 2 | 选择 te2，其他类型选择 te |
| 11 | 发送侧可带 net.health_status |
| 13 | 接收要求 ice-ufrag 与 ice-pwd，可带 certificate |

候选轮次 p2p-cand-round 属于本场通话。类型 13 的 ICE 凭据、中继 token、通话主秘密及私有 P2P 探测材料各有用途，按对应路径使用。

net 整体缺失与空 net 的处理不同，默认与错误规则集中在[网络字段表](03-signaling-fields.md#zh-shared-net)。

<a id="zh-relay"></a>

### relay 授权与端点

```xml
<relay call-id="CALL_ID" transaction-id="RELAY_TRANSACTION_ID">
  <token>RELAY_TOKEN_BYTES</token>
  <te relay_id="0" token_id="0" protocol="0">RELAY_ENDPOINT_BYTES</te>
</relay>
```

relay 将端点与授权关联：根据端点索引选择 token／auth_token，再准备对应中继路径。te 与 te2 两种列表必须恰有一种非空，并有可用地址。

| 材料 | 用途 |
| --- | --- |
| token / auth_token | 对应中继或认证流程的授权 |
| key / hbh_key | 中继控制材料；warp_mi_tag_len 非 0 时要求有效 hbh_key |
| callKey | 设备间保护通话的端到端秘密，经 Signal 分发 |

token 数量与长度、材料编码及开关见[relay 表](03-signaling-fields.md#zh-shared-relay)，索引与端点属性见[relay_te 表](03-signaling-fields.md#zh-shared-relay_te)。

<a id="zh-relaylatency"></a>

### relaylatency 测量

```xml
<call to="PEER_JID" id="LATENCY_STANZA_ID">
  <relaylatency call-id="CALL_ID" call-creator="CALLER_DEVICE_JID"
                transaction-id="MEASUREMENT_TRANSACTION_ID">
    <te latency="42" ul_bw="UPLINK_MEASUREMENT" dl_bw="DOWNLINK_MEASUREMENT">
      RELAY_ENDPOINT_BYTES
    </te>
  </relaylatency>
</call>
```

测量项关联端点，并可附带带宽、偏好和连接标识。至少一项有效、最多八项；无效地址与无法解析延迟的区别见[测量字段](03-signaling-fields.md#zh-shared-relaylatency_te)及[列表规则](03-signaling-fields.md#zh-message-relaylatency)。

relaylatency ACK 允许缺失节点或 call-id 的回退关联；显式 ID 仍需非空。详见[测量 ACK](03-signaling-fields.md#zh-message-relaylatency_ack)。

<a id="zh-updates"></a>

### ready、更新与配置

| 节点 | 处理职责 | 字段参考 |
| --- | --- | --- |
| ready | 按 type 通知对应就绪事件 | [ready](03-signaling-fields.md#zh-message-ready) |
| relay_info_update | 有 relay 时应用相应更新；缺失时允许此解析分支通过 | [relay_info_update](03-signaling-fields.md#zh-message-relay_info_update) |
| voip_settings | 读取默认配置，再按 JSON 树覆盖，可按 jid 绑定设备 | [voip_settings](03-signaling-fields.md#zh-shared-voip_settings) |

实际编码路径由能力和配置共同决定；audio 的标签与采样率规则见[音频表](03-signaling-fields.md#zh-shared-audio)。

### 直连与中继

开启通话 IP 保护时，媒体经服务端中继，以避免向通话对方暴露 IP 地址。[官方通话隐私说明](https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/)

语音加密与转发概念见[公开 Wiki 的语音通话目录](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#zh)。本篇涉及的信令分派见[伪代码](07-signaling-pseudocode.md#zh-transport-dispatch)。

[下一篇：状态、乱序与异常收尾](06-state-and-recovery.md#zh)

[English](#en)

<a id="en"></a>

## English

[Home](../README.md#en) · [中文](#zh)

### Exchanging information and validating paths

offer, accept and transport exchange candidates and network information; ACKs provide relay grants and configuration. The transport layer probes, authenticates and selects paths. Signaling tracks candidate acquisition, authorization, probe success and selected path separately. See [acknowledgment semantics](02-call-lifecycle.md#en-confirmations) for answering and media state.

<a id="en-endpoints"></a>

### te, te2 and rte

| Node | Purpose |
| --- | --- |
| te / te2 | Two endpoint-list representations; their distinction is not the same as IPv4 versus IPv6 |
| rte | The currently selected endpoint |

An endpoint body contains binary address and port data: six bytes for IPv4 or 18 for IPv6; reception also accepts a 24-byte dual-family form. Use the endpoint encoder when building nodes. Candidate limits, priority and invalid-entry rules are in the [endpoint table](03-signaling-fields.md#en-shared-te).

<a id="en-transport"></a>

### transport updates

```xml
<call to="PEER_DEVICE_JID" id="TRANSPORT_STANZA_ID">
  <transport call-id="CALL_ID" call-creator="CALLER_DEVICE_JID"
             transport-message-type="1" p2p-cand-round="1">
    <te priority="1">ENDPOINT_BYTES</te>
    <net medium="NETWORK_MEDIUM" protocol="NETWORK_PROTOCOL" />
  </transport>
</call>
```

The type and round are example values. Special branches are listed below; other fields are in the [transport table](03-signaling-fields.md#en-message-transport).

| transport-message-type | Handling |
| --- | --- |
| Missing | Defaults to 0 |
| 2 | Select te2; other types select te |
| 11 | The sender may include net.health_status |
| 13 | Reception requires ice-ufrag and ice-pwd; certificate is optional |

The p2p-cand-round belongs to this call. Type-13 ICE credentials, relay tokens, call master secrets and private P2P probe material have distinct roles and are used by their respective paths.

A missing net node and an empty net node are handled differently. Defaults and errors are collected in the [network field table](03-signaling-fields.md#en-shared-net).

<a id="en-relay"></a>

### Relay authorization and endpoints

```xml
<relay call-id="CALL_ID" transaction-id="RELAY_TRANSACTION_ID">
  <token>RELAY_TOKEN_BYTES</token>
  <te relay_id="0" token_id="0" protocol="0">RELAY_ENDPOINT_BYTES</te>
</relay>
```

relay associates endpoints with grants: endpoint indices select token/auth_token material before the corresponding relay path is prepared. Exactly one of the te and te2 lists must be nonempty, with a usable address.

| Material | Purpose |
| --- | --- |
| token / auth_token | Authorization for the corresponding relay or authentication procedure |
| key / hbh_key | Relay control material; a nonzero warp_mi_tag_len requires valid hbh_key |
| callKey | The end-to-end call secret distributed between devices using Signal |

For token counts, lengths, encodings and switches, see the [relay table](03-signaling-fields.md#en-shared-relay). Indices and endpoint attributes are in [relay_te](03-signaling-fields.md#en-shared-relay_te).

<a id="en-relaylatency"></a>

### relaylatency measurements

```xml
<call to="PEER_JID" id="LATENCY_STANZA_ID">
  <relaylatency call-id="CALL_ID" call-creator="CALLER_DEVICE_JID"
                transaction-id="MEASUREMENT_TRANSACTION_ID">
    <te latency="42" ul_bw="UPLINK_MEASUREMENT" dl_bw="DOWNLINK_MEASUREMENT">
      RELAY_ENDPOINT_BYTES
    </te>
  </relaylatency>
</call>
```

A measurement identifies an endpoint and may include bandwidth, preference and a connection identifier. At least one usable entry is required, with at most eight stored. See [measurement fields](03-signaling-fields.md#en-shared-relaylatency_te) and [list rules](03-signaling-fields.md#en-message-relaylatency) for the distinction between invalid addresses and unparseable latency.

relaylatency ACK supports fallback correlation when its node or call-id is missing. An explicit ID must still be nonempty. See [measurement ACK](03-signaling-fields.md#en-message-relaylatency_ack).

<a id="en-updates"></a>

### ready, updates and configuration

| Node | Responsibility | Field reference |
| --- | --- | --- |
| ready | Notify the corresponding readiness event by type | [ready](03-signaling-fields.md#en-message-ready) |
| relay_info_update | Apply relay information when present; its absence is accepted by this parsing branch | [relay_info_update](03-signaling-fields.md#en-message-relay_info_update) |
| voip_settings | Load defaults, apply JSON overrides and optionally bind settings to a device by jid | [voip_settings](03-signaling-fields.md#en-shared-voip_settings) |

The actual codec path depends on both capabilities and configuration. See [audio fields](03-signaling-fields.md#en-shared-audio) for label and sample-rate rules.

### Direct and relayed transport

With call IP protection enabled, media is relayed through the service to avoid revealing the IP address to the other participant. [Official call privacy explanation](https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/)

For voice encryption and forwarding concepts, see the [public Wiki voice-calling directory](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#en). The signaling dispatch used here is shown in [pseudocode](07-signaling-pseudocode.md#en-transport-dispatch).

[Next: State, reordering and cleanup](06-state-and-recovery.md#en)

[中文](#zh)
