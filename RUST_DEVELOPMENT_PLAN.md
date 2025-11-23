# SockTail Rust 版本详细开发计划
# Detailed Development Plan for SockTail Rust Implementation

**项目名称**: SockTail-RS
**目标**: 用 Rust 重新实现 SOCKS5-over-Tailscale 代理工具
**预计总时长**: 2-4 周（按每天 6-8 小时计算）
**开发模式**: 敏捷迭代，每个阶段都有可运行的版本

---

## 📋 项目概览

### 核心目标

1. ✅ 实现功能完整的 SOCKS5 代理服务器
2. ✅ 集成 Tailscale VPN 网络
3. ✅ 与 Go 版本功能对等
4. ✅ 性能优于 Go 版本
5. ✅ 二进制大小控制在 3MB 以内
6. ✅ 跨平台支持（Linux/macOS/Windows）

### 成功标准

- [ ] 通过所有功能测试
- [ ] 性能基准测试达标（> 800 Mbps）
- [ ] 内存占用 < 10 MB
- [ ] 二进制大小 < 3 MB
- [ ] 代码覆盖率 > 80%
- [ ] 无内存泄漏
- [ ] 通过 clippy 和 cargo audit 检查

---

## 🗓️ 开发阶段规划

### 阶段 0: 环境准备（第 1 天，4-6 小时）

#### 任务清单

**0.1 开发环境搭建** (1 小时)
- [ ] 安装 Rust 工具链（rustup）
- [ ] 配置 IDE（VS Code + rust-analyzer 或 CLion）
- [ ] 安装必要工具
  ```bash
  cargo install cargo-watch    # 自动重新编译
  cargo install cargo-expand   # 宏展开
  cargo install cargo-audit    # 安全审计
  cargo install cargo-bloat    # 分析二进制大小
  cargo install flamegraph     # 性能分析
  ```

**0.2 项目初始化** (1 小时)
- [ ] 创建 Git 仓库
  ```bash
  cargo new socktail-rs --bin
  cd socktail-rs
  git init
  ```
- [ ] 设置 `.gitignore`
- [ ] 创建项目结构
  ```
  socktail-rs/
  ├── Cargo.toml
  ├── .cargo/
  │   └── config.toml          # 构建配置
  ├── src/
  │   ├── main.rs
  │   ├── lib.rs
  │   ├── socks5/
  │   │   ├── mod.rs
  │   │   ├── protocol.rs
  │   │   ├── server.rs
  │   │   └── relay.rs
  │   ├── vpn/
  │   │   ├── mod.rs
  │   │   └── tailscale.rs
  │   ├── crypto/
  │   │   └── xor.rs
  │   └── utils/
  │       ├── mod.rs
  │       ├── hostname.rs
  │       └── config.rs
  ├── tests/
  │   ├── integration_tests.rs
  │   └── protocol_tests.rs
  ├── benches/
  │   └── throughput.rs
  ├── examples/
  │   └── simple_proxy.rs
  └── README.md
  ```

**0.3 依赖配置** (1 小时)
- [ ] 编写 `Cargo.toml`
  ```toml
  [package]
  name = "socktail"
  version = "0.1.0"
  edition = "2021"
  authors = ["Your Name <email@example.com>"]
  license = "MIT"
  description = "SOCKS5 proxy over Tailscale VPN"
  repository = "https://github.com/yourusername/socktail-rs"

  [dependencies]
  tokio = { version = "1.35", features = ["full"] }
  tokio-util = { version = "0.7", features = ["codec"] }
  bytes = "1.5"
  anyhow = "1.0"
  thiserror = "1.0"
  tracing = "0.1"
  tracing-subscriber = { version = "0.3", features = ["env-filter"] }
  clap = { version = "4.4", features = ["derive", "env"] }
  hex = "0.4"
  rand = "0.8"
  serde = { version = "1.0", features = ["derive"] }
  serde_json = "1.0"

  [dev-dependencies]
  criterion = { version = "0.5", features = ["html_reports"] }
  tokio-test = "0.4"
  tempfile = "3.8"

  [profile.dev]
  opt-level = 0
  debug = true

  [profile.release]
  opt-level = 3
  lto = true
  codegen-units = 1
  strip = true
  panic = "abort"

  [profile.release-small]
  inherits = "release"
  opt-level = "z"
  lto = true
  codegen-units = 1
  strip = true

  [[bench]]
  name = "throughput"
  harness = false
  ```

**0.4 CI/CD 配置** (1-2 小时)
- [ ] GitHub Actions workflow
  ```yaml
  # .github/workflows/ci.yml
  name: CI

  on: [push, pull_request]

  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - uses: dtolnay/rust-toolchain@stable
        - run: cargo test --all-features
        - run: cargo clippy -- -D warnings
        - run: cargo fmt --check

    build:
      strategy:
        matrix:
          os: [ubuntu-latest, macos-latest, windows-latest]
      runs-on: ${{ matrix.os }}
      steps:
        - uses: actions/checkout@v3
        - uses: dtolnay/rust-toolchain@stable
        - run: cargo build --release
  ```

**0.5 文档框架** (1 小时)
- [ ] 编写 README.md
- [ ] 添加 LICENSE
- [ ] 创建 CHANGELOG.md
- [ ] 设置 docs.rs 配置

**验收标准**:
- ✅ `cargo build` 成功
- ✅ `cargo test` 通过（即使测试为空）
- ✅ CI/CD 流水线正常运行
- ✅ 项目结构清晰

---

### 阶段 1: 基础 SOCKS5 实现（第 2-4 天，18-24 小时）

**目标**: 实现基本的 SOCKS5 代理功能（无 VPN）

#### 1.1 协议定义和错误处理 (4-6 小时)

**任务 1.1.1: 定义错误类型** (1 小时)
```rust
// src/socks5/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Socks5Error {
    #[error("Unsupported SOCKS version: {0}")]
    UnsupportedVersion(u8),

    #[error("Unsupported command: {0}")]
    UnsupportedCommand(u8),

    #[error("Unsupported address type: {0}")]
    UnsupportedAddressType(u8),

    #[error("Authentication failed")]
    AuthFailed,

    #[error("Connection refused")]
    ConnectionRefused,

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Invalid protocol data")]
    InvalidData,
}

pub type Result<T> = std::result::Result<T, Socks5Error>;
```

