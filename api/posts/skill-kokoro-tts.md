## 概述

Kokoro TTS 是一个高质量的开源 AI 语音合成引擎，支持本地部署，提供与 OpenAI TTS API 兼容的接口。它能够将文本转换为自然的语音，适用于聊天机器人语音消息、有声内容生成、无障碍辅助等场景。

本文以 derder-bot 项目中的 `kokoro-tts` skill 为例，介绍完整的集成方案。

### 核心特性

- **高质量语音**：基于 Kokoro 模型，生成自然流畅的人声
- **本地部署**：数据不出本地，隐私安全有保障
- **OpenAI 兼容 API**：`/v1/audio/speech` 接口，迁移成本低
- **多音色支持**：内置美式/英式男女声共 20+ 种音色
- **参数可调**：支持语速调节（0.25x ~ 4.0x）

## 工作原理

```
用户文本 → Node.js 脚本 → HTTP POST → Kokoro TTS API → MP3 音频 → 机器人发送语音消息
```

### 架构示意

```
┌─────────────┐     HTTP POST      ┌──────────────────┐     MP3      ┌──────────┐
│  tts.js     │  ──────────────→   │  Kokoro TTS API  │  ────────→   │  media/  │
│  (客户端)   │  JSON body         │  (localhost:8880) │              │  *.mp3   │
└─────────────┘                    └──────────────────┘              └──────────┘
```

derder-bot（基于 OpenClaw）通过读取脚本输出的 `MEDIA:` 行，自动将生成的 MP3 作为语音附件发送给用户。

## 环境配置

### 环境变量

Kokoro TTS skill 通过 `KOKORO_API_URL` 环境变量定位 API 服务：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `KOKORO_API_URL` | `http://localhost:8880/v1/audio/speech` | Kokoro TTS API 地址 |

### 配置方式

在项目根目录的 `.env` 文件中添加：

```env
KOKORO_API_URL=http://your-server:port/v1/audio/speech
```

若使用本地部署的 Kokoro 实例，保持默认值即可，无需额外配置。

## 使用方法

### 命令格式

```bash
node skills/kokoro-tts/scripts/tts.js "<text>" [voice] [speed]
```

### 参数说明

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `text` | 是 | — | 要朗读的文本，需用引号包裹 |
| `voice` | 否 | `af_heart` | 音色 ID |
| `speed` | 否 | `1.0` | 语速，范围 0.25 ~ 4.0 |

### 使用示例

默认音色朗读：

```bash
node skills/kokoro-tts/scripts/tts.js "Hello, this is a test message."
```

指定音色（美式女声 nova）：

```bash
node skills/kokoro-tts/scripts/tts.js "Hello Ed, this is Theosaurus speaking." af_nova
```

调整语速（0.5 倍速慢读）：

```bash
node skills/kokoro-tts/scripts/tts.js "慢速朗读示例" af_heart 0.5
```

### 输出格式

脚本执行成功后，输出一行 `MEDIA:` 前缀的文件路径：

```
MEDIA: media/tts_1706745000000.mp3
```

OpenClaw / derder-bot 会自动识别该格式，并将文件作为语音附件发送。

## 脚本源码解析

以下是 `scripts/tts.js` 的完整实现：

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');

// 配置：优先从环境变量读取 API 地址
const API_URL = process.env.KOKORO_API_URL || 'http://localhost:8880/v1/audio/speech';
const DEFAULT_VOICE = 'af_heart';
const DEFAULT_SPEED = 1.0;

// 解析命令行参数
const args = process.argv.slice(2);
if (args.length === 0) {
  console.error('Usage: node tts.js <text> [voice] [speed]');
  process.exit(1);
}

const text = args[0];
const voice = args[1] || DEFAULT_VOICE;
const speed = parseFloat(args[2] || DEFAULT_SPEED);

// 确保 media 目录存在
const mediaDir = path.join(process.cwd(), 'media');
if (!fs.existsSync(mediaDir)) {
  fs.mkdirSync(mediaDir, { recursive: true });
}

// 生成唯一文件名
const timestamp = Date.now();
const filename = `tts_${timestamp}.mp3`;
const filePath = path.join(mediaDir, filename);

async function generateSpeech() {
  try {
    // 调用 Kokoro TTS API（OpenAI 兼容格式）
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        input: text,           // 要朗读的文本
        voice: voice,          // 音色 ID
        speed: speed,          // 语速
        response_format: 'mp3', // 输出格式
        model: 'kokoro',       // 模型名称
        stream: false           // 非流式返回
      }),
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.status} ${response.statusText}`);
    }

    // 将音频写入文件
    const buffer = await response.arrayBuffer();
    fs.writeFileSync(filePath, Buffer.from(buffer));

    // 输出 OpenClaw 可识别的格式
    console.log(`MEDIA: ${path.relative(process.cwd(), filePath)}`);

  } catch (error) {
    console.error('Error generating speech:', error.message);
    process.exit(1);
  }
}

generateSpeech();
```

