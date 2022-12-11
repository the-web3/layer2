# Arbitrum 的 Nitro 项目启动和交易执行源码解析

## 一. 交易执行细节

### 1 合约部署

在节点启动之前，会先去部署 L1 上的合约，L2 上的合约是预部署的，接口部分有 solidity 来编写，实现是由 go 代码来实现的，合约实现对应的 go 代码为 precompiles 项目的代码目录，接口定义为 /nitro/contracts/src/precompiles 的代码。

### 2.1 L1 上的合约部署

合约部署代码入口为`/nitro/cmd/deploy/deploy.go`,
调用链：`arbnode.DeployOnL1` -> `deployRollupCreator` -> `deployBridgeCreator deployChallengeFactory DeployRollupAdminLogic DeployRollupUserLogic` 等
核心代码示例如下

```
 deployPtr, err := arbnode.DeployOnL1(
   ctx,
   l1client,
   l1TransactionOpts,
   sequencerAddress,
   *authorizevalidators,
   headerReaderConfig,
   machineConfig,
   arbnode.GenerateRollupConfig(*prod, common.HexToHash(*wasmmoduleroot), ownerAddress, l2ChainId, loserEscrowAddress),
)
```

```
rollupCreator, rollupCreatorAddress, validatorUtils, validatorWalletCreator, err := deployRollupCreator(ctx, l1Reader, deployAuth)
if err != nil {
   return nil, fmt.Errorf("error deploying rollup creator: %w", err)
}
```

```
func deployRollupCreator(ctx context.Context, l1Reader *headerreader.HeaderReader, auth *bind.TransactOpts) (*rollupgen.RollupCreator, common.Address, common.Address, common.Address, error) {
   bridgeCreator, err := deployBridgeCreator(ctx, l1Reader, auth)
   if err != nil {
      return nil, common.Address{}, common.Address{}, common.Address{}, err
   }

   ospEntryAddr, challengeManagerAddr, err := deployChallengeFactory(ctx, l1Reader, auth)
   if err != nil {
      return nil, common.Address{}, common.Address{}, common.Address{}, err
   }
   ......
```

### 1.2 L2 上的合约部署

预部署合约,  执行 `docker exec nitro_sequencer_1 cat /config/deployment.json` 的时候会去部署一些合约, 部署生成的合约如下:

```
{
  "l1Network": {
    "blockTime": 10,
    "chainID": 1337,
    "explorerUrl": "",
    "isCustom": true,
    "name": "EthLocal",
    "partnerChainIDs": [
      412346,
      412346
    ],
    "rpcURL": "http://localhost:8545"
  },
  "l2Network": {
    "chainID": 412346,
    "confirmPeriodBlocks": 20,
    "ethBridge": {
      "bridge": "0x815b0ce130aa4c1db18ba0c4c92fcfbf6062ab08",
      "inbox": "0x07061a11d42da58c7bd08ddbf4ef6e60232ba966",
      "outbox": "0xE7098C657B3Ee7c92939f20A4E308efCdd656163",
      "rollup": "0x532016aa3f129f35214559723aa7a0faa435f7ce",
      "sequencerInbox": "0xda7b4b25cac35e41f62cf79744b7e4d50f177b64"
    },
    "explorerUrl": "",
    "isArbitrum": true,
    "isCustom": true,
    "name": "ArbLocal",
    "partnerChainID": 1337,
    "rpcURL": "http://localhost:8547",
    "retryableLifetimeSeconds": 604800,
    "depositTimeout": 900000,
    "tokenBridge": {
      "l1CustomGateway": "0xDe67138B609Fbca38FcC2673Bbc5E33d26C5B584",
      "l1ERC20Gateway": "0x0Bdb0992B3872DF911260BfB60D72607eb22d5d4",
      "l1GatewayRouter": "0x4535771b8D5C43100f126EdACfEc7eb60d391312",
      "l1MultiCall": "0x36BeF5fD671f2aA8686023dE4797A7dae3082D5F",
      "l1ProxyAdmin": "0xF7818cd5f5Dc379965fD1C66b36C0C4D788E7cDB",
      "l1Weth": "0x24067223381F042fF36fb87818196dB4D2C56E9B",
      "l1WethGateway": "0xBa3d12E370a4b592AAF0CA1EF09971D196c27aAd",
      "l2CustomGateway": "0x0Bdb0992B3872DF911260BfB60D72607eb22d5d4",
      "l2ERC20Gateway": "0x4535771b8D5C43100f126EdACfEc7eb60d391312",
      "l2GatewayRouter": "0xF7818cd5f5Dc379965fD1C66b36C0C4D788E7cDB",
      "l2Multicall": "0x14a00f381A870878Ae6A055C656520FF2Cbff985",
      "l2ProxyAdmin": "0x8c366Cfd28bC93729e14Da4fcf94d20862A7f266",
      "l2Weth": "0x9b890cA9dE3D317b165afA7DFb8C65f2e4c95C20",
      "l2WethGateway": "0xDe67138B609Fbca38FcC2673Bbc5E33d26C5B584"
    }
  }
}
```