**任务 1.1.2: 定义协议常量** (30 分钟)
```rust
// src/socks5/protocol.rs
pub const SOCKS5_VERSION: u8 = 0x05;

// 认证方法
pub const AUTH_NO_AUTH: u8 = 0x00;
pub const AUTH_GSSAPI: u8 = 0x01;
pub const AUTH_USERNAME_PASSWORD: u8 = 0x02;
pub const AUTH_NO_ACCEPTABLE: u8 = 0xFF;

// 命令
pub const CMD_CONNECT: u8 = 0x01;
pub const CMD_BIND: u8 = 0x02;
pub const CMD_UDP_ASSOCIATE: u8 = 0x03;

// 地址类型
pub const ATYP_IPV4: u8 = 0x01;
pub const ATYP_DOMAIN: u8 = 0x03;
pub const ATYP_IPV6: u8 = 0x04;

// 响应状态
pub const REP_SUCCESS: u8 = 0x00;
pub const REP_GENERAL_FAILURE: u8 = 0x01;
pub const REP_CONNECTION_NOT_ALLOWED: u8 = 0x02;
pub const REP_NETWORK_UNREACHABLE: u8 = 0x03;
pub const REP_HOST_UNREACHABLE: u8 = 0x04;
pub const REP_CONNECTION_REFUSED: u8 = 0x05;
pub const REP_TTL_EXPIRED: u8 = 0x06;
pub const REP_COMMAND_NOT_SUPPORTED: u8 = 0x07;
pub const REP_ADDRESS_TYPE_NOT_SUPPORTED: u8 = 0x08;
```

**任务 1.1.3: 定义目标地址类型** (1 小时)
```rust
// src/socks5/protocol.rs
use std::net::{IpAddr, Ipv4Addr, Ipv6Addr, SocketAddr};

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TargetAddr {
    Ip(SocketAddr),
    Domain(String, u16),
}

impl TargetAddr {
    pub fn to_string(&self) -> String {
        match self {
            TargetAddr::Ip(addr) => addr.to_string(),
            TargetAddr::Domain(domain, port) => format!("{}:{}", domain, port),
        }
    }
}

impl std::fmt::Display for TargetAddr {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.to_string())
    }
}
```

**任务 1.1.4: 实现认证请求解析** (2 小时)
```rust
// src/socks5/protocol.rs
use bytes::{Buf, BufMut, BytesMut};

pub struct AuthRequest {
    pub version: u8,
    pub methods: Vec<u8>,
}

impl AuthRequest {
    pub fn parse(buf: &mut BytesMut) -> Result<Self> {
        if buf.len() < 2 {
            return Err(Socks5Error::InvalidData);
        }

        let version = buf.get_u8();
        let n_methods = buf.get_u8() as usize;

        if buf.len() < n_methods {
            return Err(Socks5Error::InvalidData);
        }

        let methods = buf.split_to(n_methods).to_vec();

        Ok(AuthRequest { version, methods })
    }

    pub fn supports_method(&self, method: u8) -> bool {
        self.methods.contains(&method)
    }
}

pub fn auth_response(method: u8) -> [u8; 2] {
    [SOCKS5_VERSION, method]
}
```

**任务 1.1.5: 实现 CONNECT 请求解析** (2-3 小时)
```rust
pub struct ConnectRequest {
    pub version: u8,
    pub command: u8,
    pub target: TargetAddr,
}

impl ConnectRequest {
    pub fn parse(buf: &mut BytesMut) -> Result<Self> {
        if buf.len() < 4 {
            return Err(Socks5Error::InvalidData);
        }

        let version = buf.get_u8();
        let command = buf.get_u8();
        let _reserved = buf.get_u8();
        let atyp = buf.get_u8();

        let target = match atyp {
            ATYP_IPV4 => {
                if buf.len() < 6 {
                    return Err(Socks5Error::InvalidData);
                }
                let ip = Ipv4Addr::new(
                    buf.get_u8(),
                    buf.get_u8(),
                    buf.get_u8(),
                    buf.get_u8(),
                );
                let port = buf.get_u16();
                TargetAddr::Ip(SocketAddr::new(IpAddr::V4(ip), port))
            }
            ATYP_DOMAIN => {
                if buf.is_empty() {
                    return Err(Socks5Error::InvalidData);
                }
                let len = buf.get_u8() as usize;
                if buf.len() < len + 2 {
                    return Err(Socks5Error::InvalidData);
                }
                let domain = String::from_utf8_lossy(&buf.split_to(len)).to_string();
                let port = buf.get_u16();
                TargetAddr::Domain(domain, port)
            }
            ATYP_IPV6 => {
                if buf.len() < 18 {
                    return Err(Socks5Error::InvalidData);
                }
                let mut octets = [0u8; 16];
                buf.copy_to_slice(&mut octets);
                let ip = Ipv6Addr::from(octets);
                let port = buf.get_u16();
                TargetAddr::Ip(SocketAddr::new(IpAddr::V6(ip), port))
            }
            _ => return Err(Socks5Error::UnsupportedAddressType(atyp)),
        };

        Ok(ConnectRequest {
            version,
            command,
            target,
        })
    }
}

pub fn connect_response(status: u8) -> Vec<u8> {
    vec![
        SOCKS5_VERSION,
        status,
        0x00,        // Reserved
        ATYP_IPV4,   // Address type
        0, 0, 0, 0,  // Bind address (0.0.0.0)
        0, 0,        // Bind port (0)
    ]
}
```

**测试用例** (30 分钟):
```rust
// tests/protocol_tests.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_auth_request_parsing() {
        let mut buf = BytesMut::from(&[0x05, 0x02, 0x00, 0x02][..]);
        let auth = AuthRequest::parse(&mut buf).unwrap();

        assert_eq!(auth.version, 0x05);
        assert_eq!(auth.methods, vec![0x00, 0x02]);
        assert!(auth.supports_method(AUTH_NO_AUTH));
    }

    #[test]
    fn test_connect_ipv4() {
        let mut buf = BytesMut::from(&[
            0x05, 0x01, 0x00, 0x01,  // Version, CMD, RSV, ATYP
            0x7F, 0x00, 0x00, 0x01,  // 127.0.0.1
            0x00, 0x50,              // Port 80
        ][..]);

        let req = ConnectRequest::parse(&mut buf).unwrap();
        assert_eq!(req.command, CMD_CONNECT);

        if let TargetAddr::Ip(addr) = req.target {
            assert_eq!(addr.port(), 80);
        } else {
            panic!("Expected IP address");
        }
    }

    #[test]
    fn test_connect_domain() {
        let mut buf = BytesMut::from(&[
            0x05, 0x01, 0x00, 0x03,  // Version, CMD, RSV, ATYP
            0x0B,                     // Domain length (11)
            b'e', b'x', b'a', b'm', b'p', b'l', b'e', b'.', b'c', b'o', b'm',
            0x01, 0xBB,              // Port 443
        ][..]);

        let req = ConnectRequest::parse(&mut buf).unwrap();

        if let TargetAddr::Domain(domain, port) = req.target {
            assert_eq!(domain, "example.com");
            assert_eq!(port, 443);
        } else {
            panic!("Expected domain");
        }
    }
}
```

