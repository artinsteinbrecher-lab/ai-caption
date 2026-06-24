# 小智 ESP32 实时字幕辅助设备 — 代码审查报告

审查范围：后端 3 文件 + 固件 3 文件，基于 78/xiaozhi-esp32 与 xinnan-tech/xiaozhi-esp32-server 二次开发。

---

## 一、总体结论

**不建议直接合并/部署到量产设备。** 当前代码能跑通"音频上传→ASR→字幕显示"主链路，但存在多个会导致设备卡死、连接泄漏、字幕丢失的问题。

**最大风险有三个：**

1. **静音超时后连接永远不关闭**（`no_voice_close_connect` 在 caption_mode 下走入 `startToChat` 短路分支，`close_after_chat=True` 但无 chat 发生，连接泄漏）。120 秒静音后设备将彻底卡住，无法恢复，只能重启。
2. **断线后无自动重连**（固件进入 idle 后清空所有字幕，需手动按键或说唤醒词才能恢复）。对听障辅助设备来说，这是致命的可用性缺陷。
3. **每次 ASR 中间结果都 `lv_obj_clean` 重建 LVGL 对象**，在 240×240 LCD 上每秒可能触发 5-10 次，会造成内存碎片、屏闪、丢帧。

建议在修复以上三个问题后再部署验证。

---

## 二、必须修复的问题

按严重程度从高到低排序。

### P0-1. 静音超时后连接泄漏 + 结束语被当作字幕显示

**文件**：`main/xiaozhi-server/core/handle/receiveAudioHandle.py`
**位置**：`no_voice_close_connect()` 第 114–128 行

**问题说明**：
当设备 120 秒无语音输入时，`no_voice_close_connect` 设置 `close_after_chat = True`，然后调用 `startToChat(conn, end_prompt)` 试图发送结束语。但在 caption_mode 下，`startToChat` 在第 71-74 行短路返回：

```python
if conn.config.get("caption_mode", False):
    await send_display_message(conn, actual_text)
    conn.logger.bind(tag=TAG).info(f"caption_mode: display ASR text only: {actual_text}")
    return
```

这段代码会把**结束语文本**（"请你以时间过得真快……"）当作字幕发送到设备屏幕显示，然后直接返回。`close_after_chat = True` 被设置了，但没有任何 chat 流程会触发关闭回调。

**可能后果**：
- 连接永远不会被关闭，WebSocket 资源泄漏。
- 设备屏幕上出现结束语文本作为字幕，对听障用户造成困惑。
- 后续音频不再被处理（因为连接状态异常），设备彻底卡死。

**建议修复方式**：
在 `no_voice_close_connect` 中、调用 `startToChat` 之前，增加 caption_mode 判断：

```python
if conn.config.get("caption_mode", False):
    conn.logger.bind(tag=TAG).info("caption_mode: 静音超时，直接关闭连接")
    await conn.close()
    return
```

---

### P0-2. 断线后设备不自动重连，且清空所有历史字幕

**文件**：`main/application.cc`
**位置**：`HandleStateChangedEvent()` 第 873-879 行（idle 状态）、`OnAudioChannelClosed` 回调第 514-521 行

**问题说明**：
当服务器断开或网络中断时，`OnAudioChannelClosed` 将状态设为 `kDeviceStateIdle`。在 idle 状态处理中：

```cpp
case kDeviceStateIdle:
    display->SetStatus(Lang::Strings::STANDBY);
    display->ClearChatMessages();  // ← 清空所有字幕
    display->SetEmotion("neutral");
    audio_service_.EnableVoiceProcessing(false);
    audio_service_.EnableWakeWordDetection(true);  // ← 等待唤醒词
    break;
```

设备进入待机，清空所有字幕，然后等待用户说唤醒词或按按钮才能恢复。对于一个听障辅助设备，用户可能无法说话，也不会知道什么时候网络断了。

**可能后果**：
- 网络短暂抖动即丢失所有字幕。
- 听障用户无法通过语音唤醒词恢复设备。
- 设备显示"待机"但用户不知道发生了什么。