温馨提示：看代码的时候建议执行一下 make contract，这样会生成合约对应的 go 调用代码，代码生成在  `nitro/solgen` 目录下。

### 2. 节点启动的流程

节点启动过程中会生成四个 Json 文件，用来区分 `validator`, `sequencer` 和 `poster`, 四个文件的名字为 `unsafe_staker_config`, `validator_config.json`, `sequencer_config`和 `poster_config`，这四个文件里面包含各自节点相关的配置，除此之外，还会生成deployment.json 文件，该文件用户合约的部署；文件由 testnode-scripts 下面的代码生成，这里面的代码会单独形成一个 docker 镜像，节点启动过程中会驱使该镜像去生成相关的配置文件。节点启动过程中，`testnode-scripts` 下面的 `ethcommands.ts` 会去运行 3 笔交易，L1<->L2, L2 和 L2 自己转账。需要发布 L1->L2, L2->L 相关的交易时，需要运行 `docker exec nitro_sequencer_1 cat /config/deployment.json` 不是相关的合约。

### 3. L2 上的交易

L2 复用了 geth 客户端，所以交易发进来的时候和 ETH 是一致的，调用了 SendTransaction 和 SendRawTransaction 接口进交易发进来，然后交易进入到 SubmitTransaction, 通过 
b.SendTx(ctx, tx) 

```
func (a *APIBackend) SendTx(ctx context.Context, signedTx *types.Transaction) error {
   return a.b.EnqueueL2Message(ctx, signedTx)
}
```

接下来交易进入 `b.arb.PublishTransaction(ctx, tx)` , 在 PublishTransaction 阶段，按照如下顺序执行了 `ArbInterface PublishTransaction`->`TxPreChecker PublishTransaction`->`Sequencer PublishTransaction`

`PreCheckTx`: 主要是对交易进行gas, Nonce, balance 等简单的检查，检查完成之后交易进入到 Sequencer PublishTransaction 阶段，到这个阶段，交易并没有完全执行完成，交易还需要 rollup 到一层，那么怎么 rollup 的，我们来看看下面的流程。

交易的 Rollup 由BatchPoster 将交易批量提交到 L2 上去，函数执行流程如下：`arbnode: Start()`->`BatchPoster.Start`->`b.maybePostSequencerBatch(ctx, batchSeqNum)`->`b.inboxContract.AddSequencerL2BatchFromOrigin`->`l1Reader.Client().SendTransaction(ctx, tx)`

BatchPoster 回去组装相关的 RollUp 的交易，通过 SendTransaction 发到 L1 上，整个过程涉及到交易的获取组装，交易压缩和把交易发送到一层。

### 4. L1->L2 交易