**验收标准**:
- ✅ 所有测试通过
- ✅ `cargo clippy` 无警告
- ✅ 支持 IPv4、IPv6、域名解析

---

#### 1.2 SOCKS5 服务器实现 (6-8 小时)

**任务 1.2.1: 服务器结构体** (1 小时)
```rust
// src/socks5/server.rs
use tokio::net::{TcpListener, TcpStream};
use tracing::{info, error, debug, warn};

pub struct Socks5Server {
    listen_addr: String,
}

impl Socks5Server {
    pub fn new(listen_addr: String) -> Self {
        Self { listen_addr }
    }

    pub async fn run(&self) -> anyhow::Result<()> {
        let listener = TcpListener::bind(&self.listen_addr).await?;
        info!("SOCKS5 server listening on {}", self.listen_addr);

        loop {
            match listener.accept().await {
                Ok((socket, peer_addr)) => {
                    debug!("New connection from {}", peer_addr);

                    tokio::spawn(async move {
                        if let Err(e) = handle_client(socket).await {
                            error!("Error handling client {}: {}", peer_addr, e);
                        }
                    });
                }
                Err(e) => {
                    error!("Failed to accept connection: {}", e);
                }
            }
        }
    }
}
```

**任务 1.2.2: 客户端连接处理** (3-4 小时)
```rust
// src/socks5/server.rs
use bytes::BytesMut;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

async fn handle_client(mut client: TcpStream) -> anyhow::Result<()> {
    // 1. 认证阶段
    let mut buf = BytesMut::with_capacity(512);

    // 读取认证请求
    if client.read_buf(&mut buf).await? == 0 {
        return Err(anyhow::anyhow!("Connection closed"));
    }

    let auth_req = AuthRequest::parse(&mut buf)?;

    if auth_req.version != SOCKS5_VERSION {
        return Err(Socks5Error::UnsupportedVersion(auth_req.version).into());
    }

    // 检查是否支持无认证
    if !auth_req.supports_method(AUTH_NO_AUTH) {
        client.write_all(&auth_response(AUTH_NO_ACCEPTABLE)).await?;
        return Err(Socks5Error::AuthFailed.into());
    }

    // 发送认证响应
    client.write_all(&auth_response(AUTH_NO_AUTH)).await?;
    debug!("Authentication successful");

    // 2. 请求阶段
    buf.clear();

    if client.read_buf(&mut buf).await? == 0 {
        return Err(anyhow::anyhow!("Connection closed after auth"));
    }

    let connect_req = ConnectRequest::parse(&mut buf)?;

    if connect_req.version != SOCKS5_VERSION {
        return Err(Socks5Error::UnsupportedVersion(connect_req.version).into());
    }

    if connect_req.command != CMD_CONNECT {
        client.write_all(&connect_response(REP_COMMAND_NOT_SUPPORTED)).await?;
        return Err(Socks5Error::UnsupportedCommand(connect_req.command).into());
    }

    let target_addr = connect_req.target.to_string();
    debug!("Connecting to target: {}", target_addr);

    // 3. 连接到目标
    match TcpStream::connect(&target_addr).await {
        Ok(target) => {
            debug!("Connected to {}", target_addr);
            client.write_all(&connect_response(REP_SUCCESS)).await?;

            // 4. 数据中继
            if let Err(e) = relay_data(client, target).await {
                warn!("Relay error: {}", e);
            }
        }
        Err(e) => {
            error!("Failed to connect to {}: {}", target_addr, e);
            client.write_all(&connect_response(REP_CONNECTION_REFUSED)).await?;
        }
    }

    Ok(())
}
```

**任务 1.2.3: 数据中继实现** (2-3 小时)
```rust
// src/socks5/relay.rs
use tokio::io::{self, AsyncRead, AsyncWrite};
use tokio::net::TcpStream;
use tracing::{debug, trace};

pub async fn relay_data(client: TcpStream, target: TcpStream) -> io::Result<()> {
    let (mut client_read, mut client_write) = io::split(client);
    let (mut target_read, mut target_write) = io::split(target);

    let client_to_target = async {
        let bytes = io::copy(&mut client_read, &mut target_write).await?;
        debug!("Client -> Target: {} bytes", bytes);
        target_write.shutdown().await?;
        Ok::<_, io::Error>(bytes)
    };

    let target_to_client = async {
        let bytes = io::copy(&mut target_read, &mut client_write).await?;
        debug!("Target -> Client: {} bytes", bytes);
        client_write.shutdown().await?;
        Ok::<_, io::Error>(bytes)
    };

    // 并发执行两个方向的数据传输
    match tokio::try_join!(client_to_target, target_to_client) {
        Ok((c2t, t2c)) => {
            debug!("Relay completed: {} bytes up, {} bytes down", c2t, t2c);
            Ok(())
        }
        Err(e) => {
            debug!("Relay error: {}", e);
            Err(e)
        }
    }
}
```

**验收标准**:
- ✅ 能够代理 HTTP/HTTPS 流量
- ✅ 支持多个并发连接
- ✅ 使用 curl 测试成功
  ```bash
  # 测试
  cargo run &
  curl --socks5 127.0.0.1:1080 http://example.com
  curl --socks5 127.0.0.1:1080 https://www.google.com
  ```

---

#### 1.3 日志和命令行参数 (3-4 小时)

**任务 1.3.1: 日志配置** (1-2 小时)
```rust
// src/main.rs
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

fn init_logging(verbose: bool) {
    let filter = if verbose {
        EnvFilter::new("debug")
    } else {
        EnvFilter::new("info")
    };

    tracing_subscriber::registry()
        .with(filter)
        .with(tracing_subscriber::fmt::layer())
        .init();
}
```

**任务 1.3.2: CLI 参数解析** (2 小时)
```rust
// src/main.rs
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "socktail")]
#[command(version = env!("CARGO_PKG_VERSION"))]
#[command(about = "SOCKS5 proxy over Tailscale", long_about = None)]
struct Args {
    /// SOCKS5 server listen address
    #[arg(short, long, default_value = "127.0.0.1:1080")]
    listen: String,

    /// Tailscale hostname
    #[arg(short = 'H', long)]
    hostname: Option<String>,

    /// Tailscale auth key
    #[arg(short, long, env = "TAILSCALE_AUTH_KEY")]
    authkey: Option<String>,

    /// Control server URL (for Headscale)
    #[arg(short, long)]
    control_url: Option<String>,

    /// Enable verbose logging
    #[arg(short, long)]
    verbose: bool,

    /// Skip Tailscale connection (dev mode)
    #[arg(long)]
    no_vpn: bool,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();
    init_logging(args.verbose);

    // 临时：只启动 SOCKS5 服务器
    let server = Socks5Server::new(args.listen);
    server.run().await?;

    Ok(())
}
```

