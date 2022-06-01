### 一. zkSync 简介

zkSync 是第 2 层扩展解决方案，提供比以太坊主区块链（第 1 层）更便宜、更快的交易。第 2 层解决方案将大部分活动从第 1 层移开，同时仍继承其安全性和确定性。

zkSync是一种无需信任的协议，用于在以太坊上进行可扩展的低成本支付， 底层技术方案为 zkRollup。它使用零知识证明和链上数据可用性来保证用户的资金安全，就好像他们从未离开过主网一样。

### 二. zkSync 的优势

- 低 gas：高达 L1 gas 的 1/100，并且比使用乐观汇总更便宜
- 高速：每秒 2000+ 个事务 (tps)，​​而 L1 上的 14tps
- 安全性：由以太坊主区块链保护 
- 无延迟传输：毫不费力地在 L1 和第 2 层之间移动您的加密货币，没有延迟
- 审查阻力：您可以随时将资产移回 L1

### 三. zkSync 是如何工作的

zkSync 是一组称为汇总的第 2 层解决方案的一部分。更具体地说，zkSync 是一个 ZK 汇总。（ZK 代表零知识，这是一个加密术语，表示一方能够向另一方证明某事是真实的，而无需透露任何其他信息。）

有关 zk Rollup 的信息，请参阅我们以前的文章。

在桥实现上，zkSync 和 Arbitrum 相比，主要的区别在于 Withdraw 时，对交易的验证采用了基于零知识证明的有效证明，而非欺诈证明，其基本步骤为：

#### 1. L2—>L1

用户在 L2 发起 Withdraw 交易：将交易数据编码为字节串，并使用正确的 zkSync 私钥对该字节串进行签名，并为交易描述生成 Ethereum 签名或提供 EIP-1712 签名，通过相应 JSON RPC 方法发送交易
发送交易到 L1：交易进入 zkSync operator 创建的区块，并发布到 L1 上的 zkSync 智能合约中
对区块进行验证：若干分钟后，证明该区块正确性的 ZK proof 将生成，该 proof 会通过一个验证交易发布到 L1 上，直到这个验证交易完成，Withdraw 交易完成

可以发现，在退出时间上，zk Rollup 方案明显优于 Optimistic Rollup 方案，但由于 zk Rollup 要实现完全兼容 EVM 尚需时日，所以预计 Optimistic Rollup 仍然会成为前期主流 L2 方案。也正因此，利用第三方桥来解决 Optimistic Rollup 退出期太长的问题，来带给用户更好的使用体验，成为一些团队努力的目标。


### 四. zkSync2.0

zkSync 2.0 的外观和感觉都像以太坊，但费用较低。就像在以太坊上一样，智能合约是用 Solidity/Vyper 编写的，并且可以使用与其他 EVM 兼容链相同的客户端来调用。

无需注册单独的私钥即可使用 zkSync。现有的以太坊钱包开箱即用。

参考资料
[zksync 用户文档](https://docs.zksync.io/userdocs/)
[zksync 开发者文档](https://docs.zksync.io/dev/)
[zksync 2.0](https://v2-docs.zksync.io/dev/zksync-v2/overview.html)