#### 4.1 概述

简单的 Eth 充值存在一种特殊的消息类型；即，将 Eth 从 L1 发送到 L2。Eth 可以通过调用Inbox的depositEth方法来存入。如果 L1 调用方是 EOA，则 Eth 将存入 L2 上相同的 EOA 地址；L1 调用者是一个合约，资金将存入合约的别名地址。值得注意的是，将 Eth viadepositEth存入 L2 上的合约不会触发合约的回退功能。
原则上，可重试的票据也可以用于存入以太币；这可能比特殊的 eth-deposit 消息类型更可取，例如，如果目标地址需要更大的灵活性，或者如果想要在 L2 侧触发回退功能。

虽然 Retryables 和 Eth 存款必须通过延迟收件箱提交，但原则上，任何消息都可以通过这种方式包含；这是确保 Arbitrum 链保持审查阻力的必要手段，即使 Sequencer 行为不端（参见The Sequencer and Censorship Resistance）。但是，在普通/愉快的情况下，期望/建议是客户端仅将延迟收件箱用于 Retryables 和 Eth 存款，并通过 Sequencer 处理所有其他消息。

#### 4.2. 执行细节

##### 4.2.1. L1 -> L2 (以 ETH充值 为例)

L1 合约侧

- 调用合约方法入口 Inbox.depositEth(),  根据合约逻辑该操作会触发事件 event:InboxMessageDelivered(msgNum, _messageData) 和 event:MessageDelivered
- 合约逻辑内部调用bridge.enqueueDelayedMessage() 将该交易信息写入到 bridge 合约的 delayedInboxAccs 数组中，并将 value 传递给 Bridge 合约(lock ether)
- 此时 ETH balance 锁定在 Bridge 合约中

L2 Sequencer 侧

- Arbnode 从 Inbox合约 中查询消息事件。arbnode.InboxReader 以设定高度( 默认100) 范围读取 event:MessageDelivered ，同时根据获取的 events 的最大、最小高度查询 event:InboxMessageDelivered 事件。结合两者信息解析到 DelayedInboxMessage 对象中(记为: messages) 。
- InboxTracker 将上述message 对象写入本地存储中，并更新状态数据. 
-  delayed_sequencer 根据本地 DelayedMessagesRead 缓存的高度作为起始索引位置( pos )，对比本地 db 中缓存的 Message 计数. 循环从 db 缓存按照 pos 为索引获取 inconming msgs 数据并解析为 L1IncomingMessage 数组，进一步由 TransactionStreamer 在 SequenceDelayedMessages方法中将消息事件进行排序，完成排序后调用 createBlocks打包出块。打包区块执行逻辑过程中，通过 ParseL2Transactions 按照L1消息类型执行具体的实现逻辑，在这里是构造充值交易（mint，充值交易一笔一个块，充值的message 会被解析成一笔L2交易）
- 最后在执行交易的过程中，调用 TxProcessor.StartTxHook，按照 types.ArbitrumDepositTx 的交易类型 mint 指定的 balance（stateDB.AddBalance(to, amount) 的方式）

##### 4.2.2.L1 -> L2 (以 ERC20充值 为例, Deposit ERC20)

L1 合约侧:

- 合约入口为L1GatewayRouter.outboundTransfer 或 L1GatewayRouter.outboundTransferCustomRefund ，方法逻辑中执行了 L2 gasLimit、L2 maxFeePerGas 的 txFee 的检查
-  在父类 GatewayRoute.outboundTransfer 中，根据 token 类型获取 Gateway 实现后，调用具体实例的outboundTransfer方法
- 在L1ArbitrumGateway.outboundTransfer实例中，根据 l2BeaconProxyFactory 地址、nonce、和 cloneableProxyHash 计算 L2-ERC20token 地址（同deploy流程计算方式）
- 调用 outboundEscrowTransfer 方法 对ERC20 Token 执行 transfer 操作，将 ERC20 Token 锁定在 L1ArbitrumGateway 中
- 调用 getOutboundCalldata 方法构造 L2 上 finalizeInboundTransfer调用的calldata，在逻辑方法 L1ArbitrumMessenger.sendTxToL2CustomRefund中，调用Inbox.createRetryableTicket方法，触发事件 event:TxToL2 event:InboxMessageDelivered
- 进而调用 Inbox._deliverMessage 发起一笔充值，后续操作流程同 : ETH deposit L2 Sequencer 侧
 
 L2 合约侧 :
 
