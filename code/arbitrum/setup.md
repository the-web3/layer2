# Arbitrum 的 Nitro 本地网络启动与 ETH/ERC20 充提测试教程

### 一.  本地网络启动

1. 第一步：克隆 nitro 代码

```
git clone https://github.com/OffchainLabs/nitro.git
```

2. 安装子模块

```
git submodule init
git submodule update
```

3. 启动测试

```
./test-node.bash
```

4. 启动配置项

```
force_build=true  是否强制 build
validate=true     是否启动 validate
blockscout=false  是否启动 blockscout
```

如果启动过程中报错如下：
![](/media/editor/1_20220915233711398444.png)

请把` force_init `设置成 true 就行.

启动完成之后有以下这些镜像(我的没有启动 blockscout)：

```
bj89182ml@192 nitro % docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS       PORTS                                                                       NAMES
9b4d301838ea   nitro_validator             "/usr/local/bin/nitr…"   8 hours ago   Up 8 hours   127.0.0.1:8247->8547/tcp, 127.0.0.1:8248->8548/tcp                          nitro_validator_1
00e897625661   nitro_sequencer             "/usr/local/bin/nitr…"   8 hours ago   Up 8 hours   127.0.0.1:8547-8548->8547-8548/tcp, 127.0.0.1:9642->9642/tcp                nitro_sequencer_1
393399b921f0   nitro_poster                "/usr/local/bin/nitr…"   8 hours ago   Up 8 hours   127.0.0.1:8147->8547/tcp, 127.0.0.1:8148->8548/tcp                          nitro_poster_1
f0898fa127be   redis:6.2.6                 "docker-entrypoint.s…"   8 hours ago   Up 8 hours   127.0.0.1:6379->6379/tcp                                                    nitro_redis_1
d2fc2ca998a3   ethereum/client-go:stable   "geth --keystore /ke…"   8 hours ago   Up 8 hours   127.0.0.1:8545-8546->8545-8546/tcp, 127.0.0.1:30303->30303/tcp, 30303/udp   nitro_geth_1
```

### 二. ETH 测试

使用 arbitrum-sdk 配置本地网络
克隆代码：

```
git clone https://github.com/OffchainLabs/arbitrum-sdk.git
```

安装依赖：

```
yarn install
```

配置环境变量：

```
ARB_URL="http://localhost:8547"
ETH_URL="http://localhost:8545"
ARB_KEY="e887f7d17d07cc7b8004053fb8826f6657084e88904bb61590e498ca04704cf2"
ETH_KEY="e887f7d17d07cc7b8004053fb8826f6657084e88904bb61590e498ca04704cf2"

#INFURA_KEY="2323232323232323232"
#SHOULD_FORK=""
#MAINNET_RPC="https://mainnet.infura.io/v3/2323"
#ARB_ONE_RPC="https://arb1.arbitrum.io/rpc"
#RINKEBY_RPC="https://rinkeby.infura.io/v3/232323"
#RINKARBY_RPC="https://rinkeby.arbitrum.io/rpc"
```

获取本地网络配置 

```
yarn gen:network
```

执行结果如下：
![](/media/editor/2_20220915233811833921.jpeg)

将打印出来的这些地址配置到 arbitrum-tutorials 项目中的各个模块
1. Deposit
克隆代码并安装依赖

```
git clone https://github.com/OffchainLabs/arbitrum-tutorials.git
cd arbitrum-tutorials/packages/eth-deposit
yarn install 安装依赖
```

配置环境变量