**建议修复方式**：
在 caption 设备模式下，`OnAudioChannelClosed` 不应进入 idle，而应进入一个"重连中"状态，并在短暂延迟后自动重连：

```cpp
protocol_->OnAudioChannelClosed([this, &board]() {
    board.SetPowerSaveLevel(PowerSaveLevel::LOW_POWER);
    Schedule([this]() {
        auto display = Board::GetInstance().GetDisplay();
        if (caption_mode_) {
            // 不清空字幕，显示"重连中"，自动重连
            display->SetStatus("重连中...");
            SetDeviceState(kDeviceStateConnecting);
            // 3秒后自动重连
            Schedule([this]() {
                if (GetDeviceState() == kDeviceStateConnecting) {
                    ContinueOpenAudioChannel(kListeningModeRealtime);
                }
            });
        } else {
            display->SetChatMessage("system", "");
            SetDeviceState(kDeviceStateIdle);
        }
    });
});
```

并在 idle 状态处理中对 caption_mode 跳过 `ClearChatMessages()`。

---

### P0-3. 每次 ASR 中间结果都 lv_obj_clean 重建整个内容区域

**文件**：`main/display/lcd_display.cc`
**位置**：`SetChatMessage()` caption 分支第 571-592 行（wechat style）

**问题说明**：
paraformer-realtime-v2 每句话会产生 5-20 个中间结果（每个结果增加几个字）。当前代码在每次收到 caption 消息时：

```cpp
if (strcmp(role, "caption") == 0) {
    lv_obj_clean(content_);           // ← 删除所有子对象
    lv_obj_set_flex_flow(content_, LV_FLEX_FLOW_COLUMN);  // ← 重新设置布局
    lv_obj_set_flex_align(content_, ...);                 // ← 重新设置对齐
    lv_obj_t* caption_label = lv_label_create(content_);   // ← 创建新标签
    lv_label_set_text(caption_label, content);
    ...
}
```

每秒可能执行 5-10 次完整的对象销毁+创建+布局计算。

**可能后果**：
- LVGL 内存碎片化，长时间运行后分配失败。
- 屏幕闪烁（clean 到 create 之间有空帧）。
- 在 ESP32-S3 有限 RAM 上可能触发看门狗复位。
- `chat_message_label_` 指针频繁变更，如果其他线程在此期间访问会崩溃。

**建议修复方式**：
在 caption 分支中，检查是否已有 caption 标签，有则只更新文本：

```cpp
if (strcmp(role, "caption") == 0) {
    lv_obj_add_flag(emoji_label_, LV_OBJ_FLAG_HIDDEN);
    if (emoji_image_ != nullptr) {
        lv_obj_add_flag(emoji_image_, LV_OBJ_FLAG_HIDDEN);
    }
    
    // 如果已有 caption 标签且 content_ 中只有这一个子对象，直接更新文本
    if (chat_message_label_ != nullptr && 
        lv_obj_get_parent(chat_message_label_) == content_ &&
        lv_obj_get_child_cnt(content_) == 1) {
        lv_label_set_text(chat_message_label_, content);
        return;
    }
    
    // 否才重建
    lv_obj_clean(content_);
    lv_obj_set_style_pad_all(content_, lvgl_theme->spacing(8), 0);
    lv_obj_set_flex_flow(content_, LV_FLEX_FLOW_COLUMN);
    lv_obj_set_flex_align(content_, LV_FLEX_ALIGN_CENTER, LV_FLEX_ALIGN_CENTER, LV_FLEX_ALIGN_CENTER);
    
    lv_obj_t* caption_label = lv_label_create(content_);
    lv_label_set_text(caption_label, content);
    lv_label_set_long_mode(caption_label, LV_LABEL_LONG_WRAP);
    lv_obj_set_width(caption_label, LV_HOR_RES - lvgl_theme->spacing(16));
    lv_obj_set_style_text_color(caption_label, lvgl_theme->text_color(), 0);
    lv_obj_set_style_text_align(caption_label, LV_TEXT_ALIGN_CENTER, 0);
    lv_obj_set_style_pad_all(caption_label, 0, 0);
    lv_obj_center(caption_label);
    chat_message_label_ = caption_label;
    return;
}
```