- Sequencer 定序完成进行交易执行会根据以上步骤所得 abi 调用到 L2ArbitrumGateway.finalizeInboundTransfer，触发事件 event:DepositFinalized
- 执行逻辑中，检查 L2-ERC20Token 合约地址，检查 L2 Token / L1 Token 的绑定关系（验证失败则执行 triggerWithdrawal 方法将 token 返还给 L1） 
- 执行逻辑中，调用 inboundEscrowTransfer 方法，调用 ERC20 的 bridgeMint 方法 mint 给指定用户

### 5.  L2->L1 交易

#### 5.1. 概述

Arbitrum 的发件箱系统允许任意 L2 到 L1 合约调用；即，从 L2 发起的消息最终在 L1 上执行解决。L2-to-L1 消息（又名“传出”消息）与 Arbitrum 的L1-to-L2 消息（可重试）有许多共同点，“相反”，尽管有一些差异。

协议流: Arbitrum 链的 L2 状态的一部分——因此，每个 RBlock 中断言的部分——是链历史中所有 L2 到 L1 消息的 Merkle 根。在确认断言的 RBlock 后（通常在断言后约 1 周），此 Merkle 根将发布在Outbox合约的 L1 上。然后，发件箱合约允许用户执行他们的消息——验证 Merkle 包含证明，并跟踪哪些 L2 到 L1 消息已经被使用。

客户流程: 从客户端的角度来看，L2 到 L1 的消息以调用 L2ArbSys预编译合约的sendTxToL1方法开始。一旦消息包含在断言中（通常在约 1 小时内）并确认断言（通常约 1 周），任何客户端都可以执行该消息。为此，客户端首先通过调用 Arbitrum 链的“虚拟”/precompile-esque**NodeInterface合约的constructOutboxProof方法来检索证明数据。然后可以在Outbox'executeTransaction方法中使用返回的数据来执行 L1 执行。

协议设计细节: 发件箱系统设计中的一个重要特征是调用confirmNode具有恒定的开销。要求confirmNode只更新固定大小的传出消息根哈希，用户自己执行最后一步执行，达到了这个目标；即，无论根中传出消息的数量，或者在 L1 上执行它们的 gas 成本，确认节点的成本保持不变；这确保了处理的 RBlock 确认不会被破坏。

与可以选择为自动 L2 执行提供 Ether 的 Retryables 不同，传出消息不能提供协议内自动 L1 执行，原因很简单，以太坊本身不提供预定执行功能。但是，原则上可以构建与发件箱交互的应用层合约，以提供有点类似的“执行市场”功能，用于外包最终的 L1 执行步骤。

传出消息和 Retryables 之间的另一个区别是 Retryables 的生命周期有限，在此之前它们必须被赎回（或显式延长它们的生命周期），而 L2 到 L1 消息存储在 L1 状态，因此永久存在/没有截止日期他们必须被处决。 可以执行传出消息之前的长达一周的延迟期是 Arbitrum Rollup 或任何 Optimistic Rollup 样式 L2 的本质和基础；交易在链上发布的那一刻，任何观察者都可以预测其结果；然而，为了让以太坊本身接受它的结果，协议必须给 Arbitrum 验证器时间，以便在需要时检测并证明错误。

