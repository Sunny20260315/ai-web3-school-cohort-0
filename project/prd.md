# 📄 MVP 产品需求文档 (PRD): Agent-Fi Wallet (AFW)

**版本：** v2.0 (ERC-4337 + TEE 安全架构重构版)

---

## 1. 产品定位与核心价值

* **产品名称：** Agent-Fi Wallet (AFW)
* **一句话定义：** 基于 ERC-4337 账户抽象，在本地 TEE 安全飞地中持有私钥、后端仅做 Relay 的 AI Agent 原生加密钱包。
* **核心痛点：**

| 痛点 | 现状 | AFW 解决方案 |
|------|------|-------------|
| AI 私钥安全 | 私钥写死在代码里或后端明文托管，一次泄漏全部归零 | **TEE Secure Enclave**：私钥仅在可信执行环境中持存，后端无任何私钥访问权 |
| 链上策略执行 | 限额逻辑在后端代码中，可以被绕过 | **ERC-4337 validateUserOp**：策略硬编码在智能合约 `validateUserOp()` 中，链上强制执行 |
| AI 意图安全 | LLM 解析结果无验证，直接上链执行 | **交易模拟 + 函数选择器白名单 + 多级确认**：每一笔 AI 交易先模拟、再验证、后执行 |
| 行业标准兼容 | 自研钱包合约无法与其他 AA 基础设施互通 | **完全 ERC-4337 合规**：兼容任意 Bundler、Paymaster、EntryPoint |

* **核心差异化 (Moat)：**

```
┌─────────────────────────────────────────────────────────┐
│           AFW 三大不可绕过护城河                          │
├─────────────────────────────────────────────────────────┤
│ 🔐 私钥安全：TEE Enclave → 后端零接触，被黑也无法掏空   │
│ ⛓️ 策略安全：ERC-4337 链上执行 → 合约代码不可篡改       │
│ 🤖 意图安全：模拟 + 确认 → AI 幻觉也无法造成损失         │
└─────────────────────────────────────────────────────────┘
```

> 💡 **为什么不是又一个 AA 钱包？** 现有 AA 方案（Safe、Biconomy、ZeroDev）解决了人类的 Gasless/Gas-Abstraction 需求，但**没有解决 AI Agent 的特殊安全需求**：AI 可能被提示词注入、可能产生幻觉、需要人类设置硬性资金护栏。AFW 是专门为 AI-to-Chain 场景设计的安全层。

---

## 2. 核心功能范围 (MVP Scope)

### 🔐 模块一：TEE Secure Enclave — AI 私钥安全托管

**需求描述：** 创建一个独立于后端的 Secure Signer 服务，作为 AI 的"可信执行环境"容器。私钥在此容器内存中生成并终生不离开。后端只能向 Enclave 提交**已构建好的 UserOperation** 请求签名，无法获取私钥本身。

**功能点：**
- **本地 TEE 模拟容器：** Docker 容器运行独立 Secure Signer 进程，通过本地 HTTPS/gRPC 暴露唯一端点 `POST /signUserOp`
- **私钥生成与隔离：** AI 账户的 EOA 私钥在 Enclave 内生成，仅返回公钥/地址给后端注册
- **Enclave 侧策略预检 (Pre-Flight Check)：** Enclave 在签名前对 UserOp 做本地策略复核（限额、白名单），形成**双重防护**（链上 `validateUserOp` + Enclave 预检）
- **签名响应：** 验证通过的 UserOp 返回 ECDSA 签名，后端提交给 Bundler

```
后端                      Secure Enclave (TEE)
  │                              │
  │── POST /signUserOp ────────>│
  │   {userOp, policyContext}   │
  │                              │── 1. 验证 policyContext 签名
  │                              │── 2. 复核 UserOp 金额/目标合约
  │                              │── 3. 本地 ECDSA 签名
  │<── {signature} ─────────────│
  │                              │
  │── 提交至 Bundler ───────────│  (私钥从未离开 Enclave)
```

### ⛓️ 模块二：ERC-4337 智能账户与链上策略

**需求描述：** AI 的子钱包是一个**完全 ERC-4337 兼容的智能合约账户**，实现 `IAccount` 接口。风控策略通过 `validateUserOp()` 在链上强制执行——任何人（包括后端）都无法绕过合约中写死的限额逻辑。