**验收标准**:
- ✅ 日志输出清晰
- ✅ 支持 `-v` 详细日志
- ✅ 帮助信息完整 (`--help`)

---

### 阶段 2: VPN 集成（第 5-7 天，18-24 小时）

**目标**: 集成 Tailscale VPN 功能

#### 2.1 XOR 混淆实现 (2-3 小时)

**任务 2.1.1: XOR 编解码** (1 小时)
```rust
// src/crypto/xor.rs
const XOR_KEY: &[u8] = b"747sg^8N0$";

pub fn xor_encode(data: &[u8]) -> Vec<u8> {
    data.iter()
        .enumerate()
        .map(|(i, &byte)| byte ^ XOR_KEY[i % XOR_KEY.len()])
        .collect()
}

pub fn xor_decode(data: &[u8]) -> Vec<u8> {
    xor_encode(data) // XOR is symmetric
}

pub fn deobfuscate_authkey(obfuscated: &[u8]) -> String {
    let decoded = xor_decode(obfuscated);
    String::from_utf8_lossy(&decoded).to_string()
}
```

**任务 2.1.2: 内置密钥** (30 分钟)
```rust
// src/crypto/xor.rs
const EMBEDDED_OBFUSCATED_KEY: &[u8] = &[
    0x48, 0x65, 0x6c, 0x6c, 0x6f, 0x20, 0x74, 0x68, 0x65, 0x72,
    0x65, 0x21, 0x20, 0x47, 0x65, 0x6e, 0x65, 0x72, 0x61, 0x6c,
    0x20, 0x4b, 0x65, 0x6e, 0x6f, 0x62, 0x69, 0x2e,
];

pub fn get_default_authkey() -> String {
    deobfuscate_authkey(EMBEDDED_OBFUSCATED_KEY)
}
```

**任务 2.1.3: 测试兼容性** (30-60 分钟)
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_xor_symmetry() {
        let original = b"tskey-auth-test-1234567890";
        let encoded = xor_encode(original);
        let decoded = xor_decode(&encoded);
        assert_eq!(original, decoded.as_slice());
    }

    #[test]
    fn test_go_compatibility() {
        // 与 Go 版本生成的结果对比
        let result = deobfuscate_authkey(EMBEDDED_OBFUSCATED_KEY);
        assert_eq!(result, "Hello there! General Kenobi.");
    }

    #[test]
    fn test_hex_format() {
        let key = "test-key-123";
        let encoded = xor_encode(key.as_bytes());
        let hex = hex::encode(&encoded);

        // 应该能够从 hex 还原
        let decoded_bytes = hex::decode(&hex).unwrap();
        let decoded = xor_decode(&decoded_bytes);
        assert_eq!(key.as_bytes(), decoded.as_slice());
    }
}
```

---

#### 2.2 Tailscale 集成 (8-10 小时)

**任务 2.2.1: Tailscale 管理器** (3-4 小时)
```rust
// src/vpn/tailscale.rs
use std::process::{Command, Stdio};
use anyhow::{Context, Result};
use tracing::{info, error, debug};
use serde::Deserialize;

pub struct TailscaleManager {
    hostname: String,
    authkey: String,
    control_url: Option<String>,
    connected: bool,
}

impl TailscaleManager {
    pub fn new(
        hostname: String,
        authkey: String,
        control_url: Option<String>,
    ) -> Self {
        Self {
            hostname,
            authkey,
            control_url,
            connected: false,
        }
    }

    pub fn connect(&mut self) -> Result<()> {
        info!("Connecting to Tailscale network...");

        let mut cmd = Command::new("tailscale");
        cmd.args(&["up", "--authkey", &self.authkey, "--hostname", &self.hostname]);

        if let Some(url) = &self.control_url {
            cmd.args(&["--login-server", url]);
        }

        let output = cmd
            .stdout(Stdio::piped())
            .stderr(Stdio::piped())
            .output()
            .context("Failed to execute tailscale command")?;

        if output.status.success() {
            self.connected = true;
            info!("Successfully connected to Tailscale network");
            Ok(())
        } else {
            let stderr = String::from_utf8_lossy(&output.stderr);
            error!("Tailscale connection failed: {}", stderr);
            anyhow::bail!("Tailscale connection failed: {}", stderr)
        }
    }

    pub fn status(&self) -> Result<TailscaleStatus> {
        let output = Command::new("tailscale")
            .args(&["status", "--json"])
            .output()
            .context("Failed to get Tailscale status")?;

        if !output.status.success() {
            anyhow::bail!("Failed to get status");
        }

        let status: TailscaleStatus = serde_json::from_slice(&output.stdout)
            .context("Failed to parse status JSON")?;

        Ok(status)
    }

    pub fn disconnect(&mut self) -> Result<()> {
        if !self.connected {
            return Ok(());
        }

        info!("Disconnecting from Tailscale...");

        Command::new("tailscale")
            .arg("down")
            .output()
            .context("Failed to disconnect from Tailscale")?;

        self.connected = false;
        Ok(())
    }

    pub fn is_connected(&self) -> bool {
        self.connected
    }
}

#[derive(Debug, Deserialize)]
pub struct TailscaleStatus {
    #[serde(rename = "BackendState")]
    pub backend_state: String,

    #[serde(rename = "Self")]
    pub self_node: Option<TailscaleNode>,
}

#[derive(Debug, Deserialize)]
pub struct TailscaleNode {
    #[serde(rename = "HostName")]
    pub hostname: String,

    #[serde(rename = "Online")]
    pub online: bool,
}

impl Drop for TailscaleManager {
    fn drop(&mut self) {
        if let Err(e) = self.disconnect() {
            error!("Error disconnecting Tailscale: {}", e);
        }
    }
}
```

**任务 2.2.2: 主机名生成** (2 小时)
```rust
// src/utils/hostname.rs
use rand::seq::SliceRandom;
use rand::Rng;

const PREFIXES: &[&str] = &[
    "web", "api", "cdn", "mail", "ftp", "db", "cache",
    "proxy", "gw", "vpn", "app", "svc"
];

const SUFFIXES: &[&str] = &[
    "srv", "node", "host", "box", "vm", "sys", "pod", "shard"
];