我们称之为NodeInterface“虚拟”合同；它的方法可以通过调用访问0x00000000000000000000000000000000000000C8，但它并不真正存在于链上。它并不是真正的预编译，但其行为很像无法接收来自其他合约的调用的预编译。这是一个可爱的技巧，让我们无需实现自定义 RPC 即可提供特定于 Arbitrum 的数据。

#### 5.2. 执行细节

#### 5.2.1. L2 -> L1 (以 ETH提现 为例，参考 SDK 中的提现操作)

L2 网络侧

- 调用预编译合约 ARB_SYS(0x0000000000000000000000000000000000000064)中的withdrawEth 方法，在 EVM instance 中，判断合约类型，预编译合约会调用底层的 Call 方法，将合约调用的请求转为预编译和约go代码执行，最后执行到 precompiles/ArbSys.go:198 WithdrawEth方法
- 在 WithdrawEth 方法中，调用跨网络消息传递的入口函数SendTxToL1，其中的util.BurnBalance 方法将 withdraw 请求的 ETH amount 燃烧掉（from balance减，to == nil balance 不加），触发事件 event:SendMerkleUpdate和 event:L2ToL1Tx
- 此时，用户可以根据L2ToL1Tx消息中的参数，调用预编译合约NODE_INTERFACE(0x00000000000000000000000000000000000000C8)中的 ConstructOutboxProof方法构造提款证明供 L1合约 Outbox.executeTransaction 方法使用

L1 合约侧（参考 SDK 中的提现操作）

- 调用 L1 合约方法 Outbox.executeTransaction，参数包含上述 L2 上提现的 proof 等，触发事件 event:OutBoxTransactionExecuted，event:BridgeCallTriggered
- 合约逻辑内部，调用 Outbox.recordOutputAsSpent 方法更新了 proof 的使用状态，进而调用 Bridge 合约的 Bridge.executeCall方法，此时将 ETH 转给提款地址

#### 5.2.2. L2 -> L1 (以 ERC20提现为例, Withdraw ERC20)

L2 合约侧

-  提现的合约入口为 L2ArbitrumGateway.outboundTransfer 方法, 逻辑中检查了指定 L1 Token的 L2 ERC20 Token 是否存在，检查了 L2-ERC20 Token的绑定关系，并调用 outboundEscrowTransfer 将 L2 Token 给 burn 掉
-  后续方法调用栈为: triggerWithdrawal -> createOutboundTx -> sendTxToL1 ->  ArbSys.sendTxToL1
-  ArbSys.sendTxToL1 执行会触发事件 event: SendMerkleUpdate和 event:L2ToL1Tx
- 此时，用户可以根据L2ToL1Tx消息中的参数，调用预编译合约NODE_INTERFACE(0x00000000000000000000000000000000000000C8)中的 ConstructOutboxProof方法构造提款证明供 L1合约 Outbox.executeTransaction 方法使用

L1 合约侧

- Outbox.executeTransaction 最终将调用 L1ArbitrumGateway.finalizeInboundTransfer 进行 L1 ERC20 的转账, 提现流程至此结束.

## 二. 交易提交细节深入

### 1. 交易提交过程:  

交易的 Rollup 由BatchPoster 将交易批量提交到 L1 上去，函数执行流程如下：arbnode: Start()-> BatchPoster.Start->b.maybePostSequencerBatch(ctx, batchSeqNum)->b.inboxContract.AddSequencerL2BatchFromOrigin->l1Reader.Client().SendTransaction(ctx, tx)

BatchPoster 回去组装相关的 RollUp 的交易，通过 SendTransaction 发到 L1 上，整个过程涉及到交易的获取组装，交易压缩和把交易发送到一层。

### 2. 合约事件的处理与监听

sequencer_inbox.go 和 delayed.go 里面会去监听合约事件，同步相关的事件下来进行交易的处理，处理事件ID 如下：

`sequencer_inbox.go`

