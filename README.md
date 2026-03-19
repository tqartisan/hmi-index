# 数采控制客户端 (Robot Data Collection Client)

[![License](https://img.shields.io/badge/license-GPL-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Linux-green.svg)]()

## 项目简介

数采控制客户端（HMI）是一个面向机器人/具身智能领域的端侧数据采集与处理系统，作为数据采集平台的配套组件使用。该系统采用高度模块化、可扩展的架构设计，支持多样化的数据来源(摄像头、传感器、ROS2 Topic等)和多级数据处理流程，为机器人数据采集提供完整的解决方案。

### 核心特性

- **模块化架构**: 采用"hmi-console + hmi-backen + hmi-agent + Handlers"的分层设计,职责清晰,易于扩展
- **灵活部署**: 支持全本体部署或分离部署,适应不同算力场景
- **插件式Handler**: 支持自定义Handler实现,适配不同型号机器人
- **完整数据流**: 覆盖采集、预处理、后处理、上传的全流程管理
- **实时监控**: 提供WebSocket实时预览、状态上报等能力
- **异步处理**: 采用异步任务机制,保证系统响应性能

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                         云端平台                             │
│       (数据采集管理系统 - 任务管理/数据标注/审核导出...)        │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP/HTTPS
                         │
┌────────────────────────┴────────────────────────────────────┐
│                          端侧部署区域                        │
│                                                             │
│         ┌──────────────┐      ┌──────────────┐              │
│         │  前端界面     │◄────►│     HMI      │              │
│         │  (Nginx)     │ HTTP │ (业务逻辑层)  │              │
│         └──────┬───────┘      └──────┬───────┘              │
│                │                     │ HTTP                 │
│                │              ┌──────┴───────┐              │
│                │              │    Agent     │              │
│                │              │  (核心调度器) │              │
│                │              └──────┬───────┘              │
│                │WebSocket            │ HTTP                 │
│                │（视频预览）   ┌──────┴───────┐              │
│                │              │  Handler(s)  │              │
│                │              │  (核心处理器) │              │
│                │              └──────┬───────┘              │
│                │                     │                      │
│                │              ┌──────┴───────┐              │
│                │              │  ROS2 / 硬件 │              │
│                └─────────────►│  (机器人本体) │              │
│                               └──────────────┘              │
└─────────────────────────────────────────────────────────────┘

```

### 组件说明

#### 1. [hmi-console](https://github.com/iss-artisan/hmi-console)
- **职责**: 用户交互层,负责发送控制指令和状态展示
- **技术**: Nginx + 静态HTML/JS
- **特点**: 设备类型不敏感,统一实现

#### 2. [hmi-backend](https://github.com/iss-artisan/hmi-backend)
- **职责**: 业务逻辑中转层,协调前端与Agent交互,管理任务状态
- **核心模块**:
  - `records`: 数据采集控制模块
  - `operations`: 运维管理模块(设备状态、故障管理、系统设置)
  - `tasks`: 定时任务模块(状态上报、数据清理)
- **数据存储**: SQLite本地数据库
- **特点**: 设备类型不敏感,统一实现

#### 3. [hmi-agent](https://github.com/iss-artisan/hmi-agent)
- **职责**: Handler生命周期管理、能力注册、任务调度、数据上传
- **核心功能**:
  - Handler进程管理(启动/监控/重启)
  - 能力列表维护与同步
  - HTTP接口服务
  - 配置管理
  - 数据上传模块
- **特点**: 设备类型不敏感,统一实现

#### 4. [Handler体系](https://github.com/iss-artisan/hmi-handlers) (业务执行单元)
Handler是与机器人本体直接交互的可执行程序,采用插件式设计,支持自定义实现。

**Handler必需实现的能力**:
- **robot_info**: 本体信息
- **collect**: 数据采集(摄像头、关节电机数据)
- **pre_process**: 数据预处理(格式转换、初步整理)

**Handler可选实现的能力**:
- **battery_status**: 电池状态
- **motor_status**: 电机状态
- **fault_info**: 故障信息
- **post_process**: 后处理(视频压缩、数据对齐、质量调整等)
- **audio**: 音频播放与控制
- **media**: 本体摄像头预览

## 逻辑架构

### 启动流程

1. **hmi-console与hmi-backend启动**: 等待Agent注册
2. **hmi-agent启动**: 向hmi-backend发送心跳注册(初始无能力列表)
3. **Handler拉起**: hmi-agent将`handler/`目录下的程序以子进程形式启动
4. **能力注册**: Handler启动完成后向hmi-agent注册能力列表
5. **能力同步**: hmi-agent通过心跳将能力列表同步至hmi-backend
6. **界面更新**: hmi-console根据能力列表动态展示功能模块

### 数据流转

```
采集开始 → 数据采集 → 预处理 → (可选)后处理 → 数据上传 → 云端存储
   ↓          ↓         ↓           ↓           ↓
 状态同步   实时预览   初步整理    数据对齐    进度上报
```

### 通信机制

- **hmi-console ↔ hmi-backend**: HTTP RESTful API
- **hmi-backend ↔ hmi-agent**: HTTP RESTful API + 心跳机制
- **hmi-agent ↔ Handler**: HTTP RESTful API + 异步回调
- **hmi-console ↔ Handler**: WebSocket (视频预览)
- **hmi-backend ↔ 云端**: HTTP RESTful API

## 部署方式

### 部署模式

#### 模式一: 全本体部署 (推荐)
适用于算力充足的机器人本体。

```
机器人本体
├── hmi-console (Nginx)
├── hmi-backend
└── hmi-agent
    └── Handlers
```

#### 模式二: 分离部署
适用于算力受限的机器人本体,将前端与HMI部署到外部设备。

```
外部设备(PC)              机器人本体
├── hmi-console (Nginx)      └── hmi-agent
└── hmi-backend   ◄─────────►    └── Handlers
    (网络互通)
```

### 安装目录结构

#### hmi-console
```
/var/www/my-frontend/...   # 前端构建产物
```

#### hmi-backend
```
/opt/hmi-backend/...       # hmi-backend制品及脚本
```

#### Agent与Handlers
```
/opt/hmi-agent/
├── ...                       # hmi-agent制品及脚本
├── handlers/                 # Handler存放目录
│   ├── collect-handler       # 具体Handler实现目录
│   │   ├── start.sh          # 必须包含start.sh脚本，Agent将遍历handlers下所有目录内的start.sh并启动
│   │   └── other-files
│   ├── pre-process-handler
│   │   └── ...
│   ├── status-handler
│   │   └── ...
│   ├── audio-handler
│   │   └── ...
│   └── post-process-handler
│   │   └── ...
└── data/                     # 数据存储目录
    ├── raw/                  # 原始采集数据
    ├── processed/            # 预处理数据
    └── upload/               # 待上传数据
```

### 配置说明

#### hmi-agent配置 (agent.conf)
```yaml
# HMI服务地址
hmi:
  host: "127.0.0.1"
  port: 8080

# Agent服务配置
agent:
  port: 5000

# 数据存储路径
storage:
  data_path: "./data"

# Handler健康检测
health_check:
  interval: 10  # 秒
  timeout: 5    # 秒
```

#### hmi-backend配置 (config.yaml)
```yaml
# 云端平台地址
platform:
  url: "https://platform.example.com"

# Agent地址(分离部署时需配置)
agent:
  host: "10.0.0.1"  # 机器人本体IP
  port: 5000

# 存储服务配置
storage:
  endpoint: "https://s3.example.com"
  access_key: "your-access-key"
  secret_key: "your-secret-key"
  bucket: "robot-data"
```

## 接口说明

### Handler能力接口

Handler需实现以下标准接口,Agent通过这些接口进行调度。

#### 必需接口

| 能力 | 接口 | 方法 | 描述 |
|------|------|------|------|
| 健康检测 | `/health` | GET | 健康状态检查 |
| 能力注册 | `/register` | POST | 向Agent注册能力列表 |
| 本体信息 | `/robot_info` | GET | 获取机器人基本信息 |
| 开始采集 | `/collect/start` | POST | 开始数据采集(异步) |
| 结束采集 | `/collect/stop` | POST | 结束数据采集(异步) |
| 丢弃采集 | `/collect/delete` | POST | 丢弃当前采集(异步) |
| 开始预处理 | `/pre_process/start` | POST | 开始预处理(异步) |
| 停止预处理 | `/pre_process/stop` | POST | 停止预处理(异步) |

#### 可选接口

| 能力 | 接口 | 方法 | 描述 |
|------|------|------|------|
| 电池状态 | `/battery_status` | GET | 获取电池状态 |
| 电机状态 | `/motor_status` | GET | 获取电机状态 |
| 故障信息 | `/fault_info` | GET | 获取故障信息 |
| 音频播放 | `/audio/play` | POST | 播放音频 |
| 音频状态 | `/audio/status` | GET/POST | 获取/设置音频状态 |
| 视频预览 | `/media/preview` | WebSocket | 实时视频流 |
| 开始后处理 | `/post_process/start` | POST | 开始后处理(异步) |
| 停止后处理 | `/post_process/stop` | POST | 停止后处理(异步) |

### 接口示例

#### Handler能力注册
```http
POST http://agent:5000/api/handler/register
Content-Type: application/json

{
  "handler_name": "collect-handler",
  "port": 5005,
  "base_url": "/api",
  "ws_port": 5015,
  "ws_base_url": "/media",
  "modules": ["collect", "media"]
}
```

#### 开始采集
```http
POST http://handler:5005/api/collect/start
Content-Type: application/json

{
  "episode_id": "ep_123456",
  "task_id": 1,
  "duration": 60,
  "video_quality": 1
}

Response:
{
  "code": 0,
  "message": "success",
  "data": {
    "status": "accepted"
  }
}
```

#### 异步状态回调
Handler通过回调接口向Agent报告任务状态变化:

```http
POST http://agent:5000/api/task/status
Content-Type: application/json

{
  "episode_id": "ep_123456",
  "handler_name": "collect-handler",
  "status": "collecting",  // collecting, completed, failed
  "progress": 50,
  "message": "采集进行中"
}
```

## 快速开始

### 环境要求

- **操作系统**: Linux (Ubuntu 20.04+) / Windows 10+
- **依赖**:
  - ROS2 (Humble/Foxy) - 用于ROS Topic采集
  - Python 3.8+ - 部分Handler实现
  - C++ 17+ - 部分Handler实现
  - Nginx - 前端服务
  - SQLite3 - HMI数据存储

### 安装步骤

1. **获取hmi-installer安装包**

联系软通天擎技术支持工程师获取安装包

2. **解压并启动安装脚本**
```bash
# 解压并进入安装目录
tar -zxf hmi-installer-[version].tar.gz && cd hmi-component
# 安装脚本赋权
chmod +x installer.sh
# 启动安装
bash installer.sh
```

3. **访问界面**
```
浏览器打开: http://localhost
```

### 验证部署

1. 访问前端界面,检查能力列表是否正常显示
2. 查看Agent日志,确认Handler注册成功
3. 执行一次测试采集,验证完整流程

## 自定义Handler开发

### Handler开发规范

1. **实现标准接口**: 按照接口规范实现必需接口
2. **能力注册**: 启动后向Agent注册能力列表
3. **异步回调**: 长时任务通过回调接口报告状态
4. **健康检测**: 响应Agent的健康检测请求
5. **进程管理**: 支持优雅启动和关闭

### Handler模板

```python
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)
AGENT_URL = "http://localhost:5000"

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "healthy"})