---

### P1-1. vocabulary_id 为 null 时仍被发送到阿里云 API

**文件**：`main/xiaozhi-server/core/providers/asr/aliyunbl_stream.py`
**位置**：`_build_run_task_message()` 第 156-158 行

**问题说明**：

```python
if self.model.lower().endswith("v2"):
    message["payload"]["parameters"]["vocabulary_id"] = self.vocabulary_id
```

当 `vocabulary_id` 在 config.yaml 中被注释掉时（当前状态），`self.vocabulary_id` 为 `None`。代码仍然把 `vocabulary_id: None` 放入请求 JSON，序列化后变成 `"vocabulary_id": null`。阿里云 API 可能拒绝此请求或返回错误。

**可能后果**：ASR 任务启动失败，设备收不到任何字幕。

**建议修复方式**：

```python
if self.model.lower().endswith("v2") and self.vocabulary_id:
    message["payload"]["parameters"]["vocabulary_id"] = self.vocabulary_id
```

---

### P1-2. 非 WeChat 样式的 LCD 没有任何 caption 角色处理

**文件**：`main/display/lcd_display.cc`
**位置**：`#else` 分支的 `SetChatMessage()` 第 1061-1088 行

**问题说明**：
caption 角色的特殊处理只存在于 `#if CONFIG_USE_WECHAT_MESSAGE_STYLE` 分支（第 571-592 行）。在 `#else` 分支（非 WeChat 样式），`SetChatMessage` 直接调用 `lv_label_set_text(chat_message_label_, content)`，不区分 caption 和其他角色。

如果目标硬件没有启用 `CONFIG_USE_WECHAT_MESSAGE_STYLE`，字幕会以单行滚动方式显示在底部 bottom_bar 中，而不是占据整块内容区域。长字幕会被截断或滚动过快无法阅读。

**可能后果**：在非 WeChat 样式编译的固件上，字幕完全不可读或显示异常。

**建议修复方式**：
在 `#else` 分支的 `SetChatMessage` 中也添加 caption 角色处理。至少将 `chat_message_label_` 切换到 wrap 模式并扩展宽度：

```cpp
if (strcmp(role, "caption") == 0) {
    lv_label_set_long_mode(chat_message_label_, LV_LABEL_LONG_WRAP);
    lv_obj_set_width(chat_message_label_, LV_HOR_RES - lvgl_theme->spacing(8));
    lv_obj_set_style_text_align(chat_message_label_, LV_TEXT_ALIGN_CENTER, 0);
    // 让 bottom_bar 占据更多高度
    if (bottom_bar_ != nullptr) {
        lv_obj_set_height(bottom_bar_, LV_VER_RES * 70 / 100);
        lv_obj_align(bottom_bar_, LV_ALIGN_CENTER, 0, 0);
    }
}
lv_label_set_text(chat_message_label_, content);
```

---

### P1-3. OLED 隐藏 content_left_ 后永不恢复

**文件**：`main/display/oled_display.cc`
**位置**：`SetChatMessage()` 第 163-169 行

**问题说明**：

```cpp
if (strcmp(role, "caption") == 0 && content_left_ != nullptr) {
    lv_obj_add_flag(content_left_, LV_OBJ_FLAG_HIDDEN);
    lv_label_set_long_mode(chat_message_label_, LV_LABEL_LONG_WRAP);
    lv_obj_set_style_text_align(chat_message_label_, LV_TEXT_ALIGN_CENTER, 0);
    lv_obj_set_width(chat_message_label_, width_ - 4);
    lv_obj_set_style_pad_top(chat_message_label_, 2, 0);
}
```

