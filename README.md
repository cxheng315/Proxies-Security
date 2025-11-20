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

欢迎来到 QuillAudits 的 Proxy 与可升级智能合约仓库。本仓库收录了理解 Solidity 智能合约可升级性所需的所有技术概念、理论基础、实践说明以及实现示例。

区块链常被视为具有数据不可篡改性的系统，人们长期以来认为“一旦部署，就无法更改”。这对于历史交易信息依然成立，但对于智能合约的存储与地址而言，情况并非完全如此。

#### 为什么需要 Proxy 以及如何使用它？

从设计上来看，部署在区块链上的合约代码是不可变的。虽然不可篡改性是区块链的核心特性之一，但在需要可升级性时也会带来挑战。许多新手可能会疑惑：既然区块链强调不可篡改，为何还需要“升级”？事实上，仍有许多需要修改代码的场景，例如错误修复、安全补丁、性能优化、新功能发布等。

在直接进入 Proxy 和可升级合约之前，我们需要先了解 Solidity 的 `delegatecall` 操作码是如何工作的。而在理解 `delegatecall` 之前，了解 EVM 如何在存储中保存合约变量也会很有帮助。

## 智能合约存储布局

合约存储布局是指的是管理合约中存储变量在长期存储中如何排列的规则。几乎所有的智能合约都有需要长期保存的状态变量。在 Solidity 中，开发者可以使用 3 种不同类型的存储空间来指示 EVM 将变量储存在哪里：`memory`、`calldata` 和 `storage`。这里，我们将讨论存储布局。

每个合约都有自己的存储区域，这是一个持久的可读写存储空间区域。合约只能从自己的存储中读取和写入。合约的存储被划分为 2²⁵⁶ 个槽位，每个槽位为 32 字节(bytes)。所有槽位的初始值均为 0。

### 状态变量是如何存储的？

Solidity 会按照状态变量在合约中声明的顺序，从槽位 0 开始，自动将每个已定义的状态变量映射到存储槽位中。
在这里，我们可以看到变量 a、b 和 c 是如何映射到它们各自的存储槽位的。