### 关键设计点

1. **环境变量优先**：`KOKORO_API_URL` 未设置时回退到 `localhost:8880`，本地开发零配置
2. **自动创建目录**：`media/` 目录不存在时自动递归创建
3. **时间戳命名**：用 `Date.now()` 生成唯一文件名，避免覆盖
4. **MEDIA 协议**：输出格式严格遵循 OpenClaw 的 `MEDIA:` 协议，实现机器人自动发送
5. **错误处理**：API 异常时输出错误信息并 `exit(1)`，便于上层捕获

## API 接口说明

Kokoro TTS 提供 OpenAI 兼容的 `/v1/audio/speech` 接口。

### 请求

```
POST /v1/audio/speech
Content-Type: application/json
```

请求体：

```json
{
  "input": "要朗读的文本",
  "voice": "af_heart",
  "speed": 1.0,
  "response_format": "mp3",
  "model": "kokoro",
  "stream": false
}
```

### 响应

返回二进制 MP3 音频流，`Content-Type: audio/mpeg`。

### curl 调用示例

```bash
curl -X POST http://localhost:8880/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"Hello world","voice":"af_heart","speed":1.0}' \
  --output output.mp3
```

## 可用音色

### 美式英语 · 女声（American Female）

| 音色 ID | 风格 |
|---------|------|
| `af_heart` | 默认，温暖亲切 |
| `af_alloy` | 通用 |
| `af_bella` | 活泼 |
| `af_jessica` | 清晰 |
| `af_kore` | 柔和 |
| `af_nicole` | 沉稳 |
| `af_nova` | 专业 |
| `af_river` | 自然 |
| `af_sarah` | 优雅 |
| `af_sky` | 明亮 |

### 美式英语 · 男声（American Male）

| 音色 ID | 风格 |
|---------|------|
| `am_adam` | 浑厚 |
| `am_echo` | 回响 |
| `am_eric` | 标准 |
| `am_fenrir` | 低沉 |
| `am_liam` | 年轻 |
| `am_michael` | 成熟 |
| `am_onyx` | 深沉 |
| `am_puck` | 活泼 |
| `am_santa` | 欢乐 |

### 英式英语 · 女声（British Female）

| 音色 ID | 风格 |
|---------|------|
| `bf_alice` | 优雅 |
| `bf_emma` | 清新 |
| `bf_lily` | 甜美 |

### 英式英语 · 男声（British Male）

| 音色 ID | 风格 |
|---------|------|
| `bm_daniel` | 稳重 |
| `bm_fable` | 叙事 |
| `bm_george` | 厚重 |
| `bm_lewis` | 轻快 |

### 推荐音色

| 场景 | 推荐音色 | 理由 |
|------|---------|------|
| 默认使用 | `af_heart` | 温暖亲切，适用性广 |
| 专业播报 | `af_nova` | 清晰专业 |
| 浑厚男声 | `am_adam` | 低沉有力 |
| 英式口音 | `bf_alice` | 优雅英式女声 |

## 实际应用场景

### 1. 聊天机器人语音消息

在 derder-bot 中，当用户请求"说"某段文字时，skill 自动调用并返回语音：

```
用户：说一下"你好，欢迎使用"
Bot：[语音消息] 你好，欢迎使用
```

### 2. 有声内容生成

批量将文章转为音频，配合 RSS 日报生成语音播报：

```bash
node skills/kokoro-tts/scripts/tts.js "$(cat article.md)" af_nova
```

### 3. 无障碍辅助

为视障用户朗读屏幕内容，或为阅读困难的用户提供语音版本。

### 4. 多语言内容

配合翻译 API，实现"翻译 + 朗读"一体化工作流。

## 部署 Kokoro TTS 服务

### Docker 部署（推荐）

```bash
docker run -d \
  --name kokoro-tts \
  -p 8880:8880 \
  --gpus all \
  kokoro-tts:latest
```

### 验证服务

```bash
curl http://localhost:8880/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"test","voice":"af_heart"}' \
  --output test.mp3
```

若返回有效的 MP3 文件，说明服务正常运行。

## 总结

Kokoro TTS 提供了轻量、高质量的本地语音合成方案。通过 OpenAI 兼容 API 和简洁的 Node.js 脚本，可以快速集成到各类聊天机器人项目中。核心优势在于：

- 零云端依赖，数据完全本地化
- OpenAI API 兼容，迁移成本低
- 20+ 音色覆盖主流场景
- 与 OpenClaw `MEDIA:` 协议无缝对接

## 参考

- [Kokoro TTS GitHub](https://github.com/kokoro-engage/kokoro-tts)
- [OpenAI TTS API 文档](https://platform.openai.com/docs/guides/text-to-speech)
- derder-bot 项目中的 `skills/kokoro-tts/` 目录
