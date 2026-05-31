# 📄 MVP 产品需求文档 (PRD): Agent-Fi Wallet (AFW)

**版本：** v1.1 (本地全栈网络开发版)

---

## 1. 产品定位与核心价值

* **产品名称：** Agent-Fi Wallet (AFW)
* **一句话定义：** 为 AI 智能体打造的“非托管、自动化、限额受控”的原生加密钱包（本地测试网运行版）。
* **核心痛点：** 1. 传统的 AI Agent 无法自主在链上开户和支付。
2. 如果直接把钱包私钥写死在 AI 代码里，AI 一旦遭遇“提示词注入攻击”或逻辑跑飞，钱包资金会被一把薅光。
* **解决方案：** 引入 **账户抽象（Account Abstraction）** 理念。人类主人创建母钱包，给 AI 创建一个子钱包（智能合约账户），并在本地智能合约层设定 **“单笔最高 $1$ 刀、每天最多 $10$ 刀、只能交互特定白名单合约”** 的硬性策略（Policy）。

---

## 2. 核心功能范围 (MVP Scope)

### 🛠️ 模块一：AI 专属账户创建与策略配置 (Policy Manager)

* **需求描述：** 用户（人类主人）登录网页，一键为自己的 AI 助手在本地测试网生成一个智能合约账户（子钱包），并写入限制策略。
* **功能点：**
* **本地一键开户：** 调用部署在本地私链上的 Factory 合约，为 AI 部署一个独立的账户合约。
* **风控策略配置：** 允许用户设置：
* `Max_Per_Tx` (单笔最大交易额)
* `Daily_Limit` (每日限额)
* `Whitelist_Contracts` (白名单合约：如本地部署的 Mock ERC20 / Mock DEX)


### 🤖 模块二：AI 意图解析与自动化构建交易 (Intent & Execution)

* **需求描述：** AI 接收到外部指令后，自己算好账，把交易请求发给本地钱包后端，后端验证后自动在本地私链上放行。
* **功能点：**
* **意图转 JSON：** 利用本地/在线大模型将自然语言转化为标准的链上调用参数（To, Value, Data）。
* **自动化本地执行（Auto-Sign）：** 钱包后端收到 AI 的请求后，**计算其是否触发风控策略**：
* *未触发：* 后端直接动用本地托管的代理私钥（Relayer Key）调用 AI 子钱包合约，完成链上交易。
* *已触发（如超额）：* 交易状态设为 `Pending_Approval`，挂起并等待。



### 🚨 模块三：人类干预与可视化看板 (Human-in-the-loop & Dashboard)

* **需求描述：** 人类用来查看本地资产流向、AI 花费明细，以及审批被拦截的超额交易。
* **功能点：**
* **本地流水看板：** 实时抓取本地私链上 AI 钱包的余额（Local ETH / Mock Token）与历史交易记录。
* **超额审批弹窗：** AI 发起超额请求时，前端看板弹窗提示人类，人类在前端点击“Approve”，由人类主钱包进行签名放行。

在前端做一个“时光飞逝”作弊按钮，点击后后端向本地 Anvil 私链发送 `evm_increaseTime` 指令，让本地区块链时间直接快进 24 小时！然后再次演示 AI 成功支付


---

## 3. 全栈技术架构与技术栈选型（完全本地化）

为了让你们团队在本地（Localhost）流畅跑通全流程，技术栈作如下严谨定义：

```
[前端: Next.js + Wagmi] <---> [后端: Node.js/NestJS] <---> [数据库: PostgreSQL]
                                  |                 |
                                  v                 v
                       [AI 引擎: Dify Local]  [本地私链: Hardhat/Anvil]

```

### 💻 3.1 前端技术栈 (Frontend)

* **框架：** `Next.js (App Router)` + `Tailwind CSS` + `Shadcn UI`（快速揉出高级感黑客松看板）。
* **Web3 交互库：** `Wagmi` + `Viem` + `RainbowKit/privy`。
* **配置变更：** 将 RainbowKit/Wagmi 的 `chains` 配置为 `localhost` 或 `hardhat`，RPC URL 指向 `http://127.0.0.1:8545`。

### ⚙️ 3.2 后端技术栈 (Backend)

* **框架：** `Node.js (NestJS 或 Express)`，推荐 NestJS，结构清晰好分工。
* **Web3 驱动：** `viem`（比 ethers.js 更轻量、对本地私链及账户抽象支持极其友好）。
* **AI 核心（意图解析）：** * 方案 A（极速）：本地部署 `Dify (Docker 版)`，通过 Dify 的 Workflow 编排提示词，后端只需调用 Dify 的 API。
* 方案 B（纯代码）：使用 `LangChain / LangGraph` 直接在 Node.js 中调用 OpenAI/Claude API。



### 📜 3.3 智能合约架构 (Smart Contracts)

* **开发框架：** `Hardhat` 或 `Foundry (Anvil)`。建议用 Foundry，其自带的 `Anvil` 作为本地私链启动极快（1秒），且支持快照和时间旅行模拟。
* **核心合约：**
* `AgentWallet.sol`：AI 的智能合约账户，内部包含 `checkPolicy` 逻辑（检查 limit 和 whitelist）。
* `AgentWalletFactory.sol`：用于一键为新 AI 部署新钱包账户的工厂合约。
* `MockDEX.sol` / `MockERC20.sol`：本地部署一个假的流动性池，供 Demo 演示“AI 自动去换币”。



### 🗄️ 3.4 数据库架构 (Database)

* **选型：** `PostgreSQL` (配合 `Prisma ORM`，开发体验极佳)。
* **核心数据表设计 (Schema)：**
1. **`Wallets` 表：** 记录主人地址、AI 子钱包合约地址、本地私钥索引、当前策略设置（限额等）。
2. **`Transactions` 表：** 记录交易流水。字段包括：`tx_hash`、`ai_intent_text` (输入的人话)、`action_type` (Swap/Transfer)、`status` (`Success` / `Failed` / `Pending_Approval`)。



---

## 4. 本地环境一键启动指南 (Developer Guide for Hackathon)

为了向评委证明你们的技术扎实度，项目仓库应支持本地一键跑通：

1. **启动本地私链：** 执行 `anvil` 或 `npx hardhat node`，生成 10 个自带 10000 ETH 的本地测试私钥。
2. **部署合约：** 执行 `forge script` 或 `npx hardhat run deploy.js --network localhost`，将钱包工厂和 Mock Token 部署到本地，并将合约地址自动写入后端 `.env` 文件。
3. **启动数据库与后端：** `docker-compose up -d` (一键启动 Postgres 和 Dify Local)，然后运行后端服务监听 `8545` 端口。
4. **启动前端：** `npm run dev`，打开 `localhost:3000` 即可开始流畅演示。

---