```
var messageDeliveredID common.Hash
var inboxMessageDeliveredID common.Hash
var inboxMessageFromOriginID common.Hash
var l2MessageFromOriginCallABI abi.Method
```

`delayed.go`

```
var sequencerBridgeABI *abi.ABI
var batchDeliveredID common.Hash
var addSequencerL2BatchFromOriginCallABI abi.Method
var sequencerBatchDataABI abi.Event
```

此部分的入口函数在 `inbox_reader.go` 里面的 run 函数, 

```
delayedMessages, err := ir.delayedBridge.LookupMessagesInRange(ctx, from, to)
if err != nil {
   return err
}
```

`LookupMessagesInRange` 进入到 `delay.go` 里面去解析整个延迟收件向里面的消息，具体细节请参考代码文件

```
sequencerBatches, err := ir.sequencerInbox.LookupBatchesInRange(ctx, from, to)
if err != nil {
   return err
}
if !ir.caughtUp && to.Cmp(currentHeight) == 0 {
   // TODO better caught up tracking
   ir.caughtUp = true
   ir.caughtUpChan <- true
}
```

LookupBatchesInRange 进入 sequencer_inbox.go 里面进行时间的解析，具体细节可以进入到代码里面查看。

处理完成的消息将由 transaction_streamer.go 处理生成区块和判断区块是否需要重组，细节的执行流程如下：

`Sequencer: Start`-> `s.createBlock(ctx)`->`s.txStreamer.SequenceTransactions(header, txes, hooks)`->`arbos.ProduceBlockAdvanced`(进入arbios 的块处理逻辑)->`validator.NewBlock(block, lastBlockHeader, msgWithMeta)`

### 3. 区块产生逻辑整理

ProduceBlockAdvanced-> types.NewBlock(进入 geth 块处理程序，此处先生成临时处理块，然后再生成落库的块)
这里面有几点值得注意的地方是：
A. 一个块里面包含两笔交易的解释

- 新版的 nitro 里面加了一个 startBlock 的特性，两笔交易里面有一笔交易和这个相关(在所有其他人之前添加一个 tx 以修改状态（更新 L1 块编号、定价池等）)，另一个交易是用户发过来的交易，处理核心代码逻辑如下：

```
startTx := InternalTxStartBlock(chainConfig.ChainID, l1Header.L1BaseFee, l1BlockNum, header, lastBlockHeader)
txes = append(types.Transactions{types.NewTx(startTx)}, txes...)
```

- 区块的生成另一条路径，通过 validator.NewBlock 最终回到 block_processor 处理流程，这里生成的块是经过 validator 验证过的，生成路径如下：
`validator.NewBlock`-> `v.prepareBlock`-> `BlockDataForValidation`->`RecordBlockCreation`->`arbos.ProduceBlock`->`ProduceBlockAdvanced`

- 在最终生成区块的时候，会通过 FinalizeBlock 将 Merkle root 和 size 更新到区块头

### 4.  交易产生的逻辑整理

交易的核心处理逻辑在 tx_processor 里面，核心代码在 StartTxHook 内部，L2 上的交易通过 PublishTransaction 发布到 arbnode 里面处理，再由 geth 的 hook 函数处理。L1->L2, L2->L1 通过 pareMessage 的方式把 message 转换为 L2 上的 Tx,  细节的代码可以参考 tx_processor 和 incomingmessage.go 里面的处理逻辑,incomingmessage 处理的消息有一下几种类别

```
L1MessageType_L2Message             = 3
L1MessageType_EndOfBlock            = 6
L1MessageType_L2FundedByL1          = 7
L1MessageType_RollupEvent           = 8
L1MessageType_SubmitRetryable       = 9
L1MessageType_BatchForGasEstimation = 10 // probably won't use this in practice
L1MessageType_Initialize            = 11
L1MessageType_EthDeposit            = 12
L1MessageType_BatchPostingReport    = 13
L1MessageType_Invalid               = 0xFF
```

