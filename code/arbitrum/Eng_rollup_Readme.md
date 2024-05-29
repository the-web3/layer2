#Arbitrum’s Nitro project Rollup in-depth details

Regarding rollup submitting transactions to the first layer, our [above article](https://chaineye.info/arctle_detail?article_id=584) has already been mentioned. Here we mainly analyze the interaction between the rollup contract, rollup watcher and fraud proof. and logical organization of details.

### 1. Code analysis of validator sub-modules

- block_challenge_backend and challenge_manager block challenge related codes
- block_validator: block verification logic, here is the block verification module connecting arbnode and geth
- rollup_watcher: observer of rollup contract transaction events

### 2. 2. Rollup details

The rollup protocol tracks the rollup block chain – for clarity, we’ll call these “RBlocks”. They are different from layer 1 Ethereum blocks, and they are different from layer 2 Nitro blocks. You can think of RBlocks as forming a separate chain, managed and overseen by the Arbitrum aggregation protocol.

For explanations related to the summary blockchain, please refer to: nitro/inside-arbitrum-nitro.md at master · OffchainLabs/nitro

rollup_watcher will monitor rollup-related events. The monitored events are as follows:

```
var rollupInitializedID common.Hash
var nodeCreatedID common.Hash
var challengeCreatedID common.Hash
```

These events are handled by methods such as LookupCreation, LookupNode, LookupNodeChildren and LookupChallengedNode respectively. L1Validator will call these methods to obtain relevant contract event information for verification.

RollupNode definition

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

Roles participating in rollup: WatchtowerStrategy, DefensiveStrategy, StakeLatestStrategy and MakeNodesStrategy respectively
The code is defined as follows and is located in staker.go:

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

- After the Staker start task is started, it will first update the wasm moudle root of the authentication block. The function call is as follows

```
err := s.updateBlockValidatorModuleRoot(ctx)
```

The updateBlockValidatorModuleRoot will fetch the WasmModuleRoot from the rollup and update it to the blockValidator. Of course, when the l1 validator is initialized, it will also update the corresponding WasmModuleRoot and set the WasmModuleRoot to the current ModuleRoot of the blockValidator for verification.

- Obtain the callOpts information for the following function call, which contains

```
type CallOpts struct {
    Pending bool // Whether to operate on the pending state or the last known one
    From common.Address // Optional the sender address, otherwise the first account is used
    BlockNumber *big.Int // Optional the block number on which the call should be performed
    Context context.Context // Network context to support cancellation and timeouts (nil = no timeout)
}
```

- Clear built transactions

```
func (b *ValidatorTxBuilder) ClearTransactions() {
    b.transactions = nil
}
```

- Get the latest staker information from the rollup's StakerMap, including challenge information
- Get the latest pledge data information from validatorUtils, and the contract function called is as follows:

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

- To determine whether Rblocks is forked, validatorUtils.AreUnresolvedNodesLinear -> _ValidatorUtils.contract.Call(opts, &out, "areUnresolvedNodesLinear", rollup) finally calls this contract function to determine

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

- If a fork occurs, effectiveStrategy will be switched from DefensiveStrategy to StakeLatestStrategy, otherwise it will be executed according to WatchtowerStrategy.
code show as below:

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

s.bringActiveUntilNode stores and info.LatestStakedNode < s.bringActiveUntilNode, effectiveStrategy will switch from DefensiveStrategy to StakeLatestStrategy

- Get the latest confirmed node of the rollup chain, call stack s.rollup.LatestConfirmed(callOpts)->latestConfirmed, the contract function details are as follows:

```
/// @return Index of the latest confirmed node
function latestConfirmed() public view override returns (uint64) {
     return _latestConfirmed;
}
```

- To determine whether a pledge increase is required, isRequiredStakeElevated obtains CurrentRequiredStake, and baseStake determines whether a pledge increase is required (obtains the current stake, and finally enters the currentRequiredStake function, and baseStake enters the contract processing logic)
- When effectiveStrategy is the MakeNodesStrategy strategy, or effectiveStrategy is StakeLatestStrategy and rawInfo (staker information is nil) and the stake needs to be increased, the following logic will be processed
   - Handle timeout challenges resolveTimedOutChallenges, enter the details of handling challenges, which will be discussed below
   - Processing unresolve nodes, unresolve processing logic: There are three categories of processing nodes, the code is as follows:

```
const (
    CONFIRM_TYPE_NONE ConfirmType = iota
    CONFIRM_TYPE_VALID
    CONFIRM_TYPE_INVALID
)
```

The processing logic is to first obtain confirmType and unresolvedNodeIndex, and call the checkDecidableNextNode function in ValidatorUtils and firstUnresolvedNode in rollup respectively. If it is of the CONFIRM_TYPE_INVALID category, directly reject the next node. If it is CONFIRM_TYPE_VALID, find and confirm the next node. The core logic code is as follows

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

- If stakeInfo is not nil, and the latest stake node is smaller than the latest confirmation node, the effectiveStrategy is WatchtowerStrategy and DefensiveStrategy, and the pledged coins of the pledger currently pledged on or before the latest confirmation node are returned, and the sender's possession is extracted from the summary chain For uncommitted funds, the calling methods are s.rollup.ReturnOldDeposit and s.rollup.WithdrawStakerFunds, and it is best to execute the transaction ExecuteTransactions
- Continue to retrieve the data yourself under the condition walletAddressOrZero != (common.Address{})
- The next step is to resolve the conflict handleConflict. If the conditions are met, go back to create a challenge, and then enter the challenge details.
- rawInfo != nil || !resolvingNode || !requiredStakeElevated, increase the stake,
Improve the detailed logic of pledge

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

       // We'll return early if we already have a stake
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
             log.Warn("bringing defensive validator onlinebecause of incorrect assertion")
             s.bringActiveUntilNode = action.number
             info.CanProgress = false
          } else {
             s.inactiveLastCheckedNode = &nodeAndHash{
                id: action.number,
                hash: action.hash,
             }
          }
          return nil
       }
       log.Info("staking on existing node", "node", action.number)
       // We'll return early if we already have a stake
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

The next process: createConflict->BuildingTransactionCount->ExecuteTransactions