一旦收到 caption 消息，`content_left_`（情感图标区域）被永久隐藏，`chat_message_label_` 的长模式、对齐方式、宽度被永久修改。后续如果收到 `system` 或 `assistant` 角色消息，这些属性不会恢复，导致普通消息显示异常。

**可能后果**：网络重连提示、错误信息等系统消息在 OLED 上显示异常（居中换行而非左对齐滚动）。

**建议修复方式**：
在收到非 caption 角色消息时恢复原始属性：

```cpp
if (strcmp(role, "caption") == 0 && content_left_ != nullptr) {
    lv_obj_add_flag(content_left_, LV_OBJ_FLAG_HIDDEN);
    lv_label_set_long_mode(chat_message_label_, LV_LABEL_LONG_WRAP);
    lv_obj_set_style_text_align(chat_message_label_, LV_TEXT_ALIGN_CENTER, 0);
    lv_obj_set_width(chat_message_label_, width_ - 4);
    lv_obj_set_style_pad_top(chat_message_label_, 2, 0);
} else if (content_left_ != nullptr) {
    // 恢复非 caption 模式的显示属性
    lv_obj_remove_flag(content_left_, LV_OBJ_FLAG_HIDDEN);
    lv_label_set_long_mode(chat_message_label_, LV_LABEL_LONG_SCROLL_CIRCULAR);
    lv_obj_set_style_text_align(chat_message_label_, LV_TEXT_ALIGN_LEFT, 0);
    lv_obj_set_width(chat_message_label_, width_ - 32);
    lv_obj_set_style_pad_top(chat_message_label_, 14, 0);
}
```

---

### P1-4. _cleanup 不取消 forward_task，可能造成竞态

**文件**：`main/xiaozhi-server/core/providers/asr/aliyunbl_stream.py`
**位置**：`_cleanup()` 第 298-327 行

**问题说明**：
`_cleanup()` 只设置 `self.forward_task = None` 丢弃引用，但不 cancel 正在运行的 task。如果 `_cleanup` 是从 `receive_audio`（错误分支）调用的，`_forward_results` 协程仍在 `self.asr_ws.recv()` 上等待。此时 `asr_ws` 被设为 `None`，下一轮 `recv()` 会抛出 `AttributeError`。

更严重的是，如果在新 `_start_recognition` 创建新 `forward_task` 之前旧 task 尚未结束，两个 task 可能同时操作 `self.asr_ws`。

**可能后果**：间歇性 ASR 异常、音频数据混乱、WebSocket 操作冲突。

**建议修复方式**：

```python
async def _cleanup(self):
    self.is_processing = False
    self.server_ready = False

    # 先取消 forward_task
    if self.forward_task and not self.forward_task.done():
        self.forward_task.cancel()
        try:
            await self.forward_task
        except asyncio.CancelledError:
            pass

    if self.asr_ws:
        try:
            await self._send_finish_task()
            await asyncio.sleep(0.1)
            await asyncio.wait_for(self.asr_ws.close(), timeout=2.0)
        except Exception as e:
            logger.bind(tag=TAG).error(f"关闭WebSocket连接失败: {e}")
        finally:
            self.asr_ws = None

    self.forward_task = None
    self.task_id = None
```

---

### P1-5. 每次 ASR 会话都新建 WebSocket 连接，延迟过高

**文件**：`main/xiaozhi-server/core/providers/asr/aliyunbl_stream.py`
**位置**：`_start_recognition()` 第 83-127 行

**问题说明**：
每检测到一段语音（VAD 触发），代码就创建一个全新的 WebSocket 连接到阿里云。连接流程是：TCP 握手 → TLS 握手 → WebSocket 升级 → 发送 run-task → 等待 task-started → 开始发送音频。整个过程可能耗时 300ms-1s。

在日常对话中，每句话之间会有 200-600ms 的停顿（`max_sentence_silence: 200`），每次停顿都会触发 `is_final` → `_cleanup` → 新连接。这意味着每句话开头都会丢失 300ms-1s 的音频（虽然发送了 10 个缓存帧约 600ms，但仍可能不够）。

