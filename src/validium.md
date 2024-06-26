### 一. Validium 简介

Validium运行方式类似于ZK rollup，不同之处在于数据被保存在链下。因为交易数据不是发布在区块链上的，所以有必要采用额外的信任假设，用户必须信任操作员，以便在需要时可以访问数据。

Validium的链下数据存储有很多好处，交易速度更快，并且因为公众无法访问交易，所以用户隐私性也到了提高。

然而，由于数据是在链外保存的，因此用户将无法随时看到其在智能合约中的可用金额。

因此，用户必须从中继器那里获取数据才能掌握自己的资金，并且他们必须信任中继器。

为了解决这个问题，StarkWare这样的解决方案提供了一个数据可用性委员会（DAC），其会存储所有链下数据，并在紧急情况下变为公开可访问，减少用户对中继器的依赖：由于其仍使用zkp，所以不存在广播不正确状态的危险；用户现在必须信任的只是信息的及时性。

Validium解决方案是较新的，建立在ZKR基础之上，如果扩展解决方案的有效性证明套件越来越受欢迎，其势头会不断提高。

使用Validium解决方案的项目包括DeversiFi、ZKSwap（支付、交易平台）、Sorare（足球NFT游戏）和Immutable X（NFT 市场）。

### 二. Validium 工作原理

Validium 选择将 layer 2 的交易数据放在链下，因而比 rollup 架构有着更高的扩展性。验证计算方面，Validium 不像 Plasma 依托诈欺证明，而是采用零知识证明。如先前在讨论 zkRollup 时提到的，这样做会导致 Validium 在目前的应用部署，只能局限于特定目的（普适性低），比如 StarkEx 就是面向去中心化交易所的方案。

但这些权衡使得 Validium 在某些方面优于 Plasma 。在主网进行零知识证明验证能避免执行者提供无效状态，也能降低执行者不公开数据造成的后果。举例来说，想要勾结执行者，让状态错误地转变为 “把他人的钱转到自己账户” 是不可能办到的；因此 Validium 不需要在协议中设计 “大量资金退出” 激励博弈，也不需要延长资金从 layer 2 退出的时间。

正如其他研究者指出的，零知识证明并不是解决数据可用性问题的万灵丹：比如（恶意）执行者修改自己所控制的账户的状态是没有问题的，然后积压关于这些交易的数据，这会导致某些用户想退出资金时，无法提供 Merkle proof 。

