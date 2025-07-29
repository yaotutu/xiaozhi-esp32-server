# 小智语音助手服务端API接口文档

## 目录
1. [接口概述](#接口概述)
2. [WebSocket协议](#websocket协议)
3. [HTTP API](#http-api)
4. [错误处理](#错误处理)
5. [认证机制](#认证机制)
6. [客户端示例](#客户端示例)

---

## 接口概述

### 服务端口
- **WebSocket服务**: 默认端口 8000
- **HTTP API服务**: 默认端口 8003

### 数据格式
- **WebSocket**: JSON文本消息 + 二进制音频数据
- **HTTP**: JSON格式请求/响应 + multipart/form-data文件上传

---

## WebSocket协议

### 连接建立

#### 连接URL
```
ws://服务器IP:8000/xiaozhi/v1/
```

#### 认证方式
**请求头认证 (推荐)**
```javascript
const headers = {
    "device-id": "AA:BB:CC:DD:EE:FF",      // 必填，设备唯一标识
    "client-id": "客户端ID",                // 可选，客户端标识
    "Authorization": "Bearer your-token"   // 可选，认证token
};
```

**查询参数认证**
```
ws://localhost:8000/xiaozhi/v1/?device-id=AA:BB:CC:DD:EE:FF&client-id=client123
```

### 消息协议

#### 客户端消息类型

| 类型 | 描述 | 必填字段 |
|------|------|----------|
| `hello` | 握手消息 | type |
| `listen` | 音频监听控制 | type, state |
| `abort` | 中止当前操作 | type |
| `iot` | IoT设备消息 | type |
| `mcp` | MCP协议消息 | type |
| `server` | 服务端管理 | type, action |

#### 服务端消息类型

| 类型 | 描述 | 必填字段 |
|------|------|----------|
| `hello` | 握手响应 | type, session_id |
| `tts` | 语音合成状态 | type, state, session_id |
| `stt` | 语音识别结果 | type, text, session_id |
| `llm` | 大模型情感响应 | type, session_id |
| `server` | 服务端操作响应 | type, status |

### 详细消息格式

#### hello消息 (握手协议)
**客户端发送:**
```json
{
    "type": "hello",
    "version": 1,
    "transport": "websocket",
    "audio_params": {
        "format": "opus",
        "sample_rate": 16000,
        "channels": 1,
        "frame_duration": 60
    }
}
```

**服务端响应:**
```json
{
    "type": "hello",
    "session_id": "550e8400-e29b-41d4-a716-446655440000",
    "version": 1,
    "transport": "websocket",
    "audio_params": {
        "format": "opus",
        "sample_rate": 16000,
        "channels": 1,
        "frame_duration": 60
    }
}
```

#### listen消息 (音频控制)
**客户端发送:**
```json
{
    "type": "listen",
    "state": "start",           // 枚举: start, stop, detect
    "mode": "auto",             // 可选，枚举: auto, manual
    "session_id": "uuid-string", // 可选
    "text": "用户输入的文本"       // 可选，仅当state为detect时使用
}
```

#### abort消息 (中止操作)
```json
{
    "type": "abort",
    "session_id": "uuid-string",     // 可选
    "reason": "user_interrupt"       // 可选，枚举: user_interrupt, wake_word_detected, timeout
}
```

#### iot消息 (IoT设备控制)
**设备能力声明:**
```json
{
    "type": "iot",
    "descriptors": [
        {
            "name": "living_room_light",
            "description": "客厅智能灯",
            "properties": {
                "power": {"type": "boolean", "description": "电源状态"},
                "brightness": {"type": "number", "min": 0, "max": 100}
            },
            "methods": {
                "turn_on": {"description": "打开灯光"},
                "turn_off": {"description": "关闭灯光"}
            }
        }
    ]
}
```

**设备状态上报:**
```json
{
    "type": "iot",
    "states": [
        {
            "name": "living_room_light",
            "state": {"power": true, "brightness": 80},
            "timestamp": 1699123456789
        }
    ]
}
```

**控制命令 (服务端→客户端):**
```json
{
    "type": "iot",
    "commands": [
        {
            "name": "living_room_light",
            "method": "turn_on",
            "parameters": {"brightness": 80},
            "command_id": "cmd_123456789"
        }
    ]
}
```

#### mcp消息 (MCP协议)
```json
{
    "type": "mcp",
    "payload": {
        "method": "tool_call",
        "params": {
            "tool_name": "weather",
            "arguments": {"city": "北京"}
        }
    }
}
```

#### server消息 (服务端管理)
**客户端发送:**
```json
{
    "type": "server",
    "action": "update_config",      // 枚举: update_config, restart
    "content": {
        "secret": "your-secret-key"
    }
}
```

### 服务端响应消息

#### TTS状态消息
```json
{
    "type": "tts",
    "state": "sentence_start",      // 枚举: start, sentence_start, sentence_end, stop
    "text": "今天北京天气晴朗",
    "session_id": "uuid-string"
}
```

#### STT识别消息
```json
{
    "type": "stt",
    "text": "今天天气怎么样",
    "confidence": 0.95,             // 可选，置信度0-1
    "is_final": true,               // 可选，是否为最终结果
    "session_id": "uuid-string"
}
```

#### LLM情感消息
```json
{
    "type": "llm",
    "text": "😊",                        // 表情符号
    "emotion": "happy",             // 情感类型
    "session_id": "uuid-string"
}
```

**emotion字段枚举值:**
`neutral`, `happy`, `laughing`, `funny`, `sad`, `angry`, `crying`, `loving`, `embarrassed`, `surprised`, `shocked`, `thinking`, `winking`, `cool`, `relaxed`, `delicious`, `kissy`, `confident`, `sleepy`, `silly`, `confused`

### 音频数据传输

#### 音频格式规范
| 参数 | 值 | 描述 |
|------|-----|------|
| 编码格式 | Opus | 必须使用Opus编码 |
| 采样率 | 16000 Hz | 固定采样率 |
| 声道数 | 1 | 单声道 |
| 帧时长 | 60ms | 每个音频帧的时长 |

#### 音频数据发送
```javascript
// 客户端发送音频数据
const opusAudioData = new Uint8Array([...]); // Opus音频帧
ws.send(opusAudioData);

// 接收服务端音频数据
ws.onmessage = function(event) {
    if (event.data instanceof Blob || event.data instanceof ArrayBuffer) {
        handleAudioData(event.data);  // 处理音频数据
    } else {
        const message = JSON.parse(event.data);  // 处理文本消息
    }
};
```

---

## HTTP API

### 1. OTA更新接口

#### 状态检查
```http
GET /xiaozhi/ota/
```
**响应:** `OTA接口运行正常，向设备发送的websocket地址是：ws://192.168.1.100:8000/xiaozhi/v1/`

#### 获取OTA信息
```http
POST /xiaozhi/ota/
Content-Type: application/json
device-id: AA:BB:CC:DD:EE:FF

{
    "application": {
        "version": "1.0.0",
        "build": "20240101"
    }
}
```

**成功响应 (200):**
```json
{
    "server_time": {
        "timestamp": 1699123456789,
        "timezone_offset": 480
    },
    "firmware": {
        "version": "1.0.0",
        "url": "",
        "size": 0,
        "checksum": ""
    },
    "websocket": {
        "url": "ws://192.168.1.100:8000/xiaozhi/v1/"
    }
}
```

### 2. 视觉分析接口

#### 状态检查
```http
GET /mcp/vision/explain
```

#### 图像分析
```http
POST /mcp/vision/explain
Content-Type: multipart/form-data
Authorization: Bearer your-jwt-token
Device-Id: AA:BB:CC:DD:EE:FF

# Form参数
question: 这张图片里有什么内容？
image: [图片文件]
```

**成功响应 (200):**
```json
{
    "success": true,
    "action": "RESPONSE",
    "response": "这是一张美丽的风景照片，画面中有蓝天白云、绿树成荫的山峦..."
}
```

**支持的图片格式:** JPEG, PNG, GIF, BMP, TIFF, WebP (最大5MB)

---

## 错误处理

### WebSocket错误

#### 连接级错误
```json
{
    "type": "server",
    "status": "error",
    "message": "设备认证失败",
    "error_code": "AUTH_FAILED"
}
```

#### 消息级错误
```json
{
    "type": "server",
    "status": "error",
    "message": "消息格式不正确",
    "error_code": "INVALID_JSON"
}
```

### HTTP错误

#### 错误响应格式
```json
{
    "success": false,
    "error_code": "INVALID_REQUEST",
    "message": "请求参数无效"
}
```

### 常见错误代码

| 错误代码 | HTTP状态码 | 描述 | 解决方案 |
|----------|------------|------|----------|
| `AUTH_FAILED` | 401 | 认证失败 | 检查Token是否有效 |
| `MISSING_DEVICE_ID` | 400 | 缺少设备ID | 添加device-id请求头 |
| `INVALID_JSON` | 400 | JSON格式错误 | 检查消息格式 |
| `DEVICE_OFFLINE` | 503 | 设备离线 | 检查设备连接 |
| `FILE_TOO_LARGE` | 413 | 文件过大 | 压缩到5MB以下 |

---

## 认证机制

### WebSocket认证

#### 设备ID认证 (基础)
```javascript
const headers = {
    "device-id": "AA:BB:CC:DD:EE:FF",  // 必填，设备MAC地址
    "client-id": "client123"           // 可选，客户端标识
};
```

#### Token认证 (增强)
```javascript
const headers = {
    "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "device-id": "AA:BB:CC:DD:EE:FF"
};
```

### HTTP认证

#### JWT Token格式
```http
Authorization: Bearer <JWT_TOKEN>
```

#### Token生成示例 (Python)
```python
import jwt
import time

def generate_token(device_id, secret_key, expires_in=3600):
    payload = {
        'device_id': device_id,
        'iat': int(time.time()),
        'exp': int(time.time()) + expires_in
    }
    return jwt.encode(payload, secret_key, algorithm='HS256')
```

---

## 客户端示例

### JavaScript基础示例
```html
<!DOCTYPE html>
<html>
<head>
    <title>小智语音助手客户端</title>
    <meta charset="UTF-8">
</head>
<body>
    <div id="status">状态: 未连接</div>
    <input type="text" id="messageInput" placeholder="输入消息...">
    <button onclick="sendTextMessage()">发送</button>
    <button onclick="toggleVoice()">语音</button>
    <div id="messages"></div>

    <script>
        let ws = null;
        let sessionId = null;
        let isConnected = false;

        // 建立连接
        function connect() {
            ws = new WebSocket("ws://localhost:8000/xiaozhi/v1/");
            
            ws.onopen = function() {
                document.getElementById('status').textContent = "状态: 已连接，正在握手...";
                sendHello();
            };
            
            ws.onmessage = function(event) {
                if (typeof event.data === 'string') {
                    const message = JSON.parse(event.data);
                    handleTextMessage(message);
                } else {
                    handleAudioMessage(event.data);
                }
            };
            
            ws.onclose = function() {
                isConnected = false;
                document.getElementById('status').textContent = "状态: 连接已关闭";
            };
        }

        // 发送Hello握手
        function sendHello() {
            const hello = {
                type: "hello",
                version: 1,
                transport: "websocket",
                audio_params: {
                    format: "opus",
                    sample_rate: 16000,
                    channels: 1,
                    frame_duration: 60
                }
            };
            ws.send(JSON.stringify(hello));
        }

        // 处理文本消息
        function handleTextMessage(message) {
            if (message.type === "hello") {
                sessionId = message.session_id;
                isConnected = true;
                document.getElementById('status').textContent = `状态: 已连接 (${sessionId.substr(0, 8)}...)`;
            } else if (message.type === "tts" && message.state === "sentence_start") {
                addMessage("小智", message.text || "");
            } else if (message.type === "stt") {
                addMessage("识别", message.text);
            }
        }

        // 处理音频数据
        function handleAudioMessage(audioData) {
            console.log(`收到音频数据: ${audioData.byteLength} 字节`);
        }

        // 发送文本消息
        function sendTextMessage() {
            const input = document.getElementById('messageInput');
            const text = input.value.trim();
            if (text && isConnected) {
                const message = {
                    type: "listen",
                    state: "detect",
                    text: text
                };
                ws.send(JSON.stringify(message));
                addMessage("用户", text);
                input.value = '';
            }
        }

        // 添加消息到显示区域
        function addMessage(sender, content) {
            const messages = document.getElementById('messages');
            const div = document.createElement('div');
            const timestamp = new Date().toLocaleTimeString();
            div.innerHTML = `<strong>[${timestamp}] ${sender}:</strong> ${content}`;
            messages.appendChild(div);
            messages.scrollTop = messages.scrollHeight;
        }

        // 页面加载时自动连接
        window.onload = function() {
            connect();
        };

        // 回车发送消息
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendTextMessage();
            }
        });
    </script>
</body>
</html>
```

### Python基础示例
```python
import asyncio
import websockets
import json
import logging

class XiaozhiClient:
    def __init__(self, ws_url, device_id):
        self.ws_url = ws_url
        self.device_id = device_id
        self.session_id = None
        self.ws = None
        self.is_connected = False

    async def connect(self):
        headers = {
            "device-id": self.device_id,
            "client-id": f"python-client-{self.device_id}"
        }
        
        self.ws = await websockets.connect(self.ws_url, extra_headers=headers)
        await self.send_hello()
        await self.message_loop()

    async def send_hello(self):
        hello = {
            "type": "hello",
            "version": 1,
            "transport": "websocket",
            "audio_params": {
                "format": "opus",
                "sample_rate": 16000,
                "channels": 1,
                "frame_duration": 60
            }
        }
        await self.ws.send(json.dumps(hello))

    async def send_text(self, text):
        message = {
            "type": "listen",
            "state": "detect",
            "text": text
        }
        await self.ws.send(json.dumps(message))

    async def message_loop(self):
        async for message in self.ws:
            if isinstance(message, str):
                data = json.loads(message)
                await self.handle_message(data)
            else:
                print(f"收到音频数据: {len(message)} 字节")

    async def handle_message(self, message):
        msg_type = message.get("type", "")
        
        if msg_type == "hello":
            self.session_id = message.get("session_id")
            self.is_connected = True
            print(f"连接成功，会话ID: {self.session_id}")
        elif msg_type == "tts" and message.get("state") == "sentence_start":
            print(f"小智回复: {message.get('text', '')}")
        elif msg_type == "stt":
            print(f"语音识别: {message.get('text', '')}")

# 使用示例
async def main():
    client = XiaozhiClient(
        ws_url="ws://localhost:8000/xiaozhi/v1/",
        device_id="AA:BB:CC:DD:EE:FF"
    )
    
    try:
        await client.connect()
    except Exception as e:
        print(f"连接失败: {e}")

if __name__ == "__main__":
    asyncio.run(main())
```

### IoT设备示例
```python
# IoT设备完整示例
async def iot_device_example():
    client = XiaozhiClient("ws://localhost:8000/xiaozhi/v1/", "AA:BB:CC:DD:EE:FF")
    
    # 设备描述符
    descriptors = [
        {
            "name": "living_room_light",
            "description": "客厅智能灯",
            "properties": {
                "power": {"type": "boolean", "description": "电源状态"},
                "brightness": {"type": "number", "min": 0, "max": 100}
            },
            "methods": {
                "turn_on": {"description": "打开灯光"},
                "turn_off": {"description": "关闭灯光"},
                "set_brightness": {"description": "设置亮度"}
            }
        }
    ]
    
    await client.connect()
    
    # 发送设备能力声明
    await client.ws.send(json.dumps({
        "type": "iot",
        "descriptors": descriptors
    }))
    
    # 发送初始状态
    await client.ws.send(json.dumps({
        "type": "iot",
        "states": [
            {
                "name": "living_room_light",
                "state": {"power": False, "brightness": 0}
            }
        ]
    }))
    
    print("IoT设备已注册，等待语音控制...")
    await client.message_loop()
```

---

## 总结

本API文档提供了小智语音助手服务端的核心接口规范，包括：

- **WebSocket协议**: 6种客户端消息类型，5种服务端消息类型
- **IoT设备控制**: 完整的设备能力声明和控制机制
- **MCP协议支持**: 标准MCP工具调用协议
- **HTTP API**: OTA更新和视觉分析接口
- **认证机制**: 多种认证方式和安全保障
- **客户端示例**: JavaScript、Python和IoT设备完整示例

该文档为前端开发和IoT设备开发提供了简洁明确的API规范，支持快速集成和开发。