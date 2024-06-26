### 一. Polygon 简介
Polygon 是一个区块链应用平台，提供混合权益证明和支持 Plasma 的侧链。

在架构上，Polygon 的美妙之处在于其优雅的设计，它具有一个通用的验证层，与不同的执行环境分离，如 Plasma 启用的链、成熟的 EVM 侧链，以及未来的其他第 2 层方法，如 Optimistic Rollups。

目前，开发人员可以将 Plasma 用于已编写 Plasma 谓词的特定状态转换，例如 ERC20、ERC721、资产交换或其他自定义谓词。对于任意状态转换，他们可以使用 PoS。或两者！Polygon 的混合结构使这成为可能。

为了在我们的平台上启用 PoS 机制，在以太坊上部署了一组质押管理合约，以及一组运行 Heimdall 和 Bor 节点的激励验证者。以太坊是 Polygon 支持的第一个基础链，但 Polygon 打算根据社区建议和共识为其他基础链提供支持，以实现可互操作的去中心化第 2 层区块链平台。

Polygon 是由 Heimdall 和 Bor 组成，Heimdall 是我们的 Proof-of-Stake 验证层，它负责将 Plasma 块的表示检查点到我们架构中的主链。我们通过在 Tendermint 共识引擎之上构建并更改签名方案和各种数据结构来实现这一点。Bor 节点或 Block Producer 实现基本上是侧链运营商。侧链 VM 与 EVM 兼容。目前，它是一个基本的 Geth 实现，对共识算法进行了自定义更改。但是，这将是从头开始构建的，以使其轻巧且专注。

### 二. Polygon 项目架构

#### 1. Polygon 具有三层架构
以太坊上的 Staking 和 Plasma 智能合约
Heimdall（权益证明层）
Bor（区块生产者层）

![](https://raw.githubusercontent.com/the-web3/layer2/79839bb1ee4b3ca0a345fca240678b111dd64efd/images/30.jpeg)

#### 2. Polygon 智能合约（在以太坊上）

Polygon 在以太坊上维护了一组智能合约，它们处理以下内容：

权益证明层的权益管理
授权管理，包括验证者共享
MoreVP 的等离子合约，包括侧链状态的检查点/快照
#### 3. Heimdall 节点

Heimdall 是 PoS 验证节点，它与以太坊上的 Staking 合约协同工作，以在 Polygon 上启用 PoS 机制。我们通过在 Tendermint 共识引擎之上构建并更改签名方案和各种数据结构来实现这一点。它负责区块验证、区块生产者委员会选择、在我们的架构中将侧链区块的表示检查点到以太坊，以及各种其他职责。

Heimdall 层将 Bor 产生的块聚合到 Merkle 树中，并定期将 Merkle 根发布到根链。这种定期发布称为checkpoints. 对于 Bor 上的每几个块，验证器（在 Heimdall 层上）：

- 验证自上一个检查点以来的所有块
- 创建块哈希的 Merkle 树
- 将 Merkle 根发布到主链
  检查点之所以重要，有两个原因：

- 在根链上提供最终确定性
- 在提取资产时提供燃烧证明

该过程可以解释为：

- 从池中选择一部分活跃的验证者作为跨度的块生产者。每个跨度的选择也将得到至少 ⅔ 掌权者的同意。这些块生产者负责创建块并将其广播到网络的其余部分。
  检查点包括在任何给定时间间隔内创建的所有块的根。所有节点都验证相同并将其签名附加到它。
  从验证者集中选出的提议者负责收集特定检查点的所有签名并将其提交到主链上。
  创建区块和提出检查点的责任取决于验证者在整个池中的股权比例。

Heimdall 是我们的 Proof-of-Stake 验证层，它负责将 Plasma 块的表示检查点到我们架构中的主链。我们通过在 Tendermint 共识引擎之上构建并更改签名方案和各种数据结构来实现这一点。

主链 Stake Manager 合约与 Heimdall 节点协同工作，作为 PoS 引擎的无信任权益管理机制，包括选择 Validator 集合、更新验证者等。由于 Staking 实际上是在以太坊智能合约上完成的，我们不仅仅依赖验证者的诚实，而是为这个关键部分继承了以太坊链的安全性。

##### 3.1. 验证人选择
验证者是通过链上拍卖过程选择的，该过程按照此处定义的定期发生 72. 赢得插槽后成为验证者的过程概述如下：

- 在合约上调用StakeFor()函数StakeManager来锁定你的链上状态。
- Bridgeon heimdall 监听这个事件并向 heimdall 广播
- 共识Validator加入heimdall但未激活后。
- ValidatorStartEpoch仅在（此处定义）后才开始验证 39)
- 一旦startEpoch到达验证者就被添加到validator-set并开始参与共识机制