**可能后果**：每句话开头几个字丢失，字幕不完整。

**建议修复方式**：
将 WebSocket 连接改为长连接模式。在 `__init__` 或 `open_audio_channels` 时建立连接，每次语音段只发送 run-task/finish-task，不关闭底层 WebSocket。或者使用阿里云 paraformer-realtime 的持续模式，不发送 finish-task 直到连接关闭。

这是一个较大的架构改动，详见第四节架构建议。

---

### P1-6. caption_last_text 去重可能在 ASR 文本修正时丢失最终结果

**文件**：`main/xiaozhi-server/core/providers/asr/aliyunbl_stream.py`
**位置**：`_forward_results()` 第 207-211 行

**问题说明**：

```python
if self.caption_mode and text:
    last_text = getattr(conn, "caption_last_text", "")
    if text != last_text:
        await send_display_message(conn, text)
        conn.caption_last_text = text
```

paraformer-realtime 的中间结果会逐步增长（如 "你好" → "你好世界" → "你好世界。"），通常每帧文本不同，去重没问题。

但有一个边缘情况：最终结果（`is_final=True`）的文本有时与最后一个中间结果完全相同（仅 `sentence_end` 从 False 变为 True）。此时去重会导致最终结果不发送。

虽然最终结果文本与中间结果相同时不发送不影响显示内容（屏幕已显示相同文本），但如果后续逻辑依赖最终结果的发送（比如将来添加字幕历史记录），就会出问题。

**可能后果**：当前影响较小，但限制了未来功能扩展。

**建议修复方式**：
对 `is_final` 的结果总是发送，不做去重：

```python
if self.caption_mode and text:
    last_text = getattr(conn, "caption_last_text", "")
    if text != last_text or is_final:
        await send_display_message(conn, text)
        conn.caption_last_text = text
```

---

### P1-7. config.yaml 中 caption_mode 双重定义，容易失配

**文件**：`main/xiaozhi-server/config.yaml`
**位置**：第 254 行（顶层）和第 588 行（ASR.AliyunBLStreamASR 内部）

**问题说明**：
`caption_mode` 在两个位置定义：
- 顶层 `caption_mode: true`（第 254 行）—— 被 `receiveAudioHandle.py` 的 `startToChat` 读取
- `ASR.AliyunBLStreamASR.caption_mode: true`（第 588 行）—— 被 `aliyunbl_stream.py` 的 ASR Provider 读取

两个值必须同时为 true 才能正常工作。如果只改了一个：
- 顶层 false + ASR true：ASR 会发 display 消息，但 `startToChat` 仍走 LLM/TTS 流程，设备同时收到字幕和 TTS 语音。
- 顶层 true + ASR false：ASR 不发中间结果，只有 `startToChat` 短路，但此时 `startToChat` 收到的 `actual_text` 为空（因为 `self.text` 从未被设置）。

**可能后果**：配置修改时容易漏改一处，导致行为异常。

**建议修复方式**：
只保留顶层 `caption_mode`，在 ASR Provider 中通过 `conn.config.get("caption_mode")` 读取，不在 ASR 配置节中重复定义。或者反过来，只在 ASR 配置中定义，在 `receiveAudioHandle.py` 中通过 `conn.asr.caption_mode` 读取。

---

### P2-1. stt 消息类型被硬编码为 caption 角色，破坏非 caption 模式

**文件**：`main/application.cc`
**位置**：`OnIncomingJson` stt 处理 第 552-559 行

**问题说明**：

```cpp
} else if (strcmp(type->valuestring, "stt") == 0) {
    ...
    Schedule([display, message = std::string(text->valuestring)]() {
        display->SetChatMessage("caption", message.c_str());
    });
}
```

原始固件中 stt 消息显示为 `"user"` 角色（用户气泡），改造后硬编码为 `"caption"` 角色。如果将来需要切换回对话模式，或者同一个固件被用于非 caption 设备，用户语音将被显示为字幕样式而非用户气泡。

**可能后果**：固件无法在 caption 和非 caption 模式间切换。

