# 小智 ESP32 实时字幕辅助设备 — 使用手册

## 目录

1. [产品概述](#1-产品概述)
2. [硬件清单](#2-硬件清单)
3. [服务端搭建教程](#3-服务端搭建教程)
4. [阿里云百炼 ASR 配置](#4-阿里云百炼-asr-配置)
5. [固件编译与烧录](#5-固件编译与烧录)
6. [设备连接与配网](#6-设备连接与配网)
7. [日常使用指南](#7-日常使用指南)
8. [配置参数详解](#8-配置参数详解)
9. [故障排查](#9-故障排查)
10. [修改文件清单与部署](#10-修改文件清单与部署)

---

## 1. 产品概述

本设备是一款面向听力障碍人士的实时语音转文字辅助工具。设备通过麦克风采集周围声音，上传至后端服务器调用阿里云百炼 Paraformer 实时语音识别模型，将识别到的文字以流式方式回传并在屏幕上以大字号字幕形式显示。

与原版小智 AI 对话设备不同，本设备不做任何 AI 对话、大语言模型回复、文字转语音播放。设备唯一的输出方式是屏幕字幕显示，专注于为听障用户提供清晰、实时的语音转文字服务。

设备核心特性包括：开机自动进入实时监听模式，无需唤醒词触发；说话时屏幕实时显示流式识别文字，字幕占据屏幕约 70% 区域居中显示；断线后自动指数退避重连，保留最后字幕并显示"重连中..."状态；长时间无声音（默认 120 秒）后自动断开连接节省资源。

---

## 2. 硬件清单

| 组件 | 说明 |
|------|------|
| 主控板 | 立创实战派 ESP32-S3（对应固件板型 `lichuang-dev`） |
| 显示屏 | 板载 SPI LCD ST7789，320×240 分辨率，旋转后有效分辨率 240×320（竖屏） |
| 麦克风 | 板载 MEMS 麦克风 |
| 电源 | USB-C 供电，5V/500mA 以上 |
| 网络 | WiFi 2.4GHz |

本设备不需要扬声器、按键模组或其他外设。如果需要外接更大屏幕，可以通过修改固件显示驱动适配其他 LCD 或 OLED 模组。

---

## 3. 服务端搭建教程

服务端基于 [xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server) 开源项目，修改为字幕模式后部署。支持 Docker 部署和手动部署两种方式。

### 3.1 环境要求

Python 3.10（必须使用 3.10 版本，其他版本可能导致依赖冲突）、至少 2GB 可用内存、稳定网络连接。

### 3.2 方式一：Docker 部署（推荐）

Docker 部署最简单，无需手动安装 Python 环境和依赖。

第一步，克隆原始仓库并应用修改文件：

```bash
git clone https://github.com/xinnan-tech/xiaozhi-esp32-server.git
cd xiaozhi-esp32-server
```

将修复版中的三个后端文件按路径覆盖：

```
config.yaml                                       → main/xiaozhi-server/config.yaml
core/handle/receiveAudioHandle.py                 → main/xiaozhi-server/core/handle/receiveAudioHandle.py
core/providers/asr/aliyunbl_stream.py             → main/xiaozhi-server/core/providers/asr/aliyunbl_stream.py
```

第二步，创建用户配置文件。在 `main/xiaozhi-server/data/` 目录下创建 `.config.yaml` 文件，只写入需要覆盖的配置项：

```yaml
server:
  # 改为你的服务器 IP 地址或域名
  websocket: ws://你的服务器IP:8000/xiaozhi/v1/
  vision_explain: http://你的服务器IP:8003/mcp/vision/explain

# 阿里云百炼 ASR 配置
ASR:
  AliyunBLStreamASR:
    api_key: 你的阿里云百炼API密钥
```

这种双层配置方式的好处是：`config.yaml` 作为模板保持不动，你的密钥等敏感信息只写在 `.config.yaml` 中，不会因为更新代码而丢失。

第三步，使用 Docker Compose 启动：

```bash
cd main
docker compose up -d
```

启动后查看日志确认服务正常运行：

```bash
docker compose logs -f
```

看到类似 `WebSocket server started on 0.0.0.0:8000` 的输出即表示服务端已就绪。

### 3.3 方式二：手动部署

适合需要调试或无法使用 Docker 的环境。

第一步，安装 Conda 并创建 Python 环境：

```bash
# 安装 Miniconda（如已安装可跳过）
# 下载地址：https://docs.conda.io/en/latest/miniconda.html

# 创建 Python 3.10 环境
conda create -n xiaozhi python=3.10 -y
conda activate xiaozhi
```

第二步，克隆仓库并安装系统依赖：

```bash
git clone https://github.com/xinnan-tech/xiaozhi-esp32-server.git
cd xiaozhi-esp32-server/main/xiaozhi-server

# 安装系统依赖（libopus 用于音频编解码，ffmpeg 用于音频处理）
conda install -c conda-forge libopus ffmpeg -y
```

第三步，安装 Python 依赖：

```bash
pip install -r requirements.txt
```

第四步，按 3.2 节的说明覆盖修改文件并创建 `.config.yaml` 配置。

第五步，启动服务：

```bash
python app.py
```

看到以下输出即表示服务端启动成功：

```
[2026-06-23 10:00:00] - [xiaozhi_server] - INFO - WebSocket server started on 0.0.0.0:8000
[2026-06-23 10:00:00] - [xiaozhi_server] - INFO - HTTP server started on 0.0.0.0:8003
```

### 3.4 防火墙与端口配置

服务端需要开放以下端口：

| 端口 | 用途 |
|------|------|
| 8000 | WebSocket 主服务端口，设备音频上传和文字回传 |
| 8003 | HTTP 服务端口，OTA 接口和设备激活 |

如果服务器有防火墙，需要放行这两个端口。以 Ubuntu 的 ufw 为例：

```bash
sudo ufw allow 8000/tcp
sudo ufw allow 8003/tcp
```

如果部署在云服务器上，还需要在云服务商的安全组中放行这两个端口。

### 3.5 验证服务端是否正常

启动后在浏览器访问 OTA 接口验证：

```
http://你的服务器IP:8003/xiaozhi/ota/
```

如果返回 JSON 数据（包含 websocket 地址等信息），说明 HTTP 服务正常。

---

## 4. 阿里云百炼 ASR 配置

本设备使用阿里云百炼平台的 Paraformer 实时语音识别服务。该服务基于 WebSocket 协议进行流式语音识别，支持中文、英文、日语等多种语言。

### 4.1 开通服务

第一步，访问阿里云百炼平台：https://bailian.console.aliyun.com/

第二步，使用阿里云账号登录，首次进入需要开通百炼服务。

第三步，创建 API Key。访问 https://bailian.console.aliyun.com/#/api-key ，点击"创建 API Key"，复制生成的密钥。

### 4.2 配置 ASR 参数

在 `data/.config.yaml` 中配置：

```yaml
ASR:
  AliyunBLStreamASR:
    api_key: sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    model: paraformer-realtime-v2
    format: pcm
    sample_rate: 16000
    # 可选：是否过滤语气词（"嗯""啊"等），字幕模式建议关闭以保留完整语义
    disfluency_removal_enabled: false
    # 可选：语义断句。false 使用 VAD 断句（低延迟，适合交互场景），true 使用语义断句（更准确，适合会议场景）
    semantic_punctuation_enabled: false
    # VAD 断句静音时长阈值（毫秒），范围 200-6000，值越小响应越快但可能断句过碎
    max_sentence_silence: 200
    # 防止 VAD 断句切割过长，仅 VAD 断句时生效
    multi_threshold_mode_enabled: false
    # 是否自动添加标点符号
    punctuation_prediction_enabled: true
    # 是否开启 ITN（中文数字转阿拉伯数字，如"一百二" → "120"）
    inverse_text_normalization_enabled: true
    # 可选：热词定制 ID，用于提高专有名词识别准确率
    # vocabulary_id: vocab-xxx-24ee19fa8cfb4d52902170a0xxxxxxxx
    # 可选：指定识别语言，不设置则自动检测
    # language_hints: ["zh", "en"]
```

### 4.3 热词定制（可选）

如果使用场景中有大量专有名词（如人名、地名、专业术语），可以通过热词定制提高识别准确率。配置方法参考阿里云文档：https://help.aliyun.com/zh/model-studio/custom-hot-words

创建热词表后会得到一个 vocabulary_id，填入配置即可。注意只有 paraformer-realtime-v2 模型支持热词功能。

### 4.4 费用说明

阿里云百炼实时语音识别服务按使用量计费，新用户通常有免费额度。具体计费标准请查看：https://help.aliyun.com/zh/model-studio/billing-for-model-studio

---

## 5. 固件编译与烧录

### 5.1 开发环境搭建

本固件基于 ESP-IDF 框架开发，需要安装以下工具：

ESP-IDF v5.4 或以上版本。安装指南：https://docs.espressif.com/projects/esp-idf/zh_CN/v5.4/esp32s3/get-started/

安装后激活 ESP-IDF 环境（每次打开新终端都需要执行）：

```bash
# Linux/macOS
. $HOME/esp/esp-idf/export.sh

# Windows (PowerShell)
.\esp\esp-idf\export.ps1
```

### 5.2 获取并修改固件源码

```bash
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32
```

将修复版中的四个固件文件按路径覆盖：

```
application.h          → main/application.h
application.cc         → main/application.cc
display/lcd_display.cc → main/display/lcd_display.cc
display/oled_display.cc → main/display/oled_display.cc
```

### 5.3 编译固件

激活 ESP-IDF 环境后，选择 lichuang-dev 板型编译：

```bash
# 设置目标芯片
idf.py set-target esp32s3

# 使用 lichuang-dev 板型配置编译
idf.py -DBOARD=lichuang-dev build
```

编译成功后会在 `build/` 目录下生成固件文件。

### 5.4 烧录固件

将立创实战派 ESP32-S3 通过 USB-C 线连接到电脑：

```bash
# 烧录并打开串口监控（替换为实际串口号）
# Windows: COM3 或 COM4 等
# Linux: /dev/ttyUSB0 或 /dev/ttyACM0
# macOS: /dev/cu.usbserial-xxxx

idf.py -p COM3 flash monitor
```

输入 `Ctrl+]` 退出监控。

### 5.5 OTA 地址配置

设备首次启动后需要配置服务器地址。有两种方式：

方式一，通过设备 WiFi 配网页面配置（推荐）。设备首次启动会进入配网模式，手机连接设备发出的 WiFi 热点后，在弹出的配网页面中填写 WiFi 密码和"高级选项"中的 OTA 地址：

```
http://你的服务器IP:8003/xiaozhi/ota/
```

方式二，通过修改固件源码中的默认 OTA 地址。在 `main/ota.h` 或板型配置文件中修改默认 OTA URL，然后重新编译烧录。

设备通过 OTA 接口获取 WebSocket 连接地址，因此 OTA 地址中的 IP 必须与设备在同一网络可达的地址。如果服务器部署在公网，使用公网 IP 或域名。

---

## 6. 设备连接与配网

### 6.1 首次配网

设备烧录固件后首次上电，屏幕显示初始化信息。设备会创建一个名为 `Xiaozhi-XXXX` 的 WiFi 热点。

用手机连接该热点，浏览器会自动弹出配网页面（或手动访问 `192.168.4.1`）。在页面中填写：

WiFi 名称（SSID）：你的 2.4GHz WiFi 名称
WiFi 密码：你的 WiFi 密码
高级选项 → OTA 地址：`http://你的服务器IP:8003/xiaozhi/ota/`

点击保存后设备会自动重启，连接 WiFi，然后激活并连接到服务器。

### 6.2 激活过程

设备连接 WiFi 后会自动进行以下步骤：

第一步，设备向服务器发送 OTA 请求，获取 WebSocket 地址和设备参数。
第二步，设备播放成功提示音，屏幕显示"连接中..."。
第三步，设备自动建立 WebSocket 连接并进入实时监听状态。
第四步，屏幕状态栏显示"监听中"，表示设备已就绪。

如果激活失败，屏幕会显示错误信息。常见原因是服务器未启动或 OTA 地址配置错误。

### 6.3 重新配网

如果需要更换 WiFi 或服务器地址，可以通过以下方式重新进入配网模式：

方法一，长按设备上的 BOOT 按键（根据板型不同可能需要按 3-5 秒）。
方法二，通过串口发送指令（需要连接电脑）。
方法三，擦除设备 Flash 重新烧录：`idf.py -p COM3 erase-flash`，然后重新烧录。

---

## 7. 日常使用指南

### 7.1 开机使用

设备上电后会自动完成 WiFi 连接、服务器激活、WebSocket 建连的流程，全程无需用户干预。当屏幕状态栏显示"监听中"时，设备开始工作，麦克风采集到的语音会被实时识别并显示在屏幕上。

### 7.2 字幕显示

字幕以流式方式更新，识别过程中文字会实时变化，最终结果显示为完整句子。字幕占据屏幕约 70% 的面积，居中显示，字号较大便于阅读。

每次新的语音开始时，上一句字幕会被新字幕替换。设备不会保存历史字幕记录，如果需要保存字幕内容，可以在服务端通过日志功能记录。

### 7.3 断线恢复

当网络断开时，设备会保留最后一句字幕在屏幕上，状态栏显示"重连中..."，并自动尝试重连。重连策略为指数退避：第一次 1 秒后重连，第二次 2 秒，第三次 4 秒，第四次 8 秒，第五次 16 秒，之后每 30 秒重试一次，最多重试 10 次。

网络恢复后设备会自动重新连接，无需手动操作。重连成功后字幕继续实时更新。

如果重连 10 次仍然失败，屏幕显示"连接失败，请检查网络"，此时需要手动重启设备或检查网络环境。

### 7.4 自动休眠

当环境中长时间没有声音（默认 120 秒），服务器会主动断开 WebSocket 连接以节省资源。设备检测到连接断开后，会自动启动指数退避重连流程：状态栏显示"重连中..."，1 秒后尝试重连。重连成功后设备恢复实时监听，字幕继续更新。

如果环境中持续没有声音，设备会周期性地断开和重连（每 120 秒一次断开，1 秒后重连）。这是正常行为，不影响设备工作。如果希望延长自动断开时间，可以在 `data/.config.yaml` 中修改：

```yaml
# 无语音输入多少秒后断开连接
close_connection_no_voice_time: 600  # 改为 600 秒（10 分钟）
```

---

## 8. 配置参数详解

以下是字幕设备的关键配置参数，在 `data/.config.yaml` 中按需覆盖。完整配置项请参考 `config.yaml` 模板文件。

### 8.1 服务器配置

```yaml
server:
  ip: 0.0.0.0              # 监听地址，0.0.0.0 表示所有网卡
  port: 8000                # WebSocket 服务端口
  http_port: 8003           # HTTP/OTA 服务端口
  websocket: ws://IP:8000/xiaozhi/v1/   # 设备连接的 WebSocket 地址
  vision_explain: http://IP:8003/mcp/vision/explain  # 视觉分析地址（字幕模式不使用）
```

### 8.2 字幕模式配置

```yaml
# 字幕设备模式：只流式输出 ASR 文本到设备，不进行 LLM 对话和 TTS 回复
caption_mode: true

# 选择的处理模块
selected_module:
  VAD: SileroVAD              # 语音活动检测
  ASR: AliyunBLStreamASR      # 语音识别（阿里云百炼实时流式）
  LLM: ChatGLMLLM             # LLM（字幕模式不使用，但需要配置避免报错）
  VLLM: ChatGLMVLLM           # 视觉语言模型（字幕模式不使用）
  TTS: EdgeTTS                # TTS（字幕模式不使用，但需要配置避免报错）
  Memory: nomem               # 记忆模块（字幕模式不使用）
  Intent: function_call       # 意图识别（字幕模式会短路跳过）
```

### 8.3 连接保活配置

```yaml
# WebSocket 心跳保活，防止 NAT/防火墙静默断开长连接
enable_websocket_ping: true

# 无语音输入多少秒后断开连接
close_connection_no_voice_time: 120

# 结束语配置（字幕模式下会直接关闭连接，不发送结束语）
end_prompt:
  enable: true
  prompt: |
    请你以"时间过得真快"未来头，用富有感情、依依不舍的话来结束这场对话吧！
```

### 8.4 ASR 识别参数

```yaml
ASR:
  AliyunBLStreamASR:
    type: aliyunbl_stream
    api_key: sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    model: paraformer-realtime-v2       # 推荐使用 v2 版本
    format: pcm                         # 音频格式
    sample_rate: 16000                  # 采样率
    disfluency_removal_enabled: false   # 过滤语气词
    semantic_punctuation_enabled: false # 语义断句（false=VAD断句，低延迟）
    max_sentence_silence: 200           # VAD 断句静音阈值（毫秒）
    multi_threshold_mode_enabled: false # 防止 VAD 断句切割过长
    punctuation_prediction_enabled: true # 自动标点
    inverse_text_normalization_enabled: true # ITN（数字转换）
```

### 8.5 VAD 配置

```yaml
VAD:
  SileroVAD:
    type: silero
    threshold: 0.5                      # 语音检测阈值，越高越严格
    threshold_low: 0.3                  # 低阈值（用于持续检测）
    model_dir: models/snakers4_silero-vad
    min_silence_duration_ms: 200        # 最小静音时长（毫秒）
```

---

## 9. 故障排查

### 9.1 设备无法连接服务器

现象：屏幕一直显示"连接中..."或"连接失败，请检查网络"。

排查步骤：

第一步，确认服务器已启动。在电脑浏览器访问 `http://你的服务器IP:8003/xiaozhi/ota/`，如果无法访问说明服务器未启动或端口未开放。

第二步，确认设备 WiFi 配置正确。重新进入配网模式检查 WiFi 密码和 OTA 地址。

第三步，确认 OTA 地址格式正确。必须是 `http://IP:8003/xiaozhi/ota/` 格式，注意结尾的斜杠。

第四步，检查服务器防火墙。确保 8000 和 8003 端口已开放。

第五步，查看服务器日志。如果设备能连接到 OTA 但无法建立 WebSocket，检查服务器日志中的错误信息。

### 9.2 字幕不显示

现象：设备显示"监听中"但说话后屏幕没有字幕。

排查步骤：

第一步，确认 ASR API Key 配置正确。在服务器日志中查找 ASR 相关错误，如 `API Key 无效` 或 `认证失败`。

第二步，确认阿里云百炼服务已开通且有余额。登录阿里云百炼控制台查看。

第三步，检查 VAD 是否正常工作。在服务器日志中查找 `VAD detected voice` 相关日志，如果没有说明 VAD 没有检测到声音，可能是麦克风问题。

第四步，检查 config.yaml 中 `selected_module.ASR` 是否设置为 `AliyunBLStreamASR`。

### 9.3 字幕延迟过高

现象：说话后字幕显示明显延迟（超过 3 秒）。

排查步骤：

第一步，检查网络延迟。设备到服务器的网络延迟应该低于 100ms。

第二步，调整 `max_sentence_silence` 参数。值越小断句越快，但可能过碎。建议范围 200-500。

第三步，确认 `semantic_punctuation_enabled` 设置为 `false`。语义断句模式会增加延迟。

第四步，检查服务器负载。如果 CPU 使用率过高，考虑使用更好的服务器或减少其他服务。

### 9.4 设备频繁断线重连

现象：设备每隔一段时间就显示"重连中..."然后又恢复。

排查步骤：

第一步，检查 WiFi 信号强度。设备距离路由器太远或隔墙会导致信号不稳定。

第二步，确认 `enable_websocket_ping` 设置为 `true`。心跳保活可以防止 NAT 超时断开。

第三步，检查服务器是否有定期重启或资源回收机制。

第四步，查看设备串口日志，确认断线原因。通过 `idf.py monitor` 连接设备查看实时日志。

### 9.5 屏幕显示异常

现象：字幕显示位置不对、字体太小、或者有残留内容。

排查步骤：

第一步，确认使用的是修复版固件文件。原始版本的 `lcd_display.cc` 在默认样式下没有 caption 角色处理。

第二步，确认编译时选择了 `lichuang-dev` 板型。其他板型可能使用不同的显示配置。

第三步，如果使用 OLED 屏幕（非 SPI LCD），检查 `oled_display.cc` 是否已覆盖为修复版。

### 9.6 服务器日志查看

Docker 部署：

```bash
cd main
docker compose logs -f
```

手动部署：直接查看终端输出，或查看日志文件：

```bash
tail -f main/xiaozhi-server/tmp/server.log
```

关键字段说明：`caption_mode: display ASR text only` 表示字幕模式已生效；`ASR服务连接已关闭` 表示一次识别会话结束；`启动重连定时器` 表示设备断线重连中。

---

## 10. 修改文件清单与部署

### 10.1 后端修改文件（3 个）

| 文件 | 修改路径 | 说明 |
|------|----------|------|
| `config.yaml` | `main/xiaozhi-server/config.yaml` | 启用 WebSocket 心跳、删除重复 caption_mode |
| `receiveAudioHandle.py` | `main/xiaozhi-server/core/handle/receiveAudioHandle.py` | caption_mode 下静音超时直接关闭连接 |
| `aliyunbl_stream.py` | `main/xiaozhi-server/core/providers/asr/aliyunbl_stream.py` | vocabulary_id 空值保护、forward_task 取消、is_final 去重修复、caption_mode 从 conn.config 读取、task-failed 设备通知 |

### 10.2 固件修改文件（4 个）

| 文件 | 修改路径 | 说明 |
|------|----------|------|
| `application.h` | `main/application.h` | 新增 caption_mode_ 标志、重连定时器成员声明 |
| `application.cc` | `main/application.cc` | 断线自动重连（指数退避）、idle 不清屏、stt 角色控制 |
| `lcd_display.cc` | `main/display/lcd_display.cc` | 默认样式添加 caption 全屏显示处理、WeChat 样式 label 复用 |
| `oled_display.cc` | `main/display/oled_display.cc` | 非 caption 角色恢复 content_left_ 可见性 |

### 10.3 部署步骤总结

后端部署：克隆原始仓库 → 覆盖 3 个后端文件 → 创建 `data/.config.yaml` 填入 API Key 和服务器地址 → Docker 或手动启动服务。

固件部署：克隆原始仓库 → 覆盖 4 个固件文件 → ESP-IDF 编译 lichuang-dev 板型 → USB 烧录 → 设备首次配网填写 WiFi 和 OTA 地址。

验证：访问 OTA 接口确认服务正常 → 设备上电激活 → 说话观察屏幕字幕显示 → 断开 WiFi 测试自动重连。