**功能点：**
- **ERC-4337 标准账户：** 基于 `eth-infinitism/account-abstraction` 的 `SimpleAccount` 扩展，实现：
  - `validateUserOp()` — 包含 `checkPolicy()` 逻辑（单笔限额、日限额、白名单合约+函数选择器）
  - `execute()` / `executeBatch()` — 执行实际调用
- **AgentWalletFactory：** 一键部署新 AI 账户合约，自动关联主人地址与策略参数
- **日限额重置机制：** 合约内维护 `lastResetTimestamp` 和 `dailySpent`，跨日自动归零
- **主人恢复权：** 主人地址可随时调用 `updatePolicy()` 修改策略或 `emergencyWithdraw()` 提取资金
- **本地 Bundler：** 运行一个轻量级 Bundler 服务（直接使用 `eth-infinitism/bundler` 或自定义实现），监听 UserOp 并打包上链

**策略数据结构 (合约内)：**
```solidity
struct Policy {
    uint256 maxPerTx;           // 单笔上限 (wei)
    uint256 dailyLimit;         // 每日上限 (wei)
    uint256 dailySpent;         // 今日已花费
    uint256 lastResetTimestamp; // 上次重置时间
    mapping(address => mapping(bytes4 => bool)) whitelist; // 合约+函数选择器白名单
}
```

### 🤖 模块三：AI 意图安全解析与交易模拟

**需求描述：** AI 接收到自然语言指令后，不是直接生成交易参数，而是经过**意图解析 → 交易模拟 → 人类确认（可选）** 的安全闭环。

**功能点：**
- **意图解析引擎：** 使用 Dify Workflow 或 LangChain，将自然语言转化为**结构化意图 JSON**：
  ```json
  {
    "intent": "swap",
    "inputToken": "0x...",
    "outputToken": "0x...",
    "amount": "1000000000000000000",
    "slippage": 0.005,
    "confidence": 0.94,
    "reasoning": "用户说'帮我用1 ETH换成USDC'，当前ETH/USDC=3000，预期收到3000 USDC"
  }
  ```
- **交易构建器：** 将意图 JSON 转换为标准 UserOperation 结构（含 calldata）
- **交易模拟（Dry-Run）：** 在后端使用 `eth_call` 或 `debug_traceCall` 在 Anvil 上模拟执行，捕获：
  - 预期余额变化（ETH / ERC20）
  - 是否触发 revert
  - Gas 消耗估算
  - 模拟结果返回前端展示
- **函数选择器白名单校验：** 除了合约地址白名单，还限制 AI 只能调用特定函数签名（如 `swapExactETHForTokens`，禁止调用 `approve` 无限授权）
- **置信度分级处理：**
  - `confidence > 0.9` + 策略通过 → 自动执行
  - `confidence 0.7-0.9` → 前端展示意图细节，请求人类一键确认
  - `confidence < 0.7` → 挂起，必须人工介入调整意图

### 🚨 模块四：人类干预与可视化看板 (Human-in-the-loop)

**功能点：**
- **实时看板：** AI 钱包余额（ETH + ERC20）、历史交易流水、策略使用率（当日已用/限额）
- **意图确认面板：** 中置信度意图展示"AI 想干什么、预期结果、模拟详情"，一键 Approve/Reject
- **超额审批：** 超出策略限额的交易进入 `Pending_Approval`，主人用 EOA 钱包签名后由后端构建绕过限额的单次 UserOp（主人签名覆盖策略）
- **策略管理面板：** 主人可实时修改限额、白名单合约+函数选择器，交易上链后即时生效
- **"时光飞逝" Demo 按钮：** 点击后向后端发送指令，后端调用 `anvil_mine` / `evm_increaseTime` 快进 24 小时，演示日限额重置

---

## 3. 全栈技术架构（ERC-4337 合规重构版）