**建议修复方式**：
添加一个编译时宏或运行时配置标志：

```cpp
#ifdef CONFIG_CAPTION_DEVICE_MODE
    display->SetChatMessage("caption", message.c_str());
#else
    display->SetChatMessage("user", message.c_str());
#endif
```

---

### P2-2. need_bind 检查在 caption_mode 检查之前，会触发 TTS

**文件**：`main/xiaozhi-server/core/handle/receiveAudioHandle.py`
**位置**：`startToChat()` 第 67-69 行

**问题说明**：

```python
if conn.need_bind:
    await check_bind_device(conn)
    return

if conn.config.get("caption_mode", False):
    ...
```

`check_bind_device` 会向 `conn.tts.tts_audio_queue` 放入音频数据（第 157 行），触发 TTS 播放绑定码语音。这在 caption_mode 下也会发生。

虽然设备绑定是初始化阶段的一次性操作，且绑定码语音对用户有帮助，但严格来说这违反了"caption_mode 下不触发 TTS"的原则。

**可能后果**：设备首次绑定时会播放语音，在听障设备场景中可能不必要。

**建议修复方式**：
这个问题影响不大（绑定只发生一次），可以接受。但如果要严格遵循 caption_mode，可以在 `check_bind_device` 中检查 caption_mode 并跳过 TTS 音频播放，只发送显示消息。

---

### P2-3. enable_websocket_ping 为 false，长连接可能被中间件断开

**文件**：`main/xiaozhi-server/config.yaml`
**位置**：第 75 行

**问题说明**：
`enable_websocket_ping: false`。对于需要长时间保持连接的字幕设备，如果没有 WebSocket 心跳，NAT、防火墙、反向代理等中间件可能在 30-60 秒无数据后断开连接。

**可能后果**：设备运行一段时间后连接被静默断开，字幕停止更新且无提示。

**建议修复方式**：
对于 caption 设备，设为 `true`。

---

## 三、建议优化的问题

### O-1. caption 布局修改不可逆，影响后续系统消息显示

**文件**：`lcd_display.cc` `SetChatMessage()` 第 577-592 行

caption 分支修改了 `content_` 的 padding、flex_flow、flex_align，但这些修改在收到非 caption 消息时不会恢复。后续 system 消息会在 caption 布局的 content_ 中创建气泡，显示效果与预期不同。

**建议**：在 system/assistant/user 分支入口处检查并恢复 content_ 的原始布局属性，或用一个独立的 `caption_label_` 指针替代复用 `content_`。

### O-2. forward_task 的 recv timeout 为 1 秒，静音时持续空转

**文件**：`aliyunbl_stream.py` 第 172 行

`asyncio.wait_for(self.asr_ws.recv(), timeout=1.0)` 在静音时每秒超时一次并循环。虽然不影响正确性，但每秒一次的 wake-up 增加了不必要的 CPU 使用。

**建议**：增大 timeout 到 5-10 秒，或在收到 is_final 后退出循环等待下一次语音。

### O-3. 缺少 ASR 错误状态反馈到设备

**文件**：`aliyunbl_stream.py` 第 244-248 行

`task-failed` 事件只记录日志并 break，不通知设备。设备屏幕停留在最后一帧字幕，用户不知道出了问题。

**建议**：在 task-failed 时发送 `send_display_message(conn, "[识别错误，正在恢复...]")` 或通过 system 消息通知设备。

### O-4. 字体大小未针对字幕场景优化

**文件**：`lcd_display.cc` 第 583-590 行

caption 标签使用 `lvgl_theme->text_color()` 的颜色和默认字体大小。对于 240×240 的 LCD，默认字体可能太小，听障用户难以阅读。

**建议**：为 caption 角色使用更大的字体（如果可用），或至少确保字体大小是可配置的。在 LVGL 中可以用 `lv_obj_set_style_text_font(caption_label, &lv_font_montserrat_16, 0)` 或更大的字体。

### O-5. 字幕不保留历史记录

