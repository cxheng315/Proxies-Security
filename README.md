# Proxy、可升级智能合约及其安全性

![](./public/mindmap.png)

## 目录

- [介绍](#介绍)
- [智能合约存储布局](#智能合约存储布局)
- [`delegatecall`](#delegatecall)
- [Proxy 模式](#proxy-模式)
  - [最简 Proxy](#最简-proxy)
  - [可初始化 Proxy](#可初始化-proxy)
  - [可升级 Proxy](#可升级-proxy)
  - [EIP-1967 可升级 Proxy](#eip-1967-可升级-proxy)
    - [什么是非结构化存储模式？](#什么是非结构化存储模式)
  - [透明 Proxy 模式 (TPP)](#透明-proxy-模式-tpp)
  - [通用可升级 Proxy 标准 (UUPS)](#通用可升级-proxy-标准-uups)
  - [Beacon Proxy](#beacon-proxy)
  - [Diamond Proxy](#diamond-proxy)
  - [Proxy 模式对比](#比较-proxy-模式)
- [Proxy 漏洞安全指南](#proxy-漏洞安全指南)
  - [Delegatecall 到不存在的外部合约](#1-delegatecall-到不存在的外部合约)
  - [Delegatecall 与 Selfdestruct 漏洞](#2-delegatecall-与-selfdestruct-漏洞)
  - [函数冲突漏洞](#3-函数冲突漏洞)
  - [存储冲突漏洞](#4-存储冲突漏洞)
  - [未初始化 Proxy 漏洞](#5-未初始化-proxy-漏洞)
- [Proxy 开发指南](#proxy-开发指南)
- [参考资料](#参考资料)

## 介绍

欢迎来到 QuillAudits 的 Proxy 与可升级智能合约仓库。本仓库收录了理解 Solidity 智能合约可升级性所需的技术概念、理论基础、实践指南以及实现示例。

区块链通常被视为具有数据不可篡改性的系统，人们也长期认为“合约一旦部署，就无法更改”。这一点对于历史交易记录依然成立，但当涉及智能合约的存储与地址时，情况则并非绝对如此。

#### 为什么需要 Proxy 以及如何使用它？

从设计上看，部署在区块链上的合约代码是不可变的。虽然不可篡改性是区块链的核心特性之一，但在需要可升级性时，它也会带来实际挑战。许多新手可能会疑惑：既然区块链强调不可篡改，为什么还需要“升级”？事实上，仍有许多需要修改代码的场景，例如错误修复、安全补丁、性能优化、新功能发布等。

在正式进入 Proxy 与可升级合约的主题之前，我们需要先了解 Solidity 中 `delegatecall` 操作码的工作机制。而在理解 `delegatecall` 之前，理解 EVM 如何在存储中保存并管理合约变量也会非常有帮助。

## 智能合约存储布局

合约存储布局指的是管理合约中状态变量在持久化存储中如何排列的规则。几乎所有智能合约都需要保存长期状态变量。在 Solidity 中，开发者可以使用三种不同的存储区域来指定 EVM 将变量存储的位置：`memory`、`calldata` 和 `storage`。在本节中，我们主要讨论与可升级合约密切相关的 storage 存储布局。

每个合约都有自己独立的存储空间，用于持久且可读写的数据存储。合约只能读取和写入其自身的存储区域。合约的存储被划分为 2²⁵⁶ 个槽（slots），每个槽大小为 32 字节（bytes），且所有槽的初始值均为 0。

### 状态变量是如何存储的？

Solidity 会按照状态变量在合约中声明的顺序，从槽位 0 开始，自动将每个已定义的状态变量映射到存储槽位中。
在下图中，我们可以看到变量 `a`、`b` 和 `c` 是如何映射到各自的存储槽位的：

![](https://pbs.twimg.com/media/GJs-fFZasAAQwVl?format=jpg&name=large)

当变量占用少于 32 字节（bytes）时，EVM 会使用 0 进行填充，直到占满整个槽位的 32 字节，再将填充后的值写入存储。
许多类型的变量都小于 32 字节，例如：`bool`、`uint8`、`address`。

![](https://pbs.twimg.com/media/GJs_0ozasAAZDOP?format=jpg&name=medium)

如果我们仔细设计状态变量的大小与声明顺序，EVM 就能将多个变量打包到同一个存储槽位中，从而减少存储占用。
以上图的 `PaddedContract` 为例，我们可以通过重新排序变量的声明，让 EVM 自动将它们打包。
`PackedContract` 展示了这种重新排列后的效果，它仅改变了变量声明的顺序：

![](https://pbs.twimg.com/media/GJvVjgRbQAAinx3?format=jpg&name=medium)

不过，将变量打包而不使用填充，也会带来一些潜在问题。
如果被打包的变量并不经常一起使用，虽然打包能节省存储空间，但在读写其中任意一个变量时，可能会显著增加 gas 成本。
例如，如果我们需要频繁读取某个变量，而另一个被打包在同一槽位的变量很少使用，那么将它们分开存储反而更合适。

因此，这种“存储空间节省”与“访问成本增加”之间的权衡，是开发者在编写合约时必须认真考虑的设计点。

### 映射在智能合约存储中是如何存储的？

对于映射（mapping），其声明所在的槽位仅作为标记该 mapping 存在的“基础槽位”（base slot）。
要查找某个键对应的值，EVM 会使用以下公式计算其实际存储位置：
`keccak256(key + base slot)`。

下图的示例可以帮助我们更直观地理解这一点：

![](https://pbs.twimg.com/media/GJvUO6_bUAAkXLp?format=jpg&name=medium)

## `delegatecall`

Solidity 中存在一种特殊的消息调用形式，称为 `delegatecall`。它与普通的消息调用非常相似，但不同之处在于：目标地址（target address）中的代码会在调用合约（calling contract）的上下文中执行，也就是使用调用者本身的存储、地址和余额，并且 `msg.sender` 与 `msg.value` 都保持不变。

这意味着合约可以在运行时动态加载并执行来自其他合约地址的代码。
在这种模式下，存储（storage）、地址以及余额都属于调用方合约，只有代码来自被调用合约。

得益于此特性，Solidity 才得以实现“库”（library）功能：可复用的库代码可以在多个合约之间共享，而无需每个合约都重复部署逻辑。例如用于实现复杂的数据结构。

顾名思义，`delegatecall` 是一种“委托调用”机制。调用者合约通过它执行目标合约中的函数，但在运行目标合约逻辑时，其执行上下文并不是用户，而是调用者合约本身。

![](https://pbs.twimg.com/media/GMMgUtibMAAJPsi?format=jpg&name=medium)

那么，当合约通过 `delegatecall` 调用目标合约时，存储状态会发生什么变化呢？

由于 `delegatecall` 会在调用者的存储上下文中执行目标合约代码，所有状态变量的读写操作都将作用于调用者合约的存储，而不是目标合约的存储。

例如，假设我们有一个代理（Proxy）合约和一个逻辑（Business）合约。当用户调用代理时，代理合约将使用 `delegatecall` 调用逻辑合约的函数。逻辑代码会正常执行，但所有状态变更都发生在代理合约的存储中，而不是逻辑合约本身。

### Callcode 和 Delegatecall 的比较

![](https://proxies.yacademy.dev/assets/images/Comparison_Callcode_Delegatecall.png)

`callcode` 与 `delegatecall` 在存储行为上是一致的：二者都会执行实现合约（implementation）的代码，并对代理合约（proxy）的存储进行读写。
它们的主要区别在于 `msg.value` 与 `msg.sender` 的处理方式。

- 在 `callcode` 中：

  - `msg.value` 可以在实现合约中被改变。
  - `msg.sender` 会被替换为代理合约的地址。

- 在 `delegatecall` 中：

  - `msg.value` 和 `msg.sender` 在代理与实现合约中都保持完全一致。

## Proxy 模式

## 最简 Proxy

代理合约（proxy）本身并不是天然可升级的，但它是几乎所有可升级代理模式的基础。对代理合约发起的调用会通过 `delegatecall` 被转发到实现（implementation）合约，也称为逻辑合约（logic contract）。

在某些变体中，只有当调用者与指定的“所有者”地址匹配时，代理才会执行转发逻辑。

**实现地址** - 在代理合约中是不可变的。

**升级逻辑** - 纯代理合约本身不具备可升级性。

**合约验证** - 可以正常在 Etherscan 等区块浏览器上验证。

**使用场景**

- 适用于需要部署多个逻辑相同或非常相似的合约实例的场景。

**优点**

- 部署成本低。

**缺点**

- 每次调用都需要执行一次 `delegatecall`，会增加额外 gas 开销。

**示例**

- Uniswap V1 AMM 池
- Synthetix

**已知漏洞/风险**

- 实现合约中不应使用 `delegatecall` 或 `selfdestruct`，否则可能导致不可预期的代理存储篡改或合约销毁风险。

## 可初始化 Proxy

大多数现代代理（proxy）都是可初始化的。使用代理模式的主要优势之一是：你只需部署一次实现合约（逻辑合约），然后可以部署多个指向该实现的代理合约。
然而，这种模式也带来一个问题，在创建新的代理实例时，无法使用已部署的实现合约的构造函数（constructor）。

为了解决这一限制，通常会通过调用 `initialize()` 函数来设置初始存储值。

**使用场景**

- 适用于大多数在代理部署时需要初始化存储的代理模式。

**优点**

- 允许在部署新代理时设置初始状态。

**缺点**

- 容易受到初始化相关攻击，尤其是“未初始化代理”风险。

**示例**

- 广泛用于现代代理模式，如 [TPP](#透明-proxy-模式-tpp) 与 [UUPS](#通用可升级-proxy-标准-uups)，除非该代理场景在部署时不需要进行存储初始化。

**已知漏洞/风险**

- 未初始化代理（Uninitialized Proxy）

**进一步阅读**

- [OpenZeppelin's Initializable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable)

## 可升级 Proxy

可升级代理与普通代理结构类似，不同之处在于：实现合约（implementation contract）地址是可修改的，并被存储在代理合约自身的存储中。此外，代理合约中还包含带有访问控制的升级函数。最早的可升级代理之一由 Nick Johnson 在 2016 年编写。

出于安全考虑，通常建议使用访问控制机制，以区分普通用户调用与具有升级权限的管理员调用。

**实现地址** - 保存在代理合约的存储中。

**升级逻辑** - 实现在代理合约内部。

**合约验证** - 取决于具体实现，有时无法在 Etherscan 等区块浏览器上完全验证。

**使用场景**

- 极简的可升级合约设计，适合学习与概念验证（PoC）项目。

**优点**

- 通过代理复用实现合约，可降低部署成本。
- 实现合约逻辑可升级。

**缺点**

- 容易出现存储冲突与函数冲突问题。
- 相较现代可升级模式，安全性较弱。
- 每次调用都需要执行一次 `delegatecall`，会增加额外 gas 开销。

**已知漏洞/风险**

- 实现合约中不应使用 `delegatecall` 或 `selfdestruct`，否则可能导致不可预期的代理存储篡改或合约销毁风险。
- 未初始化代理（Uninitialized Proxy）
- 存储冲突（Storage Collision）
- 函数冲突（Function Selector Clashing）

**进一步阅读**

- [The First Proxy Contract](https://ethereum-blockchain-developer.com/110-upgrade-smart-contracts/05-proxy-nick-johnson/)
- [Writing Upgradeable Contracts](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)

## EIP-1967 可升级 Proxy

EIP-1967 是为了解决 存储冲突（storage collision）而出现的“标准化方案”。

它与传统的可升级代理类似，但通过采用 非结构化存储模式（unstructured storage pattern） 来显著降低存储冲突的风险。具体来说，它不会将实现合约地址存放在槽位（slot）0 或任何常规的存储槽中。

### 什么是非结构化存储模式？

在使用代理时，一个关键问题会很快浮现：
代理合约的存储位置（尤其是 `_implementation` 等变量）可能与逻辑合约的变量发生冲突。

举例来说：

假设代理合约中用于存储实现合约地址的变量是：

```solidity
address public _implementation;
```

而逻辑合约（例如一个简单的代币合约）中声明的第一个变量为：

```solidity
address public _owner;
```

这两个变量都是 32 字节大小。在 EVM 看来，它们都占用执行上下文中的第一个槽位。当逻辑合约内部写入 `_owner` 时，由于使用的是 `delegatecall`，写入动作会作用在代理合约的存储中，并覆盖 `_implementation`。
这就是所谓的存储冲突。

![](./public/1.png)

为避免这种冲突，OpenZeppelin Upgrades 采用了 非结构化存储方案：

它不会把 `_implementation` 写入常规槽位，而是将其存储在一个“伪随机”的槽位中。
该槽位足够唯一，使逻辑合约在同一槽位声明变量的概率几乎可以忽略不计。

同样的原则也被用于代理合约可能需要存储的其他变量，例如管理员地址（具有更新 `_implementation` 权限）。

![](./public/2.png)

OpenZeppelin 合约使用以下槽位来存储实现合约的地址：
`keccak256("eip1967.proxy.implementation") - 1`\*。

由于该槽位已成为行业标准，Etherscan 等区块浏览器也能自动识别并显示代理结构。

- 减 1 是为了增加安全性。
  如果不减 1，该槽位的原像是已知字符串，可能通过 mapping 的 slot 计算机制（mapping 的 `keccak256(key + slot)`）意外覆盖。但减去 1 后，其原像未知，更难被意外写入。

[EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) 还进一步定义了管理员（auth）槽位以及 Beacon Proxy 的标准槽位。

**实现地址** - 存储在代理合约的一个唯一槽位中。

**升级逻辑** - 根据具体实现方式而有所不同。

**合约验证** - 是的，大多数 EVM 区块浏览器都支持该代理模式的验证。

**使用场景**

- 当你需要比基础可升级代理更高的安全性，并希望避免存储冲突时。

**优点**

- 显著降低存储冲突风险。
- 拥有良好的区块浏览器兼容性。

**缺点**

- 依然容易受到函数选择器冲突影响。
- 安全性仍低于更现代的方案（如 TPP / UUPS）。
- 每次调用都需要执行一次 `delegatecall`，会增加额外 gas 开销。

**已知漏洞**

- 实现合约中不应使用 `delegatecall` 或 `selfdestruct`，否则可能导致不可预期的代理存储篡改或合约销毁风险。
- 未初始化代理（Uninitialized Proxy）
- 函数冲突（Function Selector Clashing）

**进一步阅读**

- [EIP-1967 Standard Proxy Storage Slots](https://ethereum-blockchain-developer.com/110-upgrade-smart-contracts/09-eip-1967/)
- [The Proxy Delegate](https://fravoll.github.io/solidity-patterns/proxy_delegate.html)

## 透明 Proxy 模式 (TPP)

**注意** - 这模式已被 [EIP-2535 Diamond Standard](https://eips.ethereum.org/EIPS/eip-2535) 所取代。

这是函数冲突的"解决方案"。

如前所述，代理通过 `delegatecall` 将所有调用转发到逻辑合约(logic contract)，逻辑合约执行实际的代码。
然而，可升级代理自身需要一些管理函数，例如用于将代理升级到新的逻辑合约的函数：

```solidity
upgradeTo(address newImplementation)
```

这就提出了一个问题：如果逻辑合约也有一个同名和同签名的函数，该怎么办？
在调用 `upgradeTo` 时，调用者是打算调用代理的管理函数或逻辑合约的函数？
这种歧义可能导致意外的错误，甚至被恶意利用。

冲突也可能发生在名称不同的函数之间。作为合约公共接口的一部分，每个函数在字节码层面都会以一个 4 字节(byte)的标识符(function selector)来识别。该标识符由函数名称与参数列表组合计算而成，但由于它只有 4 字节，两个不同名称的不同参数的函数也有可能碰巧得到相同的值。
Solidity 编译器会在同一个合约内部侦测并报告这样的冲突，但如果冲突发生在不同合约之间(例如代理合约与逻辑合约之间)，编译器无法自动发现。

**解决方案**

我们通过透明代理模式来处理这个问题。透明代理的目标是让用户感知不到代理的存在，使其看起来就像在与实际的逻辑合约交互。因此，当用户在代理上调用 `upgradeTo` 时，应该始终执行逻辑合约中的函数，而不是代理管理函数。

那么，我们如何允许管理代理合约呢？答案是基于消息发送者(`msg.sender`)。透明代理将根据调用者地址决定哪些调用被委托给底层逻辑合约：

- 如果调用者是代理的管理员，代理将不会委托任何调用，只会处理自己具备的管理函数。
- 如果调用者是任何其他地址，代理将始终委托调用，即使它匹配代理自己的函数之一。

该模式通常搭配 EIP-1967 存储槽位来管理实现地址和管理员地址。

**实现地址** - 位于代理合约中的唯一存储槽位（EIP-1967）。

**升级逻辑** - 位于代理合约中，使用修饰符(modifier)判读调用者是否管理员，以确定是否 `delegatecall`

**合约验证** - 是的，大多数 EVM 区块浏览器都支持验证。

**使用场景**

- 广泛用于需要可升级性且避免函数冲突、存储冲突风险的项目

**优点**

- 解决管理员函数冲突的问题(管理员永远不会被 `delegatecall` 到逻辑合约)
- 由于升级逻辑存在于代理本身，即使代理未初始化或实现合约被 `selfdestruct`，仍然可以重新设置新的实现地址。
- 使用 EIP-1967 的存储槽位可降低存储冲突风险。
- 区块浏览器兼容性。

**缺点**

- 每次调用不仅会产生 `delegatecall` 的成本，还会产生检查调用者是否为管理员的 SLOAD 成本。
- 由于升级逻辑在代理上，使得字节码体积更大，因此部署成本更高。

**示例**

- dYdX
- USDC
- Aztec

**已知漏洞**

- 实现合约中不允许使用 `delegatecall` 和 `selfdestruct`
- 未初始化代理
- 存储冲突

**进一步阅读**

- [ERC-1538: Transparent Contract Standard](https://eips.ethereum.org/EIPS/eip-1538)

## 通用可升级 Proxy 标准 (UUPS)

如果我们将升级逻辑移到实现合约中会怎样？

[EIP-1822](https://eips.ethereum.org/EIPS/eip-1822) 描述了一种可升级代理模式的标准，其中升级逻辑存储在实现合约内。这样代理就不需要检查调用者是否为管理员，从而节省 gas。同时，也消除了实现合约中的函数与代理中升级逻辑发生冲突的可能性。

UUPS 的缺点是它被认为比 TPP 风险更大。如果代理未正确初始化，或实现合约被 `selfdestruct`，那么就没有办法拯救代理，因为升级逻辑位于实现合约里。

UUPS 代理在升级时还包含一个额外的检查，确保新的实现合约是可升级的。

此代理合约通常也会 EIP-1967 的存储槽位。

**实现地址** - 位于代理合约中的唯一存储槽位（EIP-1967）。

**升级逻辑** - 位于实现合约中。

**合约验证** - 是的，大多数 EVM 区块浏览器都支持验证。

**使用场景**

- 目前最广泛采用的可升级合约模式。

**优点**

- 消除实现合约中的函数与代理合约冲突的风险，因为升级逻辑存在于实现合约，代理除了 `delegatecall` 到实现合约的 `fallback()` 之外没有其他逻辑。
- 与 TPP 相比，运行时 gas 更低，因为代理不再需要检查调用者是否为管理员。
- 部署新代理的成本降低，因为代理只包含 `fallback()`，没有其他逻辑。
- 使用 EIP-1967 的存储槽位可降低存储冲突风险。
- 区块浏览器兼容性。

**缺点**

- 由于升级逻辑存在于实现合约，必须格外小心，确保实现合约不能 `selfdestruct` 或由于初始化不当而处于不良状态。
- 所有调用仍会产生 `delegatecall` 的成本。

**示例**

- Superfluid
- Synthetix

**已知漏洞**

- 未初始化代理
- 函数冲突
- `selfdestruct`

**进一步阅读**

- [EIP-1822](https://eips.ethereum.org/EIPS/eip-1822)

## Beacon Proxy

截至目前为止讨论的大多数代理都将实现合约地址存储在代理合约存储中。Beacon 模式将实现合约的地址存储在独立的"beacon"合约中。Beacon 地址使用 EIP-1967 存储模式存储在代理合约中。

对于其他类型的代理，每次升级实现合约都需要逐一更新所有代理。但对于 Beacon 代理，只需要更新 Beacon 合约本身，所有基于该 Beacon 的代理都会自动指向新实现，极大减少操作量。

代理上的 Beacon 地址以及 Beacon 上的实现合约地址都可以由管理员设置。这在需要管理大量代理，并以不同方式对其分组时非常有用。

Beacon 合约背后的核心理念是可复用性。如果多个代理指向同一个逻辑合约地址，那么每次更新逻辑合约时，都必须更新所有代理。这在 gas 成本上极其昂贵。那么拥有一个为所有代理返回逻辑合约地址的 Beacon 合约会更有意义。

因此，使用 Beacon 时，相当于在中间有另一层智能合约，它返回实际逻辑合约的地址。

**实现地址** - 位于 Beacon 合约中的唯一存储槽。Beacon 地址位于代理合约中的唯一存储槽。

**升级逻辑** - 升级逻辑通常位于 Beacon 合约中。

**合约验证** - 是的，大多数 EVM 区块浏览器都支持验证。

**使用场景**

- 当多个代理合约可以通过升级 Beacon 同时升级。
- 适用于涉及基于多个实现合约的大量代理合约的情况。Beacon 代理允许同时更新多个代理组。

**优点**

- 能够轻松同时升级多个代理合约。

**缺点**

- 从存储中获取 Beacon 合约地址、调用 Beacon 合约、然后从存储中获取实现合约地址的 gas 开销，以及使用代理所需的额外 gas。
- 增加了系统各复杂度。

**示例**

- USDC
- Dharma

**已知漏洞**

- 实现合约中不允许使用 `delegatecall` 和 `selfdestruct`
- 未初始化代理
- 函数冲突

## Diamond Proxy

[EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) "Diamonds" 是一种模块化智能合约系统，可在部署后进行升级或扩展，并几乎不存在代码大小限制。
EIP 中的定义如下：

> Diamond 是一个具有外部函数的合约，这些外部函数由称为 _facets_ 的合约提供。
> Facets 是独立的合约，它们可以共享内部函数、库和状态变量。

Diamond 模式由一个核心的 `Diamond.sol` 代理合约组成。除了其他存储外，此合约包含一个函数注册表，这些函数可以在被称为 `facets` 的外部合约上调用。

Diamond 代理的术语表使用了一套独特的专业词汇：

![](./public/3.png)

此标准是对 [EIP-1538](https://eips.ethereum.org/EIPS/eip-1538) (TPP) 的改进，其设计动机也与之相同。

一个已部署的 facet 可以被任意数量的 diamonds 使用。

下图展示了两个 diamonds 共享两个 facets 的情况：

- FacetA 被 Diamond1 使用

- FacetA 被 Diamond2 使用

- FacetB 被 Diamond1 使用

- FacetB 被 Diamond2 使用

![](https://eips.ethereum.org/assets/eip-2535/facetreuse.png)

### 术语

1. Diamond
   是一个门面(facade)智能合约，通过 `delegatecall` 调用其 facets 中的函数。
   Diamond 是有状态的。数据存储在 diamond 合约的存储中。

2. Facet
   是一个无状态的智能合约或具有外部函数的 Solidity 库。
   Facet 部署后，其一个或多个函数可以被添加到一个或多个 diamonds 中。
   Facet 自身不会在其合约存储中保存数据，但可以定义状态并读写任意 diamond 的存储。
   术语 _facet_ 来自钻石行业。它是钻石的一个侧面或平面。

3. Loupe facet
   提供内省(introspection)函数的 facet。
   在钻石行业中，loupe 是用于查看钻石的放大镜。

4. 不可变函数(Immutable function)
   指无法被替换或移除的外部函数（因为它直接在 diamond 中定义，或者因为 diamond 的逻辑不允许修改它）。

5. Mapping(在本 EIP 中)，
   映射是两个事物之间的关联，不指特定的实现。

**合约验证** - 可以使用 Louper 工具在 Etherscan 上验证合约。

**使用场景**

- 适用于需要最高级别可升级性与模块化互操作性的复杂系统。

**优点**

- 提供一个稳定且长久可用的合约地址。从单一地址发出事件(event)可以简化事件处理。
- 可用于拆分超过 Spurious Dragon 限制(24kb)的大型合约。

**缺点**

- 函数路由需要额外访问存储，导致 gas 成本增加。
- 系统结构复杂，存储冲突风险上升。
- 对仅需简单可升级性的场景，复杂性可能过高。

**示例**

- Simple DeFi
- PartyFinance

**已知漏洞**

- 实现合约中不允许使用 `delegatecall` 和 `selfdestruct`

**进一步阅读**

- [Introduction to EIP-2535 Diamonds](https://eip2535diamonds.substack.com/p/introduction-to-the-diamond-standard)
- [Dark Forest and the Diamond standard](https://blog.zkga.me/dark-forest-and-the-diamond-standard)

### 比较 Proxy 模式

下表对比了 Diamond、Transparent 和 UUPS 三种代理模式的优缺点：

![](/public/4.png)

## Proxy 漏洞安全指南

### 1. 对不存在的外部合约执行 Delegatecall

在使用 `delegatecall` 时，EVM 不会自动检查目标合约是否存在。如果被调用的外部合约地址不存在，`delegatecall` 仍然可能返回 `true`。
正如 [Solidity 官方文档所警告的](https://docs.soliditylang.org/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions)：

> 低级函数 `call`、`delegatecall` 和 `staticcall` 在目标账户不存在时，依然会将其第一个返回值设为 `true`。这是 EVM 设计上的行为。因此，如果你需要确保目标账户存在，必须在调用前先自行检查。

**测试步骤**

第一步是确认 `delegatecall` 实际要调用的外部合约地址。
如果目标地址可能不存在合约，并且在调用 `delegatecall` 之前没有对其进行部署检查，那么 `delegatecall` 可能会意外返回 `true`，从而导致错误行为无法被察觉。

### 2. Delegatecall 与 Selfdestruct 漏洞

当 `selfdestruct` 与 `delegatecall` 同时使用时，可能会出现不可预见的边缘情况(edge case)。特别是，如果合约 B 中的某个函数包含 `selfdestruct`，而合约 A 通过 `delegatecall` 调用该函数，那么被销毁的将是合约 A 本身。

**测试步骤**

识别此类漏洞相对简单。

1. 若合约包含 `delegatecall` 到用户提供的目标地址（例如作为外部函数的参数），则存在重大的整体安全风险。
2. 若 `delegatecall` 到硬编码的目标合约

   - 检查目标合约是否有 `selfdestruct`。
   - 若目标合约没有 `selfdestruct`，但有进一步的 `delegatecall` ，则继续沿着 `delegatecall` 路径检查。
   - 一旦链路中的任意合约包含 `selfdestruct` ，则最初发出 `delegatecall` 的合约可能会被删除。

3. 如果使用 EIP-1167 最小代理模式，那么由该主合约克隆出的所有代理实例都有可能被销毁。

**CTF 示例**

1. [Ethernaut Level 25 "Motorbike"](https://ethernaut.openzeppelin.com/level/0x3A78EE8462BD2e31133de2B8f1f9CBD973D6eDd6)

**进一步阅读**

1. [OpenZeppelin's Proxy Vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)

### 3. 函数冲突漏洞

当编译后的智能合约使用函数选择器(function selector)来标识函数时，就可能发生函数冲突。函数选择器是依据函数名称与参数列表哈希所得的前 4 个字节(byte)。如果两个不同函数的哈希前 32 位一致，那么它们会生成相同的 4 字节选择器，即便名称完全不同。

编译器会检测同一合约中重复的函数选择器，但如果冲突发生在不同的合约之间(例如代理与实现合约之间)，编译器无法阻止。
大多数代理都可能出现函数冲突，但并非全部如此。尤其是 UUPS 代理，由于所有自定义逻辑都集中在实现合约中，因此通常不容易受到函数选择器冲突的影响。

**测试步骤**

要测试此漏洞，你可以收集代理合约和实现合约的函数选择器，并比较它们是否存在冲突。

常用工具包括：

- solc
  `solc --hashes MyContract.sol` 将列出所有函数选择器。
- [Slither - Function ID printer](https://github.com/crytic/slither/wiki/Printer-documentation#function-id)
  可输出所有函数 ID。
- [Slither Check Upgradeability](https://github.com/crytic/slither/wiki/Upgradeability-Checks#functions-ids-collisions)
  可直接检测代理相关的函数冲突。

**进一步阅读**

1. [Tincho Function Clashing Writeup](https://forum.openzeppelin.com/t/beware-of-the-proxy-learn-how-to-exploit-function-clashing/1070)
2. [Nomic Labs' Blog Post](https://medium.com/nomic-foundation-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357)
3. [OpenZeppelin Docs Explaining Function Clashing](https://docs.openzeppelin.com/sdk/2.5/pattern#transparent-proxies-and-function-clashes)

### 4. 存储冲突漏洞

当代理合约和实现合约中的存储槽位布局不一致时，就会发生存储冲突。问题在于：实现合约中的变量定义了数据应存放的位置，但由于代理合约使用 `delegatecall` ，实现合约实际上会读写代理合约的存储空间。如果两者的存储槽位未对齐，就可能导致存储被错误覆盖，从而产生存储冲突。

**测试步骤**

要测试此漏洞的方法有很多。

1. 可视化存储布局
   使用 [sol2uml](https://github.com/naddison36/sol2uml) 工具，将代理合约和实现合约的存储布局可视化，以直接检查是否有任何不匹配。

2. 程序化方式提取槽位并比较
   使用 [slither-read-storage](https://github.com/crytic/slither/blob/master/slither/tools/read_storage/README.md) 提取两个合约使用的存储槽位，再进行比对。

3. 使用专门比对存储布局的的工具
   Slither 提供 [slither-check-upgradeability](https://github.com/crytic/slither/wiki/Upgradeability-Checks) 工具，包含多个检测器，可检查代理相关的存储布局问题。

**CTF 示例**

1. [Solidity by Example](https://solidity-by-example.org/hacks/delegatecall/)
2. [Ethernaut Level 6 "Delegation"](https://ethernaut.openzeppelin.com/level/0x73379d8B82Fda494ee59555f333DF7D44483fD58)
3. [Ethernaut Level 16 "Preservation"](https://ethernaut.openzeppelin.com/level/0x7ae0655F0Ee1e7752D7C62493CEa1E69A810e2ed)
4. [Ethernaut Level 24 "Puzzle Wallet"](https://ethernaut.openzeppelin.com/level/0x725595BA16E76ED1F6cC1e1b65A88365cC494824)

**进一步阅读**

1. [MixBytes Storage Collision Audit](https://mixbytes.io/blog/collisions-solidity-storage-layouts)

### 5. 未初始化 Proxy 漏洞

当合约构造函数（constructor）在部署时已自动执行，为什么代理合约仍然需要一个 `initialize` 函数？OpenZeppelin 在[此处](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat)对此给出了详细解释。

合约的构造函数代码只会在部署时运行一次；然而，实现合约（也称逻辑合约）的构造函数无法在代理合约的执行上下文中运行。由于实现合约的构造函数始终在其自身环境中执行，因此无法通过构造函数来初始化代理合约中的状态。

因此，实现合约必须在代理的存储上下文中写入 `_initialized` 变量，这也是实现合约必须提供 `initialize` 函数的根本原因。初始化调用必须通过代理执行，而不是通过实现合约本身，从而引入了一个潜在的竞争条件（race condition）：初始化必须在部署之后、且在代理被外部使用之前安全完成。

为避免这种竞争条件，可以为 `initialize` 函数加入基于地址的权限控制修饰符（modifier），将其调用权限限制为特定的 `msg.sender`。

在 [OpenZeppelin 4.1.0 至 4.3.2 之间](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/security/advisories/GHSA-q4h9-46xg-m3x9)，曾出现过未初始化 UUPS Proxy 漏洞的特定变体。

**测试步骤**

要检测此漏洞，可以查找用于初始化检查的状态变量（例如 [`_initialized`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/25aabd286e002a1526c345c8db259d57bdf0ad28/contracts/proxy/utils/Initializable.sol#L62)）在代理合约中的存储槽位，或其他用于判断“是否为首次调用”的类似变量。

在使用 OpenZeppelin 的 `Initializable.sol` 时，该文件中的私有变量 `_initialized` 会用于记录初始化状态：

- `_initialized == 0` → 合约尚未初始化
- `_initialized == 1` → 合约已完成初始化

Slither 的 [slither-check-upgradeability](https://github.com/crytic/slither/wiki/Upgradeability-Checks) 工具包含多个与初始化相关的漏洞检测器，可用于辅助检查。

**进一步阅读**

1. [OpenZeppelin Proxy Vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)
2. [iosiro Disclosure of OpenZeppelin Vulnerability](https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)

## Proxy 开发指南

以下资源可作为使用代理构建可升级智能合约的开发参考：

1. [Writing Upgradeable Contracts by OpenZeppelin](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)
2. [Proxy API by OpenZeppelin](https://docs.openzeppelin.com/contracts/4.x/api/proxy)
3. [How to create a Beacon Proxy](https://medium.com/coinmonks/how-to-create-a-beacon-proxy-3d55335f7353)
4. [How to use the UUPS proxy pattern to upgrade smart contracts](https://blog.logrocket.com/using-uups-proxy-pattern-upgrade-smart-contracts/#setup-with-hardhat-and-openzeppelin)

## 参考资料

1. Offical ERC Releases
2. OpenZeppelin
3. yAcademy
4. Solidity documentation
