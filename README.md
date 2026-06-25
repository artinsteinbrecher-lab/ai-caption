# 小智 ESP32 实时字幕辅助设备

面向听障人士的实时语音转文字硬件设备。基于小智 ESP32 开源项目二次开发，硬件采用立创实战派 ESP32-S3 成品开发板（自带 SPI LCD 屏、MEMS 麦克风、扬声器）。

## 工作原理

设备开机后自动进入实时监听模式，无需唤醒词触发。麦克风采集环境声音 → 通过 WebSocket 上传至后端服务器 → 后端调用阿里云百炼 Paraformer 实时语音识别模型（paraformer-realtime-v2）→ 流式识别文本回传设备 → 屏幕以全屏大字号字幕形式显示。

设备不进行 AI 对话、LLM 回复、TTS 语音播放，唯一的输出方式是屏幕字幕。

## 关键特性

- **开机即用**：上电自动连接 WiFi → 激活 → 进入实时监听，全程无需用户干预
- **全屏字幕**：字幕占据屏幕约 70% 区域，居中换行显示，字号较大便于阅读
- **流式更新**：识别过程中文字实时刷新，最终结果显示为完整句子
- **断线自动重连**：指数退避策略（1s→2s→4s→8s→16s→30s），屏幕保留最后字幕并显示"重连中..."
- **静音自动断开**：120 秒无声音后服务器主动断开，设备自动重连恢复
- **无唤醒词**：caption 模式下关闭唤醒词检测，持续监听

## 硬件

| 组件 | 说明 |
|------|------|
| 主控板 | 立创实战派 ESP32-S3（嘉立创商城成品板，自带屏幕/麦克风/扬声器） |
| 芯片 | ESP32-S3，Xtensa 双核 240MHz，512KB SRAM + 8MB PSRAM + 16MB Flash |
| 显示屏 | 板载 SPI LCD ST7789，320×240，旋转后有效分辨率 240×320（竖屏） |
| 音频 | 板载 ES8311 编解码芯片 + ES7210 + MEMS 麦克风 + 扬声器 |
| 电源 | USB-C 5V/500mA 以上 |
| 网络 | WiFi 2.4GHz |

## 文件结构

```
├── README.md                          # 本文件
├── server/                            # 后端修改文件（3个）
│   └── main/xiaozhi-server/
│       ├── config.yaml                # 服务器配置（启用心跳、caption_mode）
│       └── core/
│           ├── handle/receiveAudioHandle.py   # 静音超时连接泄漏修复
│           └── providers/asr/aliyunbl_stream.py # ASR Provider 多项修复
├── firmware/                          # 固件修改文件（4个）
│   └── main/
│       ├── application.h              # 新增 caption_mode_、重连定时器成员
│       ├── application.cc             # 自动重连、idle 不清屏、stt 角色控制
│       └── display/
│           ├── lcd_display.cc         # 默认样式添加全屏字幕显示
│           └── oled_display.cc        # 非 caption 角色恢复显示区域
├── xiaozhi-caption-review.md           # 代码审查报告
└── docs/
    └── xiaozhi-caption-device-manual.md  # 完整使用手册（含服务端搭建教程）
```

## 完整源码仓库

本仓库只包含修改过的文件。如需获取已合并修改的完整源码，请克隆以下仓库：