**文件**：`lcd_display.cc` 第 577 行

`lv_obj_clean(content_)` 清除了所有历史内容，只显示当前句。对于听障用户，可能需要看到最近 2-3 句话作为上下文。

**建议**：保留最近 3 条字幕，新的字幕在底部，旧的向上滚动。可以用一个有限长度的消息列表替代 `lv_obj_clean`。

### O-6. 240×240 屏幕未做差异化布局

**文件**：`lcd_display.cc`

当前 caption 布局对分辨率无感知。240×240 是方形屏幕，字幕可以占据下方 70% 区域，上方 30% 保留状态信息。128x64 OLED 已做差异化（128x64 vs 128x32），但 LCD 没有。

**建议**：根据 `width_` 和 `height_` 动态调整 caption 区域大小和字体。

### O-7. README 中"Verification Done"部分不准确

**文件**：`README.md` 第 79 行

写着 "Firmware changes are limited to `main/application.cc`"，但实际还修改了 `lcd_display.cc` 和 `oled_display.cc`。

**建议**：更新 README 反映实际修改的文件列表。

### O-8. api_key 仍为占位符

**文件**：`config.yaml` 第 590 行

`api_key: 你的阿里云百炼API密钥` 是占位符。部署前必须替换为真实 key。建议在启动时检查是否仍为占位符并拒绝启动。

---

## 四、针对实时字幕辅助设备的架构建议

### 后端 ASR 流式策略

当前方案是"每段语音一个 WebSocket 连接"。建议改为**单一长连接 + 持续音频流**模式：

1. 在设备 WebSocket 连接建立时，同时建立到阿里云的 ASR WebSocket 长连接。
2. 设备音频持续发送到 ASR，不依赖 VAD 触发连接建立。
3. ASR 的 `max_sentence_silence` 负责断句，断句结果直接转发到设备。
4. 不发送 `finish-task`，直到设备 WebSocket 断开时才关闭 ASR 连接。

这样每句话开头不会丢失音频，延迟降低 300ms-1s。

如果需要节省 ASR 费用（按时长计费），可以用 VAD 控制：有语音时保持连接，静音超过阈值后发送 finish-task 但保持 WebSocket 打开，下次有语音时发新的 run-task。

### 设备端显示策略

1. **保留 3 条字幕历史**：当前句在底部高亮，上面 2 句灰色渐淡。这比只显示当前句对听障用户友好得多。
2. **状态栏显示连接状态**：在顶部状态栏用图标或文字指示"聆听中 / 重连中 / 识别错误"。
3. **字体大小**：240×240 屏幕建议至少 16px 字体，可配置。
4. **中间结果更新策略**：用 `lv_label_set_text` 更新现有标签，不要 `lv_obj_clean` 重建。只有句子切换时才创建新标签。
5. **标点和断句**：利用 ASR 的 `punctuation_prediction_enabled` 自动添加标点。当前已开启，保持。

### 断线重连策略

1. **指数退避重连**：1s → 2s → 4s → 8s → 16s，上限 30s。
2. **重连时不清屏**：保留当前字幕，在状态栏显示"重连中"。
3. **重连失败超过 5 次显示错误**：提示用户检查网络。
4. **网络恢复后自动恢复监听**：不需要用户干预。

实现位置：在 `OnAudioChannelClosed` 回调中启动重连定时器，而不是进入 idle 状态。

### 配置管理

1. **单一 caption_mode**：只在一个位置定义，建议放在顶层，ASR Provider 通过 `conn.config` 读取。
2. **设备端配置同步**：通过 OTA 下发 caption_mode 配置，而不是硬编码在固件中。
3. **热词配置**：通过 config.yaml 的 `vocabulary_id` 支持热词，对专业场景（医疗、教育）的识别准确率提升很大。当前被注释掉了，需要取消注释并填入真实的 vocabulary_id。
4. **API Key 安全**：不要在 config.yaml 中放真实 key，使用 `data/.config.yaml` 覆盖（config.yaml 第 1-4 行已说明此机制）。