```
                        ┌──────────────────────┐
                        │   前端 (Next.js)      │
                        │   Wagmi + Viem +      │
                        │   RainbowKit          │
                        └─────┬────────────────┘
                              │ HTTP/WS
                              ▼
┌─────────────┐     ┌───────────────────┐     ┌──────────────────┐
│   Dify/LLM  │────▶│   后端 (NestJS)    │────▶│ Secure Enclave   │
│  意图解析   │     │  - UserOp 构建     │     │ (TEE 签名服务)   │
│             │     │  - 交易模拟        │     │ - AI 私钥托管    │
└─────────────┘     │  - 策略管理        │     │ - 策略预检      │
                    │  - API Gateway     │     │ - ECDSA 签名     │
                    └───┬───────┬───────┘     └──────────────────┘
                        │       │
                        │       ▼
                        │  ┌──────────┐
                        │  │PostgreSQL│
                        │  │+ Prisma  │
                        │  └──────────┘
                        ▼
              ┌────────────────────┐
              │   本地 Bundler      │
              │  (eth-infinitism)  │
              └─────────┬──────────┘
                        │ submit UserOp
                        ▼
              ┌────────────────────┐
              │  Anvil 本地私链     │
              │  (Foundry)         │
              │                    │
              │  ┌──────────────┐  │
              │  │ EntryPoint   │  │
              │  │ (ERC-4337)   │  │
              │  └──────┬───────┘  │
              │         │          │
              │  ┌──────▼───────┐  │
              │  │AgentWallet   │  │
              │  │(IAccount)    │  │
              │  │+ Policy      │  │
              │  └──────────────┘  │
              │                    │
              │  ┌──────────────┐  │
              │  │AgentWallet   │  │
              │  │Factory       │  │
              │  └──────────────┘  │
              │                    │
              │  ┌──────────────┐  │
              │  │ MockERC20    │  │
              │  │ MockDEX      │  │
              │  └──────────────┘  │
              └────────────────────┘
```

### 💻 3.1 前端技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 框架 | Next.js (App Router) | SSR + API Routes |
| 样式 | Tailwind CSS + Shadcn UI | 黑客松级看板 |
| Web3 交互 | Wagmi v2 + Viem v2 | Hooks 驱动, 类型安全 |
| 钱包连接 | RainbowKit / ConnectKit | 支持 localhost chain |
| 状态管理 | TanStack Query (Wagmi 内置) | 余额/交易实时刷新 |
| Chain 配置 | `localhost` chain, RPC → `http://127.0.0.1:8545` | 指向 Anvil |

### ⚙️ 3.2 后端技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 框架 | NestJS (TypeScript) | Module/Controller/Service 分层 |
| Web3 驱动 | Viem v2 | 轻量, ERC-4337 UserOp 构建原生支持 |
| UserOp 构建 | `permissionless.js` | 简化 ERC-4337 账户交互 |
| 交易模拟 | `eth_call` + `debug_traceCall` | Anvil 原生支持 |
| Bundler | `@account-abstraction/sdk` 或自建轻量 Bundler | 本地运行 |
| AI 引擎 | Dify (Docker) 或 LangChain/LangGraph | 意图解析 Workflow |
| 数据库 | PostgreSQL + Prisma ORM | 索引交易/钱包/策略 |

### 🔐 3.3 Secure Enclave（TEE 签名服务）

| 组件 | 选型 | 说明 |
|------|------|------|
| 运行环境 | 独立 Docker 容器 | 网络隔离 |
| 框架 | 轻量 Express / Fastify | 单端点服务 |
| 私钥存储 | 仅内存中 (never persisted) | 容器重启=私钥丢失（测试网可接受） |
| 通信协议 | HTTP + HMAC 签名验证 | 本地 localhost 通信 |
| 对外端点 | `POST /signUserOp` | 唯一入口 |

**Enclave 安全设计原则：**
1. 后端发送的请求必须携带 HMAC 签名（shared secret），防止容器内其他进程伪造请求
2. Enclave 内对 UserOp 做**独立的策略复核**（不信任后端传入的 policy 判断结果）
3. Enclave 只返回签名，绝不返回私钥
4. 私钥在内存中以 `Uint8Array` 形式持有，不序列化、不落盘

> 📌 **生产环境升级路径**：本地 Docker Enclave 是 MVP 模拟方案。生产环境可替换为 AWS Nitro Enclaves、Intel SGX、或 MPC-TSS 分布式签名网络。

### 📜 3.4 智能合约架构（ERC-4337 合规）

| 合约 | 继承/依赖 | 职责 |
|------|----------|------|
| `AgentWallet.sol` | `IAccount`, `BaseAccount` (eth-infinitism) | ERC-4337 智能账户，实现 `validateUserOp` + `execute` |
| `AgentWalletFactory.sol` | `Create2` | 一键部署 AgentWallet，CREATE2 确定性地址 |
| `EntryPoint.sol` | (导入 eth-infinitism 官方合约) | ERC-4337 标准入口，处理 UserOp 验证与执行 |
| `MockERC20.sol` | OpenZeppelin ERC20 | Demo 代币 |
| `MockDEX.sol` | 自定义 AMM | 简易 Uniswap V2 风格，用于 AI 换币演示 |