pub fn generate() -> String {
    let mut rng = rand::thread_rng();

    let prefix = PREFIXES.choose(&mut rng).unwrap();
    let suffix = SUFFIXES.choose(&mut rng).unwrap();
    let num = rng.gen_range(1..100);

    format!("{}-{}-{:02}", prefix, suffix, num)
}

pub fn get_system_hostname() -> Option<String> {
    hostname::get()
        .ok()
        .and_then(|h| h.into_string().ok())
        .map(|h| {
            h.split('.')
                .next()
                .unwrap_or(&h)
                .replace('_', "-")
                .to_string()
        })
        .filter(|h| !h.is_empty() && h.len() <= 63)
}

pub fn get_or_generate() -> String {
    get_system_hostname().unwrap_or_else(generate)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_generate_format() {
        let hostname = generate();
        assert!(hostname.contains('-'));
        assert!(hostname.len() <= 63);

        let parts: Vec<&str> = hostname.split('-').collect();
        assert_eq!(parts.len(), 3);
    }

    #[test]
    fn test_generate_randomness() {
        let h1 = generate();
        let h2 = generate();
        // 大概率不同（除非非常不走运）
        // 这个测试可能偶尔失败，但概率极低
    }
}
```

**任务 2.2.3: 集成到主程序** (3-4 小时)
```rust
// src/main.rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();
    init_logging(args.verbose);

    // 获取配置
    let hostname = args.hostname
        .unwrap_or_else(|| utils::hostname::get_or_generate());

    let authkey = args.authkey
        .unwrap_or_else(|| crypto::xor::get_default_authkey());

    info!("Starting SockTail v{}", env!("CARGO_PKG_VERSION"));
    info!("Hostname: {}", hostname);

    // 连接 Tailscale（除非在开发模式）
    if !args.no_vpn {
        let mut ts_manager = vpn::tailscale::TailscaleManager::new(
            hostname,
            authkey,
            args.control_url,
        );

        ts_manager.connect()?;

        // 验证连接状态
        match ts_manager.status() {
            Ok(status) => {
                info!("Tailscale status: {}", status.backend_state);
                if let Some(node) = status.self_node {
                    info!("Node online: {}", node.online);
                }
            }
            Err(e) => {
                error!("Failed to get Tailscale status: {}", e);
            }
        }

        // 设置优雅关闭
        let ts_manager = std::sync::Arc::new(tokio::sync::Mutex::new(ts_manager));
        let ts_manager_clone = ts_manager.clone();

        tokio::spawn(async move {
            tokio::signal::ctrl_c().await.ok();
            info!("Shutting down...");
            if let Ok(mut mgr) = ts_manager_clone.lock().await.disconnect() {
                // 清理完成
            }
            std::process::exit(0);
        });
    }

    // 启动 SOCKS5 服务器
    let server = socks5::server::Socks5Server::new(args.listen);
    server.run().await?;

    Ok(())
}
```

**验收标准**:
- ✅ 能够连接到 Tailscale
- ✅ 能够通过 Tailscale 网络代理流量
- ✅ Ctrl+C 优雅关闭
- ✅ XOR 混淆与 Go 版本兼容

---

#### 2.3 构建时嵌入密钥 (4-6 小时)

**任务 2.3.1: 构建脚本** (2-3 小时)
```rust
// build.rs
use std::env;

fn main() {
    // 从环境变量获取认证密钥
    if let Ok(auth_key) = env::var("AUTH_KEY") {
        let obfuscated = xor_encode(auth_key.as_bytes());
        let hex = hex::encode(&obfuscated);

        println!("cargo:rustc-env=EMBEDDED_AUTH_KEY={}", hex);
    }

    // 从环境变量获取控制服务器 URL
    if let Ok(control_url) = env::var("CONTROL_URL") {
        println!("cargo:rustc-env=EMBEDDED_CONTROL_URL={}", control_url);
    }
}

fn xor_encode(data: &[u8]) -> Vec<u8> {
    const XOR_KEY: &[u8] = b"747sg^8N0$";
    data.iter()
        .enumerate()
        .map(|(i, &byte)| byte ^ XOR_KEY[i % XOR_KEY.len()])
        .collect()
}
```

**任务 2.3.2: 更新 Cargo.toml** (30 分钟)
```toml
[build-dependencies]
hex = "0.4"
```

**任务 2.3.3: 读取嵌入的密钥** (1-2 小时)
```rust
// src/crypto/xor.rs
pub fn get_default_authkey() -> String {
    // 优先使用构建时嵌入的密钥
    if let Ok(embedded_hex) = std::env::var("EMBEDDED_AUTH_KEY") {
        if let Ok(obfuscated) = hex::decode(&embedded_hex) {
            return deobfuscate_authkey(&obfuscated);
        }
    }

    // 回退到硬编码的默认密钥
    deobfuscate_authkey(EMBEDDED_OBFUSCATED_KEY)
}

pub fn get_default_control_url() -> Option<String> {
    std::env::var("EMBEDDED_CONTROL_URL").ok()
}
```

**任务 2.3.4: Makefile** (1 小时)
```makefile
# Makefile
.PHONY: build build-with-key build-all clean

# 默认构建
build:
	cargo build --release

# 使用自定义密钥构建
build-with-key:
	@if [ -z "$(AUTH_KEY)" ]; then \
		echo "Error: AUTH_KEY not specified"; \
		exit 1; \
	fi
	AUTH_KEY=$(AUTH_KEY) cargo build --release

# 使用自定义密钥和控制服务器构建
build-with-config:
	@if [ -z "$(AUTH_KEY)" ]; then \
		echo "Error: AUTH_KEY not specified"; \
		exit 1; \
	fi
	AUTH_KEY=$(AUTH_KEY) CONTROL_URL=$(CONTROL_URL) cargo build --release

# 交叉编译所有平台
build-all-with-key:
	@if [ -z "$(AUTH_KEY)" ]; then \
		echo "Error: AUTH_KEY not specified"; \
		exit 1; \
	fi
	# Linux
	AUTH_KEY=$(AUTH_KEY) cargo build --release --target x86_64-unknown-linux-musl
	# macOS
	AUTH_KEY=$(AUTH_KEY) cargo build --release --target x86_64-apple-darwin
	# Windows
	AUTH_KEY=$(AUTH_KEY) cargo build --release --target x86_64-pc-windows-gnu

# 清理
clean:
	cargo clean

# 运行测试
test:
	cargo test --all-features

# 代码检查
lint:
	cargo clippy -- -D warnings
	cargo fmt --check

# 运行（开发模式）
run:
	cargo run -- --no-vpn --verbose
