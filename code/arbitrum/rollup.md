# Arbitrum 的 Nitro 项目 Rollup 细节深入

关于 rollup 向一层提交交易的我们[上面文章](https://chaineye.info/arctle_detail?article_id=584)已经由提到，此处主要主要剖析 rollup 合约，rollup watcher 和欺诈证明之间的交互和细节的逻辑整理。

### 一. validator 各子模块代码解析

- block_challenge_backend 和 challenge_manager 区块挑战相关的代码
- block_validator：区块验证逻辑，这里是衔接 arbnode 和 geth 的区块验证模块
- rollup_watcher：rollup 合约交易事件的观察者

### 二. 2. Rollup 细节

汇总协议跟踪汇总块链——为了清楚起见，我们将这些称为“RBlocks”。它们与第 1 层以太坊区块不同，也与第 2 层 Nitro 区块不同。您可以将 RBlocks 视为形成一个单独的链，由 Arbitrum 汇总协议管理和监督。

关于汇总块链相关的解释请参考：nitro/inside-arbitrum-nitro.md at master · OffchainLabs/nitro

rollup_watcher 会去监听 rollup 相关的事件，监听的事件如下：

```
var rollupInitializedID common.Hash
var nodeCreatedID common.Hash
var challengeCreatedID common.Hash
```

这些事件分别由 LookupCreation LookupNode LookupNodeChildren 和 LookupChallengedNode 等方法处理，L1Validator 将这些调用这里面的方法获取相关的合约事件信息验证。

RollupNode 定义

```
 struct Node {
    // Hash of the state of the chain as of this node
    bytes32 stateHash;
    // Hash of the data that can be challenged
    bytes32 challengeHash;
    // Hash of the data that will be committed if this node is confirmed
    bytes32 confirmData;
    // Index of the node previous to this one
    uint64 prevNum;
    // Deadline at which this node can be confirmed
    uint64 deadlineBlock;
    // Deadline at which a child of this node can be confirmed
    uint64 noChildConfirmedBeforeBlock;
    // Number of stakers staked on this node. This includes real stakers and zombies
    uint64 stakerCount;
    // Number of stakers staked on a child node. This includes real stakers and zombies
    uint64 childStakerCount;
    // This value starts at zero and is set to a value when the first child is created. After that it is constant until the node is destroyed or the owner destroys pending nodes
    uint64 firstChildBlock;
    // The number of the latest child of this node to be created
    uint64 latestChildNumber;
    // The block number when this node was created
    uint64 createdAtBlock;
    // A hash of all the data needed to determine this node's validity, to protect against reorgs
    bytes32 nodeHash;
}
```

参加 rollup 的角色：分别由 WatchtowerStrategy，DefensiveStrategy， StakeLatestStrategy 和 MakeNodesStrategy
代码定义如下，该代码位于 staker.go 里面：

```
const (
   // Watchtower: don't do anything on L1, but log if there's a bad assertion
   WatchtowerStrategy StakerStrategy = iota
   // Defensive: stake if there's a bad assertion
   DefensiveStrategy
   // Stake latest: stay staked on the latest node, challenging bad assertions
   StakeLatestStrategy
   // Make nodes: continually create new nodes, challenging bad assertions
   MakeNodesStrategy
)
```

- Staker start 任务启动之后，先会去更新认证区块的 wasm moudle root, 函数调用如下

```
err := s.updateBlockValidatorModuleRoot(ctx)
```

updateBlockValidatorModuleRoot 里面会去从 rollup 的中取到 WasmModuleRoot 更新到 blockValidator 里面， 当然 l1 validator 初始化的时候也会去更新相应的 WasmModuleRoot，并把 WasmModuleRoot 设置成 blockValidator 当前的 ModuleRoot，以供验证使用。

-  获取到 callOpts 的信息，供下面的函数调用，该信息包含

```
type CallOpts struct {
   Pending     bool            // Whether to operate on the pending state or the last known one
   From        common.Address  // Optional the sender address, otherwise the first account is used
   BlockNumber *big.Int        // Optional the block number on which the call should be performed
   Context     context.Context // Network context to support cancellation and timeouts (nil = no timeout)
}
```

- 清除构建的交易

```
func (b *ValidatorTxBuilder) ClearTransactions() {
   b.transactions = nil
}
```

- 从 rollup 的 StakerMap 里面获取最新质押者的信息，包含挑战的信息
- 从 validatorUtils 中拿到最新的质押数据信息，调用的合约函数如下：

```
function latestStaked(IRollupCore rollup, address staker)
    external
    view
    returns (uint64, Node memory)
{
    uint64 num = rollup.latestStakedNode(staker);
    if (num == 0) {
        num = rollup.latestConfirmed();
    }
    Node memory node = rollup.getNode(num);
    return (num, node);
}
```

- 判断 Rblocks 是否分叉，validatorUtils.AreUnresolvedNodesLinear -> _ValidatorUtils.contract.Call(opts, &out, "areUnresolvedNodesLinear", rollup)最终调用到这个合约函数进行判断

```
function areUnresolvedNodesLinear(IRollupCore rollup) external view returns (bool) {
    uint256 end = rollup.latestNodeCreated();
    for (uint64 i = rollup.firstUnresolvedNode(); i <= end; i++) {
        if (i > 0 && rollup.getNode(i).prevNum != i - 1) {
            return false;
        }
    }
    return true;
}
```

- 如果发生了分叉，effectiveStrategy将由 DefensiveStrategy 切换成 StakeLatestStrategy，否则将按照 WatchtowerStrategy 往下执行
代码如下：

```
nodesLinear, err := s.validatorUtils.AreUnresolvedNodesLinear(callOpts, s.rollupAddress)
if err != nil {
   return nil, err
}
if !nodesLinear {
   log.Warn("rollup assertion fork detected")
   if effectiveStrategy == DefensiveStrategy {
      effectiveStrategy = StakeLatestStrategy
   }
   s.inactiveLastCheckedNode = nil
}
```

s.bringActiveUntilNode 存储并且 info.LatestStakedNode < s.bringActiveUntilNode, effectiveStrategy 会由 DefensiveStrategy 切换到 StakeLatestStrategy

- 获取 rollup 链最新确认的节点，调用栈 s.rollup.LatestConfirmed(callOpts)->latestConfirmed, 合约函数细节如下：

```
/// @return Index of the latest confirmed node
function latestConfirmed() public view override returns (uint64) {
    return _latestConfirmed;
}
```

- 判断是否需要做质押提升, isRequiredStakeElevated获取CurrentRequiredStake, 和 baseStake 判定是否要做质押提升，(获取现在的 stake, 最终进入 currentRequiredStake 函数， baseStake 进入合约处理逻辑)
- effectiveStrategy 为 MakeNodesStrategy 策略，或者 effectiveStrategy 为 StakeLatestStrategy 并且 rawInfo(staker 的信息为 nil) 并且需要提升质押的情况，会去处理以下这些逻辑
  - 处理超时的挑战 resolveTimedOutChallenges, 进入处理挑战的细节，下面的内容会说到
  - 处理 unresolve 节点， unresolve 处理逻辑：处理节点的类别有三种，代码如下：

```
const (
   CONFIRM_TYPE_NONE ConfirmType = iota
   CONFIRM_TYPE_VALID
   CONFIRM_TYPE_INVALID
)
```

处理逻辑是先去获取 confirmType 和 unresolvedNodeIndex，分别调用  ValidatorUtils 里面的 checkDecidableNextNode 函数和 rollup 的 firstUnresolvedNode，如果是 CONFIRM_TYPE_INVALID 类别的，直接拒绝下一个节点，如果是 CONFIRM_TYPE_VALID，查找并确认下一个节点。核心逻辑代码如下

```
func (v *L1Validator) resolveNextNode(ctx context.Context, info *StakerInfo, latestConfirmedNode *uint64) (bool, error) {
   callOpts := v.getCallOpts(ctx)
   confirmType, err := v.validatorUtils.CheckDecidableNextNode(callOpts, v.rollupAddress)
   if err != nil {
      return false, err
   }
   unresolvedNodeIndex, err := v.rollup.FirstUnresolvedNode(callOpts)
   if err != nil {
      return false, err
   }
   switch ConfirmType(confirmType) {
   case CONFIRM_TYPE_INVALID:
      addr := v.wallet.Address()
      if info == nil || addr == nil || info.LatestStakedNode <= unresolvedNodeIndex {
         // We aren't an example of someone staked on a competitor
         return false, nil
      }
      log.Info("rejecing node", "node", unresolvedNodeIndex)
      _, err = v.rollup.RejectNextNode(v.builder.Auth(ctx), *addr)
      return true, err
   case CONFIRM_TYPE_VALID:
      nodeInfo, err := v.rollup.LookupNode(ctx, unresolvedNodeIndex)
      if err != nil {
         return false, err
      }
      afterGs := nodeInfo.AfterState().GlobalState
      _, err = v.rollup.ConfirmNextNode(v.builder.Auth(ctx), afterGs.BlockHash, afterGs.SendRoot)
      if err != nil {
         return false, err
      }
      *latestConfirmedNode = unresolvedNodeIndex
      return true, nil
   default:
      return false, nil
   }
}
```

- 若 stakeInfo 的不为 nil, 并且最新节点 stake 节点小于最新确认节点，effectiveStrategy 为 WatchtowerStrategy 和 DefensiveStrategy，退还当前在最新确认节点上或之前质押的质押者的质押币，从汇总链中提取发件人拥有的未承诺资金, 调用方法为 s.rollup.ReturnOldDeposit 和 s.rollup.WithdrawStakerFunds，最好执行交易 ExecuteTransactions
- walletAddressOrZero != (common.Address{})条件下继续做自己取回处理
- 接下来是解决冲突 handleConflict, 条件满足的情况下回去创建挑战，然后进入挑战细节
- rawInfo != nil || !resolvingNode || !requiredStakeElevated, 提升质押，
提升质押的细节逻辑

```
func (s *Staker) advanceStake(ctx context.Context, info *OurStakerInfo, effectiveStrategy StakerStrategy) error {
   active := effectiveStrategy >= StakeLatestStrategy
   action, wrongNodesExist, err := s.generateNodeAction(ctx, info, effectiveStrategy, s.config.MakeAssertionInterval)
   if err != nil {
      return err
   }
   if wrongNodesExist && effectiveStrategy == WatchtowerStrategy {
      log.Error("found incorrect assertion in watchtower mode")
   }
   if action == nil {
      info.CanProgress = false
      return nil
   }

   switch action := action.(type) {
   case createNodeAction:
      if wrongNodesExist && s.config.DisableChallenge {
         log.Error("refusing to challenge assertion as config disables challenges")
         info.CanProgress = false
         return nil
      }
      if !active {
         if wrongNodesExist && effectiveStrategy >= DefensiveStrategy {
            log.Warn("bringing defensive validator online because of incorrect assertion")
            s.bringActiveUntilNode = info.LatestStakedNode + 1
         }
         info.CanProgress = false
         return nil
      }

      // Details are already logged with more details in generateNodeAction
      info.CanProgress = false
      info.LatestStakedNode = 0
      info.LatestStakedNodeHash = action.hash

      // We'll return early if we already havea stake
      if info.StakeExists {
         _, err = s.rollup.StakeOnNewNode(s.builder.Auth(ctx), action.assertion.AsSolidityStruct(), action.hash, action.prevInboxMaxCount)
         return err
      }

      // If we have no stake yet, we'll put one down
      stakeAmount, err := s.rollup.CurrentRequiredStake(s.getCallOpts(ctx))
      if err != nil {
         return err
      }
      _, err = s.rollup.NewStakeOnNewNode(
         s.builder.AuthWithAmount(ctx, stakeAmount),
         action.assertion.AsSolidityStruct(),
         action.hash,
         action.prevInboxMaxCount,
      )
      if err != nil {
         return err
      }
      info.StakeExists = true
      return nil
   case existingNodeAction:
      info.LatestStakedNode = action.number
      info.LatestStakedNodeHash = action.hash
      if !active {
         if wrongNodesExist && effectiveStrategy >= DefensiveStrategy {
            log.Warn("bringing defensive validator online because of incorrect assertion")
            s.bringActiveUntilNode = action.number
            info.CanProgress = false
         } else {
            s.inactiveLastCheckedNode = &nodeAndHash{
               id:   action.number,
               hash: action.hash,
            }
         }
         return nil
      }
      log.Info("staking on existing node", "node", action.number)
      // We'll return early if we already havea stake
      if info.StakeExists {
         _, err = s.rollup.StakeOnExistingNode(s.builder.Auth(ctx), action.number, action.hash)
         return err
      }

      // If we have no stake yet, we'll put one down
      stakeAmount, err := s.rollup.CurrentRequiredStake(s.getCallOpts(ctx))
      if err != nil {
         return err
      }
      _, err = s.rollup.NewStakeOnExistingNode(
         s.builder.AuthWithAmount(ctx, stakeAmount),
         action.number,
         action.hash,
      )
      if err != nil {
         return err
      }
      info.StakeExists = true
      return nil
   default:
      panic("invalid action type")
   }
}
```

接下来的流程：createConflict->BuildingTransactionCount->ExecuteTransactions
