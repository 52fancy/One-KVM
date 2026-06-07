# One-KVM 添加 frp (frpc) 内网穿透扩展 — 完整代码变更说明

## 概述

为 One-KVM 项目添加 frp 内网穿透客户端 (frpc) 作为新的 Extension，遵循现有 GOSTC / EasyTier 的实现模式，使用命令行参数（不使用配置文件），适配 frp v0.52+ 的 Cobra CLI 新格式。

---

## 一、Rust 后端修改（6 个文件）

### 1. `src/extensions/types.rs` — 类型定义

**新增枚举 FrpProxyType（7 种代理类型）：**

```rust
pub enum FrpProxyType {
    Tcp,    // TCP 代理
    Udp,    // UDP 代理
    Http,   // HTTP 代理（支持自定义域名）
    Https,  // HTTPS 代理（支持自定义域名）
    Stcp,   // Secret TCP（加密 TCP）
    Sudp,   // Secret UDP（加密 UDP）
    Xtcp,   // P2P 直连模式
}
```

**新增结构体 FrpcConfig：**

```rust
pub struct FrpcConfig {
    pub enabled: bool,               // 是否启用
    pub proxy_name: String,          // 代理名称（-n 参数，必填，用户自定义）
    pub proxy_type: FrpProxyType,    // 代理类型（子命令）
    pub server_addr: String,         // frps 服务器地址（-s 参数）
    pub server_port: u16,            // frps 端口（-P 参数，默认 7000）
    pub token: String,               // 认证令牌（-t 参数）
    pub local_ip: String,            // 本地转发 IP（-i 参数，默认 127.0.0.1）
    pub local_port: u16,             // 本地端口（-l 参数，默认 8080）
    pub remote_port: Option<u16>,    // 远程端口（-r 参数，可选，仅 TCP/UDP/STCP/SUDP/XTCP）
    pub custom_domain: Option<String>, // 自定义域名（--domain 参数，可选，仅 HTTP/HTTPS）
    pub tls: bool,                   // TLS 开关（--tls-enable 参数）
}
```

**新增结构体 FrpcInfo：**

```rust
pub struct FrpcInfo {
    pub available: bool,
    pub status: ExtensionStatus,
    pub config: FrpcConfig,
}
```

**新增结构体 FrpcConfigUpdate（Partial Update，支持三种状态）：**

```rust
pub struct FrpcConfigUpdate {
    pub enabled: Option<bool>,
    pub proxy_name: Option<String>,
    pub proxy_type: Option<FrpProxyType>,
    pub server_addr: Option<String>,
    pub server_port: Option<u16>,
    pub token: Option<String>,
    pub local_ip: Option<String>,
    pub local_port: Option<u16>,
    pub remote_port: Option<Option<u16>>,      // Option<Option>：不修改/设置值/清空
    pub custom_domain: Option<Option<String>>, // Option<Option>：不修改/设置值/清空
    pub tls: Option<bool>,
}
```

> 注意：`remote_port` 和 `custom_domain` 使用 `Option<Option<T>>`，外层 None 表示不修改该字段，内层 None 表示清空该值。

**修改已有类型：**

| 类型 | 修改内容 |
|------|---------|
| `ExtensionId` | 新增 `Frpc` 变体（Display: `"frpc"`，FromStr: `"frpc"`） |
| `ExtensionId::all()` | 返回数组新增 `Self::Frpc` |
| `ExtensionsConfig` | 新增 `pub frpc: FrpcConfig` 字段 |
| `ExtensionsStatus` | 新增 `pub frpc: FrpcInfo` 字段 |

### 2. `src/extensions/manager.rs` — 进程管理

**修改位置：is_enabled_for_config、build_args、redact_args_for_log**

is_enabled_for_config 新增 Frpc 分支：
```
enabled && server_addr 非空 && token 非空
```

build_args 新增 Frpc 分支（完整参数构建逻辑）：

| Config 字段 | CLI 参数 | 条件 |
|------------|---------|------|
| proxy_type | 子命令位置参数 | 必填 |
| proxy_name | -n | 必填 |
| server_addr | -s | 必填 |
| server_port | -P | 必填 |
| token | -t | 必填（日志脱敏） |
| local_ip | -i | 必填 |
| local_port | -l | 必填 |
| remote_port | -r | 仅 TCP/UDP/STCP/SUDP/XTCP |
| custom_domain | --domain | 仅 HTTP/HTTPS |
| tls | --tls-enable | flag 型 |

redact_args_for_log 增脱敏：新增对 `-t` / `--token` 参数的 `****` 脱敏处理

### 3. `src/extensions/software_linux.rs` — Linux 二进制路径

```
ExtensionId::Frpc => "/usr/bin/frpc"
```