```

**验收标准**:
- ✅ `make build-with-key AUTH_KEY=xxx` 成功
- ✅ 编译后的二进制包含嵌入的密钥
- ✅ 密钥被正确混淆
- ✅ 与 Go 版本的构建流程一致

---

### 阶段 3: 优化和测试（第 8-10 天，18-24 小时）

**目标**: 性能优化、测试、文档完善

#### 3.1 性能优化 (6-8 小时)

**任务 3.1.1: 二进制大小优化** (2-3 小时)

添加优化配置:
```toml
# Cargo.toml
[profile.release-small]
inherits = "release"
opt-level = "z"      # 优化大小
lto = true           # 链接时优化
codegen-units = 1    # 单个代码生成单元
strip = true         # 去除符号
panic = "abort"      # 使用 abort 而非 unwind
```

分析二进制大小:
```bash
cargo install cargo-bloat
cargo bloat --release -n 20
cargo build --profile release-small
ls -lh target/release-small/socktail
```

**任务 3.1.2: 运行时性能优化** (2-3 小时)

使用缓冲区池:
```rust
// src/socks5/relay.rs
use bytes::BytesMut;

const BUFFER_SIZE: usize = 8192;

pub async fn relay_data_buffered(
    client: TcpStream,
    target: TcpStream
) -> io::Result<()> {
    let (mut client_read, mut client_write) = client.into_split();
    let (mut target_read, mut target_write) = target.into_split();

    let client_to_target = async {
        let mut buf = BytesMut::with_capacity(BUFFER_SIZE);
        loop {
            buf.clear();
            let n = client_read.read_buf(&mut buf).await?;
            if n == 0 {
                break;
            }
            target_write.write_all(&buf[..n]).await?;
        }
        target_write.shutdown().await?;
        Ok::<_, io::Error>(())
    };

    let target_to_client = async {
        let mut buf = BytesMut::with_capacity(BUFFER_SIZE);
        loop {
            buf.clear();
            let n = target_read.read_buf(&mut buf).await?;
            if n == 0 {
                break;
            }
            client_write.write_all(&buf[..n]).await?;
        }
        client_write.shutdown().await?;
        Ok::<_, io::Error>(())
    };

    tokio::try_join!(client_to_target, target_to_client)?;
    Ok(())
}
```

**任务 3.1.3: 性能基准测试** (2 小时)
```rust
// benches/throughput.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, Throughput};
use tokio::runtime::Runtime;

fn relay_benchmark(c: &mut Criterion) {
    let rt = Runtime::new().unwrap();
    let mut group = c.benchmark_group("relay");

    // 设置吞吐量测量
    group.throughput(Throughput::Bytes(1024 * 1024));

    group.bench_function("tokio_copy_1mb", |b| {
        b.to_async(&rt).iter(|| async {
            // 基准测试代码
            black_box(1024 * 1024)
        });
    });

    group.finish();
}

criterion_group!(benches, relay_benchmark);
criterion_main!(benches);
```

运行基准测试:
```bash
cargo bench
# 结果在 target/criterion/report/index.html
```

**验收标准**:
- ✅ 二进制大小 < 3 MB
- ✅ 吞吐量 > 800 Mbps
- ✅ CPU 占用 < 15%
- ✅ 内存占用 < 10 MB

---

#### 3.2 全面测试 (6-8 小时)

**任务 3.2.1: 单元测试** (2-3 小时)

补充所有模块的测试:
```rust
// src/socks5/protocol.rs
#[cfg(test)]
mod tests {
    // 已有测试 +

    #[test]
    fn test_ipv6_parsing() { /* ... */ }

    #[test]
    fn test_invalid_version() { /* ... */ }

    #[test]
    fn test_buffer_underflow() { /* ... */ }
}
```

测试覆盖率:
```bash
cargo install cargo-tarpaulin
cargo tarpaulin --out Html --output-dir coverage
```

**任务 3.2.2: 集成测试** (2-3 小时)
```rust
// tests/integration_tests.rs
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::test]
async fn test_socks5_connect() {
    // 启动测试服务器
    let server = tokio::spawn(async {
        let server = Socks5Server::new("127.0.0.1:10800".to_string());
        server.run().await
    });

    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;

    // 连接到 SOCKS5 服务器
    let mut stream = TcpStream::connect("127.0.0.1:10800").await.unwrap();

    // 发送认证请求
    stream.write_all(&[0x05, 0x01, 0x00]).await.unwrap();

    // 读取响应
    let mut buf = [0u8; 2];
    stream.read_exact(&mut buf).await.unwrap();

    assert_eq!(buf, [0x05, 0x00]);

    // ... 更多测试
}

#[tokio::test]
async fn test_proxy_http_request() {
    // 测试完整的 HTTP 代理流程
    /* ... */
}
```

**任务 3.2.3: 压力测试** (2 小时)
```bash
# 使用 wrk 进行压力测试
wrk -t12 -c400 -d30s --socks5 127.0.0.1:1080 http://example.com

# 或使用自定义脚本
# tests/stress_test.sh
```

**验收标准**:
- ✅ 所有单元测试通过
- ✅ 集成测试通过
- ✅ 代码覆盖率 > 80%
- ✅ 压力测试稳定（1000+ 并发连接）

---

#### 3.3 文档和示例 (4-6 小时)

**任务 3.3.1: API 文档** (2 小时)
```rust
//! SockTail - SOCKS5 proxy over Tailscale VPN
//!
//! # Examples
//!
//! ```no_run
//! use socktail::socks5::server::Socks5Server;
//!
//! #[tokio::main]
//! async fn main() -> anyhow::Result<()> {
//!     let server = Socks5Server::new("127.0.0.1:1080".to_string());
//!     server.run().await?;
//!     Ok(())
//! }
//! ```

/// SOCKS5 protocol implementation
pub mod socks5;

/// VPN integration (Tailscale)
pub mod vpn;

/// Cryptographic utilities
pub mod crypto;

/// Utility functions
pub mod utils;
```

生成文档:
```bash
cargo doc --no-deps --open
```

**任务 3.3.2: README.md** (1-2 小时)
```markdown
# SockTail-RS 🦀

Rust implementation of SockTail - A SOCKS5 proxy over Tailscale VPN.

## Features

- ✅ Fast SOCKS5 proxy (async/await with Tokio)
- ✅ Tailscale VPN integration
- ✅ XOR key obfuscation
- ✅ Cross-platform (Linux/macOS/Windows)
- ✅ Small binary size (~1-3 MB)
- ✅ Low memory footprint (~5 MB)

## Quick Start

### Installation

```bash
cargo install socktail
```

### Usage

```bash
# Basic usage (uses embedded key)
socktail

# Custom hostname
socktail -H my-proxy

# Custom auth key
socktail -a tskey-auth-xxxx

# With Headscale
socktail -a tskey-auth-xxxx -c https://headscale.example.com
```