```
# This is a sample .env file for use in local development.
# Duplicate this file as .env here

# Your Private key
DEVNET_PRIVKEY="e887f7d17d07cc7b8004053fb8826f6657084e88904bb61590e498ca04704cf2"

# Hosted Aggregator Node (JSON-RPC Endpoint). This is Arbitrum Goerli Testnet, can use any Arbitrum chain
L2RPC="http://127.0.0.1:8547"

# Ethereum RPC; i.e., for Goerli https://goerli.infura.io/v3/<your infura key>
L1RPC="http://127.0.0.1:8545"
将上面的 sdk 里面获取到的本地网络配置配置到代码中 arbitrum-tutorials/packages/eth-deposit/scripts/exec.js
await arbLog('Deposit Eth via Arbitrum SDK')
await addCustomNetwork({
  customL1Network: {
    blockTime: 10,
    chainID: 1337,
    explorerUrl: '',
    isCustom: true,
    name: 'EthLocal',
    partnerChainIDs: [412346],
    rpcURL: 'http://localhost:8545',
  },
  customL2Network: {
    chainID: 412346,
    confirmPeriodBlocks: 20,
    ethBridge: {
      bridge: '0x1702782edb6c1a0a20a4cf81bf7159005c7717c4',
      inbox: '0xa24b6c27d7e12a53f951f1e3f964f02a50acb700',
      outbox: '0xb6bE947DCA31b096Dc1041aC30e806A6F13Bf871',
      rollup: '0xcc0f356cbeded5db896f8905e856366510aba533',
      sequencerInbox: '0x33c04c208c6a8e644a33e104f0baaf3d48345396',
    },
    explorerUrl: '',
    isArbitrum: true,
    isCustom: true,
    name: 'ArbLocal',
    partnerChainID: 1337,
    rpcURL: 'http://localhost:8547',
    retryableLifetimeSeconds: 604800,
    tokenBridge: {
      l1CustomGateway: '0xacd66ABF571E20aaB88c45dEA436E1F34ed97f79',
      l1ERC20Gateway: '0x99FF57ee073255f26669D703697bca8DDc797d98',
      l1GatewayRouter: '0x508A0F1EFAA8664e3c85BC57eC52433DEC682fa1',
      l1MultiCall: '0xC6f7A186231F4f541Ab9FA1cc3092Fae0CBCA815',
      l1ProxyAdmin: '0x1Ffb0bCa2F62290fd9E237F1bD4969A513B28195',
      l1Weth: '0xC6f7A186231F4f541Ab9FA1cc3092Fae0CBCA815',
      l1WethGateway: '0x28B3a95f8e6508dc2C0097Ac543C8B2eFff21bF9',
      l2CustomGateway: '0xF0B003F9247f2DC0e874710eD55e55f8C63B14a3',
      l2ERC20Gateway: '0x78a6dC8D17027992230c112432E42EC3d6838d74',
      l2GatewayRouter: '0x7b650845242a96595f3a9766D4e8e5ab0887936A',
      l2Multicall: '0x9b890cA9dE3D317b165afA7DFb8C65f2e4c95C20',
      l2ProxyAdmin: '0x7F85fB7f42A0c0D40431cc0f7DFDf88be6495e67',
      l2Weth: '0x36BeF5fD671f2aA8686023dE4797A7dae3082D5F',
      l2WethGateway: '0x2E76efCC2518CB801E5340d5f140B1c1911b4F4B',
    },
  },
})
const l2Network = await getL2Network(l2Provider)
const ethBridger = new EthBridger(l2Network)
```

执行充值命令

```
yarn run depositETH
```

执行结果如下：

```
yarn run v1.22.17
$ hardhat run scripts/exec.js
Environmental variables properly set 👍
                            🔵🔵
                           🔵  🔵
                          🔵    🔵
                         🔵      🔵
                        🔵        🔵
                       🔵          🔵
                      🔵            🔵
                     🔵              🔵
                    🔵                🔵
                   🔵                  🔵
                  🔵                    🔵
                 🔵                      🔵
                🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵
               🔵                          🔵
              🔵                            🔵
             🔵                              🔵
            🔵                                🔵
           🔵                                  🔵
          🔵                                    🔵
         🔵                                      🔵
        🔵                                        🔵
       🔵                                          🔵
      🔵                                            🔵
     🔵                                              🔵
    🔵                                                🔵
Arbitrum Demo: Deposit Eth via Arbitrum SDK
Lets
Go ➡️
l2Wallet = 0x683642c22feDE752415D4793832Ab75EFdF6223c
l2WalletInitialEthBalance = BigNumber { _hex: '0x1565edfa392c81df50e0', _isBigNumber: true }
deposit L1 receipt is: 0xac1b9d790004dba342bbcd97a9c0e8f59e84b1c987203f16eb28d67cf50c5892
Now we wait for L2 side of the transaction to be executed ⏳
L2 message successful: status: undefined
your L2 ETH balance is updated from 101049965373101701026016 to 101059965373101701026016
✨  Done in 34.99s.
```

2 Withdraw

进行入提现代码目录并安装依赖

```
 cd packages/eth-withdraw
 yarn install
```

配置环境变量和本地网络同上，然后运行测试脚本


```
yarn run withdrawETH
```