### 日志和隐私处理

1. **音频不落盘**：当前 `delete_audio: true` 会删除音频文件。但 ASR Provider 中的 `output_dir` 仍指向 `tmp/`。确认 ASR Provider 不会在 caption_mode 下保存音频。
2. **文本日志脱敏**：当前 `receiveAudioHandle.py` 第 73 行 `conn.logger.info(f"caption_mode: display ASR text only: {actual_text}")` 会把识别文本写入日志。在隐私敏感场景，应改为只记录长度或哈希。
3. **API Key 不入日志**：`aliyunbl_stream.py` 第 97 行 `f"Bearer {self.api_key}"` 用于 header，不会直接出现在日志中。但异常消息可能包含 header 信息，建议在异常处理中脱敏。
4. **设备端不保存音频**：ESP32-S3 的麦克风数据直接通过 WebSocket 发送，不落盘。这是正确的。

---

## 五、建议的补丁方向

不需要完整代码，以下明确应该改哪些文件、改成什么逻辑：

### 后端

**1. `receiveAudioHandle.py` — `no_voice_close_connect()`**

在 `close_after_chat = True` 之后、`end_prompt` 逻辑之前，增加：
```
if caption_mode:
    await conn.close()
    return
```

**2. `aliyunbl_stream.py` — `_build_run_task_message()`**

第 157 行改为：
```
if self.model.lower().endswith("v2") and self.vocabulary_id:
```

**3. `aliyunbl_stream.py` — `_cleanup()`**

在状态重置之前，增加取消 `forward_task` 的逻辑：
```
if self.forward_task and not self.forward_task.done():
    self.forward_task.cancel()
    try: await self.forward_task
    except asyncio.CancelledError: pass
```

**4. `aliyunbl_stream.py` — `_forward_results()` task-failed 分支**

增加向设备发送错误提示：
```
if self.caption_mode:
    await send_display_message(conn, "[识别异常，正在恢复...]")
```

**5. `aliyunbl_stream.py` — caption 中间结果转发**

考虑将 WebSocket 连接改为长连接模式：在 `open_audio_channels` 时建立连接，`receive_audio` 只发送音频和 run-task，不在每次语音段重建连接。

**6. `config.yaml`**

- `enable_websocket_ping: true`
- 删除 `ASR.AliyunBLStreamASR.caption_mode`，只保留顶层 `caption_mode`
- 在 `startToChat` 中改为从 `conn.config` 读取 caption_mode 并传给 ASR Provider

### 固件

**7. `application.cc` — `OnAudioChannelClosed` 回调**

不进入 idle，而是进入重连流程：
```
if caption_mode:
    display->SetStatus("重连中...")
    StartReconnectTimer()  // 指数退避 1s→2s→4s...
```

**8. `application.cc` — `HandleStateChangedEvent` idle 分支**

caption_mode 下跳过 `ClearChatMessages()`。

**9. `application.cc` — stt 消息处理**

用编译宏或运行时 flag 控制是显示为 "caption" 还是 "user" 角色。

**10. `lcd_display.cc` — `SetChatMessage()` caption 分支**

改为复用现有 label 更新文本，只在角色切换时才 `lv_obj_clean`：
```
if 已有 caption label 且 parent == content_ 且子对象数 == 1:
    lv_label_set_text(chat_message_label_, content)
    return
否则:
    lv_obj_clean + 创建新 label
```

**11. `lcd_display.cc` — `#else` 分支 `SetChatMessage()`**

添加 caption 角色处理，至少切换到 wrap 模式并扩展宽度。

**12. `oled_display.cc` — `SetChatMessage()`**

非 caption 角色时恢复 `content_left_` 可见性和 label 原始属性。

**13. 新增重连逻辑**

在 `application.cc` 中添加 `ReconnectTimer` 方法，实现指数退避重连。

---

*审查完毕。以上问题按优先级实施，P0 必须在部署前修复，P1 建议尽快修复，P2 和优化项可分批迭代。*