## Building from Source

```bash
# Development build
cargo build

# Release build
cargo build --release

# With embedded auth key
make build-with-key AUTH_KEY=tskey-auth-xxxxx
```

## Performance

| Metric | Value |
|--------|-------|
| Binary Size | 1-3 MB |
| Memory Usage | ~5 MB |
| Throughput | 900+ Mbps |
| Startup Time | <500ms |

## License

MIT
```

**任务 3.3.3: CHANGELOG.md** (30 分钟)
```markdown
# Changelog

## [0.1.0] - 2025-11-XX

### Added
- Initial Rust implementation
- SOCKS5 proxy server
- Tailscale integration
- XOR key obfuscation
- Cross-platform support

### Performance
- 30x smaller binary than Go version
- 5-10x less memory usage
- Comparable network throughput
```

**任务 3.3.4: 示例代码** (1 小时)
```rust
// examples/simple_proxy.rs
use socktail::socks5::server::Socks5Server;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();

    let server = Socks5Server::new("127.0.0.1:1080".to_string());
    println!("🚀 SOCKS5 proxy listening on 127.0.0.1:1080");

    server.run().await?;
    Ok(())
}
```

运行示例:
```bash
cargo run --example simple_proxy
```

**验收标准**:
- ✅ 文档清晰完整
- ✅ 示例可运行
- ✅ README 包含所有必要信息
- ✅ `cargo doc` 无警告

---

### 阶段 4: 跨平台和发布（第 11-14 天，18-24 小时）

**目标**: 跨平台支持、发布准备

#### 4.1 跨平台编译 (6-8 小时)

**任务 4.1.1: 交叉编译环境** (2-3 小时)
```bash
# 安装 cross
cargo install cross

# 添加目标平台
rustup target add x86_64-unknown-linux-musl
rustup target add x86_64-unknown-linux-gnu
rustup target add x86_64-apple-darwin
rustup target add aarch64-apple-darwin
rustup target add x86_64-pc-windows-gnu
rustup target add aarch64-unknown-linux-musl
```

**任务 4.1.2: 构建脚本** (2-3 小时)
```bash
#!/bin/bash
# scripts/build-all.sh

set -e

VERSION=$(cargo metadata --no-deps --format-version 1 | jq -r '.packages[0].version')
AUTH_KEY=${AUTH_KEY:-""}

TARGETS=(
    "x86_64-unknown-linux-musl"
    "x86_64-unknown-linux-gnu"
    "x86_64-apple-darwin"
    "aarch64-apple-darwin"
    "x86_64-pc-windows-gnu"
    "aarch64-unknown-linux-musl"
)

for target in "${TARGETS[@]}"; do
    echo "Building for $target..."

    if [ -n "$AUTH_KEY" ]; then
        AUTH_KEY="$AUTH_KEY" cross build --release --target "$target"
    else
        cross build --release --target "$target"
    fi

    # 重命名二进制
    binary_name="socktail-$VERSION-$target"
    if [[ $target == *"windows"* ]]; then
        cp "target/$target/release/socktail.exe" "dist/$binary_name.exe"
    else
        cp "target/$target/release/socktail" "dist/$binary_name"
    fi

    # 压缩
    cd dist
    if [[ $target == *"windows"* ]]; then
        zip "$binary_name.zip" "$binary_name.exe"
        rm "$binary_name.exe"
    else
        tar czf "$binary_name.tar.gz" "$binary_name"
        rm "$binary_name"
    fi
    cd ..

    echo "✅ Built $binary_name"
done

echo "🎉 All builds completed!"
```

**任务 4.1.3: 平台特定测试** (2 小时)
```rust
// src/vpn/tailscale.rs
#[cfg(target_os = "windows")]
fn get_tailscale_command() -> &'static str {
    "tailscale.exe"
}

#[cfg(not(target_os = "windows"))]
fn get_tailscale_command() -> &'static str {
    "tailscale"
}
```

在各平台测试:
```bash
# Linux
cargo test --target x86_64-unknown-linux-gnu

# macOS (如果可用)
cargo test --target x86_64-apple-darwin

# Windows (使用 cross)
cross test --target x86_64-pc-windows-gnu
```

**验收标准**:
- ✅ 所有目标平台编译成功
- ✅ 二进制大小符合预期
- ✅ 功能测试通过

---

#### 4.2 CI/CD 完善 (4-6 小时)

**任务 4.2.1: GitHub Actions 完整流程** (2-3 小时)
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-and-release:
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: x86_64-unknown-linux-musl
            cross: true
          - os: ubuntu-latest
            target: aarch64-unknown-linux-musl
            cross: true
          - os: macos-latest
            target: x86_64-apple-darwin
            cross: false
          - os: macos-latest
            target: aarch64-apple-darwin
            cross: false
          - os: windows-latest
            target: x86_64-pc-windows-gnu
            cross: false

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v3

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install cross
        if: matrix.cross
        run: cargo install cross

      - name: Build
        run: |
          if [ "${{ matrix.cross }}" = "true" ]; then
            cross build --release --target ${{ matrix.target }}
          else
            cargo build --release --target ${{ matrix.target }}
          fi

      - name: Package
        shell: bash
        run: |
          cd target/${{ matrix.target }}/release
          if [[ "${{ matrix.os }}" == "windows-latest" ]]; then
            7z a ../../../socktail-${{ matrix.target }}.zip socktail.exe
          else
            tar czf ../../../socktail-${{ matrix.target }}.tar.gz socktail
          fi

      - name: Upload Release Asset
        uses: softprops/action-gh-release@v1
        with:
          files: socktail-${{ matrix.target }}.*
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**任务 4.2.2: 自动化测试流程** (1-2 小时)
```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: dtolnay/rust-toolchain@stable

      - name: Run tests
        run: cargo test --all-features --verbose

      - name: Run clippy
        run: cargo clippy -- -D warnings

      - name: Check formatting
        run: cargo fmt --check

      - name: Security audit
        run: |
          cargo install cargo-audit
          cargo audit

  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: dtolnay/rust-toolchain@stable

      - name: Install tarpaulin
        run: cargo install cargo-tarpaulin

      - name: Generate coverage
        run: cargo tarpaulin --out Xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

**验收标准**:
- ✅ CI/CD 流水线完整
- ✅ 自动化测试覆盖所有平台
- ✅ 发布流程自动化

---

#### 4.3 发布准备 (4-6 小时)

