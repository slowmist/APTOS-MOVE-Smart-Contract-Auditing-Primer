# APTOS MOVE 智能合约审计入门

## 前言

本文旨在对已经有其他语言的智能合约审计基础的审计人员进行 APTOS 智能合约审计的简单入门引导。我们将从账户模型开始认识 APTOS，随后将简述 Move 语言在 APTOS 中一些特性，最后将通过 checklist 进行简单的审计入门。

## 账户模型 Account

`APTOS 账户是资源与模块的容器，且具有在区块链上执行特定操作(如交易发起)的功能，每个账户都由一个 32 字节的账户地址进行标识。`

### 模块&脚本 Module&Script

Move 有两种不同类型的程序：模块(Module)和脚本(Script)。

模块相当于智能合约，我们可以在模块中编写各种函数逻辑等，与以太坊智能合约的不同之处在于模块虽然是全局存储但本身并不存储任何资产，资产都以资源的形式全局存储于用户账户中。Move 可以并行执行([Blcoks-STM](https://medium.com/aptoslabs/block-stm-how-we-execute-over-160k-transactions-per-second-on-the-aptos-blockchain-3b003657e4ba))而不会使不同账户存储的资产受到影响。同时模块并非是一个单独的地址，而是存储在用户账户地址下(密钥轮换不会改变地址，只是使用新的密钥访问该地址)。

`访问调用示例：0x93bb93f633ad80fedd608b36c846f70b54ced5c6140fd3eca227e4eeb3f3206a::lend::borrow`

脚本相当于 action 集，其只是一个暂时用于指令执行的代码片段不会存储到全局存储中(即一次性的)。一个脚本中只能有一个函数，一般用于对模块的调用。

### 资源 Resources

资源是一种可以作为全局存储中的键的特殊结构体，即资源是具有全局存储能力的结构体，反之亦成立。资源也是存储在用户账户下，如用户持有的 APT、USDT 代币是资源，用户所有的 Capabilities 也是资源。

资源具有以下特性：

* 不可能被复制与丢弃，只具有 key 与 store 两种能力。因此审计中需要特别关注其创建与销毁

* 必须存储在账户下。因此，只有在分配帐户后才会存在，并且只能通过该帐户访问

*  一个帐户同一时刻只能容纳一个某类型的资源。如果一个账户需要持有多个资源实例，比如存储多个 NFT 都属于同一个集合，那么可以创建一个容器资源用于储存，其本质上是一个向量(SUI 则允许账户拥有无限数量的同类型资源)

### 签名者 Signer

Signer 是 Move 内置的资源类型，但只有 drop 能力。与 Solidity 中的 sender 类似，但并不是一个地址类型，它包含着代表此地址行使权利的能力(如：此地址有着铸币能力，那么接收到 signer 的 module 也可以通过 signer 使用此能力)。但它并不能通过用户指令等方式进行创建，只能通过虚拟机创建，在虚拟机运行带有 signer 类型参数的脚本之前，它会自动创建 signer 值并将它们传递给脚本。

*因此如果我们将 signer 给到了不明 module 将可能导致我们账户下的资源被恶意操作*

### 签名方案 Signature schemes

APTOS 原生支持单签与多签方案(原子交换会变得更容易)，以下将进行简单说明。

#### 单签

APTOS 在 Ed25519 算法的基础上使用 PureEdDSA 方案生成公私钥对 (privkey_A, pubkey_A)。

使用公钥派生一个 32 字节的认证密钥：

```js
auth_key = sha3-256(pubkey_A | 0x00) // 其中`|`表示连接，`0x00`是单字节的单签策略标识
```

在第一次生成的情况下此认证密钥也是账户地址(密钥轮换时会发生改变但地址还是初始的这个)，在进行单签时即直接使用生成的私钥进行签名即可(可以直接使用 SDK)。

#### 多签

APTOS 支持任意数量的签名者对交易进行签名(K-N)，一个账户具有 N 个签名人，至少其中的 K 个人签署之后，交易才会被认证为已签发，并且所有签名必须与交易一起提交(链上聚合)。

其通过 MultiED25519 标准进行密钥对生成，由 N 个公钥派生出认证密钥：

```js
auth_key = sha3-256(p_1 | . . . | p_n | K | 0x01) // 其中 | 表示连接， 0x01 是单字节的多签策略标识
```

多签账户也支持密钥[轮换](https://github.com/aptos-labs/aptos-core/pull/3626)，在第一次生成的情况下此认证密钥也是账户地址。



我们可以简单梳理下：

* 密钥对：成对存在并一起生成。
  - 私钥：用于签署交易。
  - 公钥：适合代表单签帐户，不适合多签帐户。

- 身份验证密钥：概括了一种表示单签名者和多签名帐户的方式。密钥轮换后将发生改变。
- 地址：用于表示单签名和多签名帐户，密钥轮换后也保持不变，是基本数据类型之一。

## 语言基础

以下语言基础将只讲述审计中 Move 最常见的特性，建议审计员在审计前应完整阅读 Damir 撰写的[快速语言入门文档](https://move-book.com/cn/index.html)，在实际接触 Move 时再搭配 [Move Book](https://move-dao.github.io/move-book-zh/) 进行深入了解。

### 类型

Move 语言的基本类型包含整型(u8、u64、u128)、布尔(bool)、地址(address)，不支持字符串和浮点数，动态类型将由向量(vector)进行表示。

除地址外，其余类型定于基本都与 Rust 相同，以下我们将对[地址类型](https://move-dao.github.io/move-book-zh/address.html#%E5%9C%B0%E5%9D%80address)进行简单介绍：

* 地址即为账户，可以理解为全局储存的索引，其表示着全局存储的位置(资源与模块都在地址下进行存储)

* 任何 uint128 类型的数值都可以用来表示地址，但在表达式中使用需要加 @ 前缀

  ```rust
  let a1: address = @0x1;
  ```

* 可以在 toml 文件中设置地址别名

  ```rust
  swap = "aa16ed61ecbc91e151cffd7c34bbf9a551235d977eca8f7aed5d9d45e8790b1f"
  ```

### 函数

`函数使用 fun 关键字声明，后跟函数名称、类型参数、参数、返回类型、获取标注，最后是函数体`

#### 可见性

APTOS Move 中函数有三种可见性：内部私有、公开可见 `public`、友元可见 `public(friend)`

* 内部私有：仅能在同一模块中调用，不能从其他模块、脚本、终端进行调用。
* 公开可见 `public`：不仅在同一模块中可调用，任意其他模块、脚本都可以进行调用，但不能直接通过终端发起交易调用。
* 友元可见 `public(friend)`：只能在同一模块中以及在模块中被声明为友元模块才可调用。(例如：friend 0x1::n)

#### 修饰符

APTOS Move 中函数只有 `entry` 修饰符，其表示我们可以像执行脚本一样直接发起一笔交易进行调用，但其也受到可见性的约束。

#### 函数调用

APTOS Move 中进行函数调用时，名称可以通过别名或完全限定名指定，且对外部 module 进行调用是需先进行 use，因此暂时没有类似 solidity 开放式 call 的方法，这将减少很多未知外部调用导致安全问题。

```rust
    use 0x42::example::{Self, zero};
    fun call_zero() {
        0x42::example::zero();
        example::zero();
        zero();
    }
```

#### 资源访问

APTOS Move 中函数在对本模块资源创建的进行访问时需要使用 acquires 标识符进行声明，没有 acquires 标识将无法对资源进行操作。审计时注意是否有无效的 acquire。

```rust
public fun extract_balance(addr: address): u64 acquires Balance {...}
```

#### 资源操作

APTOS Move 中可以通过下面五种指令创建、删除、更新全局存储中的资源：

| 操作符 | 描述 |
| :-----------------------------------: | :------------------: |
|       move_to<T>(&signer,T)        | 资源转给 signer |
|       move_from<T>(address): T        | 取出(删除)资源并返回 |
| borrow_global_mut<T>(address): &mut T |     可变引用资源     |
|     borrow_global<T>(address): &T     |    不可变引用资源    |
|       exists<T>(address): bool        |   返回资源是否存在   |

在审计中需注意资源创建必须 move_to 到某个账户下，移出时必须被解构或者存储在其他账户下。

### 能力 Abilities

能力是 Move 语言中的一种类型特性，用于控制对给定类型的值所允许的操作，一共有四种类型：`copy`、`drop`、`store`、`key`。这些能力是通过对某些字节码指令的进行访问控制来实现的，这为值在全局存储中如何使用提供了细粒度控制。

#### copy 复制

copy 能力允许具有此能力的类型的值被复制，如果一个值具有 copy 能力，那么这个值内部的所有值都有 copy 能力。

#### drop 丢弃

drop 能力允许类型的值被丢弃，即程序执行后值会被有效的销毁而不必担心被转移。同样的，如果一个值具有 drop 能力，那么这个值内部的所有值都有 drop 能力。

#### store 存储

store 能力允许具有这种能力的类型的值被存储到全局存储中。同样的，如果一个值具有 store 能力，那么这个值内部的所有值都有 store 能力。

#### key 键值

key 能力允许此类型作为全局存储中的键(我们可以通过键对全局状态进行访问)。如果有一个值有 key 能力，那么这个值包含的所有字段值也都具有 store 能力。

*Note: 几乎所有内置的基本类型具都有 copy、drop、store 能力，但 singer 除外，它只有 drop 能力*

### 其他

以上仅列出了 APTOS Move 语言的一些特性，审计人员仍应尽可能的熟悉并掌握其语言文档中的其他部分。

快速入门可以阅读 Damir Shamanaev 撰写的语言文档([EN](https://move-book.com/),[CN](https://move-book.com/cn/index.html))，但由于撰写时期较早文档中缺少了很多细节。

推荐阅读 Move Book([EN](https://move-language.github.io/move/),[CN](https://move-dao.github.io/move-book-zh/))，虽然其撰写是为 Diem 进撰写，但其详细阐述了 Move 的语言各种基础语法。

## 审计入门

经上述账户模型与语言基础的学习，我们对 APTOS Move 已经有了初步的了解，接下来将通过 Checklist 将 Move 与 Solidity 进行比较以快速审计入门

### 溢出审计 Overflow Audit

**说明**

Move 与 Solidity^0.8.0 相同，在进行数学运算时会进行溢出检查，运算溢出交易将失败。但需要注意的是位运算并不会进行检查。

**定位**

寻找代码中进行位运算的位置。

### 重入攻击审计 Reentrancy Attack Audit

**说明**

我们都知道在 Solidity 中重入是在一笔正常的合约调用交易中，被插入一笔非预期的(外部)调用，从而改变整体的业务调用流程然后进行非法获利的问题。涉及到外部合约调用的位置，都可能存在潜在的重入风险。目前的重入问题我们可以分为三类：单函数重入、跨函数重入以及跨合约重入。

但在 Move 中发生重入的可能性极低，为什么呢？

* Move 中无动态调用，其外部调用都需先通过 use 进行导入，即外部调用都是预期、确定的
* 无 Native 代币转账触发 Fallback 功能

### 条件竞争审计 Race Conditions Audit

#### 重排攻击审计 Reordering Attack Audit

**说明**

与以太坊一样，APTOS 中的验证者也可以对用户提交的交易进行排序，因此在审计中我们仍需要注意在同一个区块中对交易进行排序而进行获利的问题。

**定位**

* 是否有对`函数调用前`合约的数据状态进行预期管理。

* 是否有对`函数执行中`的合约数据状态进行预期管理。

* 是否有对`函数调用后`的合约数据状态进行预期的管理。

### 闪电贷攻击审计 Flashloan Attack Audit

**说明**

与 Solidity 一样，Move 也有闪电贷的用法(SUI中称`Hot Potato`)。用户可以在一笔交易中借取大量的资金任意使用，只需在这笔交易内归还资金即可。恶意用户通常使用闪电贷放大自身的资金规模进行价格操控等大资金攻击。

**定位**

分析协议本身的算法(奖励、利率等)、预言机依赖是否合理。

### 权限漏洞审计 Permission Vulnerability Audit

**说明**

与 Solidity 一样，权限漏洞这部分和业务的需求以及功能设计关系较大，所以遇见较为复杂的 Module 的时候，就需要和项目方一一确认各个方法的调用权限。这里的权限一般是指函数的可见性和函数的调用权限。

**定位**

* 检查和确认所有函数方法的可见性及调用权限，在项目评估阶段就需要项目方提供设计文档，审计时候根据设计文档中的描述一一确认权限。

* 梳理项目方角色的权限范围，如果项目方角色的权限会影响用户的资产，则存在权限过大的风险。

### 安全设计审计 Safety Design Audit

#### 外部模块使用安全审计 External Module Safe Use Audit

**说明**

在 Move 中在使用外部模块时会使用 `use` 关键字进行声明。但需要注意的是，APTOS Module 是默认可升级的，但升级不能破环现有的存储结构、函数签名。因此我们审计中需要关注所依赖的外部模块是否安全稳定。

*升级策略可以在 `Move.toml` 中指定 `upgrade_policy` 为`可升级 compatible` 或 `不可升级 immutable`*

**定位**

寻找代码中使用了 use 模块的位置。

#### 外部调用安全审计 External Call Function Security Audit

与外部模块使用审计项相同，由于 Move 中进行外部调用需要先将外部模块导入，因此理论上外部调用的结果都是开发人员所预期，所需主要的是外部模块的稳定性。

#### 区块数据依赖安全审计 Block data Dependence Security Audit

**说明**

区块数据中常见的时间戳、区块高度等资源都存在 [0x1](https://apscan.io/account/0x1?tab=resources) 账户下，但根据 [Aptos-framework](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/framework/aptos-framework/sources) 这些数据都可以由 vm_reserved 进行更改(应该在共识时写入)且没有对写入的数据进行严格的限制。因此当模块中使用此类区块数据时需要分析其被操控或预测是否会对协议造成破坏。

**定位**

寻找模块中使用区块数据的位置。

### 拒绝服务审计 Denial of Service Audit

**说明**

可以导致智能合约无法正常使用的代码逻辑错误，兼容性错误等安全问题。(目前还未了解到 APTOS 有 gaslimit 的限制)

**定位**

* 关注业务逻辑的可用性

* 关注与外部模块交互时的兼容性问题

### Gas 优化审计 Gas Optimization Audit

**说明**

与以太坊一样，APTOS 也有 Gas 机制，任何模块脚本调用都会消耗 Gas。因此对一些冗长且高复杂度的代码进行优化是有必要的。

**定位**

* 涉及到复杂的调用看是否可以解耦。

* 涉及到高频率的调用看是否可以优化函数内部执行的效率。

### 设计逻辑审计 Design Logic Audit

**说明**

主要关注代码逻辑上的问题，如：业务流程、实现上是否存在缺陷，是否会导致的非预期的情况发生，代码实现是否与预期不一致等。

**定位**

根据角色的范围，影响的数据范围来梳理可能的调用路径，然后根据梳理出来的调用路径与业务上预期内的调用路径进行比较，发现非预期的调用情况。

* 【角色的涉及范围】需要列出业务设计上该`流程`所涉及的角色。

* 【影响的数据范围】需要列出业务设计上该业务`实现`中所影响的数据范围。

* 【流程调用的路径】需要列出业务设计上该业务`流程`调用的路径。

### 算术精度误差审计 Arithmetic Accuracy Deviation Audit

**说明**

与 Solidity 一样，在 Move 中是没有浮点类型的，所以在算术运算过程中如果出现的结果是浮点型的，那么就会有算术精度上的误差，算术精度的误差很难避免，但是可以缓解。

**定位**

寻找代码中进行算术运算的位置。

### **能力安全使用审计** Capability Safe Use Audit

**说明**

Capability 本质上也是“资源”，其是具有执行特定操作能力的帐户所拥有的“资源”，拥有此资源的账户通过此资源进行特权操作(如：代币铸造，合约升级等)。但需要注意的是根据具体的业务需要这些能力资源也是可以被定义为可复制的，因此梳理这些能力资源是否按照预期的使用是很重要的。

**定位**

梳理代码中对定义的 Capability 的使用，寻找对 cap 能力的非预期的使用问题。

### **资源安全使用审计** Resource Security Usage Audit

**说明**

Move 中所有全局存储的数据都以资源的形式体现，资源都存储于各个账户下并且可以通过模块进行传递。因此在涉及其他模块资源传递时需要注意其资源使用是否符合预期是否存在兼容性问题。

**定位**

寻在代码中 acquires 资源的位置，并检查所有对资源的操作是否符合预期。

### 其他 Others

未在上述表述中体现的内容。

## 参考

[Aptos Move Framework Code](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move)

[Aptos Developer Documentation](https://aptos.dev/)

[Move book maintained by @damirka](https://move-book.com/cn/index.html)

[Move book maintained by the Move core team](https://move-dao.github.io/move-book-zh/introduction.html)