- 完整固件：[ai-caption-firmware](https://github.com/artinsteinbrecher-lab/ai-caption-firmware)（基于 78/xiaozhi-esp32 完整源码 + 4 个修改文件）
- 完整后端：[ai-caption-server](https://github.com/artinsteinbrecher-lab/ai-caption-server)（基于 xinnan-tech/xiaozhi-esp32-server 完整源码 + 3 个修改文件）

## 部署步骤

### 1. 后端部署

```bash
# 克隆原始后端仓库
git clone https://github.com/xinnan-tech/xiaozhi-esp32-server.git
cd xiaozhi-esp32-server

# 用本仓库的 server/ 下的文件覆盖对应文件
cp -r /path/to/xiaozhi-caption-device/server/main/xiaozhi-server/* main/xiaozhi-server/

# 创建用户配置文件（只写需要覆盖的配置项）
mkdir -p main/xiaozhi-server/data
cat > main/xiaozhi-server/data/.config.yaml << 'EOF'
server:
  websocket: ws://你的服务器IP:8000/xiaozhi/v1/
  vision_explain: http://你的服务器IP:8003/mcp/vision/explain
ASR:
  AliyunBLStreamASR:
    api_key: 你的阿里云百炼API密钥
EOF

# Docker 部署（推荐）
cd main && docker compose up -d

# 或手动部署
cd main/xiaozhi-server && pip install -r requirements.txt && python app.py
```

阿里云百炼 API Key 获取：https://bailian.console.aliyun.com/#/api-key

### 2. 固件编译

```bash
# 克隆原始固件仓库
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32

# 用本仓库的 firmware/ 下的文件覆盖对应文件
cp -r /path/to/xiaozhi-caption-device/firmware/main/* main/

# 编译（需要 ESP-IDF v5.4+）
idf.py set-target esp32s3
idf.py -DBOARD=lichuang-dev build

# 烧录
idf.py -p COM3 flash monitor
```

### 3. 设备配网

设备首次上电会创建 WiFi 热点，手机连接后在配网页面填写：
- WiFi 名称和密码
- 高级选项 → OTA 地址：`http://你的服务器IP:8003/xiaozhi/ota/`

设备自动激活后屏幕显示"监听中"即表示就绪。

## 修复清单

### P0 级修复（影响设备可用性）

| 编号 | 问题 | 文件 | 修复 |
|------|------|------|------|
| P0-1 | 静音超时后连接泄漏，结束语被当作字幕显示 | receiveAudioHandle.py | caption_mode 下直接关闭连接 |
| P0-2 | 断线后设备不自动重连，清空所有历史字幕 | application.h/cc | 指数退避自动重连，不清屏 |
| P0-3 | 每次 ASR 中间结果都 lv_obj_clean 重建 LVGL 对象 | lcd_display.cc | 检测已有 label 直接更新文本 |

### P1 级修复（影响稳定性和显示）

| 编号 | 问题 | 文件 | 修复 |
|------|------|------|------|
| P1-1 | vocabulary_id 为 null 时仍发送到阿里云 API | aliyunbl_stream.py | 增加空值判断 |
| P1-2 | **默认样式无 caption 处理（最关键）** | lcd_display.cc | 在 #else 分支添加全屏字幕显示 |
| P1-3 | OLED 隐藏 content_left_ 后永不恢复 | oled_display.cc | 添加 else 分支恢复 |
| P1-4 | _cleanup 不取消 forward_task 造成竞态 | aliyunbl_stream.py | 先 cancel 再关闭 WebSocket |
| P1-6 | 最终结果被去重逻辑丢弃 | aliyunbl_stream.py | is_final 总发送不去重 |
| P1-7 | caption_mode 在 config.yaml 重复定义 | config.yaml + aliyunbl_stream.py | 删除重复，从 conn.config 读取 |

### P2 级修复

| 编号 | 问题 | 文件 | 修复 |
|------|------|------|------|
| P2-1 | stt 消息硬编码为 caption 角色 | application.cc | 用 caption_mode_ 控制 |
| P2-3 | WebSocket 心跳未启用 | config.yaml | enable_websocket_ping: true |

## 技术栈

- **固件**：ESP-IDF v5.4+，C++，LVGL v9，Opus 编解码，WebSocket 协议
- **后端**：Python 3.10，asyncio，websockets，SileroVAD
- **ASR**：阿里云百炼 Paraformer Realtime v2（WebSocket 流式语音识别）
- **硬件**：ESP32-S3，ST7789 SPI LCD 320×240，ES8311 音频编解码

## 原始项目

本项目基于以下开源项目二次开发：

- 固件：[78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)
- 后端：[xinnan-tech/xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server)

## 许可证

遵循原始项目许可证。

## 完整文档

- [使用手册](docs/xiaozhi-caption-device-manual.md) — 搭建教程、配置参数、故障排查
- [Windows Server 部署指南](docs/WINDOWS-DEPLOYMENT-GUIDE.md) — 腾讯云 Windows Server 部署全流程（含 opus.dll、FFmpeg、安全组等踩坑记录）
- [代码审查报告](xiaozhi-caption-review.md) — 完整的问题分析与修复方案