**任务 4.3.1: Cargo 包发布准备** (2 小时)
```toml
# Cargo.toml
[package]
name = "socktail"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <email@example.com>"]
license = "MIT"
description = "Fast SOCKS5 proxy over Tailscale VPN, written in Rust"
repository = "https://github.com/yourusername/socktail-rs"
homepage = "https://github.com/yourusername/socktail-rs"
documentation = "https://docs.rs/socktail"
readme = "README.md"
keywords = ["socks5", "proxy", "tailscale", "vpn", "networking"]
categories = ["network-programming", "command-line-utilities"]

[package.metadata.docs.rs]
all-features = true
```

检查包:
```bash
cargo package --list
cargo package --allow-dirty
cargo publish --dry-run
```

**任务 4.3.2: Docker 镜像** (2-3 小时)
```dockerfile
# Dockerfile
FROM rust:1.75-alpine AS builder

RUN apk add --no-cache musl-dev

WORKDIR /app
COPY . .

RUN cargo build --release --target x86_64-unknown-linux-musl

# Runtime 镜像
FROM alpine:latest

RUN apk add --no-cache ca-certificates tailscale

COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/socktail /usr/local/bin/

EXPOSE 1080

CMD ["socktail"]
```

构建和发布:
```bash
docker build -t socktail:latest .
docker tag socktail:latest yourusername/socktail:0.1.0
docker push yourusername/socktail:0.1.0
```

**任务 4.3.3: 版本标记和发布** (1 小时)
```bash
# 打标签
git tag -a v0.1.0 -m "Initial Rust release"
git push origin v0.1.0

# 发布到 crates.io
cargo publish

# 创建 GitHub Release
gh release create v0.1.0 \
    --title "v0.1.0 - Initial Release" \
    --notes "See CHANGELOG.md for details" \
    dist/*
```

**验收标准**:
- ✅ Cargo 包发布成功
- ✅ Docker 镜像可用
- ✅ GitHub Release 完整
- ✅ 所有平台二进制可下载

---

## 📊 时间和资源分配

### 总体时间表

| 阶段 | 时长 | 工作日 | 关键交付物 |
|------|------|--------|----------|
| 阶段 0: 环境准备 | 4-6h | 1天 | 项目框架 |
| 阶段 1: SOCKS5 实现 | 18-24h | 2-4天 | 可工作的代理 |
| 阶段 2: VPN 集成 | 18-24h | 3-4天 | Tailscale 支持 |
| 阶段 3: 优化测试 | 18-24h | 3-4天 | 生产级质量 |
| 阶段 4: 跨平台发布 | 18-24h | 3-4天 | 所有平台发布 |
| **总计** | **76-102h** | **12-17天** | **完整产品** |

### 资源需求

**人力**：
- 1 名熟悉 Rust 的开发者（全职）
- 或 2 名开发者（兼职）

**硬件**：
- 开发机器（Linux/macOS）
- 测试用的 Tailscale 账号
- 可选：多平台测试设备

**软件/服务**：
- GitHub 账号（CI/CD）
- Crates.io 账号
- Docker Hub 账号（可选）

---

## 🎯 里程碑和检查点

### 里程碑 1: 本地 SOCKS5 代理可用（第 4 天）
- ✅ 能够代理 HTTP/HTTPS 流量
- ✅ 通过基本功能测试
- ✅ curl 测试成功

### 里程碑 2: Tailscale 集成完成（第 7 天）
- ✅ 连接到 Tailscale 网络
- ✅ 通过 VPN 代理流量
- ✅ XOR 混淆与 Go 兼容

### 里程碑 3: 生产就绪（第 10 天）
- ✅ 所有测试通过
- ✅ 性能达标
- ✅ 文档完整

### 里程碑 4: 发布（第 14 天）
- ✅ 所有平台编译成功
- ✅ CI/CD 完整
- ✅ 公开发布

---

## 🚧 风险和缓解措施

### 技术风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| Tailscale CLI 不可用 | 高 | 低 | 提供 --no-vpn 模式 |
| 性能不达标 | 中 | 低 | 早期基准测试 |
| 跨平台兼容性问题 | 中 | 中 | 持续集成测试 |
| 学习曲线陡峭 | 低 | 中 | 参考现有代码 |

### 进度风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| 时间估算不准 | 中 | 中 | 20% 缓冲时间 |
| 测试发现重大 bug | 高 | 低 | 早期持续测试 |
| 第三方依赖问题 | 中 | 低 | 锁定依赖版本 |

---

## ✅ 验收标准

### 功能性

- [ ] 支持 SOCKS5 协议（IPv4/IPv6/域名）
- [ ] Tailscale 集成正常工作
- [ ] XOR 混淆与 Go 版本兼容
- [ ] 命令行参数完整
- [ ] 优雅关闭

### 性能

- [ ] 吞吐量 > 800 Mbps
- [ ] 内存占用 < 10 MB
- [ ] 二进制大小 < 3 MB
- [ ] 启动时间 < 500ms
- [ ] CPU 占用 < 15%

### 质量

- [ ] 代码覆盖率 > 80%
- [ ] 无 clippy 警告
- [ ] 无安全审计问题
- [ ] 文档完整
- [ ] 所有平台编译成功

### 发布

- [ ] GitHub Release 完整
- [ ] Crates.io 发布成功
- [ ] Docker 镜像可用
- [ ] 所有二进制可下载

---

## 📚 参考资源

### 必读文档

1. [Rust Book](https://doc.rust-lang.org/book/)
2. [Tokio Tutorial](https://tokio.rs/tokio/tutorial)
3. [SOCKS5 RFC 1928](https://tools.ietf.org/html/rfc1928)
4. [Tailscale Docs](https://tailscale.com/kb/)

### 示例项目

1. [tokio-socks](https://github.com/sticnarf/tokio-socks)
2. [shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)
3. [boringtun](https://github.com/cloudflare/boringtun)

### 工具

- [rust-analyzer](https://rust-analyzer.github.io/)
- [cargo-edit](https://github.com/killercup/cargo-edit)
- [cargo-watch](https://github.com/watchexec/cargo-watch)

---

## 🎉 总结

这个开发计划提供了：

1. **明确的时间表**：12-17 个工作日
2. **详细的任务分解**：每个任务都有具体的代码示例
3. **清晰的验收标准**：每个阶段都有可测量的目标
4. **风险管理**：识别并缓解潜在问题
5. **质量保证**：测试、文档、CI/CD 完整

**下一步**：
1. 确认资源可用性
2. 设置开发环境
3. 开始阶段 0 的工作
4. 每日站会跟踪进度

**成功关键**：
- 遵循敏捷迭代原则
- 每个阶段都有可运行的版本
- 持续测试和集成
- 及时调整计划

祝开发顺利！🚀