**AgentWallet 核心接口：**
```solidity
interface IAgentWallet is IAccount {
    // ERC-4337 必须实现
    function validateUserOp(UserOperation calldata userOp, 
        bytes32 userOpHash, uint256 missingAccountFunds)
        external returns (uint256 validationData);
    
    // 策略管理 (仅主人调用)
    function updatePolicy(Policy calldata newPolicy) external;
    function addToWhitelist(address target, bytes4 selector) external;
    function removeFromWhitelist(address target, bytes4 selector) external;
    
    // 紧急提款 (仅主人调用，绕过限额)
    function emergencyWithdraw(address token, uint256 amount, address to) external;
    
    // 查询
    function getPolicy() external view returns (Policy memory);
    function getOwner() external view returns (address);
}
```

**validateUserOp 内的策略检查流程：**
```
1. 解析 userOp.callData → 提取 target 合约 + function selector
2. 检查 target + selector 是否在白名单
3. 检查 userOp.callValue / callData 中的转账金额 ≤ maxPerTx
4. 检查 dailySpent + 本次金额 ≤ dailyLimit
5. 检查 lastResetTimestamp → 跨日重置 dailySpent
6. 检查 userOp.signature → 验证 AI 私钥签名
7. 全部通过 → return 0（SIG_VALIDATION_SUCCESS）
```

### 🗄️ 3.5 数据库 Schema

| 表 | 关键字段 | 说明 |
|----|---------|------|
| `Wallets` | `id`, `ownerAddress`, `agentWalletAddress`, `agentEOAAddress`, `policyJSON`, `chainId`, `createdAt` | AI 钱包注册信息（**不含私钥**） |
| `Transactions` | `id`, `walletId`, `userOpHash`, `txHash`, `intentJSON`, `simulationResult`, `confidence`, `status` (Success/Failed/PendingApproval/Simulated), `createdAt` | 交易全生命周期追踪 |
| `PolicyChanges` | `id`, `walletId`, `oldPolicy`, `newPolicy`, `txHash`, `changedAt` | 策略变更审计日志 |

---

## 4. AI 意图安全性设计（完整闭环）

```
自然语言输入
     │
     ▼
┌─────────────┐
│ 意图解析引擎 │ ← Dify / LangChain
│ (LLM)       │
└──────┬──────┘
       │ Intent JSON + confidence
       ▼
┌─────────────┐
│ 交易构建器   │ → Intent → UserOperation
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────────────┐
│ 交易模拟     │────▶│ 模拟结果:            │
│ (Dry-Run)   │     │ - 余额变化           │
└──────┬──────┘     │ - 是否 revert        │
       │            │ - Gas 估算           │
       │            └─────────────────────┘
       ▼
┌─────────────────────────────┐
│ 安全决策矩阵                │
├────────────┬────────────────┤
│ 置信度≥0.9│ 限额内 → 自动   │
│            │ 超额   → 挂起   │
├────────────┼────────────────┤
│ 置信度0.7-│ 显示意图       │
│ 0.9       │ 请求人类确认    │
├────────────┼────────────────┤
│ 置信度<0.7│ 强制人工介入    │
└────────────┴────────────────┘
       │
       ▼
┌─────────────┐
│ Enclave 签名│ → 双重策略复核 → 签名 UserOp
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Bundler 提交│ → EntryPoint → AgentWallet.validateUserOp()
└──────┬──────┘
       │
       ▼
  链上执行 / Revert
```

**多层防护总结：**

| 层级 | 防护点 | 防什么 |
|------|--------|--------|
| L1 | LLM 输出置信度阈值 | AI 幻觉/歧义 |
| L2 | 交易 Dry-Run 模拟 | 逻辑错误/滑点过大/revert |
| L3 | 函数选择器白名单 | 恶意重编码/未授权函数调用 |
| L4 | Enclave 策略预检 | 后端被黑/伪造 UserOp |
| L5 | 链上 `validateUserOp` | 合约层面硬性拦截 |
| L6 | 人类审批（超限+低置信） | 异常交易兜底 |