![](https://pbs.twimg.com/media/GJs-fFZasAAQwVl?format=jpg&name=large)

为了在存储中保存占用少于 32 字节(bytes)的变量，EVM 会用 0 进行填充，直到占满槽位的所有 32 字节(bytes)，然后再将填充后的值写入存储。
许多变量类型都小于 32 字节(bytes)槽位的大小，例如：`bool`、`uint8` 和 `address`。

![](https://pbs.twimg.com/media/GJs_0ozasAAZDOP?format=jpg&name=medium)

如果我们仔细考虑合约状态变量的大小和声明顺序，EVM 会将变量打包到存储槽位中，以减少使用的存储空间。
以上面的 PaddedContract 为例，我们可以重新排序状态变量的声明，让 EVM 将变量打包到存储槽位中。
PackedContract 展示了这个例子，它只是 PaddedContract 中变量的重新排序：

![](https://pbs.twimg.com/media/GJvVjgRbQAAinx3?format=jpg&name=medium)

然而，将变量打包在一起而不使用填充，也存在一个需要注意的问题。
如果打包的变量并不是经常一起使用，那么虽然打包可以节省存储占用，但在读取或写入这些变量时，反而可能显著增加 gas 成本。
例如，如果我们需要经常读取一个变量而不读取打包的另一个变量，那么最好不要打包这些变量。
这是开发者在编写合约时必须考虑的设计权衡。

### 映射在智能合约存储中是如何存储的？

对于映射(mapping)，标记槽位仅用于标记该 mapping 的存在（即基础槽位 base slot）。
要查找某个键对应的的值，使用公式 `keccak256(key + base slot)`。
我们可以通过以下示例更好地理解它：

![](https://pbs.twimg.com/media/GJvUO6_bUAAkXLp?format=jpg&name=medium)

## `delegatecall`

存在一种名为 `delegatecall` 的特殊消息调用形式。它与普通消息调用几乎相同，不同之处在于：目标地址(target address)中的代码会在调用合约(calling contract)的上下文（即调用者的地址）中执行，并且 `msg.sender` 和 `msg.value` 的值保持不变。

这意味着合约可以在运行时动态加载来自其他地址的代码。存储(storage)、当前地址和余额仍然指向调用合约，只有代码是取自被调用的地址。

这使得在 Solidity 中实现"库"(library)功能成为可能：即可复用的库代码可以应用于某个合约的存储，例如用于实现复杂的数据结构。

`delegatecall`，顾名思义，是一种调用机制。调用者合约通过它来调用目标合约中的函数，但当目标合约执行其逻辑时，执行上下文并不是发起调用的用户，而是调用者合约本身。

![](https://pbs.twimg.com/media/GMMgUtibMAAJPsi?format=jpg&name=medium)

那么当合约使用 `delegatecall` 调用目标合约时，存储状态将会如何发生变化呢？

因为在使用 `delegatecall` 调用目标合约时，执行上下文在调用者合约上，因此所有状态变更逻辑都会反映在调用者的存储上。

例如，假设有代理(Proxy)合约和逻辑(Business)合约。代理合约 `delegatecall` 到逻辑合约函数。如果用户调用代理合约，代理合约将 `delegatecall` 到逻辑合约，函数将被执行。但所有状态变更都将反映在代理合约存储中，而不是逻辑合约。

### Callcode 和 Delegatecall 的比较

![](https://proxies.yacademy.dev/assets/images/Comparison_Callcode_Delegatecall.png)

`callcode` 和 `delegatecall` 在存储上具有相同的行为。也就是说，它们都可以执行实现合约(implementation)的代码并对代理合约(proxy)的存储进行操作。它们之间的区别在于 `msg.value` 和 `msg.sender`。
在 `callcode` 中，`msg.value` 可以在实现合约中被自定义为新的值，`msg.sender` 会被改为代理合约的地址。
在 `delegatecall` 中，`msg.value` 和 `msg.sender` 在代理合约和实现合约中都保持不变。

## Proxy 模式

## 最简 Proxy

代理合约(proxy)本身并不是天然可升级的，但它是几乎所有可升级代理模式的基础。对代理合约发起的调用会通过 `delegatecall` 被转发到实现(implementation)合约。实现合约也被称为逻辑合约。

在某些变体中，只有当调用者与指定的"所有者"地址匹配时，代理才会转发调用。

**实现地址** - 在代理合约中不可变。

**升级逻辑** - 纯代理合约不具备可升级性。

**合约验证** - 可在 Etherscan 等区块浏览器上正常验证。

**使用场景**

- 适用于需要部署多个代码基本相同的合约的情况。

**优点**

- 部署成本低。

**缺点**

- 每次调用都会产生一次 `delegatecall` 的成本。

**示例**

- Uniswap V1 AMM 池
- Synthetix

**已知漏洞**

- 实现合约中不允许使用 `delegatecall` 和 `selfdestruct`。

## 可初始化 Proxy

大多数现代代理(proxy)都是可初始化的。使用代理的主要优势之一是，您只需要部署一次实现合约（也称为逻辑合约），然后可以部署许多指向它的代理合约。然而，这样做的缺点是，在创建新的代理时，您无法在已部署的实现合约中使用构造函数(constructor)。

为了解决这个问题，会使用 `initialize()` 函数来设置初始存储值。

**使用场景**

- 适用于在代理合约部署时需要设置存储的多数代理模式。

**优点**

- 允许在新代理部署时设置初始存储。

**缺点**

- 易受到与初始化相关的攻击，特别是未初始化的代理。

**示例**

- 此功能用于大多数现代代理类型，例如 [TPP](#透明-proxy-模式-tpp) 和 [UUPS](#通用可升级-proxy-标准-uups)，除了在代理部署时不需要设置存储的使用场景。

**已知漏洞**

- 未初始化代理

**进一步阅读**

- [OpenZeppelin's Initializable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable)

## 可升级 Proxy

可升级代理与普通代理类似，不同之处在于实现合约(implementation contract)地址是可设置的，并保存在代理合约的存储中。代理合约还包含带权限控制的升级函数。最早的可升级代理合约之一由 Nick Johnson 于 2016 年编写。

出于安全考虑，通常建议使用某种访问控制机制，以区分普通调用者与具有权升级权限的管理员。

**实现地址** - 位于代理存储中。

**升级逻辑** - 位于代理合约中。

**合约验证** - 根据具体实现方式，可能无法在 Etherscan 等区块浏览器上正常验证。

**使用场景**

- 一个极简的可升级合约。适用于学习类项目。

**优点**

- 通过代理复用实现合约，可降低部署成本。
- 实现合约是可升级的。

**缺点**

- 容易出现存储冲突和函数冲突。
- 相比现代方案安全性较低。
- 每次调用都会产生一次 `delegatecall` 的成本。

**已知漏洞**

- 实现合约中不允许使用 `delegatecall` 和 `selfdestruct`
- 未初始化代理
- 存储冲突
- 函数冲突

**进一步阅读**

- [The First Proxy Contract](https://ethereum-blockchain-developer.com/110-upgrade-smart-contracts/05-proxy-nick-johnson/)
- [Writing Upgradeable Contracts](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)

## EIP-1967 可升级 Proxy

这是存储冲突的"解决方案"。

它与普通的可升级代理类似，但通过采用非结构化存储模式(unstructured storage pattern)来减少存储冲突的风险。具体来说，它不会把实现合约地址存放在槽位(slot) 0 或任何标准存储槽中。

### 什么是非结构化存储模式？

使用代理时，很快会出现一个问题：变量在代理合约中的存储位置可能与逻辑合约的变量发生冲突。
举例：
假设代理合约中存储实现合约地址的变量为：

```solidity
address public _implementation;
```

而逻辑合约是一个简单的代币，它的第一个变量为

```solidity
address public _owner;
```

这两个变量都是 32 字节大小，就 EVM 所知，它们占据代理调用的执行流程的第一个槽位。当逻辑合约写入 `_owner` 时，它在 Proxy 状态的范围内执行，实际上写入的是 `_implementation`。这个问题可以称为"存储冲突"。

![](./public/1.png)

解决这个问题的方法有很多，而 OpenZeppelin Upgrades 所采用的"非结构化存储"方案的工作原理如下：
它不是将 `_implementation` 地址存储在代理合约的第一个存储槽位(storage slot)，而是选择一个伪随机槽位。
这个槽位足够随机，因此逻辑合约在同一槽位声明变量的概率可以忽略不计。
同样的随机化存储槽位的原则，也被应用在代理合约可能拥有的其他变量上，例如管理员地址（拥有更新 `_implementation` 的权限）等。

![](./public/2.png)

OpenZeppelin 合约使用 keccak256("eip1967.proxy.implementation") - 1\*作为实现合约地址的存储槽位。由于这个槽位被行业广泛采用，区块浏览器也能够识别、检测代理的使用情况。

\*减一提供了额外的安全性，因为如果没有它，槽位有一个已知的原像，但减去 1 后，原像是未知的。
对于已知的原像，存储槽位可能通过映射被覆盖，例如，其键的存储槽位是使用 keccak-256 哈希确定的。

[EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) 也指定了管理员存储（auth）的槽位以及 Beacon Proxy。

**实现地址** - 位于代理合约中的唯一存储槽位。

**升级逻辑** - 根据实现方式而有所不同。

**合约验证** - 是的，大多数 EVM 区块浏览器都支持这种代理模式的验证。

**使用场景**

- 当您需要比基础 可升级代理 更高的安全性时。

**优点**

- 降低存储冲突的风险。
- 区块浏览器兼容性

**缺点**

- 容易受到函数冲突的影响。
- 相比现代方案安全性较低。
- 每次调用都会产生一次 `delegatecall` 的成本。

**已知漏洞**

- 实现合约中不允许使用 `delegatecall` 和 `selfdestruct`
- 未初始化代理
- 函数冲突

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

在使用 `delegatecall` 时，EVM 不会自动验证外部合约是否存在。如果调用的外部合约不存在，返回值将为 `true`。[Solidity 文档中已对此给出了警告说明](https://docs.soliditylang.org/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions)，其中提到：

> 低级函数 `call`、`delegatecall` 和 `staticcall` 在调用的账户不存在时，会将第一个返回值设为 `true`，这是 EVM 的设计行为。因此，如果需要确保账户存在，必须在调用前先进行检查。

**测试步骤**

第一步是确定 `delegatecall` 实际调用的外部合约地址。
如果目标地址可能不存在合约，而在调用 `delegatecall` 之前又没有检查该合约是否部署，则 `delegatecall` 可能会意外返回 `true`，导致错误行为未被发现。

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

### 5. Uninitialized Proxy Vulnerability

When a contract constructor is called automatically, why are proxies still required to have an initialize function? OpenZeppelin provides an explanation of the cause [here](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat). The constructor code of a contract is executed only once during deployment; however, the implementation contract's (also known as the logic contract) constructor code cannot be executed within the framework of the proxy contract. The constructor cannot be used for this purpose since the implementation contract's constructor function will always run in the context of the implementation contract. Instead, the implementation contract must save the value of the \_initialized variable in the proxy contract context. Because the initialize call needs to go via the proxy, this is the reason the implementation contract has an initialize method. There is a potential race condition that should be addressed because the initialize call needs to occur independently of the implementation contract deployment. One way to prevent this race condition is to protect the initialize function with an address control modifier, limiting its ability to be initialized to a specific msg.sender.

A specific variant of the uninitialized UUPS proxy vulnerability is found in [the OpenZeppelin library between version 4.1.0 and 4.3.2](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/security/advisories/GHSA-q4h9-46xg-m3x9).

**Testing Procedure**

Find the storage slot of the [initialized state variable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/25aabd286e002a1526c345c8db259d57bdf0ad28/contracts/proxy/utils/Initializable.sol#L62) or a comparable variable that the initialization function uses to reverse if this is not the function's first call in order to test for this vulnerability. When using the OpenZeppelin private \_initialized variable from Initializable.sol, the contract has not been initialized if the \_initialized value is zero, and it has been initialized if the value is one.

Slither has a [slither-check-upgradeability](https://github.com/crytic/slither/wiki/Upgradeability-Checks) tool that has several initializer issue detectors.

**Further Reading**

1. [OpenZeppelin Proxy Vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)
2. [iosiro Disclosure of OpenZeppelin Vulnerability](https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)

## Development Guide to Proxies

The below listed resources can be used as development guides to create upgradeable smart contracts using proxies:

1. [Writing Upgradeable Contracts by OpenZeppelin](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)
2. [Proxy API by OpenZeppelin](https://docs.openzeppelin.com/contracts/4.x/api/proxy)
3. [How to create a Beacon Proxy](https://medium.com/coinmonks/how-to-create-a-beacon-proxy-3d55335f7353)
4. [How to use the UUPS proxy pattern to upgrade smart contracts](https://blog.logrocket.com/using-uups-proxy-pattern-upgrade-smart-contracts/#setup-with-hardhat-and-openzeppelin)

## References

1. Offical ERC Releases
2. OpenZeppelin
3. yAcademy
4. Solidity documentation