```
const (
   L2MessageKind_UnsignedUserTx  = 0
   L2MessageKind_ContractTx      = 1
   L2MessageKind_NonmutatingCall = 2
   L2MessageKind_Batch           = 3
   L2MessageKind_SignedTx        = 4
   // 5 is reserved
   L2MessageKind_Heartbeat          = 6 // deprecated
   L2MessageKind_SignedCompressedTx = 7
   // 8 is reserved for BLS signed batch
)
```

上面各种类别的消息处理逻辑有细微的差距，具体可以查看 `incomingmessage` 内的 `ParseIncomingL1Message` `ParseL2Transactions`  `ParseInitMessage` 等方法，这些方法的作用就是将 Message 转换成相关的交易, 解析交易相关的函数由生成区块的函数内部调起进入。
交易处理逻辑需要注意的一个点：

- Tx 处理分为不同的类别，Retryables 的交易如果不被处理，是可以不断地修改延长交易的处理周期的，核心代码如下：

```
func (rs *RetryableState) Keepalive(
   ticketId common.Hash,
   currentTimestamp,
   limitBeforeAdd,
   timeToAdd uint64,
) (uint64, error) {
   retryable, err := rs.OpenRetryable(ticketId, currentTimestamp)
   if err != nil {
      return 0, err
   }
   if retryable == nil {
      return 0, errors.New("ticketId not found")
   }
   timeout, err := retryable.CalculateTimeout()
   if err != nil {
      return 0, err
   }
   if timeout > limitBeforeAdd {
      return 0, errors.New("timeout too far into the future")
   }

   // Add a duplicate entry to the end of the queue (only the last one deletes the retryable)
   err = rs.TimeoutQueue.Put(retryable.id)
   if err != nil {
      return 0, err
   }
   if _, err := retryable.timeoutWindowsLeft.Increment(); err != nil {
      return 0, err
   }
   newTimeout := timeout + RetryableLifetimeSeconds

   // Pay in advance for the work needed to reap the duplicate from the timeout queue
   return newTimeout, rs.retryables.Burner().Burn(RetryableReapPrice)
}
```

### 5 区块 ReorgBlock 逻辑

reorg 触发的条件：L1 层发生交易回滚或者分叉等时，可能会导致 Post 上去的数据发生问题，因此二层上的数据也需要回滚，这种时候就会产生区块的 reorg，reorg 需要两个块，一个旧链和一个新链，并将重建这些块并将它们插入到新的规范链中，并累积潜在的丢失交易并发布有关它们的事件。 注意这里不会处理新的 head 块，调用者需要在外部处理它。

## 三. arbnode  和 arbos 代码解读
### 1. Arbnode

- arb_interface：TransactionPublisher 接口层，衔接 go-ethereum
- batch_poster：批量提交交易到 L1 的相关的代码
- delayed 和 delayed_sequencer: 延迟收件箱相关的消息处理
- sequencer 和 sequencer_inbox：sequencer 收件箱相关的事件处理
- inbox_reader：链接延迟收件箱和 sequencer 收件箱的模块代码
- transaction_streamer：交易生成区块和重组的逻辑
- inbox_tracker：本地数据库收件箱的管理
- tx_pre_checker：交易的预检查

### 2. Arbos

- tx_processor 交易 hook 处理的核心逻辑
- block_processor 二层区块链处理逻辑，终进入到 geth 去生成区块
- retryables： retryables 交易的特性处理代码，如果可以一直更新 retryables 交易的活性问题
- merkleAccumulator：merkle 处理逻辑
- incomingmessage： 各种交易的 message 处理
- internal_tx： 内部交易处理逻辑，例如：startblock 相关的交易处理
- l1pricing 和 l2pricing：L1 和 L2 的 price 计算模型
- burn：和资产的 burn 相关
- blockhash：区块 Hash 处理逻辑
- arbosState：arbos state 管理


