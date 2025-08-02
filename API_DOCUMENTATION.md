# API接口文档

## WebSocket API

### 连接信息
- **URL**: `ws://localhost:8848`
- **协议**: WebSocket
- **编码**: UTF-8 + Binary (音频数据)

## 消息类型定义

### 1. 控制消息

#### Ping/Pong (心跳检测)
```json
// 客户端发送
{
  "type": "ping",
  "timestamp": 1672531200000
}

// 服务端响应
{
  "type": "pong", 
  "timestamp": 1672531200000
}
```

#### 录音控制
```json
// 开始录音
{
  "type": "start_recording",
  "timestamp": 1672531200000
}

// 停止录音
{
  "type": "stop_recording",
  "timestamp": 1672531200000
}
```

### 2. 音频数据消息

#### 音频块传输
```javascript
// 二进制消息 (ArrayBuffer)
// 音频格式: PCM 16-bit, 16kHz, 单声道
// 块大小: 4096 samples
```

#### 音频结束标记
```json
{
  "type": "audio_end",
  "timestamp": 1672531200000
}
```

### 3. AI响应消息

#### 语音转录结果
```json
{
  "type": "transcript",
  "content": "请解释一下JavaScript中的闭包概念",
  "isFinal": true,
  "timestamp": 1672531200000,
  "confidence": 0.95
}
```

#### 思考引擎输出
```json
{
  "type": "thinking",
  "entries": [
    {
      "id": "keyword_javascript",
      "timestamp": 1672531200000,
      "type": "keyword",
      "content": "🔍 JavaScript",
      "keywords": ["javascript"],
      "confidence": 0.9
    },
    {
      "id": "point_1", 
      "timestamp": 1672531200001,
      "type": "point",
      "content": "💡 解释闭包的定义和特性",
      "keywords": [],
      "confidence": 0.8
    }
  ]
}
```

#### AI对话响应
```json
{
  "type": "ai_response",
  "content": "闭包是JavaScript中的一个重要概念...",
  "messageId": "msg_123456",
  "timestamp": 1672531200000,
  "isComplete": true
}
```

#### TTS音频响应
```javascript
// 二进制消息 (ArrayBuffer)
// 音频格式: MP3 或 WAV
// 采样率: 22050Hz
// 声道: 单声道
```

### 4. 系统消息

#### 错误信息
```json
{
  "type": "error",
  "code": "ASR_FAILED",
  "message": "语音识别服务暂时不可用",
  "timestamp": 1672531200000,
  "details": {
    "provider": "openai",
    "originalError": "Request timeout"
  }
}
```

#### 状态更新
```json
{
  "type": "status",
  "status": "processing",
  "message": "正在处理语音...",
  "timestamp": 1672531200000
}
```

#### 延迟监控
```json
{
  "type": "latency",
  "metrics": {
    "asr_latency": 150,
    "llm_latency": 800,
    "tts_latency": 300,
    "total_latency": 1250
  },
  "timestamp": 1672531200000
}
```

## 数据结构定义

### ThinkingEntry
```typescript
interface ThinkingEntry {
  id: string;                    // 唯一标识符
  timestamp: number;             // Unix时间戳
  type: 'keyword' | 'concept' | 'thought' | 'point' | 'basic';
  content: string;               // 显示内容
  keywords?: string[];           // 相关关键词
  confidence?: number;           // 置信度 (0-1)
}
```

### ChatMessage
```typescript
interface ChatMessage {
  role: 'user' | 'assistant' | 'transcript';
  content: string;
  timestamp?: number;
  messageId?: string;
  isFinal?: boolean;
}
```

### LogEntry
```typescript
interface LogEntry {
  timestamp: number;
  message: string;
  type: 'info' | 'error' | 'latency' | 'llm';
  level?: 'debug' | 'info' | 'warn' | 'error';
}
```

## 错误码定义

| 错误码 | 描述 | 处理建议 |
|--------|------|----------|
| `CONNECTION_FAILED` | WebSocket连接失败 | 检查网络连接 |
| `ASR_FAILED` | 语音识别失败 | 重试或切换提供商 |
| `LLM_FAILED` | 大语言模型调用失败 | 检查API密钥和配额 |
| `TTS_FAILED` | 语音合成失败 | 重试或降级到文本 |
| `VAD_FAILED` | 语音活动检测失败 | 检查音频输入 |
| `RATE_LIMITED` | API调用频率限制 | 等待后重试 |
| `INVALID_AUDIO` | 音频格式不支持 | 检查音频参数 |
| `CONFIG_ERROR` | 配置错误 | 检查配置文件 |

## 配置参数

### 音频参数
```javascript
{
  AUDIO_SAMPLE_RATE: 16000,        // 采样率
  AUDIO_CHUNK_SIZE: 4096,          // 音频块大小
  VAD_THRESHOLD: 0.5,              // VAD阈值
  VAD_MIN_SPEECH_DURATION: 250,    // 最小语音时长(ms)
  VAD_MAX_SILENCE_DURATION: 500    // 最大静音时长(ms)
}
```

### 服务器参数
```javascript
{
  LISTEN_PORT: 8848,               // 监听端口
  LISTEN_HOST: '0.0.0.0',         // 监听地址
  CANCEL_PLAYBACK_TIME_THRESHOLD: 3000  // 播放取消阈值(ms)
}
```

## 使用示例

### JavaScript客户端示例
```javascript
const ws = new WebSocket('ws://localhost:8848');

// 连接建立
ws.onopen = () => {
  console.log('WebSocket连接已建立');
  
  // 发送心跳
  setInterval(() => {
    ws.send(JSON.stringify({
      type: 'ping',
      timestamp: Date.now()
    }));
  }, 30000);
};

// 接收消息
ws.onmessage = (event) => {
  if (event.data instanceof ArrayBuffer) {
    // 处理音频数据
    handleAudioData(event.data);
  } else {
    // 处理JSON消息
    const message = JSON.parse(event.data);
    handleMessage(message);
  }
};

// 发送音频数据
function sendAudioChunk(audioBuffer) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(audioBuffer);
  }
}

// 处理不同类型的消息
function handleMessage(message) {
  switch(message.type) {
    case 'transcript':
      updateTranscript(message.content);
      break;
    case 'thinking':
      updateThinking(message.entries);
      break;
    case 'ai_response':
      displayAIResponse(message.content);
      break;
    case 'error':
      handleError(message);
      break;
  }
}
```

## 性能指标

### 延迟要求
- **音频传输延迟**: < 50ms
- **语音识别延迟**: < 500ms  
- **思考引擎延迟**: < 200ms
- **LLM响应延迟**: < 2000ms
- **TTS合成延迟**: < 800ms
- **端到端延迟**: < 3000ms

### 吞吐量指标
- **并发连接数**: 100+
- **音频处理速率**: 实时 (1x)
- **消息处理速率**: 1000+ msg/s

---

*API文档版本: v1.0*  
*最后更新: 2025-08-02*
