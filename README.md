# VoHive

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

VoHive 是一个面向高通 4G/LTE/5G 模组场景（如 Quectel EC20/EC25/EC21/EG25/EM20 高通 410 等）的高级综合管理与代理服务平台。它将复杂的底层通信与设备管理抽象为统一的服务，包含设备热插拔管理、代理实例编排、短信与 VoWiFi/IMS 能力、eSIM 管理，并提供现代化的响应式 Web 管理界面。

VoHive 的核心目标是解决 4G 模组在真实生产环境里最难落地的痛点，打破单脚本拼接的局限，提供开箱即用的工业级多模组一体化管理方案。

## 🌟 核心特性

- **多模组并发管理**：支持 USB 热插拔、自动发现设备（ttyUSB 等）、多设备实时状态管理与监控。
- **轻量级代理引擎**：内建 SOCKS5 / HTTP 代理内核，支持多实例并发。按设备网卡严格绑定出站流量（基于 `SO_BINDTODEVICE`），确保流量精确走指定模组物理链路。
- **通信与短信中心**：通过统一界面及 API 处理 AT 短信收发、会话与联系人管理、USSD 指令交互。内置短信数据落库，再也不用担心漏看验证码。
- **VoWiFi / IMS 接入**：完整支持 ePDG/IPSec、IMS 注册、SMS over IP，内置语音网关（Voice Gateway），可通过标准 SIP 协议对接 Linphone 等软电话，在零蜂窝信号下使用 Wi-Fi 实现正常电话及短信收发。
- **eSIM支持**：深度集成 eSIM 能力，直接通过 AT 指令通道管理 eSIM 芯片。支持 Profile 下载、启用/停用、重命名及删除等全生命周期操作。
- **全渠道消息通知**：重要短信及系统告警可极速推送至 Telegram、Email、PushPlus、Bark、飞书(Lark/Feishu)、QQ 等主流平台。
- **多架构构建**：原生支持跨平台编译（amd64, arm64, arm7），从路由器到边缘计算节点均可无缝部署。

## 🧩 典型应用场景

- **私有 IP 代理池**：单主机挂载多张物理 SIM 卡或切换多张 eSIM，为每张网卡分配独立的 SOCKS5/HTTP 实例，组建自己的移动网络代理池。
- **统一短信与验证码接码中心**：通过 Web 界面或 API 并行接收和管理多卡的短信，并通过 Webhook/Bot 实时推送至个人终端。
- **VoWiFi 零信号通信**：针对处于地下室、无蜂窝信号覆盖的设备，利用宽带网络隧道建立 IMS 连接，保障业务永不掉线。
- **无人值守自动化运维**：结合内置 API 和通知机制实现自动化网络拨测、自动 USSD 查费或流量重载。

## 🏗 架构与技术栈

- **Backend**: Go `1.26+` (Gin, GORM, Viper, sipgo, euicc-go)
- **Frontend**: Vue 3 + Vite + TailwindCSS + Element Plus
- **Database**: SQLite (`vohive.db`)
- **部署发布**: GitHub Actions 自动化多架构 Docker 镜像分发。

关键目录：
- `cmd/vohive`: 主服务入口
- `internal/api`: REST API 控制平面
- `internal/device`: 设备发现、拨号与状态生命周期
- `internal/proxy`: 代理实例管理与流量统计监控
- `internal/vowifi`: ePDG/IMS 底层协议栈与软电话网关
- `internal/notify`: 统一告警与消息推送机制
- `web`: 管理后台 Web 源码

## 🚀 快速开始

### 1) 源码本地构建

建议使用 Go 1.26 及以上版本，Node.js 18+。

```bash
# 构建前端静态文件
cd web
npm ci
npm run build
cd ..

# 拷贝静态文件用于嵌入
mkdir -p internal/web/dist
cp -r web/dist/* internal/web/dist/

# 编译后端二进制 (禁用 CGO 实现完全静态编译)
CGO_ENABLED=0 go build -trimpath -buildvcs=false -tags "with_utls nomsgpack" -ldflags "-s -w" -o vohive ./cmd/vohive
```

### 2) 运行服务

```bash
# 使用默认配置运行 (读取 config/config.yaml)
./vohive

# 指定配置文件运行
./vohive -c /path/to/custom_config.yaml
```

- 默认管理后台：`http://127.0.0.1:7575`
- 默认账密：`admin / admin` (可在配置中修改)

### 3) 容器化部署 (Docker)

为使代理流量精确路由及读取底层硬件串口，Docker 部署强烈建议使用 `host` 网络及特权模式。

```bash
docker run -d \
  --name vohive \
  --network host \
  --privileged \
  -v /dev:/dev \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/data:/app/data \
  vohive:latest
```
*(提示：亦可通过 Docker Compose 部署，详情参考仓库内的 docker-compose.yml)*

## ⚙️ 配置文件示例

VoHive 采用 YAML 作为配置文件，运行后会自动读取/生成 `config/config.yaml`。环境变量前缀为 `PROXY_`（例如 `PROXY_WEB_USERNAME=admin` 可覆盖 YAML 配置）。

```yaml
server:
  port: ":7575"
  debug: false

web:
  username: "admin"
  password: "admin"

proxy:
  instances:
    - id: "proxy-socks-1"
      name: "SOCKS5-Dev1"
      device_id: "ec20_1"
      enabled: true
      mode: "socks5"
      listen_addr: "0.0.0.0"
      listen_port: 10800
      auth_enabled: false

notifications:
  telegram:
    enabled: true
    bot_token: "xxx"
    chat_id: 123456
  email:
    enabled: true
    smtp_server: "smtp.example.com"
  # ... 等等
```

## 🔌 API 接入指引

Web 管理平台的所有操作均可通过 REST API 实现。接口调用需传递 Header: `Authorization: Bearer <token>`。

常用端点：
- `GET /api/proxy/overview` - 代理总览与实时流量统计
- `GET /api/devices` - 已连接模组列表及运行状态
- `GET /api/devices/:id/esim/profiles` - 查询特定设备的 eSIM 卡配置
- `POST /api/sms/send` - 指示设备发送短信
- `POST /api/vowifi/call` - 发起 VoWiFi 语音呼叫

## 📄 License

本项目采用 [MIT License](https://opensource.org/licenses/MIT) 开源协议。