![](https://raw.githubusercontent.com/the-web3/layer2/79839bb1ee4b3ca0a345fca240678b111dd64efd/images/29.png)

有没有一种方法可以阻止Validium中的数据保留攻击？从2016年提出Plasma概念以来，这个问题就被大家讨论。同时，zkRollup也是在这一研究结果上诞生出来的。Non-rollup试图无须信任地确保数据可用性，这将导致失去Validium的大部分竞争优势。

尽管不能完全解决问题，但StarkEx通过引入许可的数据可用性委员会（DAC）来缓解这一情况。

DAC必须通过其委员会成员的法定人数签署对状态的每次更新，以此来确认它已经接收到数据。在StarkEx中，DAC由8位参与者组成（添加太多成员会不利于系统的活性）。它们都是在已建立的法律管辖区中众所周知的富有声望的组织。对他们来说，几乎不太可能去尝试滥用其权力。这就是其构建的逻辑。

矛盾的是，众所周知、富有声望、且处在强大国家的司法管辖区正是让它们变得脆弱的原因。一种可能的麻烦情况是：运营者要求执行KYC/AML法规，并有义务冻结（可能是永远）交易记录超过1万美元的账户的所有资金。

随着研究的深入，StarkEx实施了“验证者合约升级”机制，它允许运营者立即将新项添加到链上的验证者合约。它不能使任何旧的逻辑失效，例如，你不能删除用户签名检查。相反，它允许增加其他约束（就Solidity而言，你可以将约束视为 `require()`语句）。这是很好的安全功能：如果在StarkEx的STARK circuit逻辑中发现任何缺失的约束，则可以快速修复它，同时不引入新的漏洞。但是，这一功能可以用作为隐藏的审查后门。简言之，StarkEx运营者始终可以部署合约逻辑的扩展，这样就存在引入黑名单的扩展可能，而无须事先警告用户。

从其文档中还无法完全弄清楚这一点，但是看上去执行新规则似乎并不需要得到DAC（蓝狐笔记：数据可用性委员会）的同意。如果你将StarkEX看作为完全去中心化的交易协议，那么这没有多大意义。想象一下如果Vitalik Buterin拥有一个开关可以即时冻结任何以太坊账户，那会是什么结果？另外，如果你将StarkEX看作为加密交易所安全功能的增强（其创建者应该这么做），则它就有意义了。StarkEX Validium的运营者能没收用户的资金让我们扩展思想试验。不管出于何种假想原因（很可能是由于运营者无法掌控的原因），很多用户的资产已被冻结。那么，用户在StarkEx的资金也可以被没收吗？事实上，它是可能的。跟其他很多加密项目一样，StarkEx实现了最新的升级机制。在部署新版本之前，会提前28天通知用户，任何人只要不喜欢都可以提取退出。除了那些资金被冻结的人。可以在合约上部署新逻辑，这样在宽限期结束后，通过新逻辑可以将冻结资金转移由指定方保管。不幸的是，受影响的用户对此毫无办法。还存在一些合理担心，升级提醒周期可能并不足以让每位不同意改变的用户退出（所谓的“大量退出”场景）。但，这个问题是通用的合约升级问题，并非Validium独有的问题。贾斯汀·德雷克描述了对Validium的加密经济攻击在后续的讨论中，贾斯汀·德雷克指出数据可用性可能会导致意外的攻击向量：如果DAC（数据可用性委员会）的法定人数的签名密钥遭到破坏（考虑到这些密钥保持在线状态，这让它们很难保证完全安全），攻击者可以将Validium转换为只有他们知道的状态，从而冻结所有资产，然后要求解锁资产的赎金。

理论上讲，合约升级机制可以减轻此类攻击。Validium的运营者可以启动新版本的部署，并在28天的升级通知期后，将状态恢复为最新的已知版本。这将是为期一个月的资本锁定，这当然有很大的成本，但是如果DAC拒绝谈判，攻击者将得不到一分钱。但是，事实证明，攻击者有一种方法可以迫使运营者在丢失所有和允许攻击者进行双花之间做决定。可以通过如下例子说明：想象一下，你可以按照某种方式对ATM进行黑客攻击，以在提款完成后擦除整个银行数据库。你只能从自己的账户中提款，但当数据库消失时，操作的详细信息也将丢失。银行员工可以在一个月内完成复杂的数据库恢复过程。但是，既然他们无法知道是谁提了款，因此通过返回上个检查点，他们还将恢复你已提过款的余额。（蓝狐笔记：也就是攻击者可以通过操作自己的账户，实现双花攻击）当然，这个双花攻击将仅限于攻击者的账户余额。但是，构建无须信任的合约并从匿名鲸鱼那里借入必要的资产并不是难事。在zkRollup中数据可用性保护了用户的资产免遭扣押、审查和黑客攻击，但其吞吐量有所降低。

对于zkrollup用户来说，rollup的状态是可用的，只要有一个以太坊全节点在线。它是这样工作的：对于每个zkRollup区块，必须将重建状态变化所需的信息作为以太坊交易的调用数据提交，否则zkRollup智能合约将拒绝进行状态转换。zkRollups上的状态更改将导致每笔交易的gas成本较低，这个成本随着交易数量呈线性增长。借助手头的Merkle树数据，被审查的用户始终可以直接从主网上的zkRollup合约中索取其资金。他们需要做的是提供其账号上的Merkle所有权证明。因此，链上的数据可用性可以确保没有人（包括zkRollup运营者）能够冻结或捕获用户资金。

数据可用性的链上存储导致吞吐量受到限制，zkRollup在如今的以太坊上有2000tps的上限，而StarkEx Validium声称可以达到9000tps。这种差异可能会导致在确定选择两者技术在应用领域和用例方面变得关键。例如zkRollup非常适合于扩展去中心化的加密支付（VISA在全球平均的tps为2000），以及那些对无须信任有严格要求的不可篡改的智能合约；而对于Validium来说，它可能更适合于传统的高频交易或具有较低信任假设的好游戏。


结论已经证明zkRollups和Validium（StarkEX）在工作方式上相对相似，但其主要区别在于数据是在链上还是链下可用。这对于理解它们以及在什么场景使用它们至关重要。这种差异也意味着，尽管zkRollup是完全无须许可的去中心化扩展协议，不过Validium展示了托管性的PoA系统的更多属性（不管是吞吐量还是风险特征），尽管其安全性已经得到极大提高。两者技术发展都在减轻对信任的需求，并为用户提供更多对其资产的控制权，都是朝赋予个人更多能力的方向发展，为了取得进展，我们总是需要作出权衡取舍。不过，在加密社区中，越来越多的共识是技术已经过了“不要作恶”的阶段，而进入了“无法作恶”的阶段。我们可以通过自我托管、抗审查性、隐私以及消除单点故障来达成目的。这些想法构成了我们正在为之奋斗的系统的基本价值。