### 4. `src/extensions/software_windows.rs` — Windows 二进制路径

```
ExtensionId::Frpc => "frpc.exe"
```

### 5. `src/web/handlers/extensions.rs` — API 处理器

新增：
- `validate_frpc_enabled()` — 校验 proxy_name / server_addr / token 必填
- `FrpcConfigUpdate` 结构体（请求体）
- `update_frpc_config()` — Partial Update 合并 + 校验 + 自动启停
- `list_extensions()` 新增 frpc 信息返回

### 6. `src/web/routes.rs` — API 路由

```
PATCH /api/extensions/frpc/config → update_frpc_config
```

---

## 二、前端 Web UI 修改（5 个文件）

### 1. `web/src/types/generated.ts` — TypeScript 类型定义

新增 `FrpProxyType` 枚举、`FrpcConfig`、`FrpcInfo`、`FrpcConfigUpdate` 接口。

修改 `ExtensionsConfig`、`ExtensionsStatus`、`ExtensionId`。

### 2. `web/src/api/config.ts` — API 调用

extensionsApi 新增 `updateFrpc(config: FrpcConfigUpdate)` 方法。

### 3. `web/src/views/SettingsView.vue` — 设置页面

共 15 处修改：

1. 导入 `FrpProxyType`（值导入，非 type import）
2. 不新增独立导航入口（FRPC 放在"远程访问"页面内）
3. `sectionSubtitleKey` / `loadSectionData` 添加 ext-frpc case
4. `extensionLogs` / `showLogs` / `extConfig` 添加 frpc 数据
5. `frpcValidationMessage` 计算属性（校验 proxy_name、server_addr、token）
6. `showFrpcRemotePort` / `showFrpcCustomDomain` 条件显示计算属性
7. `loadExtensions` 添加 frpc 加载
8. 所有 extension 操作函数类型签名扩展为包含 'frpc'
9. `validateExtensionConfig` 添加 frpc 分支
10. FRPC UI 卡片（放在 ext-remote-access 页内，GOSTC/EasyTier 之后）：
    - 状态指示器 + 启动/停止按钮
    - 自动启动开关
    - 代理类型 RadioGroup（7 种类型）
    - 代理名称输入框（必填，含校验提示）
    - 服务器地址 + 端口
    - Token 密码输入框（必填，含校验提示）
    - 本地 IP + 端口
    - 远程端口（条件显示：TCP/UDP/STCP/SUDP/XTCP）
    - 自定义域名（条件显示：HTTP/HTTPS）
    - TLS 开关
    - 日志查看（可折叠，支持刷新）
11. 保存按钮

### 4-5. `web/src/i18n/en-US.ts` 和 `zh-CN.ts` — 国际化

新增约 20 条翻译 key：

| Key | en-US | zh-CN |
|-----|-------|-------|
| extensions.frpc.title | FRP Client | FRP 客户端 |
| extensions.frpc.desc | NAT penetration via FRP (frpc) | 通过 FRP (frpc) 实现内网穿透 |
| extensions.frpc.proxyType | Proxy Type | 代理类型 |
| extensions.frpc.proxyName | Proxy Name | 代理名称 |
| extensions.frpc.proxyNamePlaceholder | my-proxy | my-proxy |
| extensions.frpc.proxyNameRequired | Enter a proxy name | 请填写代理名称 |
| extensions.frpc.serverAddr | Server Address | 服务器地址 |
| extensions.frpc.serverAddrPlaceholder | frps.example.com | frps.example.com |
| extensions.frpc.serverAddrRequired | Enter the FRP server address | 请填写 FRP 服务器地址 |
| extensions.frpc.serverPort | Server Port | 服务器端口 |
| extensions.frpc.token | Token | Token 密钥 |
| extensions.frpc.tokenRequired | Enter the FRP authentication token | 请填写 FRP 认证 Token |
| extensions.frpc.localIp | Local IP | 本地 IP |
| extensions.frpc.localPort | Local Port | 本地端口 |
| extensions.frpc.remotePort | Remote Port | 远程端口 |
| extensions.frpc.remotePortHint | Optional. Random if not specified | 可选，不填则随机分配 |
| extensions.frpc.customDomain | Custom Domain | 自定义域名 |
| extensions.frpc.customDomainPlaceholder | example.com | example.com |
| extensions.frpc.tls | Enable TLS | 启用 TLS |
| extFrpcSubtitle | NAT penetration via FRP client | 通过 FRP 客户端实现内网穿透 |

---

## 三、frpc 命令行参数映射（frp v0.52+ Cobra CLI）

```bash
frpc <proxy_type> [flags]
```