执行结果如下：

```
 yarn run withdrawETH
yarn run v1.22.17
$ hardhat run scripts/exec.js
Environmental variables properly set 👍
                            🔵🔵
                           🔵  🔵
                          🔵    🔵
                         🔵      🔵
                        🔵        🔵
                       🔵          🔵
                      🔵            🔵
                     🔵              🔵
                    🔵                🔵
                   🔵                  🔵
                  🔵                    🔵
                 🔵                      🔵
                🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵
               🔵                          🔵
              🔵                            🔵
             🔵                              🔵
            🔵                                🔵
           🔵                                  🔵
          🔵                                    🔵
         🔵                                      🔵
        🔵                                        🔵
       🔵                                          🔵
      🔵                                            🔵
     🔵                                              🔵
    🔵                                                🔵
Arbitrum Demo: Withdraw Eth via Arbitrum SDK
Lets
Go ➡️
...🚀

Wallet properly funded: initiating withdrawal now
Ether withdrawal initiated! 🥳 0x2ee2a85461f3303be2d769a35163ebc4048535545dab6064e27e9ee7fac5862a
Withdrawal data: [
  [
    '0x683642c22feDE752415D4793832Ab75EFdF6223c',
    '0x683642c22feDE752415D4793832Ab75EFdF6223c',
    BigNumber {
      _hex: '0x88b79364b9c67a6821a96c885a742e95d81baa87a47365305d9b505a1b1b06a2',
      _isBigNumber: true
    },
    BigNumber { _hex: '0x03', _isBigNumber: true },
    BigNumber { _hex: '0x32', _isBigNumber: true },
    BigNumber { _hex: '0x7add', _isBigNumber: true },
    BigNumber { _hex: '0x63232a78', _isBigNumber: true },
    BigNumber { _hex: '0xe8d4a51000', _isBigNumber: true },
    '0x',
    caller: '0x683642c22feDE752415D4793832Ab75EFdF6223c',
    destination: '0x683642c22feDE752415D4793832Ab75EFdF6223c',
    hash: BigNumber {
      _hex: '0x88b79364b9c67a6821a96c885a742e95d81baa87a47365305d9b505a1b1b06a2',
      _isBigNumber: true
    },
    position: BigNumber { _hex: '0x03', _isBigNumber: true },
    arbBlockNum: BigNumber { _hex: '0x32', _isBigNumber: true },
    ethBlockNum: BigNumber { _hex: '0x7add', _isBigNumber: true },
    timestamp: BigNumber { _hex: '0x63232a78', _isBigNumber: true },
    callvalue: BigNumber { _hex: '0xe8d4a51000', _isBigNumber: true },
    data: '0x'
  ]
]
To to claim funds (after dispute period), see outbox-execute repo ✌️
✨  Done in 9.71s.
```

3. Cliam Withdraw

进行入 outbox-execute 目录并安装依赖

```
cd packages/outbox-execute
yarn install
```

配置环境变量和本地网络同上，然后运行测试脚本

```
yarn run withdrawETH
```

执行命令

```
yarn outbox-exec --txhash 0x2ee2a85461f3303be2d769a35163ebc4048535545dab6064e27e9ee7fac5862a
```

结果如下：

```
yarn run v1.22.17
$ hardhat outbox-exec --txhash 0x2ee2a85461f3303be2d769a35163ebc4048535545dab6064e27e9ee7fac5862a
Environmental variables properly set 👍
                            🔵🔵
                           🔵  🔵
                          🔵    🔵
                         🔵      🔵
                        🔵        🔵
                       🔵          🔵
                      🔵            🔵
                     🔵              🔵
                    🔵                🔵
                   🔵                  🔵
                  🔵                    🔵
                 🔵                      🔵
                🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵
               🔵                          🔵
              🔵                            🔵
             🔵                              🔵
            🔵                                🔵
           🔵                                  🔵
          🔵                                    🔵
         🔵                                      🔵
        🔵                                        🔵
       🔵                                          🔵
      🔵                                            🔵
     🔵                                              🔵
    🔵                                                🔵
Arbitrum Demo: Outbox Execution
Lets
Go ➡️
...🚀

Waiting for the outbox entry to be created. This only happens when the L2 block is confirmed on L1, ~1 week after it's creation.
```

已经 cliam 完成的命令如下：

