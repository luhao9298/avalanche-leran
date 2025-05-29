视频地址 https://www.bilibili.com/video/BV1FTjZzQEqe/?vd_source=f39e9989a273d6500ad53adb9371a883

# Avalanche L1 主权链部署指南

## 一、环境准备

- **操作系统**：macOS
- **核心工具**：Avalanche CLI（`avalanche` 命令行工具）
- **版本要求**：avalanchego-v1.13.0（建议通过 `avalanche --version` 检查版本）

---

## 二、L1 部署步骤

### 2.1 执行部署命令

```bash
avalanche blockchain deploy myblockchain --local
```

### 2.2 关键配置参数（部署后自动生成）

| 参数项           | 配置值                                                                              | 说明                     |
| ---------------- | ----------------------------------------------------------------------------------- | ------------------------ |
| 网络类型         | Local Network                                                                       | 本地测试网络             |
| 链 ID            | 43119                                                                               | 链的唯一标识             |
| 代币符号         | AVAX_L1                                                                             | 原生代币简称             |
| 验证机制         | Proof of Authority (PoA)                                                            | 测试网常用轻量级共识     |
| VM ID            | qDNV9vtxZYYNqm7TN1mYBuaaknLdefDbFK8bFmMLTJQJKaWjV                                   | 虚拟机唯一标识           |
| RPC 端点         | http://127.0.0.1:63480/ext/bc/StdSixArd4pUZwNeDGuHu6QZMDRiebThq49fZ3F8svQ2Un5vt/rpc | 链的 RPC 交互入口        |
| 区块链 ID (CB58) | StdSixArd4pUZwNeDGuHu6QZMDRiebThq49fZ3F8svQ2Un5vt                                   | 区块链的 Base58 编码标识 |
| 子网 ID          | 2W9boARgCWL25z6pMFNtkCfNA5v28VGg9PmBgUJfuKndEdhrvw                                  | 子网的唯一标识           |

### 2.3 验证部署成功

部署完成后，通过以下命令确认节点运行状态（需保持终端不关闭）：

```bash
curl -X POST -H "content-type:application/json" \
  --data '{"jsonrpc":"2.0","id":1,"method":"info.getNodeVersion"}' \
  http://127.0.0.1:63480/ext/info
```

**预期输出**（包含版本信息）：

```json
{
  "jsonrpc": "2.0",
  "result": {
    "version": "avalanche/1.13.0",
    ...
  },
  "id": 1
}
```

---

## 三、智能合约开发（以 ERC20 代币为例）

### 3.1 合约代码（Solidity）

创建 `contracts/MyAVAXToken.sol`：

```solidity:my-avalanche-project/contracts/MyAVAXToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// 导入OpenZeppelin标准ERC20合约
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/**
 * @title 自定义AVAX测试代币
 * @dev 基于OpenZeppelin ERC20实现，支持初始铸币
 */
contract MyAVAXToken is ERC20 {
    /**
     * @param initialSupply 初始发行总量（18位小数精度）
     */
    constructor(uint256 initialSupply) ERC20("MyAVAXToken", "mAVAX") {
        // 向合约部署者地址铸币
        _mint(msg.sender, initialSupply);
    }
}
```

### 3.2 开发环境配置（Hardhat）

#### 3.2.1 初始化项目

```bash
mkdir my-avalanche-project && cd my-avalanche-project
npm init -y
npm install --save-dev hardhat @nomiclabs/hardhat-ethers ethers @openzeppelin/contracts
```

#### 3.2.2 配置 Hardhat 网络（`hardhat.config.js`）

修改项目根目录下的 `hardhat.config.js`：

```javascript:my-avalanche-project/hardhat.config.js
require("@nomiclabs/hardhat-ethers");

module.exports = {
  // 编译Solidity版本与合约匹配
  solidity: "0.8.0",
  networks: {
    // 配置本地L1网络
    myblockchain: {
      url: "http://127.0.0.1:63480/ext/bc/StdSixArd4pUZwNeDGuHu6QZMDRiebThq49fZ3F8svQ2Un5vt/rpc", // L1 RPC端点
      chainId: 43119, // 链ID与部署时一致
      // 测试账户私钥（通过Avalanche CLI生成测试账户，或使用MetaMask创建的本地账户私钥）
      accounts: ["0x你的测试钱包私钥"] // 注意：仅测试用，切勿暴露主网私钥！
    }
  }
};
```

### 3.3 部署脚本

创建 `scripts/deploy.js`：

```javascript:my-avalanche-project/scripts/deploy.js
async function main() {
  // 获取合约工厂（与MyAVAXToken.sol中的合约名一致）
  const MyAVAXToken = await ethers.getContractFactory("MyAVAXToken");

  // 部署合约：初始发行100万枚（18位小数精度，即1000000 * 10^18）
  const token = await MyAVAXToken.deploy(1000000n * (10n ** 18n));

  // 等待部署完成
  await token.deployed();

  console.log("MyAVAXToken 合约已部署，地址：", token.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error("部署失败：", error);
    process.exit(1);
  });
```

### 3.4 执行部署

```bash
npx hardhat run scripts/deploy.js --network myblockchain
```

**成功输出示例**：

```
MyAVAXToken 合约已部署，地址：0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

## 四、交互操作示例（以 MetaMask 转账为例）

### 4.1 钱包连接 L1 网络

1. 打开 MetaMask → 点击顶部网络选择 → 点击「添加网络」→ 点击「自定义 RPC」。
2. 填写以下信息：
   - **网络名称**：myblockchain（与 Hardhat 配置一致）
   - **新 RPC URL**：http://127.0.0.1:63480/ext/bc/StdSixArd4pUZwNeDGuHu6QZMDRiebThq49fZ3F8svQ2Un5vt/rpc（L1 RPC 端点）
   - **链 ID**：43119（与部署时一致）
   - **货币符号**：AVAX_L1（原生代币简称）
3. 点击「保存」，完成网络添加。

### 4.2 导入测试账户

1. 在 MetaMask 点击「账户详情」→ 点击「导入账户」。
2. 输入部署脚本中使用的测试私钥（`hardhat.config.js` 中的 `accounts` 字段）。
3. 导入后，钱包会显示初始的 AVAX_L1 余额（由 L1 节点自动分配）。

### 4.3 执行 ERC20 转账

1. 在 MetaMask 中选择「MyAVAXToken」代币（若未自动显示，手动添加合约地址：部署时输出的 `0x...` 地址）。
2. 点击「转账」→ 输入目标钱包地址 → 输入转账金额（如 `10 mAVAX`）。
3. 确认交易，等待链上确认（测试网通常 1-2 秒完成）。

### 4.4 验证交易状态（可选）

通过 curl 调用 L1 RPC 接口查询交易收据：

```bash
curl -X POST -H "content-type:application/json" \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_getTransactionReceipt","params":["0x你的交易哈希"]}' \
  http://127.0.0.1:63480/ext/bc/StdSixArd4pUZwNeDGuHu6QZMDRiebThq49fZ3F8svQ2Un5vt/rpc
```

**成功结果示例**（`status: "0x1"` 表示交易成功）：

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "0x1",
    "transactionHash": "0x...",
    ...
  },
  "id": 1
}
```

---