@app.route('/collect/start', methods=['POST'])
def collect_start():
    data = request.json
    # 启动采集逻辑
    # ...

    # 异步回调Agent
    requests.post(f"{AGENT_URL}/api/task/status", json={
        "episode_id": data['episode_id'],
        "handler_name": "my-handler",
        "status": "collecting"
    })

    return jsonify({"code": 0, "message": "success"})

if __name__ == '__main__':
    # 注册能力
    requests.post(f"{AGENT_URL}/api/handler/register", json={
        "handler_name": "my-handler",
        "port": 5010,
        "base_url": "/api",
        "modules": ["collect"]
    })

    app.run(port=5010)
```

## 常见问题

### Q: Handler启动失败怎么办?
A: 检查以下几点:
1. 端口是否被占用
2. 配置文件路径是否正确
3. 依赖库是否安装完整
4. 查看Agent日志获取详细错误信息

### Q: 如何适配新型号机器人?
A: 只需实现对应的Handler即可:
1. 根据机器人SDK实现数据采集逻辑
2. 按照接口规范封装HTTP接口
3. 将Handler可执行文件放入`handler/`目录
4. Agent会自动拉起并注册

### Q: 支持哪些视频质量?
A: 系统支持4种质量等级:
1. 无损 (lossless)
2. 超清 (ultra_clear)
3. 高清 (high_definition)
4. 流畅 (smooth)

### Q: 数据上传失败如何处理?
A: 系统具备断点续传能力:
1. 上传失败的数据会保留在本地
2. Agent会自动重试上传
3. 可在HMI界面查看上传进度和状态

## 贡献指南

我们欢迎社区贡献!

### 贡献方式

1. **提交Issue**: 报告Bug或提出新功能建议
2. **提交PR**: 修复Bug或实现新功能
3. **完善文档**: 改进文档或添加示例
4. **分享Handler**: 贡献适配不同机器人的Handler实现

### 开发流程

1. Fork本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交Pull Request

## 许可证

本项目采用 GPL v3.0 许可证 - 详见 [LICENSE](LICENSE) 文件

## 联系方式

- **项目主页**: https://github.com/tqartisan/hmi-index
- **问题反馈**: https://github.com/tqartisan/hmi-index/issues
- **邮件**: support@tqartisan.com

## 致谢

感谢所有为本项目做出贡献的开发者！

---

**注意**: 本项目处于开发中，API可能会有变动。建议关注Release Notes获取最新更新。