```
Lets
Go ➡️
...🚀
Waiting for the outbox entry to be created. This only happens when the L2 block is confirmed on L1, ~1 week after it's creation.
Outbox entry exists! Trying to execute now
Done! Your transaction is executed {
  to: '0xFfA9084d0527FA89EbB4e917F936565aa7Bf95F6',
  from: '0x683642c22feDE752415D4793832Ab75EFdF6223c',
  contractAddress: null,
  transactionIndex: 0,
  gasUsed: BigNumber { _hex: '0x014837', _isBigNumber: true },
  logsBloom: '0x00000000000000000000000000000800000000000000000000000008000000000020000400000000020000100000000000000000000000000000000800000000000000000000000000004000000000000000000000800000000000000000000000000000020000000000000000000800000000000000000000000000000010000000000000040000000000400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000002000000000000000000000000020008008000000000000000000000000008000000000000000000000000000000000',
  blockHash: '0xc27029955f16233ed53d0405f7a5ef5609b62e2b80e3bb2784b758139dd6e4a1',
  transactionHash: '0x34401cff2b5262a480e481871cbbd8b6e61cb2807701c2be9cb24daad171f832',
  logs: [
    {
      transactionIndex: 0,
      blockNumber: 31668,
      transactionHash: '0x34401cff2b5262a480e481871cbbd8b6e61cb2807701c2be9cb24daad171f832',
      address: '0xFfA9084d0527FA89EbB4e917F936565aa7Bf95F6',
      topics: [Array],
      data: '0x0000000000000000000000000000000000000000000000000000000000000002',
      logIndex: 0,
      blockHash: '0xc27029955f16233ed53d0405f7a5ef5609b62e2b80e3bb2784b758139dd6e4a1'
    },
    {
      transactionIndex: 0,
      blockNumber: 31668,
      transactionHash: '0x34401cff2b5262a480e481871cbbd8b6e61cb2807701c2be9cb24daad171f832',
      address: '0x1702782EDb6C1A0a20A4cF81BF7159005C7717c4',
      topics: [Array],
      data: '0x000000000000000000000000000000000000000000000000000000e8d4a5100000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000',
      logIndex: 1,
      blockHash: '0xc27029955f16233ed53d0405f7a5ef5609b62e2b80e3bb2784b758139dd6e4a1'
    }
  ],
  blockNumber: 31668,
  confirmations: 3,
  cumulativeGasUsed: BigNumber { _hex: '0x014837', _isBigNumber: true },
  effectiveGasPrice: BigNumber { _hex: '0x59682f07', _isBigNumber: true },
  status: 1,
  type: 2,
  byzantium: true,
  events: [
    {
      transactionIndex: 0,
      blockNumber: 31668,
      transactionHash: '0x34401cff2b5262a480e481871cbbd8b6e61cb2807701c2be9cb24daad171f832',
      address: '0xFfA9084d0527FA89EbB4e917F936565aa7Bf95F6',
      topics: [Array],
      data: '0x0000000000000000000000000000000000000000000000000000000000000002',
      logIndex: 0,
      blockHash: '0xc27029955f16233ed53d0405f7a5ef5609b62e2b80e3bb2784b758139dd6e4a1',
      args: [Array],
      decode: [Function (anonymous)],
      event: 'OutBoxTransactionExecuted',
      eventSignature: 'OutBoxTransactionExecuted(address,address,uint256,uint256)',
      removeListener: [Function (anonymous)],
      getBlock: [Function (anonymous)],
      getTransaction: [Function (anonymous)],
      getTransactionReceipt: [Function (anonymous)]
    },
    {
      transactionIndex: 0,
      blockNumber: 31668,
      transactionHash: '0x34401cff2b5262a480e481871cbbd8b6e61cb2807701c2be9cb24daad171f832',
      address: '0x1702782EDb6C1A0a20A4cF81BF7159005C7717c4',
      topics: [Array],
      data: '0x000000000000000000000000000000000000000000000000000000e8d4a5100000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000',
      logIndex: 1,
      blockHash: '0xc27029955f16233ed53d0405f7a5ef5609b62e2b80e3bb2784b758139dd6e4a1',
      removeListener: [Function (anonymous)],
      getBlock: [Function (anonymous)],
      getTransaction: [Function (anonymous)],
      getTransactionReceipt: [Function (anonymous)]
    }
  ]
}
✨  Done in 12.83s.
```

### 二. ERC20 测试