#### 4. bor 节点

Bor 是 Polygon 的区块生产者层——负责将交易聚合成区块的实体。目前，它是一个基本的 Geth 实现，对共识算法进行了自定义更改。

区块生产者通过 Heimdall 上的委员会选择定期改组，持续时间称为spanPolygon 中的 a。块在Bor节点生成，侧链 VM 与 EVM 兼容。在 Bor 上产生的块也由 Heimdall 节点定期验证，并且由 Bor 上一组块的 Merkle 树哈希组成的检查点定期提交给以太坊。

Bor 节点或 Block Producer 实现基本上是侧链运营商。侧链 VM 与 EVM 兼容。目前，它是一个基本的 Geth 实现，对共识算法进行了自定义更改。但是，这将是从头开始构建的，以使其轻巧且专注。

区块生产者是从验证者集中选择的，并出于相同目的使用历史以太坊区块哈希进行洗牌。

##### 4.1 生产者选择

生产者是 BOR 层的区块生产者，他们是根据权益从验证者池中选出的小委员会。

- 验证人根据质押获得插槽，因此如果验证人有 100 个 Matic 代币质押，并且每个插槽为 10，他将总共获得 10 个插槽。
- 所有验证者都有这些插槽 [ A, A, A, B, B, C ]
- 使用历史以太坊区块作为种子，我们对这个数组进行洗牌。
- 使用种子对插槽进行洗牌后，我们得到了这个数组 [ A, B, A, C, B, A, A]
- 现在我们从顶部弹出验证器，例如，如果我们想选择 3 个生产者，我们将生产者设置为 [ A, B, A]
- 因此，下一个跨度的生产者集定义为 [ A: 2, B:1 ]
- 使用这个验证器集和tendermint 的提议者选择算法，我们为 BOR 上的每个 sprint 选择一个生产者。

![](https://raw.githubusercontent.com/the-web3/layer2/79839bb1ee4b3ca0a345fca240678b111dd64efd/images/31.png)

#### 5. 检查点

Matic 的 PoS 系统与其他人的主要区别在于 Matic 不是 Layer 1 平台，它依赖于以太坊作为 Layer 1 结算层。因此，所有的质押机制也需要与以太坊链智能合约同步。

检查点的提议者最初是通过 Tendermint 的加权循环算法选择的。根据检查点提交成功实施进一步的自定义检查。这使我们能够与 Tendermint 提议者选择解耦，并为我们提供仅在以太坊主网上的检查点交易成功时选择提议者的能力。

在 Tendermint 上成功提交检查点是一个两阶段的提交过程；通过上述算法选择的提议者，在提议者字段中发送带有其地址的检查点，所有其他提议者在将其添加到其状态之前对其进行验证。

然后下一个提议者发送一个确认交易来证明之前的检查点交易在以太坊主网中已经成功。每个验证器集更改都将由嵌入到验证器节点的 Heimdall 上的验证器节点中继。这允许 Heimdall 始终与以太坊主链上的 Matic 合约状态保持同步。

部署在主链上的 Matic 合约被认为是最终的真相来源，因此所有验证都是通过查询主链合约来完成的。

### 三. 总结

本文咱们是初略的说一下 ploygon, 更多关于 polygon 的东西可以去阅读官方文档和源码