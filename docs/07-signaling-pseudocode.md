# XMPP 交互伪代码 · XMPP Interaction Pseudocode

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

下面以 Python 风格串联一对一语音信令。阶段名属于本模型，原生状态编号见[状态附录](06-state-and-recovery.md#zh-native-state-map)。各状态入口由同一事件队列调度，异步结果回到队列后重新核对上下文。

### 组件约定

| 组件 | 契约 |
| --- | --- |
| directory / identity | 提供可信设备目录、PN/LID 映射、来源与当前设备范围校验 |
| signal / protobuf | 按设备处理会话、业务填充与 Message.call.callKey；封装见[密钥篇](04-device-routing-and-keys.md#zh-signal-envelope) |
| jobs / queue | 后台操作以成功或失败结果回调；queue.post 排入通话队列 |
| writer / pending | 二进制节点写入与待确认记录，失败交给对应策略 |
| confirmations / signaling | 专用确认、限长解析及[完整字段规则](03-signaling-fields.md#zh) |
| engine / transport / media | 通话事件、链路更新与音频生命周期；本篇不实现媒体包保护 |
| policy / recent_results | 分支策略、重试和结束结果缓存 |

Node 的 body 为原始字节；ids 分别生成通话 ID 和 stanza ID。ctx 保存当前通话、设备密钥映射和异步任务代次。示例中的辅助解析器遵守字段参考，不在各段重复字段校验。

### 1. 发起与按设备封装

```python
import asyncio
import inspect

def current(ctx, saved):
    return (calls.current is ctx and not ctx.ending
            and (ctx.call_id, ctx.generation) == saved)

def deliver(ctx, saved, result, apply):
    if not current(ctx, saved):
        result.release_call_material()
    elif result.failed:
        begin_finish(ctx, result.reason, send_end=False)
    else:
        apply(result)

def start_outgoing(peer):
    preflight.require_available_and_permitted()
    ctx = calls.create(ids.new_call_id(), peer, role="caller")
    saved = (ctx.call_id, ctx.generation)
    def prepared(result):
        ctx.device_keys = result.keys
        ctx.allowed_devices = result.devices
        ctx.phase = "CALLING"
        send_call(ctx, peer, result.offer)
    jobs.submit(prepare_offer(ctx.snapshot()), completed=lambda result:
        queue.post(lambda: deliver(ctx, saved, result, prepared)))

async def prepare_offer(snapshot):
    devices = await directory.resolve_callable_devices(snapshot.peer)
    identity.require_valid_target_set(snapshot.peer, devices)
    keys, targets = {}, []
    for device in devices:
        await signal.ensure_session_ready(device)
        keys[device] = secure_random.bytes(32)
        plain = protobuf.encode_message_call(call_key=keys[device])
        cipher = await signal.encrypt_call_message(device, plain)
        targets.append(Node("to", {"jid": device}, children=[Node(
            "enc", {"v": "2", "type": cipher.wire_type},
            body=cipher.serialized_bytes)]))
    offer = Node("offer", common_attrs(snapshot), children=
        capabilities.offer_children() + [Node("destination", children=targets)])
    return PreparedOffer(offer, keys, devices)
```

Signal 适配层负责一次业务填充，type 来自实际密文结果。PreKey 等待和过期回调规则见[待发送 offer](04-device-routing-and-keys.md#zh-pending-offer)。单设备直接 enc 形式由同一构造层提供。

### 2. 发送与 ACK 关联

```python
def send_call(ctx, target, payload):
    request_id = ids.new_stanza_id()
    record = pending.add(request_id, payload.tag, ctx.call_id, ctx.generation,
                         expected_route=policy.ack_route(target, payload.tag))
    writer.enqueue(Node("call", {"to": target, "id": request_id},
                        children=[payload]),
        failed=lambda error: queue.post(lambda: pending.write_failed(record, error)))
    return request_id

def on_call_ack(node):
    ack = signaling.parse_ack_envelope(node)  # 校验 from/id/class/type，解析 error。
    record = pending.find(ack.id, ack.type)
    if record is None:
        return
    identity.require_expected_ack_source(ack.source, record.expected_route)
    outcome = signaling.parse_ack_payload(ack)
    if outcome.ignored:
        return
    if outcome.call_id is not None:
        require(outcome.call_id == record.call_id)
    pending.note_ack(record, outcome)
    ctx = calls.match(record.call_id, record.generation)
    if ctx is None or ctx.ending:
        return
    if outcome.failed:
        policy.handle_signaling_failure(ctx, record, outcome)
    else:
        engine.apply_ack_configuration(ctx, outcome)
```

先登记事务再写入，避免快速返回的 ACK 丢失关联。内容解析按以下分支进行；确认与接听的区别见[确认语义](02-call-lifecycle.md#zh-confirmations)。

| ACK 类型 | 解析器必须保留的规则 |
| --- | --- |
| offer | 按 group_info→relay→首个 error 选择非空 call-id；成功一对一要求 relay 和至少一份 voip_settings；保留外层与各目标错误 |
| accept | 成功非等候室要求 accept@call-id 和 relay，settings 可选；缺 accept 按忽略处理；错误分支读取 error@call-id、jid |
| relaylatency | 缺节点或 call-id 时允许回退关联；显式 ID 非空，保留外层错误 |

各分支完整字段见[offer ACK](03-signaling-fields.md#zh-message-offer_ack)、[accept ACK](03-signaling-fields.md#zh-message-accept_ack)和[测量 ACK](03-signaling-fields.md#zh-message-relaylatency_ack)。

### 3. 接收、确认与来电解密

```python
def on_node(node):
    signaling.require_envelope_limits(node)
    if node.tag == "ack" and node.attr("class") == "call":
        return on_call_ack(node)
    if node.tag == "receipt":
        receipt = confirmations.parse_receipt(node)
        confirmations.ack_receipt(receipt)
        return confirmations.note_receipt(receipt)
    if node.tag != "call":
        return other_stanzas.dispatch(node)
    event = signaling.parse_call_header(node)
    identity.require_valid_call_source(event)
    confirmations.schedule_for_received_signal(event)
    if received_signals.is_exact_duplicate(event):
        return
    received_signals.remember_bounded(event)
    if recent_results.suppresses(event):
        return recent_results.apply_late_event_if_needed(event)
    if event.tag == "offer":
        return receive_offer(event)
    ctx = calls.find_by_call_and_peer(event.call_id, event.peer_user)
    if ctx is None:
        return policy.buffer_or_ignore_without_offer(event)
    if not ctx.ending:
        identity.require_allowed_event(ctx, event)
        dispatch_current_event(ctx, event)

def receive_offer(event):
    offer = signaling.parse_offer(event)
    decision = policy.check_existing_call_and_offer(offer)
    if decision.is_retry:
        return policy.handle_offer_retry(offer)
    if decision.reject_busy:
        return signaling.send_reject_for_offer(offer, reason="busy")
    ctx = calls.reserve_incoming(offer, phase="DECRYPTING")
    saved = (ctx.call_id, ctx.generation)
    jobs.submit(decrypt_offer(offer), completed=lambda result: queue.post(
        lambda: deliver(ctx, saved, result,
                        lambda material: engine.receive_clear_offer(ctx, offer, material.key))))

async def decrypt_offer(offer):
    enc = signaling.select_unique_local_enc(offer, identity.local_device_aliases())
    require(enc.attr("v") == "2" and enc.attr("type") in {"msg", "pkmsg"})
    plain = await signal.decrypt_call_message(offer.source_device, enc.attr("type"), enc.body)
    key = protobuf.parse_message(plain).require_call_key(length=32)
    return DecryptedCallMaterial(key)
```

jobs 将结果交付为带 failed、reason 和材料释放方法的对象，并清理失败任务未交付的临时材料。普通 keygen2 路径读取 32 字节材料；其他扩展交给相应处理器。重复业务信令仍可再次确认；Signal 重复密文、身份错误与忙线由各自分支处理。

confirmations 按标签与静默条件选择专用或通用路径，普通非静默 offer 可先安排 receipt 再安排专用 ACK。入口的设备范围还包含合法的自己账户其他设备通知，随后由策略核对 handled-elsewhere 条件。

### 4. 预接受、接听与设备选择

```python
def on_incoming_offer_ready(ctx):
    if ctx.ending:
        return
    ctx.phase = "INCOMING"
    send_call(ctx, ctx.peer_device, Node("preaccept", common_attrs(ctx),
              children=capabilities.preaccept_children()))
    ui.present_incoming_according_to_platform_policy(ctx)

def user_accept(call_id):
    ctx = calls.require_current(call_id)
    require(not ctx.ending and ctx.phase == "INCOMING")
    engine.accept_incoming(ctx)  # 引擎产生本次 accept 节点。

def on_local_accept_payload(ctx, payload):
    if not ctx.ending:
        send_call(ctx, ctx.peer_device, payload)
        ctx.phase = "ACCEPT_SENT"

def on_peer_accept(ctx, event):
    identity.require_offered_device(ctx, event.source_device)
    if ctx.chosen_device is None:
        ctx.chosen_device = event.source_device
        ctx.active_material = ctx.device_keys[event.source_device]
        engine.choose_callee_device(ctx, event.source_device)
    elif ctx.chosen_device != event.source_device:
        return policy.handle_nonselected_device_event(ctx, event)
    if not ctx.preaccept_ready:
        ctx.pending_accept = event.deep_copy()
        return
    engine.apply_accept_and_optional_new_material(ctx, event)
    ctx.phase = "ACCEPT_RECEIVED"
    transport.prepare_from_current_signaling(ctx)

def on_peer_preaccept(ctx, event):
    engine.update_device_capability(ctx, event)
    ctx.preaccept_ready = True
    cached, ctx.pending_accept = ctx.pending_accept, None
    if cached is not None:
        queue.post(lambda: dispatch_if_current(ctx, cached))

def dispatch_if_current(ctx, event):
    if calls.current is ctx and not ctx.ending and event.call_id == ctx.call_id:
        identity.require_allowed_event(ctx, event)
        dispatch_current_event(ctx, event)
```

新 enc 先完成对应解密再交给引擎；重投继续核对当前通话和 ending。正式设备选择及其他设备结束通知见[设备篇](04-device-routing-and-keys.md#zh-device-selection)，缓存顺序见[状态篇](06-state-and-recovery.md#zh-accept-ordering)。

<a id="zh-transport-dispatch"></a>

### 5. 传输更新与结束事件分派

```python
def dispatch_current_event(ctx, event):
    if policy.is_handled_elsewhere_event(ctx, event):
        return engine.handle_other_self_device(ctx, event)
    tag = event.tag
    if tag == "preaccept":
        return on_peer_preaccept(ctx, event)
    if tag == "accept":
        require(ctx.role == "caller")
        return on_peer_accept(ctx, event)
    if tag == "transport":
        return transport.apply_candidates(ctx, signaling.parse_transport(event))
    if tag == "relaylatency":
        return transport.record_measurements(ctx, signaling.parse_relaylatency(event))
    if tag == "ready":
        return engine.on_ready(ctx, signaling.parse_ready_type(event))
    if tag == "relay_info_update":
        update = signaling.parse_relay_update(event)
        if update.relay is not None:
            transport.apply_relay_update(ctx, update.relay)
    elif tag in {"reject", "terminate"}:
        identity.require_end_event_applies(ctx, event)
        recent_results.record_end_event(event)
        begin_finish(ctx, event.reason, send_end=False)
```

transport 解析保留类型 2 的 te2、类型 13 的 ICE 凭据和候选轮次；测量解析保留有效项与错误条件。ready 的 type 默认规则及可缺 relay 的更新规则见[协商篇](05-transport-negotiation.md#zh-updates)。它们更新各自状态，不直接统一设为 ACTIVE。

<a id="zh-finish"></a>

### 6. 异常也继续收尾

首个结束事件先发布共享完成句柄并置 ending，重复结束返回同一句柄。信令准备、各次发送及各个清理步骤分别收集错误；finally 保证进入资源清理，传输清理失败后仍尝试媒体和材料释放。

```python
def user_hangup(call_id):
    return begin_finish(calls.require_current(call_id), "local_hangup", send_end=True)

def begin_finish(ctx, reason, send_end):
    if ctx.ending:
        return ctx.finish_future
    loop = asyncio.get_running_loop()
    ctx.finish_future = loop.create_future()
    ctx.ending = True
    ctx.generation += 1
    ctx.cleanup_task = loop.create_task(finish_call(ctx, reason, send_end))
    return ctx.finish_future

async def finish_call(ctx, reason, send_end):
    errors = []
    async def step(label, action):
        try:
            value = action()
            return await value if inspect.isawaitable(value) else value
        except (Exception, asyncio.CancelledError) as error:
            errors.append((label, error))
            return None
    try:
        await step("停止计时", lambda: timers.cancel_for_call(ctx.call_id))
        await step("停止跟踪", lambda: policy.stop_offer_and_accept_tracking(ctx))
        plan = await step("准备结束信令", lambda:
                          policy.prepare_end_signaling(ctx, reason, send_end))
        if plan is not None:
            for target, payload in plan.payloads:
                await step("发送结束信令", lambda target=target, payload=payload:
                           send_call(ctx, target, payload))
            await step("保存结束结果", lambda:
                       recent_results.record_final(ctx.call_id, plan.outcome, reason))
    except (Exception, asyncio.CancelledError) as error:
        errors.append(("处理结束计划", error))
    finally:
        for label, action in [
            ("停止媒体", lambda: media.begin_stop(ctx)),
            ("停止传输", lambda: transport.begin_stop(ctx)),
            ("等待传输退出", lambda: transport.join_and_release(ctx)),
            ("等待媒体退出", lambda: media.join_and_release(ctx)),
            ("释放通话材料", lambda: ctx.release_call_material_references()),
        ]:
            await step(label, action)
        if calls.current is ctx:
            await step("清除当前通话", lambda: calls.clear_current(ctx))
        await step("发布结束事件", lambda:
                   ui.publish_call_ended(ctx.call_id, reason, tuple(errors)))
        ctx.finish_result = FinishResult(reason, tuple(errors))
        if not ctx.finish_future.done():
            ctx.finish_future.set_result(ctx.finish_result)
```

调用者用 `await asyncio.shield(done)` 等待返回的句柄，取消等待不会取消共享收尾任务。若某个清理协程返回取消，也记录为错误并继续其他步骤。组件在自身停止标志下拒绝新工作，等待在途回调退出后释放其资源；失败结果如实保留，通话队列不自动重试这些副作用。

prepare_end_signaling 按分支决定 reject、terminate 或带结束提示且移除密钥的 pending offer；local_hangup 是本地原因，由该策略映射到线上 reason。清除引用不等于高级语言内存物理清零。正常挂断保留消息连接，完成结果同时携带业务原因与收尾错误。

[术语与常见问题](08-glossary-and-faq.md#zh) · [完整字段参考](03-signaling-fields.md#zh)

[English](#en)

<a id="en"></a>

## English

[Home](../README.md#en) · [中文](#zh)

The following Python-style pseudocode connects the signaling flow for one-to-one voice. Phase names belong to this model; native values are in the [state appendix](06-state-and-recovery.md#en-native-state-map). State entry points use the same event queue, and asynchronous results return to that queue for context revalidation.

### Component contracts

| Component | Contract |
| --- | --- |
| directory / identity | Trusted device directory, PN/LID mapping, source and current-device-scope validation |
| signal / protobuf | Per-device sessions, application padding and Message.call.callKey; see the [key chapter](04-device-routing-and-keys.md#en-signal-envelope) |
| jobs / queue | Background operations report success or failure; queue.post schedules a call-queue callback |
| writer / pending | Binary node writing and pending-acknowledgment records, with failure passed to the applicable policy |
| confirmations / signaling | Dedicated acknowledgments, bounded parsing and the [complete field rules](03-signaling-fields.md#en) |
| engine / transport / media | Call events, path updates and audio lifetimes; media packet protection is outside this pseudocode |
| policy / recent_results | Branch policy, retries and final-outcome caching |

Node bodies contain raw bytes. ids generates call and stanza IDs separately. ctx holds the current call, per-device key mapping and task generation. Parsing helpers follow the field reference without repeating every field check in each section.

### 1. Initiation and per-device envelopes

```python
import asyncio
import inspect

def current(ctx, saved):
    return (calls.current is ctx and not ctx.ending
            and (ctx.call_id, ctx.generation) == saved)

def deliver(ctx, saved, result, apply):
    if not current(ctx, saved):
        result.release_call_material()
    elif result.failed:
        begin_finish(ctx, result.reason, send_end=False)
    else:
        apply(result)

def start_outgoing(peer):
    preflight.require_available_and_permitted()
    ctx = calls.create(ids.new_call_id(), peer, role="caller")
    saved = (ctx.call_id, ctx.generation)
    def prepared(result):
        ctx.device_keys = result.keys
        ctx.allowed_devices = result.devices
        ctx.phase = "CALLING"
        send_call(ctx, peer, result.offer)
    jobs.submit(prepare_offer(ctx.snapshot()), completed=lambda result:
        queue.post(lambda: deliver(ctx, saved, result, prepared)))

async def prepare_offer(snapshot):
    devices = await directory.resolve_callable_devices(snapshot.peer)
    identity.require_valid_target_set(snapshot.peer, devices)
    keys, targets = {}, []
    for device in devices:
        await signal.ensure_session_ready(device)
        keys[device] = secure_random.bytes(32)
        plain = protobuf.encode_message_call(call_key=keys[device])
        cipher = await signal.encrypt_call_message(device, plain)
        targets.append(Node("to", {"jid": device}, children=[Node(
            "enc", {"v": "2", "type": cipher.wire_type},
            body=cipher.serialized_bytes)]))
    offer = Node("offer", common_attrs(snapshot), children=
        capabilities.offer_children() + [Node("destination", children=targets)])
    return PreparedOffer(offer, keys, devices)
```

The Signal adapter applies application padding once; type comes from the actual ciphertext result. See [pending offers](04-device-routing-and-keys.md#en-pending-offer) for PreKey waiting and stale-callback rules. The same construction layer also supports a direct enc for a single device.

### 2. Sending and ACK correlation

```python
def send_call(ctx, target, payload):
    request_id = ids.new_stanza_id()
    record = pending.add(request_id, payload.tag, ctx.call_id, ctx.generation,
                         expected_route=policy.ack_route(target, payload.tag))
    writer.enqueue(Node("call", {"to": target, "id": request_id},
                        children=[payload]),
        failed=lambda error: queue.post(lambda: pending.write_failed(record, error)))
    return request_id

def on_call_ack(node):
    ack = signaling.parse_ack_envelope(node)  # Validate from/id/class/type and parse error.
    record = pending.find(ack.id, ack.type)
    if record is None:
        return
    identity.require_expected_ack_source(ack.source, record.expected_route)
    outcome = signaling.parse_ack_payload(ack)
    if outcome.ignored:
        return
    if outcome.call_id is not None:
        require(outcome.call_id == record.call_id)
    pending.note_ack(record, outcome)
    ctx = calls.match(record.call_id, record.generation)
    if ctx is None or ctx.ending:
        return
    if outcome.failed:
        policy.handle_signaling_failure(ctx, record, outcome)
    else:
        engine.apply_ack_configuration(ctx, outcome)
```

Register the transaction before writing to avoid losing correlation with a fast ACK. Payload parsing keeps the branches below; see [acknowledgment semantics](02-call-lifecycle.md#en-confirmations) for the distinction from answering.

| ACK type | Rules the parser must retain |
| --- | --- |
| offer | Select a nonempty call-id by group_info→relay→first error; successful one-to-one requires relay and at least one voip_settings; retain outer and per-target errors |
| accept | Successful non-waiting-room requires accept@call-id and relay; settings are optional; missing accept is ignored; the error branch reads error@call-id and jid |
| relaylatency | Missing node or call-id permits fallback correlation; an explicit ID is nonempty, and the outer error is retained |

Full fields are in [offer ACK](03-signaling-fields.md#en-message-offer_ack), [accept ACK](03-signaling-fields.md#en-message-accept_ack) and [measurement ACK](03-signaling-fields.md#en-message-relaylatency_ack).

### 3. Reception, acknowledgment and offer decryption

```python
def on_node(node):
    signaling.require_envelope_limits(node)
    if node.tag == "ack" and node.attr("class") == "call":
        return on_call_ack(node)
    if node.tag == "receipt":
        receipt = confirmations.parse_receipt(node)
        confirmations.ack_receipt(receipt)
        return confirmations.note_receipt(receipt)
    if node.tag != "call":
        return other_stanzas.dispatch(node)
    event = signaling.parse_call_header(node)
    identity.require_valid_call_source(event)
    confirmations.schedule_for_received_signal(event)
    if received_signals.is_exact_duplicate(event):
        return
    received_signals.remember_bounded(event)
    if recent_results.suppresses(event):
        return recent_results.apply_late_event_if_needed(event)
    if event.tag == "offer":
        return receive_offer(event)
    ctx = calls.find_by_call_and_peer(event.call_id, event.peer_user)
    if ctx is None:
        return policy.buffer_or_ignore_without_offer(event)
    if not ctx.ending:
        identity.require_allowed_event(ctx, event)
        dispatch_current_event(ctx, event)

def receive_offer(event):
    offer = signaling.parse_offer(event)
    decision = policy.check_existing_call_and_offer(offer)
    if decision.is_retry:
        return policy.handle_offer_retry(offer)
    if decision.reject_busy:
        return signaling.send_reject_for_offer(offer, reason="busy")
    ctx = calls.reserve_incoming(offer, phase="DECRYPTING")
    saved = (ctx.call_id, ctx.generation)
    jobs.submit(decrypt_offer(offer), completed=lambda result: queue.post(
        lambda: deliver(ctx, saved, result,
                        lambda material: engine.receive_clear_offer(ctx, offer, material.key))))

async def decrypt_offer(offer):
    enc = signaling.select_unique_local_enc(offer, identity.local_device_aliases())
    require(enc.attr("v") == "2" and enc.attr("type") in {"msg", "pkmsg"})
    plain = await signal.decrypt_call_message(offer.source_device, enc.attr("type"), enc.body)
    key = protobuf.parse_message(plain).require_call_key(length=32)
    return DecryptedCallMaterial(key)
```

jobs delivers result objects with failed, reason and a call-material release method, and cleans up undelivered temporary material from failed tasks. The normal keygen2 path reads 32-byte material; other extensions use their corresponding handlers. Duplicate business signals may still be acknowledged; duplicate Signal ciphertext, identity failures and busy state take distinct branches.

confirmations selects dedicated or generic paths according to tag and silent-call conditions. A normal non-silent offer may schedule its receipt before the dedicated ACK. The allowed device scope also includes legitimate notifications from other devices of the local account; policy then checks handled-elsewhere conditions.

### 4. Preaccept, answering and device selection

```python
def on_incoming_offer_ready(ctx):
    if ctx.ending:
        return
    ctx.phase = "INCOMING"
    send_call(ctx, ctx.peer_device, Node("preaccept", common_attrs(ctx),
              children=capabilities.preaccept_children()))
    ui.present_incoming_according_to_platform_policy(ctx)

def user_accept(call_id):
    ctx = calls.require_current(call_id)
    require(not ctx.ending and ctx.phase == "INCOMING")
    engine.accept_incoming(ctx)  # The engine produces the accept node for this attempt.

def on_local_accept_payload(ctx, payload):
    if not ctx.ending:
        send_call(ctx, ctx.peer_device, payload)
        ctx.phase = "ACCEPT_SENT"

def on_peer_accept(ctx, event):
    identity.require_offered_device(ctx, event.source_device)
    if ctx.chosen_device is None:
        ctx.chosen_device = event.source_device
        ctx.active_material = ctx.device_keys[event.source_device]
        engine.choose_callee_device(ctx, event.source_device)
    elif ctx.chosen_device != event.source_device:
        return policy.handle_nonselected_device_event(ctx, event)
    if not ctx.preaccept_ready:
        ctx.pending_accept = event.deep_copy()
        return
    engine.apply_accept_and_optional_new_material(ctx, event)
    ctx.phase = "ACCEPT_RECEIVED"
    transport.prepare_from_current_signaling(ctx)

def on_peer_preaccept(ctx, event):
    engine.update_device_capability(ctx, event)
    ctx.preaccept_ready = True
    cached, ctx.pending_accept = ctx.pending_accept, None
    if cached is not None:
        queue.post(lambda: dispatch_if_current(ctx, cached))

def dispatch_if_current(ctx, event):
    if calls.current is ctx and not ctx.ending and event.call_id == ctx.call_id:
        identity.require_allowed_event(ctx, event)
        dispatch_current_event(ctx, event)
```

A new enc completes the applicable decryption before engine handoff. Redispatch rechecks the current call and ending. Formal selection and termination notices to other devices are in the [device chapter](04-device-routing-and-keys.md#en-device-selection); cache ordering is in the [state chapter](06-state-and-recovery.md#en-accept-ordering).

<a id="en-transport-dispatch"></a>

### 5. Dispatching transport updates and end events

```python
def dispatch_current_event(ctx, event):
    if policy.is_handled_elsewhere_event(ctx, event):
        return engine.handle_other_self_device(ctx, event)
    tag = event.tag
    if tag == "preaccept":
        return on_peer_preaccept(ctx, event)
    if tag == "accept":
        require(ctx.role == "caller")
        return on_peer_accept(ctx, event)
    if tag == "transport":
        return transport.apply_candidates(ctx, signaling.parse_transport(event))
    if tag == "relaylatency":
        return transport.record_measurements(ctx, signaling.parse_relaylatency(event))
    if tag == "ready":
        return engine.on_ready(ctx, signaling.parse_ready_type(event))
    if tag == "relay_info_update":
        update = signaling.parse_relay_update(event)
        if update.relay is not None:
            transport.apply_relay_update(ctx, update.relay)
    elif tag in {"reject", "terminate"}:
        identity.require_end_event_applies(ctx, event)
        recent_results.record_end_event(event)
        begin_finish(ctx, event.reason, send_end=False)
```

Transport parsing preserves type-2 te2 handling, type-13 ICE credentials and candidate rounds. Measurement parsing retains usable-entry and error rules. The default ready type and optional relay update are described in [negotiation](05-transport-negotiation.md#en-updates). These events update their own state rather than all assigning ACTIVE.

<a id="en-finish"></a>

### 6. Cleanup continues after errors

The first end event publishes a shared completion handle and sets ending; repeated ending returns that handle. End-plan preparation, each send and each cleanup step collect errors separately. finally guarantees entry into resource cleanup, and transport-cleanup failure still allows media and call-material release to be attempted.

```python
def user_hangup(call_id):
    return begin_finish(calls.require_current(call_id), "local_hangup", send_end=True)

def begin_finish(ctx, reason, send_end):
    if ctx.ending:
        return ctx.finish_future
    loop = asyncio.get_running_loop()
    ctx.finish_future = loop.create_future()
    ctx.ending = True
    ctx.generation += 1
    ctx.cleanup_task = loop.create_task(finish_call(ctx, reason, send_end))
    return ctx.finish_future

async def finish_call(ctx, reason, send_end):
    errors = []
    async def step(label, action):
        try:
            value = action()
            return await value if inspect.isawaitable(value) else value
        except (Exception, asyncio.CancelledError) as error:
            errors.append((label, error))
            return None
    try:
        await step("stop timers", lambda: timers.cancel_for_call(ctx.call_id))
        await step("stop tracking", lambda: policy.stop_offer_and_accept_tracking(ctx))
        plan = await step("prepare end signaling", lambda:
                          policy.prepare_end_signaling(ctx, reason, send_end))
        if plan is not None:
            for target, payload in plan.payloads:
                await step("send end signaling", lambda target=target, payload=payload:
                           send_call(ctx, target, payload))
            await step("record final outcome", lambda:
                       recent_results.record_final(ctx.call_id, plan.outcome, reason))
    except (Exception, asyncio.CancelledError) as error:
        errors.append(("process end plan", error))
    finally:
        for label, action in [
            ("stop media", lambda: media.begin_stop(ctx)),
            ("stop transport", lambda: transport.begin_stop(ctx)),
            ("join transport", lambda: transport.join_and_release(ctx)),
            ("join media", lambda: media.join_and_release(ctx)),
            ("release call material", lambda: ctx.release_call_material_references()),
        ]:
            await step(label, action)
        if calls.current is ctx:
            await step("clear current call", lambda: calls.clear_current(ctx))
        await step("publish ended event", lambda:
                   ui.publish_call_ended(ctx.call_id, reason, tuple(errors)))
        ctx.finish_result = FinishResult(reason, tuple(errors))
        if not ctx.finish_future.done():
            ctx.finish_future.set_result(ctx.finish_result)
```

Callers await the returned handle using `await asyncio.shield(done)`, so cancelling a waiter does not cancel shared cleanup. Cancellation returned by a cleanup coroutine is also recorded and the remaining steps continue. Components reject new work under their own stopping flags and wait for in-flight callbacks before releasing resources. Failures remain in the result, and the call queue does not automatically retry these side effects.

prepare_end_signaling chooses reject, terminate or a pending offer with an ending hint and its key removed. local_hangup is a local reason mapped to the wire reason by that policy. Releasing references does not guarantee physical zeroing of all high-level-runtime memory. Normal hangup retains the messaging connection; the completion result carries both the business reason and cleanup errors.

[Glossary and FAQ](08-glossary-and-faq.md#en) · [Complete field reference](03-signaling-fields.md#en)

[中文](#zh)