erc20 测试代码对应的是 token-deposit 和 token-withdraw
安装依赖和环境配置和上面一致
1. Deposit

```
yarn run token-deposit
```

执行命令之后结果如下, 会转 0.5 DAPP Token 到 L2：

```
yarn run v1.22.17
$ hardhat run scripts/exec.js
Environmental variables properly set 👍
                            🔵🔵
                           🔵  🔵
                          🔵    🔵
                         🔵      🔵
                        🔵        🔵
                       🔵          🔵
                      🔵            🔵
                     🔵              🔵
                    🔵                🔵
                   🔵                  🔵
                  🔵                    🔵
                 🔵                      🔵
                🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵
               🔵                          🔵
              🔵                            🔵
             🔵                              🔵
            🔵                                🔵
           🔵                                  🔵
          🔵                                    🔵
         🔵                                      🔵
        🔵                                        🔵
       🔵                                          🔵
      🔵                                            🔵
     🔵                                              🔵
    🔵                                                🔵
Arbitrum Demo: Deposit token using Arbitrum SDK
Lets
Go ➡️
...🚀

Deploying the test DappToken to L1:
DappToken is deployed to L1 at 0x3A85e361917180567F6a0fb8c68B2b5065126aCA
Approving:
You successfully allowed the Arbitrum Bridge to spend DappToken 0xd62f25efb8d37f84c292cd0ed547c13a5fc97b14fe0ca8d64635f9eb06ceafff
L2 message successful: status: REDEEMED
l2TokenAddress= 0xe48d6eD6aCE60A5C69425a5EDDa46c7CbacD7cFf
Duplicate definition of Transfer (Transfer(address,address,uint256,bytes), Transfer(address,address,uint256))
```

2. Withdraw
执行提现命令

```
yarn withdraw-token
```

结果如下：

```
yarn run v1.22.17
$ hardhat run scripts/exec.js
Warning: SPDX license identifier not provided in source file. Before publishing, consider adding a comment containing "SPDX-License-Identifier: <SPDX-License>" to each source file. Use "SPDX-License-Identifier: UNLICENSED" for non-open-source code. Please see https://spdx.org for more information.
--> contracts/DappToken.sol


Warning: Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
  --> contracts/DappToken.sol:23:5:
   |
23 |     constructor(uint256 _initialSupply) public {
   |     ^ (Relevant source part starts here and spans across multiple lines).


Compiled 1 Solidity file successfully
Environmental variables properly set 👍
                            🔵🔵
                           🔵  🔵
                          🔵    🔵
                         🔵      🔵
                        🔵        🔵
                       🔵          🔵
                      🔵            🔵
                     🔵              🔵
                    🔵                🔵
                   🔵                  🔵
                  🔵                    🔵
                 🔵                      🔵
                🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵🔵
               🔵                          🔵
              🔵                            🔵
             🔵                              🔵
            🔵                                🔵
           🔵                                  🔵
          🔵                                    🔵
         🔵                                      🔵
        🔵                                        🔵
       🔵                                          🔵
      🔵                                            🔵
     🔵                                              🔵
    🔵                                                🔵
Arbitrum Demo: Withdraw token using Arbitrum SDK
Lets
Go ➡️
🚀
Deploying the test DappToken to L1:
DappToken is deployed to L1 at 0x878E8Cc740C493Ad951dDAaceFF9a9D9f6F41413
Approving:
You successfully allowed the Arbitrum Bridge to spend DappToken 0xa0f8c4f0b8ee441052b7f25fd7060da8ec6a4ec4501309152679145ba7667111
Transferring DappToken to L2:
Deposit initiated: waiting for L2 retryable (takes < 10 minutes; current time: 21:45:01 GMT+0800 (中国标准时间))
Setup complete
L2 message successful: status: REDEEMED
Withdrawing:
Token withdrawal initiated! 🥳 0x878d8f0f78b4c397c77068be68fb05ed006724e40884b8560ef62660bd376363
Duplicate definition of Transfer (Transfer(address,address,uint256,bytes), Transfer(address,address,uint256))
To to claim funds (after dispute period), see outbox-execute repo ✌️
✨  Done in 48.55s.
```

3. Cliam withdraw

和 ETH 的 Cliam 类似

### 四. 可以参考配置项目

- arbitrum-tutorials配置参考： https://github.com/guoshijiang/arbitrum-tutorials