| FrpcConfig 字段 | CLI 参数 | 类型 | 说明 | 适用代理类型 |
|---|---|---|---|---|
| proxy_type | 子命令 | — | tcp/udp/http/https/stcp/sudp/xtcp | 全部 |
| proxy_name | -n / --proxy-name | String | 代理名称，必填，用户自定义 | 全部 |
| server_addr | -s / --server-addr | String | frps 地址，默认 127.0.0.1 | 全部 |
| server_port | -P / --server-port | Int | frps 端口，默认 7000 | 全部 |
| token | -t / --token | String | 认证令牌，日志脱敏 | 全部 |
| local_ip | -i / --local-ip | String | 本地 IP，默认 127.0.0.1 | 全部 |
| local_port | -l / --local-port | Int | 本地端口 | 全部 |
| remote_port | -r / --remote-port | Int | 远程端口，可选 | TCP/UDP/STCP/SUDP/XTCP |
| custom_domain | --domain | String | 自定义域名，可选 | HTTP/HTTPS |
| tls | --tls-enable | Flag | 启用 TLS，默认 true | 全部 |

### 命令示例

```bash
# TCP 代理（指定远程端口）
frpc tcp -n my-kvm -s frps.example.com -P 7000 -t xxx \
    -i 192.168.1.100 -l 22 -r 6000 --tls-enable

# HTTP 代理（自定义域名）
frpc http -n my-web -s frps.example.com -P 7000 -t xxx \
    -i 127.0.0.1 -l 8080 --domain example.com

# 不指定远程端口，由 frps 自动分配
frpc tcp -n my-ssh -s frps.example.com -P 7000 -t xxx \
    -i 127.0.0.1 -l 22
```

---

## 四、文件清单

```
One-KVM-frpc/
├── README.md                       # 本文件（完整变更说明）
├── One-KVM-frpc-changes.diff       # Git diff 补丁
├── src/
│   ├── extensions/
│   │   ├── types.rs                # 修改：新增 Frpc 类型定义
│   │   ├── manager.rs              # 修改：新增 Frpc 进程管理
│   │   ├── software_linux.rs       # 修改：新增 frpc 路径
│   │   └── software_windows.rs     # 修改：新增 frpc.exe 路径
│   └── web/
│       ├── routes.rs               # 修改：新增 frpc/config 路由
│       └── handlers/
│           └── extensions.rs       # 修改：新增 Frpc handler
└── web/
    └── src/
        ├── api/
        │   └── config.ts           # 修改：新增 updateFrpc API
        ├── i18n/
        │   ├── en-US.ts            # 修改：新增英文翻译
        │   └── zh-CN.ts            # 修改：新增中文翻译
        ├── types/
        │   └── generated.ts        # 修改：新增 Frpc TypeScript 类型
        └── views/
            └── SettingsView.vue    # 修改：新增 FRPC UI
```

---

## 五、设计决策

| # | 决策 | 说明 |
|---|------|------|
| 1 | 纯命令行参数 | 不生成 frpc.toml 配置文件，与 GOSTC/EasyTier 保持一致 |
| 2 | frp v0.52+ Cobra CLI | 子命令模式（`frpc tcp` 而非 `frpc -t tcp`） |
| 3 | 条件字段显示 | remote_port 仅 TCP/UDP/STCP/SUDP/XTCP 显示；custom_domain 仅 HTTP/HTTPS 显示 |
| 4 | 安全性 | Token 日志脱敏 + 前端密码输入框 |
| 5 | proxy_name 用户自定义 | 不由系统生成固定值，用户自行填写 |
| 6 | Partial Update | remote_port/custom_domain 用 Option<Option<T>> 支持不修改/设置/清空三种状态 |
| 7 | 向后兼容 | 不影响原有 Ttyd/Gostc/Easytier |
| 8 | UI 位置 | FRPC 放在「远程访问」页面，与 GOSTC、EasyTier 并列 |

---

## 六、部署

1. 部署 frpc 二进制：
   - Linux: `/usr/bin/frpc`
   - Windows: `frpc.exe`（与 one-kvm.exe 同目录）
2. 复制修改文件到项目对应目录
3. 重新编译：`cd web && npm ci && npm run build && cd .. && cargo build --release`

## 七、API 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/extensions | 获取所有扩展状态（含 frpc） |
| GET | /api/extensions/frpc | 获取 frpc 状态 |
| POST | /api/extensions/frpc/start | 启动 frpc |
| POST | /api/extensions/frpc/stop | 停止 frpc |
| GET | /api/extensions/frpc/logs | 获取日志 |
| PATCH | /api/extensions/frpc/config | 更新配置 |

## 八、相关资源

- [frp GitHub](https://github.com/fatedier/frp)
- [frp 文档](https://gofrp.org/)
- [One-KVM 项目](https://github.com/mofeng-git/One-KVM)
