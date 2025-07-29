# 小智语音助手服务端API接口文档

## 目录
1. [接口概述](#接口概述)
2. [WebSocket协议详解](#websocket协议详解)
3. [HTTP API接口](#http-api接口)
4. [错误处理](#错误处理)
5. [认证机制](#认证机制)
6. [客户端示例](#客户端示例)

---

## 接口概述

小智语音助手服务端提供两套API接口：
- **WebSocket API**: 实时双向通信，支持语音流传输和文本聊天
- **HTTP API**: RESTful接口，支持OTA更新和视觉分析

### 服务端口
- **WebSocket服务**: 默认端口 8000
- **HTTP API服务**: 默认端口 8003

### 数据格式
- **WebSocket**: JSON文本消息 + 二进制音频数据
- **HTTP**: JSON格式请求/响应 + multipart/form-data文件上传

---

## WebSocket协议详解

### 1. 连接建立

#### 连接URL
```
ws://服务器IP:8000/xiaozhi/v1/
```

#### 认证方式

**方式1: 请求头认证 (推荐)**
```javascript
const headers = {
    "device-id": "设备MAC地址",      // 必填，设备唯一标识
    "client-id": "客户端ID",         // 可选，客户端标识
    "Authorization": "Bearer your-token"  // 可选，认证token
};

const ws = new WebSocket("ws://localhost:8000/xiaozhi/v1/", [], {
    headers: headers
});
```

**方式2: 查询参数认证**
```javascript
const url = "ws://localhost:8000/xiaozhi/v1/?device-id=AA:BB:CC:DD:EE:FF&client-id=client123";
const ws = new WebSocket(url);
```

#### 连接状态检查
```javascript
// 连接成功
ws.onopen = function(event) {
    console.log("WebSocket连接已建立");
};

// 连接错误
ws.onerror = function(error) {
    console.error("WebSocket连接错误:", error);
};

// 连接关闭
ws.onclose = function(event) {
    console.log("WebSocket连接已关闭");
};
```

### 2. 消息协议规范

#### 2.1 消息类型枚举

##### 客户端发送消息类型 (type字段)

| 类型 | 描述 | 必填字段 | 可选字段 |
|------|------|----------|----------|
| `hello` | 握手消息，建立连接 | type | audio_params, features |
| `listen` | 音频监听控制 | type, state | mode, text, session_id |
| `abort` | 中止当前操作 | type | - |
| `iot` | IoT设备相关消息 | type | descriptors, states |
| `mcp` | MCP协议消息 | type | payload |
| `server` | 服务端管理消息 | type, action | content |

##### 服务端发送消息类型 (type字段)

| 类型 | 描述 | 必填字段 | 可选字段 |
|------|------|----------|----------|
| `hello` | 握手响应 | type, session_id | audio_params, 其他配置 |
| `tts` | 语音合成状态 | type, state, session_id | text |
| `stt` | 语音识别结果 | type, text, session_id | confidence, is_final |
| `llm` | 大模型响应 | type, session_id | text, emotion |
| `server` | 服务端操作响应 | type, status | message, content |

#### 2.2 详细消息结构

##### 2.2.1 hello消息 (握手协议)

**客户端发送:**
```json
{
    "type": "hello",                    // 必填，固定值
    "version": 1,                       // 可选，协议版本号，数字
    "transport": "websocket",           // 可选，传输协议，字符串
    "audio_params": {                   // 可选，音频参数配置
        "format": "opus",               // 音频格式，枚举: "opus"
        "sample_rate": 16000,           // 采样率，数字，支持: 16000
        "channels": 1,                  // 声道数，数字，支持: 1
        "frame_duration": 60            // 帧时长(ms)，数字，支持: 60
    },
    "features": {                       // 可选，客户端特性声明
        "mcp": true                     // 是否支持MCP，布尔值
    }
}
```

**服务端响应:**
```json
{
    "type": "hello",                    // 响应类型标识
    "session_id": "550e8400-e29b-41d4-a716-446655440000",  // 会话ID，UUID字符串
    "version": 1,                       // 协议版本号
    "transport": "websocket",           // 传输协议确认
    "audio_params": {                   // 音频参数确认
        "format": "opus",
        "sample_rate": 16000,
        "channels": 1,
        "frame_duration": 60
    }
}
```

##### 2.2.2 listen消息 (音频监听控制)

**客户端发送:**
```json
{
    "type": "listen",                   // 必填，固定值
    "state": "start",                   // 必填，监听状态，枚举值见下表
    "mode": "auto",                     // 可选，拾音模式，枚举值见下表
    "session_id": "uuid-string",        // 可选，会话ID
    "text": "用户输入的文本"             // 可选，仅当state为detect时使用
}
```

**state字段枚举值:**
| 值 | 描述 | 使用场景 |
|----|------|----------|
| `start` | 开始监听 | 开始语音输入 |
| `stop` | 停止监听 | 结束语音输入 |
| `detect` | 检测模式 | 文本输入或唤醒词检测 |

**mode字段枚举值:**
| 值 | 描述 | 行为 |
|----|------|------|
| `auto` | 自动模式 | VAD自动检测语音结束 |
| `manual` | 手动模式 | 需要手动发送stop停止 |

**使用示例:**
```json
// 开始语音输入
{
    "type": "listen",
    "state": "start",
    "mode": "auto",
    "session_id": "your-session-id"
}

// 停止语音输入
{
    "type": "listen", 
    "state": "stop",
    "session_id": "your-session-id"
}

// 文本输入
{
    "type": "listen",
    "state": "detect",
    "text": "今天天气怎么样？"
}
```

##### 2.2.3 abort消息 (中止操作)

**客户端发送:**
```json
{
    "type": "abort",                    // 必填，固定值
    "session_id": "uuid-string",        // 可选，会话ID
    "reason": "user_interrupt"          // 可选，中止原因
}
```

**reason字段可能值:**
| 值 | 描述 |
|----|------|
| `user_interrupt` | 用户主动中断 |
| `wake_word_detected` | 检测到唤醒词 |
| `timeout` | 操作超时 |

**服务端响应:**
```json
{
    "type": "tts",
    "state": "stop",                    // 固定为stop
    "session_id": "uuid-string"
}
```

##### 2.2.4 server消息 (服务端管理)

**客户端发送:**
```json
{
    "type": "server",                   // 必填，固定值
    "action": "update_config",          // 必填，操作类型，枚举值见下表
    "content": {                        // 可选，操作内容
        "secret": "your-secret-key"     // 验证密钥
    }
}
```

**action字段枚举值:**
| 值 | 描述 | 权限要求 |
|----|------|----------|
| `update_config` | 更新服务器配置 | 需要secret验证 |
| `restart` | 重启服务器 | 需要secret验证 |

**服务端响应:**
```json
{
    "type": "server",
    "status": "success",                // 操作状态，枚举: "success", "error"
    "message": "配置更新成功",           // 结果描述信息
    "content": {
        "action": "update_config"       // 回传操作类型
    }
}
```

##### 2.2.5 iot消息 (IoT设备控制)

IoT消息用于智能设备控制，支持设备能力声明、状态上报和控制命令接收。主要用于ESP32等硬件设备与服务端的双向通信。

**IoT消息类型说明:**

| 消息类型 | 用途 | 发送方向 | 使用场景 |
|----------|------|----------|----------|
| descriptors | 设备能力声明 | 客户端→服务端 | 设备连接时声明功能 |
| states | 设备状态上报 | 客户端→服务端 | 状态变化时主动上报 |
| commands | 控制命令接收 | 服务端→客户端 | 语音控制设备操作 |

#### 2.2.5.1 设备能力声明 (descriptors)

**客户端发送 - 完整设备描述符:**
```json
{
    "type": "iot",                      // 必填，固定值
    "descriptors": [                    // 必填，设备描述符数组
        {
            "name": "living_room_light",     // 必填，设备唯一标识
            "description": "客厅智能灯",      // 必填，设备人类可读描述
            "properties": {                  // 必填，设备属性定义
                "power": {
                    "type": "boolean",       // 属性类型：boolean/number/string
                    "description": "电源状态", // 属性描述
                    "readable": true,        // 是否可读取
                    "writable": false        // 是否可写入
                },
                "brightness": {
                    "type": "number",
                    "description": "亮度值(0-100)",
                    "min": 0,               // 数值类型的最小值
                    "max": 100,             // 数值类型的最大值
                    "readable": true,
                    "writable": true
                },
                "color": {
                    "type": "string",
                    "description": "灯光颜色",
                    "enum": ["red", "green", "blue", "white"], // 枚举值
                    "readable": true,
                    "writable": true
                }
            },
            "methods": {                     // 必填，设备方法定义
                "turn_on": {
                    "description": "打开灯光",
                    "parameters": {          // 方法参数定义
                        "brightness": {
                            "type": "number",
                            "description": "设置亮度(可选)",
                            "min": 0,
                            "max": 100,
                            "required": false
                        }
                    },
                    "response_success": "客厅灯已打开，亮度{brightness}%",
                    "response_failure": "客厅灯打开失败，请检查设备连接"
                },
                "turn_off": {
                    "description": "关闭灯光",
                    "parameters": {},
                    "response_success": "客厅灯已关闭",
                    "response_failure": "客厅灯关闭失败"
                },
                "set_brightness": {
                    "description": "设置亮度",
                    "parameters": {
                        "value": {
                            "type": "number",
                            "description": "亮度值",
                            "min": 0,
                            "max": 100,
                            "required": true
                        }
                    },
                    "response_success": "客厅灯亮度已调整到{value}%",
                    "response_failure": "亮度调节失败"
                },
                "set_color": {
                    "description": "设置颜色",
                    "parameters": {
                        "color": {
                            "type": "string",
                            "description": "灯光颜色",
                            "enum": ["red", "green", "blue", "white"],
                            "required": true
                        }
                    },
                    "response_success": "客厅灯颜色已设置为{color}",
                    "response_failure": "颜色设置失败"
                }
            }
        }
    ]
}
```

**支持的设备类型和属性:**

| 设备类型 | 常用属性 | 常用方法 | 示例设备 |
|----------|----------|----------|----------|
| `light` | power, brightness, color, temperature | turn_on, turn_off, set_brightness, set_color | 智能灯泡、LED灯带 |
| `switch` | power | turn_on, turn_off | 智能开关、插座 |
| `sensor` | temperature, humidity, motion, light_level | get_reading | 温湿度传感器、人体感应器 |
| `curtain` | position, state | open, close, stop, set_position | 智能窗帘、百叶窗 |
| `fan` | power, speed, direction | turn_on, turn_off, set_speed | 智能风扇、吊扇 |
| `air_conditioner` | power, temperature, mode, fan_speed | turn_on, turn_off, set_temperature, set_mode | 空调、新风系统 |
| `robot_vacuum` | state, battery, cleaning_mode | start, stop, pause, return_home | 扫地机器人 |
| `security` | armed, triggered | arm, disarm | 安防系统、门锁 |

**属性类型定义:**

| 类型 | 描述 | 额外字段 | 示例值 |
|------|------|----------|--------|
| `boolean` | 布尔值 | - | true, false |
| `number` | 数值 | min, max, step | 0-100, 16-30 |
| `string` | 字符串 | enum, pattern | "red", "auto" |

#### 2.2.5.2 设备状态上报 (states)

**客户端发送 - 状态更新:**
```json
{
    "type": "iot",                      // 必填，固定值
    "states": [                         // 必填，状态数组
        {
            "name": "living_room_light", // 必填，设备名称（对应descriptors中的name）
            "state": {                  // 必填，当前状态值
                "power": true,          // 布尔属性
                "brightness": 80,       // 数值属性
                "color": "white"        // 字符串属性
            },
            "timestamp": 1699123456789  // 可选，状态更新时间戳（毫秒）
        }
    ]
}
```

**状态上报时机:**
1. **设备连接后**: 上报初始状态
2. **状态变化时**: 硬件状态改变时主动上报
3. **控制命令执行后**: 执行控制命令后上报新状态
4. **定期心跳**: 可选的定期状态同步

#### 2.2.5.3 控制命令接收 (commands)

**服务端发送 - 控制命令:**
```json
{
    "type": "iot",                      // 固定值
    "commands": [                       // 控制命令数组
        {
            "name": "living_room_light", // 设备名称
            "method": "turn_on",        // 调用的方法名
            "parameters": {             // 方法参数
                "brightness": 80
            },
            "command_id": "cmd_123456789" // 可选，命令ID用于追踪
        }
    ]
}
```

**客户端响应 - 执行结果:**
```json
{
    "type": "iot",
    "command_results": [                // 命令执行结果数组
        {
            "command_id": "cmd_123456789", // 对应的命令ID
            "name": "living_room_light",
            "method": "turn_on",
            "success": true,            // 执行是否成功
            "result": "灯光已打开",      // 执行结果描述
            "new_state": {              // 执行后的新状态
                "power": true,
                "brightness": 80
            }
        }
    ]
}
```

#### 2.2.5.4 IoT工具自动注册机制

当服务端接收到IoT描述符后，会自动为每个设备生成对应的LLM函数调用工具：

**自动生成的查询工具:**
- `get_{device_name}_{property_name}()` - 查询设备属性
- 例如：`get_living_room_light_brightness()`, `get_living_room_light_power()`

**自动生成的控制工具:**
- `{device_name}_{method_name}()` - 执行设备方法
- 例如：`living_room_light_turn_on()`, `living_room_light_set_brightness()`

**语音控制示例流程:**
```
用户语音: "把客厅灯调到50%亮度"
    ↓ ASR识别
LLM理解意图，调用: living_room_light_set_brightness(value=50)
    ↓ IoT执行器
发送命令: {"type":"iot", "commands":[{"name":"living_room_light", "method":"set_brightness", "parameters":{"value":50}}]}
    ↓ 设备执行
硬件调节亮度 + 状态上报: {"type":"iot", "states":[{"name":"living_room_light", "state":{"brightness":50}}]}
    ↓ TTS合成响应
播放: "客厅灯亮度已调整到50%"
```

#### 2.2.5.5 IoT集成方式

**方式一: 设备端IoT (推荐)**
- ESP32等硬件设备直接通过WebSocket连接
- 支持即插即用，动态能力发现
- 实时双向通信，状态同步及时

**方式二: HomeAssistant集成**
- 通过REST API或MCP协议集成HomeAssistant
- 支持现有HA生态的所有设备
- 可配置为独立LLM工具或HA语音助手

**方式三: MCP协议扩展**
- 通过Model Context Protocol扩展IoT功能
- 支持更复杂的设备控制逻辑
- 与其他MCP工具无缝集成

#### 2.2.5.6 错误处理

**设备离线检测:**
```json
{
    "type": "server",
    "status": "error",
    "message": "设备 living_room_light 离线或无响应",
    "error_code": "DEVICE_OFFLINE"
}
```

**参数验证失败:**
```json
{
    "type": "server", 
    "status": "error",
    "message": "亮度值超出范围，应为0-100",
    "error_code": "INVALID_PARAMETER"
}
```

**命令执行失败:**
```json
{
    "type": "iot",
    "command_results": [
        {
            "command_id": "cmd_123456789",
            "success": false,
            "error": "设备硬件故障",
            "error_code": "HARDWARE_ERROR"
        }
    ]
}
```

##### 2.2.6 mcp消息 (Model Context Protocol)

**客户端发送:**
```json
{
    "type": "mcp",                      // 必填，固定值
    "payload": {                        // MCP协议负载
        "method": "tool_call",          // MCP方法
        "params": {                     // MCP参数
            "tool_name": "weather",
            "arguments": {"city": "北京"}
        }
    }
}
```

#### 2.3 服务端响应消息详解

##### 2.3.1 TTS状态消息

**消息格式:**
```json
{
    "type": "tts",                      // 必填，固定值
    "state": "sentence_start",          // 必填，TTS状态，枚举值见下表
    "text": "今天北京天气晴朗",           // 可选，合成的文本内容
    "session_id": "uuid-string"         // 必填，会话ID
}
```

**state字段枚举值:**
| 值 | 描述 | 何时发送 |
|----|------|----------|
| `start` | TTS开始 | 开始语音合成时 |
| `sentence_start` | 句子开始 | 每个句子开始合成时 |
| `sentence_end` | 句子结束 | 每个句子合成完成时 |
| `stop` | TTS停止 | 全部合成完成或中止时 |

##### 2.3.2 STT识别消息

**消息格式:**
```json
{
    "type": "stt",                      // 必填，固定值
    "text": "今天天气怎么样",             // 必填，识别的文本
    "confidence": 0.95,                 // 可选，识别置信度，范围0-1
    "is_final": true,                   // 可选，是否为最终结果
    "session_id": "uuid-string"         // 必填，会话ID
}
```

##### 2.3.3 LLM情感消息

**消息格式:**
```json
{
    "type": "llm",                      // 必填，固定值
    "text": "😊",                      // 可选，表情符号
    "emotion": "happy",                 // 可选，情感分析结果
    "session_id": "uuid-string"         // 必填，会话ID
}
```

**emotion字段枚举值:**
| 值 | 中文含义 | 表情符号 |
|----|----------|----------|
| `neutral` | 中性 | 😐 |
| `happy` | 开心 | 😊 |
| `laughing` | 大笑 | 😂 |
| `funny` | 有趣 | 😄 |
| `sad` | 伤心 | 😢 |
| `angry` | 生气 | 😠 |
| `crying` | 哭泣 | 😭 |
| `loving` | 爱心 | 😍 |
| `embarrassed` | 尴尬 | 😳 |
| `surprised` | 惊讶 | 😲 |
| `shocked` | 震惊 | 😱 |
| `thinking` | 思考 | 🤔 |
| `winking` | 眨眼 | 😉 |
| `cool` | 酷 | 😎 |
| `relaxed` | 放松 | 😌 |
| `delicious` | 美味 | 😋 |
| `kissy` | 亲吻 | 😘 |
| `confident` | 自信 | 😏 |
| `sleepy` | 困倦 | 😴 |
| `silly` | 傻笑 | 😜 |
| `confused` | 困惑 | 😕 |

### 3. 音频数据传输协议

#### 3.1 音频格式规范

| 参数 | 值 | 描述 |
|------|-----|------|
| 编码格式 | Opus | 必须使用Opus编码 |
| 采样率 | 16000 Hz | 固定采样率 |
| 声道数 | 1 | 单声道 |
| 帧时长 | 60ms | 每个音频帧的时长 |
| 传输方式 | WebSocket二进制帧 | 使用二进制消息发送 |

#### 3.2 音频数据发送

**客户端发送音频:**
```javascript
// 音频数据为Opus编码的二进制数据
const opusAudioData = new Uint8Array([...]); // Opus音频帧
ws.send(opusAudioData);
```

**服务端发送音频:**
```javascript
// 接收服务端音频数据
ws.onmessage = function(event) {
    if (event.data instanceof Blob || event.data instanceof ArrayBuffer) {
        // 处理音频数据
        handleAudioData(event.data);
    } else {
        // 处理文本消息
        const message = JSON.parse(event.data);
        handleTextMessage(message);
    }
};
```

### 4. 完整交互流程

#### 4.1 连接建立流程

```
1. 客户端连接 WebSocket
2. 客户端发送 hello 消息
3. 服务端返回 hello 响应 (包含 session_id)
4. 连接建立完成，可以开始通信
```

**代码示例:**
```javascript
// 1. 建立连接
const ws = new WebSocket("ws://localhost:8000/xiaozhi/v1/");
let sessionId = null;

// 2. 连接成功后发送hello
ws.onopen = function() {
    const hello = {
        "type": "hello",
        "version": 1,
        "transport": "websocket",
        "audio_params": {
            "format": "opus",
            "sample_rate": 16000,
            "channels": 1,
            "frame_duration": 60
        }
    };
    ws.send(JSON.stringify(hello));
};

// 3. 处理hello响应
ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    if (message.type === "hello") {
        sessionId = message.session_id;
        console.log("连接建立成功，会话ID:", sessionId);
    }
};
```

#### 4.2 文本聊天流程

```
1. 发送 listen 消息 (state: detect, text: 用户输入)
2. 服务端处理并返回 tts 消息 (state: sentence_start)
3. 服务端发送音频数据流
4. 服务端发送 tts 消息 (state: sentence_end)
```

**代码示例:**
```javascript
function sendTextMessage(text) {
    const message = {
        "type": "listen",
        "state": "detect",
        "text": text
    };
    ws.send(JSON.stringify(message));
}

// 处理服务端响应
ws.onmessage = function(event) {
    if (typeof event.data === 'string') {
        const message = JSON.parse(event.data);
        if (message.type === "tts" && message.state === "sentence_start") {
            console.log("AI回复:", message.text);
        }
    } else {
        // 播放音频数据
        playAudioData(event.data);
    }
};
```

#### 4.3 语音对话流程

```
1. 发送 listen 消息 (state: start, mode: auto)
2. 开始发送音频数据流
3. 语音结束时发送 listen 消息 (state: stop)
4. 服务端返回 stt 消息 (语音识别结果)
5. 服务端返回 tts 消息和音频流 (AI回复)
```

**代码示例:**
```javascript
// 开始语音输入
function startVoiceInput() {
    const message = {
        "type": "listen",
        "state": "start",
        "mode": "auto",
        "session_id": sessionId
    };
    ws.send(JSON.stringify(message));
    
    // 开始录音并发送音频数据
    startRecording();
}

// 停止语音输入
function stopVoiceInput() {
    const message = {
        "type": "listen",
        "state": "stop",
        "session_id": sessionId
    };
    ws.send(JSON.stringify(message));
    
    // 停止录音
    stopRecording();
}

// 发送音频数据
function sendAudioData(opusData) {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(opusData);
    }
}
```

#### 4.4 中止操作流程

```
1. 发送 abort 消息
2. 服务端停止当前所有操作
3. 服务端返回 tts 消息 (state: stop)
4. 清理相关状态
```

**代码示例:**
```javascript
function abortCurrentOperation() {
    const message = {
        "type": "abort",
        "session_id": sessionId,
        "reason": "user_interrupt"
    };
    ws.send(JSON.stringify(message));
}
```

### 5. 状态管理

#### 5.1 连接状态

| 状态 | 描述 | 允许的操作 |
|------|------|------------|
| 未连接 | WebSocket连接未建立 | 建立连接 |
| 已连接未握手 | 连接建立但未发送hello | 发送hello |
| 已握手 | 完成hello握手，获得session_id | 所有操作 |
| 监听中 | 正在进行语音输入 | 发送音频、停止监听、中止 |
| TTS中 | 正在进行语音合成 | 接收音频、中止 |

#### 5.2 错误状态处理

**认证失败:**
```json
{
    "type": "server",
    "status": "error", 
    "message": "设备认证失败",
    "error_code": "AUTH_FAILED"
}
```

**会话过期:**
```json
{
    "type": "server",
    "status": "error",
    "message": "会话已过期，请重新连接",
    "error_code": "SESSION_EXPIRED"
}
```

**消息格式错误:**
```json
{
    "type": "server",
    "status": "error",
    "message": "消息格式不正确",
    "error_code": "INVALID_MESSAGE"
}
```

---

## HTTP API接口

### 1. OTA更新接口

#### 1.1 OTA状态检查

**接口信息:**
- **URL**: `GET /xiaozhi/ota/`
- **描述**: 检查OTA接口运行状态
- **认证**: 无需认证
- **响应格式**: text/plain

**请求示例:**
```bash
curl -X GET http://localhost:8003/xiaozhi/ota/
```

**响应示例:**
```
OTA接口运行正常，向设备发送的websocket地址是：ws://192.168.1.100:8000/xiaozhi/v1/
```

#### 1.2 OTA信息获取

**接口信息:**
- **URL**: `POST /xiaozhi/ota/`
- **描述**: 获取设备OTA信息和WebSocket配置
- **认证**: 需要device-id请求头
- **请求格式**: application/json
- **响应格式**: application/json

**请求头:**
```http
Content-Type: application/json
device-id: AA:BB:CC:DD:EE:FF
```

**请求体结构:**
```json
{
    "application": {                    // 必填，应用信息
        "version": "1.0.0",             // 必填，当前版本号，字符串
        "build": "20240101"             // 可选，构建号，字符串
    }
}
```

**成功响应 (200):**
```json
{
    "server_time": {                    // 服务器时间信息
        "timestamp": 1699123456789,     // 时间戳，毫秒，数字
        "timezone_offset": 480          // 时区偏移，分钟，数字
    },
    "firmware": {                       // 固件信息
        "version": "1.0.0",             // 固件版本号，字符串
        "url": "",                      // 下载地址，字符串（空表示无更新）
        "size": 0,                      // 文件大小，字节，数字
        "checksum": ""                  // 文件校验和，字符串
    },
    "websocket": {                      // WebSocket配置
        "url": "ws://192.168.1.100:8000/xiaozhi/v1/"  // WebSocket地址
    }
}
```

**错误响应 (400):**
```json
{
    "success": false,                   // 固定为false
    "message": "request error."         // 错误描述
}
```

**完整请求示例:**
```bash
curl -X POST http://localhost:8003/xiaozhi/ota/ \
  -H "Content-Type: application/json" \
  -H "device-id: AA:BB:CC:DD:EE:FF" \
  -d '{
    "application": {
      "version": "1.0.0",
      "build": "20240101"
    }
  }'
```

### 2. 视觉分析接口

#### 2.1 视觉分析状态检查

**接口信息:**
- **URL**: `GET /mcp/vision/explain`
- **描述**: 检查视觉分析接口状态
- **认证**: 无需认证
- **响应格式**: text/plain

**请求示例:**
```bash
curl -X GET http://localhost:8003/mcp/vision/explain
```

**响应示例:**
```
MCP Vision 接口运行正常，视觉解释接口地址是：http://localhost:8003/mcp/vision/explain
```

#### 2.2 图像视觉分析

**接口信息:**
- **URL**: `POST /mcp/vision/explain`
- **描述**: 上传图片进行AI视觉分析
- **认证**: Bearer Token + 设备ID
- **请求格式**: multipart/form-data
- **响应格式**: application/json
- **文件大小限制**: 最大 5MB

**请求头:**
```http
Content-Type: multipart/form-data
Authorization: Bearer your-jwt-token        # 必填，JWT认证token
Device-Id: AA:BB:CC:DD:EE:FF               # 必填，设备ID
Client-Id: client123                       # 可选，客户端ID
```

**请求参数:**

| 参数名 | 类型 | 必填 | 描述 |
|--------|------|------|------|
| question | string | 是 | 用户对图片的问题 |
| image | file | 是 | 图片文件 |

**支持的图片格式:**

| 格式 | MIME类型 | 文件扩展名 | 备注 |
|------|----------|------------|------|
| JPEG | image/jpeg | .jpg, .jpeg | 推荐格式 |
| PNG | image/png | .png | 支持透明度 |
| GIF | image/gif | .gif | 支持动画 |
| BMP | image/bmp | .bmp | 位图格式 |
| TIFF | image/tiff | .tiff, .tif | 高质量格式 |
| WebP | image/webp | .webp | 现代格式 |

**成功响应 (200):**
```json
{
    "success": true,                        // 固定为true
    "action": "RESPONSE",                   // 动作类型，枚举: "RESPONSE"
    "response": "这是一张美丽的风景照片，画面中有蓝天白云、绿树成荫的山峦，还有一条清澈的小溪从山间流过。整体色彩搭配和谐，给人一种宁静祥和的感觉。"
}
```

**错误响应格式:**

| HTTP状态码 | 错误原因 | 响应格式 |
|------------|----------|----------|
| 400 | 请求参数错误 | {"success": false, "message": "缺少必填参数"} |
| 401 | 认证失败 | {"success": false, "message": "Token无效"} |
| 403 | 权限不足 | {"success": false, "message": "设备无权限"} |
| 413 | 文件过大 | {"success": false, "message": "文件大小超过5MB限制"} |
| 415 | 格式不支持 | {"success": false, "message": "不支持的图片格式"} |
| 500 | 服务器错误 | {"success": false, "message": "视觉分析服务异常"} |

**请求示例 (curl):**
```bash
curl -X POST http://localhost:8003/mcp/vision/explain \
  -H "Authorization: Bearer your-jwt-token" \
  -H "Device-Id: AA:BB:CC:DD:EE:FF" \
  -H "Client-Id: client123" \
  -F "question=这张图片里有什么内容？请详细描述。" \
  -F "image=@/path/to/your/image.jpg"
```

**请求示例 (JavaScript):**
```javascript
async function analyzeImage(imageFile, question) {
    const formData = new FormData();
    formData.append('question', question);
    formData.append('image', imageFile);

    try {
        const response = await fetch('http://localhost:8003/mcp/vision/explain', {
            method: 'POST',
            headers: {
                'Authorization': 'Bearer your-jwt-token',
                'Device-Id': 'AA:BB:CC:DD:EE:FF',
                'Client-Id': 'client123'
            },
            body: formData
        });

        const result = await response.json();
        
        if (result.success) {
            console.log('视觉分析结果:', result.response);
            return result.response;
        } else {
            throw new Error(result.message);
        }
    } catch (error) {
        console.error('视觉分析失败:', error);
        throw error;
    }
}

// 使用示例
const fileInput = document.getElementById('imageInput');
const file = fileInput.files[0];
analyzeImage(file, '这张图片里有什么？').then(result => {
    console.log(result);
});
```

**请求示例 (Python):**
```python
import requests

def analyze_image(image_path, question, token, device_id):
    url = 'http://localhost:8003/mcp/vision/explain'
    
    headers = {
        'Authorization': f'Bearer {token}',
        'Device-Id': device_id,
        'Client-Id': 'python-client'
    }
    
    files = {
        'image': open(image_path, 'rb')
    }
    
    data = {
        'question': question
    }
    
    try:
        response = requests.post(url, headers=headers, files=files, data=data)
        result = response.json()
        
        if result.get('success'):
            return result.get('response')
        else:
            raise Exception(result.get('message', '分析失败'))
    
    finally:
        files['image'].close()

# 使用示例
result = analyze_image(
    image_path='/path/to/image.jpg',
    question='这张图片的主要内容是什么？',
    token='your-jwt-token',
    device_id='AA:BB:CC:DD:EE:FF'
)
print(result)
```

### 3. CORS跨域支持

所有HTTP接口都支持跨域请求，配置如下：

**支持的HTTP方法:**
- GET
- POST  
- OPTIONS (预检请求)

**CORS响应头:**
```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: client-id, content-type, device-id, authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

**预检请求示例:**
```bash
curl -X OPTIONS http://localhost:8003/mcp/vision/explain \
  -H "Origin: https://your-domain.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: authorization,device-id"
```

---

## 错误处理

### 1. WebSocket错误

#### 1.1 连接级错误

**认证失败:**
```json
{
    "type": "server",
    "status": "error",
    "message": "设备认证失败",
    "error_code": "AUTH_FAILED"
}
```

**设备ID缺失:**
```json
{
    "type": "server",
    "status": "error", 
    "message": "缺少设备ID",
    "error_code": "MISSING_DEVICE_ID"
}
```

#### 1.2 消息级错误

**JSON格式错误:**
```json
{
    "type": "server",
    "status": "error",
    "message": "消息格式不正确",
    "error_code": "INVALID_JSON"
}
```

**未知消息类型:**
```json
{
    "type": "server", 
    "status": "error",
    "message": "不支持的消息类型",
    "error_code": "UNKNOWN_MESSAGE_TYPE"
}
```

**会话ID无效:**
```json
{
    "type": "server",
    "status": "error",
    "message": "会话ID无效或已过期", 
    "error_code": "INVALID_SESSION"
}
```

#### 1.3 业务级错误

**音频格式不支持:**
```json
{
    "type": "server",
    "status": "error",
    "message": "不支持的音频格式",
    "error_code": "UNSUPPORTED_AUDIO_FORMAT"
}
```

**请求频率过高:**
```json
{
    "type": "server",
    "status": "error", 
    "message": "请求频率过高，请稍后重试",
    "error_code": "RATE_LIMIT_EXCEEDED"
}
```

### 2. HTTP错误

#### 2.1 4xx客户端错误

**400 Bad Request:**
```json
{
    "success": false,
    "error_code": "INVALID_REQUEST",
    "message": "请求参数无效",
    "details": {
        "missing_fields": ["device-id"],
        "invalid_fields": []
    }
}
```

**401 Unauthorized:**
```json
{
    "success": false,
    "error_code": "UNAUTHORIZED", 
    "message": "认证失败，Token无效或已过期"
}
```

**403 Forbidden:**
```json
{
    "success": false,
    "error_code": "FORBIDDEN",
    "message": "设备无访问权限"
}
```

**413 Payload Too Large:**
```json
{
    "success": false,
    "error_code": "FILE_TOO_LARGE",
    "message": "文件大小超过5MB限制"
}
```

**415 Unsupported Media Type:**
```json
{
    "success": false,
    "error_code": "UNSUPPORTED_FORMAT",
    "message": "不支持的文件格式，仅支持: JPEG, PNG, GIF, BMP, TIFF, WebP"
}
```

#### 2.2 5xx服务器错误

**500 Internal Server Error:**
```json
{
    "success": false,
    "error_code": "INTERNAL_ERROR",
    "message": "服务器内部错误",
    "request_id": "req_123456789"
}
```

**503 Service Unavailable:**
```json
{
    "success": false,
    "error_code": "SERVICE_UNAVAILABLE", 
    "message": "视觉分析服务暂时不可用"
}
```

### 3. 错误代码枚举

| 错误代码 | HTTP状态码 | 描述 | 解决方案 |
|----------|------------|------|----------|
| `AUTH_FAILED` | 401 | 认证失败 | 检查Token是否有效 |
| `MISSING_DEVICE_ID` | 400 | 缺少设备ID | 添加device-id请求头 |
| `INVALID_JSON` | 400 | JSON格式错误 | 检查消息格式 |
| `UNKNOWN_MESSAGE_TYPE` | 400 | 未知消息类型 | 检查type字段值 |
| `INVALID_SESSION` | 401 | 会话无效 | 重新建立连接 |
| `UNSUPPORTED_AUDIO_FORMAT` | 400 | 音频格式错误 | 使用Opus格式 |
| `RATE_LIMIT_EXCEEDED` | 429 | 请求过频 | 降低请求频率 |
| `FILE_TOO_LARGE` | 413 | 文件过大 | 压缩图片到5MB以下 |
| `UNSUPPORTED_FORMAT` | 415 | 格式不支持 | 使用支持的图片格式 |
| `INTERNAL_ERROR` | 500 | 服务器错误 | 联系技术支持 |
| `SERVICE_UNAVAILABLE` | 503 | 服务不可用 | 稍后重试 |

---

## 认证机制

### 1. WebSocket认证

#### 1.1 设备ID认证 (基础)

**请求头方式:**
```javascript
const headers = {
    "device-id": "AA:BB:CC:DD:EE:FF",  // 必填，设备MAC地址
    "client-id": "client123"           // 可选，客户端标识
};
```

**查询参数方式:**
```
ws://localhost:8000/xiaozhi/v1/?device-id=AA:BB:CC:DD:EE:FF&client-id=client123
```

#### 1.2 Token认证 (增强)

**Bearer Token格式:**
```javascript
const headers = {
    "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "device-id": "AA:BB:CC:DD:EE:FF"
};
```

#### 1.3 设备白名单

服务端可配置设备白名单，白名单中的设备无需Token即可连接：

```yaml
# config.yaml
server:
  auth:
    enabled: true
    allowed_devices:
      - "AA:BB:CC:DD:EE:FF"
      - "11:22:33:44:55:66"
```

### 2. HTTP认证

#### 2.1 JWT Token认证

**Token格式:**
```http
Authorization: Bearer <JWT_TOKEN>
```

**Token生成示例 (Python):**
```python
import jwt
import time

def generate_token(device_id, secret_key, expires_in=3600):
    """生成JWT Token
    
    Args:
        device_id: 设备ID
        secret_key: 签名密钥
        expires_in: 过期时间(秒)，默认1小时
    """
    payload = {
        'device_id': device_id,
        'iat': int(time.time()),           # 签发时间
        'exp': int(time.time()) + expires_in  # 过期时间
    }
    token = jwt.encode(payload, secret_key, algorithm='HS256')
    return token

# 使用示例
token = generate_token("AA:BB:CC:DD:EE:FF", "your-secret-key")
```

**Token验证示例 (Python):**
```python
def verify_token(token, secret_key):
    """验证JWT Token"""
    try:
        payload = jwt.decode(token, secret_key, algorithms=['HS256'])
        return payload['device_id']
    except jwt.ExpiredSignatureError:
        raise Exception("Token已过期")
    except jwt.InvalidTokenError:
        raise Exception("Token无效")
```

#### 2.2 设备ID匹配

HTTP请求中的Token必须与device-id请求头匹配：

```python
def validate_request(token, device_id_header, secret_key):
    """验证请求合法性"""
    # 1. 验证Token
    token_device_id = verify_token(token, secret_key)
    
    # 2. 检查设备ID匹配
    if token_device_id != device_id_header:
        raise Exception("设备ID不匹配")
    
    return True
```

### 3. 设备绑定流程

#### 3.1 未绑定设备检测

当设备首次连接或未绑定时，服务端返回绑定码：

```json
{
    "success": false,
    "error_code": "DEVICE_NOT_BOUND",
    "message": "设备未绑定，请使用绑定码进行绑定",
    "bind_code": "123456"
}
```

#### 3.2 设备绑定请求

**绑定接口 (假设):**
```bash
curl -X POST http://localhost:8003/api/device/bind \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "AA:BB:CC:DD:EE:FF",
    "bind_code": "123456",
    "device_name": "客厅音箱"
  }'
```

**绑定成功响应:**
```json
{
    "success": true,
    "message": "设备绑定成功",
    "device_info": {
        "device_id": "AA:BB:CC:DD:EE:FF",
        "device_name": "客厅音箱",
        "bind_time": "2024-01-01T12:00:00Z"
    }
}
```

---

## 客户端示例

### 1. JavaScript完整示例

```html
<!DOCTYPE html>
<html>
<head>
    <title>小智语音助手客户端</title>
    <meta charset="UTF-8">
</head>
<body>
    <div id="app">
        <h1>小智语音助手测试客户端</h1>
        
        <!-- 连接状态 -->
        <div id="status">状态: 未连接</div>
        
        <!-- 文本聊天 -->
        <div>
            <h3>文本聊天</h3>
            <input type="text" id="messageInput" placeholder="输入消息..." style="width: 300px;">
            <button onclick="sendTextMessage()">发送</button>
        </div>
        
        <!-- 语音控制 -->
        <div>
            <h3>语音控制</h3>
            <button id="voiceBtn" onclick="toggleVoice()">开始语音</button>
            <button onclick="abortOperation()">中止</button>
        </div>
        
        <!-- 消息显示 -->
        <div>
            <h3>消息记录</h3>
            <div id="messages" style="height: 300px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px;"></div>
        </div>
    </div>

    <script>
        class XiaozhiClient {
            constructor(config) {
                this.config = config;
                this.ws = null;
                this.sessionId = null;
                this.isConnected = false;
                this.isVoiceActive = false;
                this.mediaRecorder = null;
                this.audioChunks = [];
            }

            // 连接到服务器
            async connect() {
                try {
                    this.updateStatus("正在连接...");
                    
                    // 创建WebSocket连接
                    this.ws = new WebSocket(this.config.wsUrl);
                    
                    // 设置事件处理器
                    this.ws.onopen = () => this.onOpen();
                    this.ws.onmessage = (event) => this.onMessage(event);
                    this.ws.onclose = () => this.onClose();
                    this.ws.onerror = (error) => this.onError(error);
                    
                } catch (error) {
                    this.updateStatus(`连接失败: ${error.message}`);
                }
            }

            // 连接成功
            onOpen() {
                this.updateStatus("已连接，正在握手...");
                this.sendHello();
            }

            // 发送Hello握手消息
            sendHello() {
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
                this.sendMessage(hello);
            }

            // 处理收到的消息
            onMessage(event) {
                if (typeof event.data === 'string') {
                    // 处理JSON消息
                    try {
                        const message = JSON.parse(event.data);
                        this.handleTextMessage(message);
                    } catch (error) {
                        console.error("JSON解析失败:", error);
                    }
                } else {
                    // 处理音频数据
                    this.handleAudioMessage(event.data);
                }
            }

            // 处理文本消息
            handleTextMessage(message) {
                console.log("收到消息:", message);
                
                switch (message.type) {
                    case "hello":
                        this.sessionId = message.session_id;
                        this.isConnected = true;
                        this.updateStatus(`已连接 (会话: ${this.sessionId.substr(0, 8)}...)`);
                        this.addMessage("系统", "连接成功，可以开始对话");
                        break;
                        
                    case "tts":
                        this.handleTTSMessage(message);
                        break;
                        
                    case "stt":
                        this.addMessage("识别", message.text);
                        break;
                        
                    case "llm":
                        if (message.emotion) {
                            this.addMessage("情感", `${message.emotion} ${message.text || ''}`);
                        }
                        break;
                        
                    case "server":
                        this.handleServerMessage(message);
                        break;
                        
                    default:
                        console.log("未处理的消息类型:", message.type);
                }
            }

            // 处理TTS消息
            handleTTSMessage(message) {
                switch (message.state) {
                    case "sentence_start":
                        if (message.text) {
                            this.addMessage("小智", message.text);
                        }
                        break;
                    case "stop":
                        this.addMessage("系统", "TTS已停止");
                        break;
                }
            }

            // 处理服务器消息
            handleServerMessage(message) {
                if (message.status === "error") {
                    this.addMessage("错误", message.message);
                } else {
                    this.addMessage("服务器", message.message);
                }
            }

            // 处理音频数据
            handleAudioMessage(audioData) {
                console.log(`收到音频数据: ${audioData.byteLength} 字节`);
                // 这里可以实现Opus解码和音频播放
                // 由于浏览器环境限制，暂时只记录日志
                this.addMessage("音频", `收到 ${audioData.byteLength} 字节音频数据`);
            }

            // 连接关闭
            onClose() {
                this.isConnected = false;
                this.sessionId = null;
                this.updateStatus("连接已关闭");
                this.addMessage("系统", "连接已断开");
            }

            // 连接错误
            onError(error) {
                console.error("WebSocket错误:", error);
                this.updateStatus("连接错误");
                this.addMessage("错误", "连接发生错误");
            }

            // 发送JSON消息
            sendMessage(message) {
                if (this.ws && this.ws.readyState === WebSocket.OPEN) {
                    this.ws.send(JSON.stringify(message));
                    return true;
                }
                return false;
            }

            // 发送文本消息
            sendTextMessage(text) {
                if (!this.isConnected) {
                    this.addMessage("错误", "未连接到服务器");
                    return;
                }

                const message = {
                    type: "listen",
                    state: "detect",
                    text: text
                };

                if (this.sendMessage(message)) {
                    this.addMessage("用户", text);
                }
            }

            // 开始语音输入
            async startVoiceInput() {
                if (!this.isConnected) {
                    this.addMessage("错误", "未连接到服务器");
                    return;
                }

                try {
                    // 请求麦克风权限
                    const stream = await navigator.mediaDevices.getUserMedia({ 
                        audio: {
                            sampleRate: 16000,
                            channelCount: 1,
                            echoCancellation: true,
                            noiseSuppression: true
                        }
                    });

                    // 发送开始监听消息
                    const message = {
                        type: "listen",
                        state: "start",
                        mode: "auto",
                        session_id: this.sessionId
                    };

                    if (this.sendMessage(message)) {
                        this.isVoiceActive = true;
                        this.addMessage("系统", "开始语音输入");
                        
                        // 这里应该实现Opus编码和实时传输
                        // 由于浏览器限制，暂时模拟
                        this.simulateVoiceInput();
                    }

                } catch (error) {
                    this.addMessage("错误", `麦克风权限被拒绝: ${error.message}`);
                }
            }

            // 停止语音输入
            stopVoiceInput() {
                if (!this.isVoiceActive) return;

                const message = {
                    type: "listen",
                    state: "stop",
                    session_id: this.sessionId
                };

                if (this.sendMessage(message)) {
                    this.isVoiceActive = false;
                    this.addMessage("系统", "停止语音输入");
                }
            }

            // 模拟语音输入 (实际项目中需要实现Opus编码)
            simulateVoiceInput() {
                // 模拟3秒后自动停止
                setTimeout(() => {
                    if (this.isVoiceActive) {
                        this.stopVoiceInput();
                    }
                }, 3000);
            }

            // 中止当前操作
            abortOperation() {
                if (!this.isConnected) return;

                const message = {
                    type: "abort",
                    session_id: this.sessionId,
                    reason: "user_interrupt"
                };

                if (this.sendMessage(message)) {
                    this.isVoiceActive = false;
                    this.addMessage("系统", "已中止当前操作");
                }
            }

            // 更新状态显示
            updateStatus(status) {
                document.getElementById('status').textContent = `状态: ${status}`;
            }

            // 添加消息到显示区域
            addMessage(sender, content) {
                const messages = document.getElementById('messages');
                const div = document.createElement('div');
                const timestamp = new Date().toLocaleTimeString();
                div.innerHTML = `<strong>[${timestamp}] ${sender}:</strong> ${content}`;
                messages.appendChild(div);
                messages.scrollTop = messages.scrollHeight;
            }

            // 断开连接
            disconnect() {
                if (this.ws) {
                    this.ws.close();
                    this.ws = null;
                }
                this.isConnected = false;
                this.sessionId = null;
                this.isVoiceActive = false;
            }
        }

        // 初始化客户端
        const client = new XiaozhiClient({
            wsUrl: "ws://localhost:8000/xiaozhi/v1/"
        });

        // 页面加载完成后自动连接
        window.onload = function() {
            client.connect();
        };

        // 发送文本消息
        function sendTextMessage() {
            const input = document.getElementById('messageInput');
            const text = input.value.trim();
            if (text) {
                client.sendTextMessage(text);
                input.value = '';
            }
        }

        // 切换语音状态
        function toggleVoice() {
            const btn = document.getElementById('voiceBtn');
            if (client.isVoiceActive) {
                client.stopVoiceInput();
                btn.textContent = '开始语音';
            } else {
                client.startVoiceInput();
                btn.textContent = '停止语音';
            }
        }

        // 中止操作
        function abortOperation() {
            client.abortOperation();
            const btn = document.getElementById('voiceBtn');
            btn.textContent = '开始语音';
        }

        // 回车发送消息
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendTextMessage();
            }
        });

        // 页面关闭时断开连接
        window.onbeforeunload = function() {
            client.disconnect();
        };
    </script>
</body>
</html>
```

### 2. Python客户端示例

```python
import asyncio
import websockets
import json
import aiohttp
import logging

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class XiaozhiPythonClient:
    def __init__(self, ws_url, http_url, device_id, token=None):
        self.ws_url = ws_url
        self.http_url = http_url
        self.device_id = device_id
        self.token = token
        self.session_id = None
        self.ws = None
        self.is_connected = False
        self.is_voice_active = False
        
    async def connect(self):
        """建立WebSocket连接"""
        try:
            # 构建请求头
            headers = {
                "device-id": self.device_id,
                "client-id": f"python-client-{self.device_id}"
            }
            
            if self.token:
                headers["Authorization"] = f"Bearer {self.token}"
            
            logger.info(f"正在连接: {self.ws_url}")
            
            # 建立连接
            self.ws = await websockets.connect(
                self.ws_url,
                extra_headers=headers
            )
            
            logger.info("WebSocket连接建立成功")
            
            # 发送Hello握手
            await self.send_hello()
            
            # 启动消息循环
            await self.message_loop()
            
        except Exception as e:
            logger.error(f"连接失败: {e}")
            raise

    async def send_hello(self):
        """发送Hello握手消息"""
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
        await self.send_message(hello)
        logger.info("已发送Hello握手消息")

    async def send_message(self, message):
        """发送JSON消息"""
        if self.ws:
            message_str = json.dumps(message, ensure_ascii=False)
            await self.ws.send(message_str)
            logger.debug(f"发送消息: {message_str}")
        else:
            logger.error("WebSocket连接未建立")

    async def send_text(self, text):
        """发送文本消息"""
        message = {
            "type": "listen",
            "state": "detect",
            "text": text
        }
        await self.send_message(message)
        logger.info(f"发送文本: {text}")

    async def start_voice_input(self, mode="auto"):
        """开始语音输入"""
        if not self.is_connected:
            logger.error("未连接到服务器")
            return
            
        message = {
            "type": "listen",
            "state": "start",
            "mode": mode,
            "session_id": self.session_id
        }
        await self.send_message(message)
        self.is_voice_active = True
        logger.info("开始语音输入")

    async def stop_voice_input(self):
        """停止语音输入"""
        if not self.is_voice_active:
            return
            
        message = {
            "type": "listen",
            "state": "stop",
            "session_id": self.session_id
        }
        await self.send_message(message)
        self.is_voice_active = False
        logger.info("停止语音输入")

    async def abort_operation(self, reason="user_interrupt"):
        """中止当前操作"""
        message = {
            "type": "abort",
            "session_id": self.session_id,
            "reason": reason
        }
        await self.send_message(message)
        self.is_voice_active = False
        logger.info(f"中止操作: {reason}")

    async def message_loop(self):
        """消息接收循环"""
        async for message in self.ws:
            if isinstance(message, str):
                # 处理文本消息
                try:
                    data = json.loads(message)
                    await self.handle_text_message(data)
                except json.JSONDecodeError as e:
                    logger.error(f"JSON解析失败: {e}")
            else:
                # 处理音频数据
                await self.handle_audio_message(message)

    async def handle_text_message(self, message):
        """处理文本消息"""
        msg_type = message.get("type", "")
        
        if msg_type == "hello":
            self.session_id = message.get("session_id")
            self.is_connected = True
            logger.info(f"握手成功，会话ID: {self.session_id}")
            
        elif msg_type == "tts":
            await self.handle_tts_message(message)
            
        elif msg_type == "stt":
            text = message.get("text", "")
            logger.info(f"语音识别: {text}")
            
        elif msg_type == "llm":
            emotion = message.get("emotion", "")
            text = message.get("text", "")
            if emotion:
                logger.info(f"情感分析: {emotion} {text}")
                
        elif msg_type == "server":
            await self.handle_server_message(message)
            
        elif msg_type == "iot":
            await self.handle_iot_message(message)
            
        else:
            logger.warning(f"未知消息类型: {msg_type}")

    async def handle_tts_message(self, message):
        """处理TTS消息"""
        state = message.get("state", "")
        text = message.get("text", "")
        
        if state == "sentence_start" and text:
            logger.info(f"小智回复: {text}")
        elif state == "stop":
            logger.info("TTS已停止")

    async def handle_server_message(self, message):
        """处理服务器消息"""
        status = message.get("status", "")
        msg = message.get("message", "")
        
        if status == "error":
            logger.error(f"服务器错误: {msg}")
        else:
            logger.info(f"服务器消息: {msg}")

    async def handle_iot_message(self, message):
        """处理IoT消息"""
        if "commands" in message:
            # 处理控制命令
            commands = message.get("commands", [])
            logger.info(f"收到IoT控制命令: {len(commands)}个")
            await self.handle_iot_commands(commands)
        elif "descriptors" in message:
            # 处理设备描述符（通常不会从服务端接收）
            logger.info("收到IoT设备描述符")
        elif "states" in message:
            # 处理状态更新确认（通常不会从服务端接收）
            logger.info("收到IoT状态更新确认")
        else:
            logger.warning("未知的IoT消息格式")

    async def handle_audio_message(self, audio_data):
        """处理音频数据"""
        logger.info(f"收到音频数据: {len(audio_data)} 字节")
        # 这里可以实现音频播放逻辑

    async def vision_analyze(self, image_path, question):
        """视觉分析接口调用"""
        url = f"{self.http_url}/mcp/vision/explain"
        
        headers = {
            "Device-Id": self.device_id,
            "Client-Id": f"python-client-{self.device_id}"
        }
        
        if self.token:
            headers["Authorization"] = f"Bearer {self.token}"
        
        data = aiohttp.FormData()
        data.add_field('question', question)
        
        with open(image_path, 'rb') as f:
            data.add_field('image', f, filename=image_path.split('/')[-1])
        
        async with aiohttp.ClientSession() as session:
            async with session.post(url, headers=headers, data=data) as response:
                result = await response.json()
                
                if result.get('success'):
                    return result.get('response')
                else:
                    raise Exception(result.get('message', '视觉分析失败'))

    async def send_iot_descriptors(self, descriptors):
        """发送IoT设备描述符"""
        message = {
            "type": "iot",
            "descriptors": descriptors
        }
        await self.send_message(message)
        logger.info("已发送IoT设备描述符")

    async def send_iot_states(self, states):
        """发送IoT设备状态"""
        message = {
            "type": "iot", 
            "states": states
        }
        await self.send_message(message)
        logger.info("已发送IoT设备状态")

    async def handle_iot_commands(self, commands):
        """处理IoT控制命令"""
        results = []
        for command in commands:
            try:
                # 模拟设备控制逻辑
                device_name = command.get("name")
                method = command.get("method") 
                parameters = command.get("parameters", {})
                command_id = command.get("command_id")
                
                logger.info(f"执行IoT命令: {device_name}.{method}({parameters})")
                
                # 这里应该实现具体的硬件控制逻辑
                # 例如：GPIO控制、串口通信、I2C/SPI等
                success = True  # 模拟执行成功
                
                result = {
                    "command_id": command_id,
                    "name": device_name,
                    "method": method,
                    "success": success,
                    "result": f"{device_name}的{method}操作已执行"
                }
                
                if success:
                    # 模拟新状态（实际应用中应该读取硬件状态）
                    if method == "turn_on":
                        result["new_state"] = {"power": True}
                    elif method == "turn_off":
                        result["new_state"] = {"power": False}
                    elif method == "set_brightness":
                        result["new_state"] = {"brightness": parameters.get("value", 100)}
                
                results.append(result)
                
            except Exception as e:
                results.append({
                    "command_id": command.get("command_id"),
                    "success": False,
                    "error": str(e),
                    "error_code": "EXECUTION_ERROR"
                })
        
        # 发送执行结果
        await self.send_message({
            "type": "iot",
            "command_results": results
        })
        
        return results

    async def disconnect(self):
        """断开连接"""
        if self.ws:
            await self.ws.close()
            self.ws = None
        self.is_connected = False
        self.session_id = None
        self.is_voice_active = False
        logger.info("连接已断开")

# 使用示例
async def main():
    # 创建客户端实例
    client = XiaozhiPythonClient(
        ws_url="ws://localhost:8000/xiaozhi/v1/",
        http_url="http://localhost:8003",
        device_id="AA:BB:CC:DD:EE:FF",
        token="your-jwt-token"  # 可选
    )
    
    try:
        # 启动连接
        await client.connect()
    except KeyboardInterrupt:
        logger.info("用户中断")
    except Exception as e:
        logger.error(f"客户端异常: {e}")
    finally:
        await client.disconnect()

# 交互式测试函数
async def interactive_test():
    client = XiaozhiPythonClient(
        ws_url="ws://localhost:8000/xiaozhi/v1/",
        http_url="http://localhost:8003", 
        device_id="AA:BB:CC:DD:EE:FF"
    )
    
    # 连接任务
    connect_task = asyncio.create_task(client.connect())
    
    # 等待连接建立
    await asyncio.sleep(2)
    
    try:
        # 发送文本消息
        await client.send_text("你好，今天天气怎么样？")
        await asyncio.sleep(3)
        
        # 测试语音功能
        await client.start_voice_input()
        await asyncio.sleep(2)
        await client.stop_voice_input()
        await asyncio.sleep(1)
        
        # 测试中止功能
        await client.abort_operation()
        
    except Exception as e:
        logger.error(f"测试异常: {e}")
    finally:
        connect_task.cancel()
        await client.disconnect()

# IoT设备测试函数
async def iot_device_test():
    """IoT设备功能完整测试示例"""
    client = XiaozhiPythonClient(
        ws_url="ws://localhost:8000/xiaozhi/v1/",
        http_url="http://localhost:8003",
        device_id="AA:BB:CC:DD:EE:FF"
    )
    
    # 定义设备描述符
    device_descriptors = [
        {
            "name": "living_room_light",
            "description": "客厅智能灯",
            "properties": {
                "power": {
                    "type": "boolean",
                    "description": "电源状态",
                    "readable": True,
                    "writable": False
                },
                "brightness": {
                    "type": "number", 
                    "description": "亮度值(0-100)",
                    "min": 0,
                    "max": 100,
                    "readable": True,
                    "writable": True
                },
                "color": {
                    "type": "string",
                    "description": "灯光颜色",
                    "enum": ["red", "green", "blue", "white"],
                    "readable": True,
                    "writable": True
                }
            },
            "methods": {
                "turn_on": {
                    "description": "打开灯光",
                    "parameters": {
                        "brightness": {
                            "type": "number",
                            "description": "设置亮度(可选)",
                            "min": 0,
                            "max": 100,
                            "required": False
                        }
                    },
                    "response_success": "客厅灯已打开，亮度{brightness}%",
                    "response_failure": "客厅灯打开失败"
                },
                "turn_off": {
                    "description": "关闭灯光",
                    "parameters": {},
                    "response_success": "客厅灯已关闭",
                    "response_failure": "客厅灯关闭失败"
                },
                "set_brightness": {
                    "description": "设置亮度",
                    "parameters": {
                        "value": {
                            "type": "number",
                            "description": "亮度值",
                            "min": 0,
                            "max": 100,
                            "required": True
                        }
                    },
                    "response_success": "客厅灯亮度已调整到{value}%",
                    "response_failure": "亮度调节失败"
                }
            }
        }
    ]
    
    # 连接任务
    connect_task = asyncio.create_task(client.connect())
    
    # 等待连接建立
    await asyncio.sleep(2)
    
    try:
        # 1. 发送设备描述符
        logger.info("=== 步骤1: 发送设备描述符 ===")
        await client.send_iot_descriptors(device_descriptors)
        await asyncio.sleep(1)
        
        # 2. 发送初始状态
        logger.info("=== 步骤2: 发送初始设备状态 ===")
        initial_states = [
            {
                "name": "living_room_light",
                "state": {
                    "power": False,
                    "brightness": 0,
                    "color": "white"
                },
                "timestamp": int(time.time() * 1000)
            }
        ]
        await client.send_iot_states(initial_states)
        await asyncio.sleep(1)
        
        # 3. 测试语音控制
        logger.info("=== 步骤3: 测试语音控制 ===")
        await client.send_text("打开客厅灯")
        await asyncio.sleep(3)
        
        await client.send_text("把客厅灯调到80%亮度")
        await asyncio.sleep(3)
        
        await client.send_text("把客厅灯设置为红色")
        await asyncio.sleep(3)
        
        await client.send_text("关闭客厅灯")
        await asyncio.sleep(3)
        
        # 4. 模拟状态变化上报
        logger.info("=== 步骤4: 模拟设备状态变化 ===")
        status_updates = [
            {
                "name": "living_room_light",
                "state": {
                    "power": True,
                    "brightness": 50,
                    "color": "blue"
                },
                "timestamp": int(time.time() * 1000)
            }
        ]
        await client.send_iot_states(status_updates)
        await asyncio.sleep(1)
        
        logger.info("IoT设备测试完成")
        
    except Exception as e:
        logger.error(f"IoT测试异常: {e}")
    finally:
        connect_task.cancel()
        await client.disconnect()

if __name__ == "__main__":
    # 运行主程序
    # asyncio.run(main())
    
    # 运行交互式测试
    # asyncio.run(interactive_test())
    
    # 运行IoT设备测试
    asyncio.run(iot_device_test())
```

---

## 总结

本API文档详细定义了小智语音助手服务端的完整接口规范，包括：

### WebSocket协议
- **6种客户端消息类型**：hello、listen、abort、iot、mcp、server
- **5种服务端消息类型**：hello、tts、stt、llm、server
- **完整字段枚举**：所有type、state、mode等字段的可选值
- **详细数据结构**：每个消息的必填/可选字段说明
- **状态流转关系**：完整的对话流程和状态管理

### IoT设备控制
- **设备能力声明**：完整的descriptors定义，支持8种设备类型
- **状态实时同步**：双向状态更新机制，支持主动上报
- **自动工具注册**：基于描述符动态生成LLM函数调用工具
- **语音智能控制**：自然语言转设备操作，支持复杂参数传递
- **多集成方式**：设备端IoT、HomeAssistant、MCP协议扩展
- **类型安全验证**：参数类型检查和错误处理机制

### HTTP API
- **OTA接口**：设备固件更新和配置获取
- **视觉分析接口**：图像上传和AI分析，支持6种图片格式
- **完整错误处理**：详细的错误代码和HTTP状态码

### 认证机制
- **多种认证方式**：设备ID、JWT Token、设备白名单
- **设备绑定流程**：完整的设备注册和绑定机制

### 客户端示例
- **JavaScript客户端**：完整的浏览器端实现
- **Python客户端**：异步的服务端客户端实现
- **IoT设备示例**：ESP32等硬件设备完整集成方案

该文档为前端开发和IoT设备开发提供了完整的API规范，每个字段都有明确的类型、取值范围和使用说明，支持从简单的文本聊天到复杂的智能家居控制等全场景应用开发。