---

## 5. 竞争差异化分析

| 维度 | Coinbase AgentKit | Safe{Core} | Crossmint | **AFW (本项目)** |
|------|-------------------|------------|-----------|-------------------|
| AA 标准 | ERC-4337 部分 | ERC-4337 | 自研 | ✅ **完全 ERC-4337** |
| 私钥安全 | 后端托管 | 多签 | MPC 托管 | ✅ **TEE Enclave 隔离** |
| 链上策略 | 无 | 模块化(需定制) | 无 | ✅ **validateUserOp 内置** |
| AI 意图安全 | 无验证 | 无 | 无 | ✅ **模拟+置信度+确认** |
| 部署难度 | 依赖 Coinbase | 需要 Bundler 集群 | SaaS 平台 | ✅ **本地一键启动** |
| 策略粒度 | N/A | 模块级 | 无 | ✅ **地址+函数选择器级** |

**AFW 不可替代的价值主张：**
> 即使后端服务器被完全攻破，攻击者也**无法提取 AI 私钥**（私钥在 Enclave）、**无法绕过策略**（策略在链上合约）、**无法伪造交易**（Enclave 签名前的策略复核 + 链上双重防护）。这是现有所有 AI Wallet 方案都不具备的三重安全冗余。

---

## 6. 本地一键启动指南

```bash
# 1. 启动本地私链
anvil --host 0.0.0.0 --port 8545

# 2. 部署 ERC-4337 合约套件
forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast
# 输出: EntryPoint 地址、AgentWalletFactory 地址、MockERC20 地址 → 自动写入 .env

# 3. 启动基础设施（PostgreSQL + Dify + Secure Enclave + Bundler）
docker-compose up -d

# 4. 启动后端
cd backend && npm run dev  # NestJS → localhost:3001

# 5. 启动前端
cd frontend && npm run dev # Next.js → localhost:3000
```

**服务端口映射：**
| 服务 | 端口 |
|------|------|
| Anvil (本地私链) | `8545` |
| 后端 API | `3001` |
| 前端 | `3000` |
| PostgreSQL | `5432` |
| Dify | `3003` |
| Secure Enclave | `3004`（仅 localhost，不对外暴露） |
| Bundler | `3005` |

---

## 7. 开发阶段划分 (2-3 人团队, 预计 5-7 天)

| 阶段 | 内容 | 产出 |
|------|------|------|
| **Day 1-2** | 智能合约开发 + 部署脚本 | AgentWallet.sol, Factory, Mock 合约 + Anvil 部署 |
| **Day 2-3** | Secure Enclave + 后端核心 | Docker Enclave 签名服务, NestJS UserOp 构建+模拟 |
| **Day 3-4** | AI 意图引擎 | Dify Workflow 或 LangChain 意图解析 + 置信度输出 |
| **Day 4-5** | 前端看板 + 联调 | Wagmi 钱包连接、Dashboard、策略面板、意图确认 |
| **Day 5-6** | Bundler 集成 + 全链路测试 | UserOp → Bundler → EntryPoint → AgentWallet 端到端 |
| **Day 6-7** | Demo 脚本打磨 + 时光飞逝 | 演示流畅度优化、边界 case 处理、PPT/视频素材 |

---

## 8. 技术风险与缓解

| 风险 | 缓解方案 |
|------|---------|
| ERC-4337 EntryPoint 与 Anvil 兼容性 | 使用 eth-infinitism 审计过的 EntryPoint v0.6/v0.7，Anvil 完全兼容 |
| Secure Enclave Docker 内网络延迟 | 本地通信 <1ms，不影响用户体验 |
| LLM 意图解析准确率不稳定 | 使用 Dify Workflow 严格约束 JSON Schema 输出，加上 few-shot examples |
| Bundler 实现复杂度高 | MVP 可使用简化版 Bundler（单线程打包），不追求高并发 |
| Create2 地址冲突 | 使用唯一 salt（walletId + owner），低概率 |

---

> **一句话总结：AFW 不是一个"又一个 AA 钱包"，而是专门为 AI Agent 设计的、具备 TEE 私钥隔离 + ERC-4337 链上策略 + AI 意图安全闭环的三重安全钱包。在黑客松场景下，它是一个技术扎实、安全模型完整、差异化明确的项目。**