---
title: 合约语言
tags: [publishing]
keywords: building, serving, serve, build
last_updated: July 3, 2016
sidebar: mydoc_sidebar_cn
permalink: mydoc_contract_language.cn.html
folder: mydoc
---

# Equity 语言入门

## 目录

  - [Equity 简介](#equity-简介)
  - [合约(contract)](#合约contract)
  - [条款函数(clause)](#条款函数clause)
  - [合约与条款函数的参数(Parameters)](#合约与条款函数的参数parameters)
  - [条款的支付需求(Required payments)](#条款的支付需求required-payments)
  - [语句 (Statements)](#语句-statements)
      - [Verify 语句 (Verify statements)](#verify-语句-verify-statements)
      - [Unlock 语句 (Unlock statements)](#unlock-语句-unlock-statements)
      - [Lock 语句 (Lock statements)](#lock-语句-lock-statements)
  - [Equity 数据类型](#equity-数据类型)
  - [表达式 (Expressions)](#表达式-expressions)
  - [函数 (Functions)](#函数-functions)
  - [Equity 合约的规则 (Rules for contracts)](#equity-合约的规则-rules-for-contracts)
  - [示例 (Examples)](#示例-examples)
      - [单公钥锁定的合约](#单公钥锁定的合约)
      - [借贷合约](#借贷合约)

## Equity 简介

Equity 是一种高级语言，专门用来编写运行在 [Bytom](https://github.com/Bytom/bytom) 上的合约程序。通过编写、部署 Equity 智能合约，可以对 Bytom 上的各类资产进行操作。

为方便下文的理解，在正式介绍 Equity 语言之前，首先对 Bytom 上的资产做简要介绍：

* Bytom 采用 BUTXO 结构，即区块链上记录由多种不同类型的 UTXO 构成的账本。

* 每一笔 UTXO 都有两个重要属性：`asset_id` 和 `amount`，即 `资产编号` 和 `资产数量`。一般将这里的 `资产数量` 称作 `value`，有时也用 `value` 抽象地指代一笔 UTXO。

* 所有 `value(UTXO)` 都被其对应的合约程序 `program` 锁定，只有满足 `program` 中定义的条件的输入，才能解锁 `program` 锁定的 `value`。

因此，用 Equity 编写智能合约，其目的就是 "描述用智能合约锁定哪些资产，以及定义在哪些条件下可以解锁指定的资产"。

## 合约(contract)

Equity 合约程序由一个用 `contract` 关键字定义的合约(contract)构成。合约的构成形式为：

- `contract` _ContractName_ `(` _parameters_ `)` `locks` _value_ `{` _clauses_ `}`

具体来讲：

* _ContractName_ 是标识符，代表合约的名称，在编写合约时自定义。

* _parameters_ 是合约的参数列表，这些参数的类型应当在 [Equity 数据类型](#equity-数据类型) 的范围内。

* _value_ 是标识符，表示锁定在合约中的 value，编写合约时自定义。

* _clauses_ 是一个或多个条款函数(clause)组成的列表。

## 条款函数(clause)

条款函数(clause)描述一种解锁合约中的 value 的方法，以及所需的任何数据(有时还需要支付新的 value)。clause 的结构是：

- `clause` _ClauseName_ `(` _parameters_ `)` `{` _statements_ `}`

或者这样：

- `clause` _ClauseName_ `(` _parameters_ `)` `requires` _payments_ `{` _statements_ `}`

具体来讲：

* _ClauseName_ 是标识符，表示 clause 的名称，编写合约时自定义。

* _parameters_ 是 clause 的参数列表，这些参数的类型应当在 [Equity 数据类型](#equity-数据类型) 的范围内。

* _payments_ 是**支付需求**组成的列表，在 [条款的支付需求](#条款的支付需求required-payment) 中详细讲解。

* _statements_ 由一个或多个**语句**组成。每个语句都应是 `verify`、`lock` 或 `unlock` 中的一种，在 [语句](#语句-statements) 中详细讲解。

## 合约与条款函数的参数(Parameters) ##

合约和条款的参数需指明 **名称(name)** 和 **数据类型(type)**。参数的写法是：

- _name_ `:` _TypeName_

有多个参数时的写法为：

- _name1_ `:` _TypeName1_ `,` _name2_ `:` _TypeName2_ `,` ...

为了简洁，可以像这样合并相同类型的相邻参数：

- _name1_ `,` _name2_ `,` ... `:` _TypeName_

所以这两种合约的声明是等价的：

- `contract LockWithMultiSig(key1: PublicKey, key2: PublicKey, key3: PublicKey)`
- `contract LockWithMultiSig(key1, key2, key3: PublicKey)`

可用的数据类型有：

- `Integer` `Amount` `Boolean` `String` `Hash` `Asset` `PublicKey` `Signature` `Program`

将在 [Equity 数据类型](#equity-数据类型) 中介绍这些数据类型。

## 条款的支付需求(Required payments)

有时候，条款函数(clause)会要求支付一些其他的 value 才能解锁合约中已有的 value。例如用美元交易换取欧元的时候，就需要支付美元以解锁合约中的欧元。

在这种情况下，必须在 clause 中使用 `requires` 语法来为所需的 value 指定名称并指定其数量和资产类型，形式为：

- `clause` _ClauseName_ `(` _parameters_ `)` `requires` _name_ `:` _amount_ `of` _asset_

在这里，_name_ 是标识符，表示支付的 value 的名称。_amount_ 是 `Amount` 数据类型的[表达式 (expressions)](#表达式-expressions)。而 _asset_ 是 `Asset` 数据类型的表达式。

还有的 clause 需要支付两种或更多种的 value 才能解锁合约。这时，可以在 `requires` 后按这种格式来写：

- ... `requires` _name1_ `:` _amount1_ `of` _asset1_ `,` _name2_ `:` _amount2_ `of` _asset2_ `,` ...

## 语句 (Statements)

条款函数(clause)的主体部分包含一个或多个语句：

* `verify` 语句用来验证表达式的结果是否为真。

* `unlock` 语句用来解锁合约中锁定的 value。

* `lock` 语句可以将原合约中的 value 以及支付给条款函数的 value 锁定至新的合约中。

### Verify 语句 (Verify statements)

`verify` 语句的格式如下：

- `verify` _expression_

expression(表达式)的值必须是布尔型(`Boolean`)。只有当每个 `verify` 语句的检测结果都为 true 时，cluase 才能被成功执行。

一些例子如下：

- `verify above(blockNumber)` 检测当前块的区块高度是否高于 `blockNumber`。
- `verify checkTxSig(key, sig)` 检测给定的签名，是否与预先设定的公钥以及解锁合约的交易匹配。
- `verify newBid > currentBid` 检测某个数值，是否(严格)大于另一个数值。

### Unlock 语句 (Unlock statements)

Unlock 语句只有一种格式：

- `unlock` _value_

其中，_value_ 是在 `contract` 声明时、 `locks` 关键字后指定的、合约中锁定的 value 的名称。该语句释放了合约中的 value，使得 value 可被用于任何用途。即没有指定一个新的合约来锁定释放的 value。

### Lock 语句 (Lock statements)

Lock 语句的格式如下：

- `lock` _value_ `with` _program_

该语句用 _program_ 锁定了 _value_ (此处的 _value_ 可以是合约中锁定的 value 的名称，也可以是向 clause 支付的其他 value)。_program_ 处的表达式必须是 `Program` 类型。

## Equity 数据类型     

Equity 编译器支持以下数据类型：

* **integer - 整数类型**

  * **Amount**: 资产的数量 (更确切地说，是一些以最小单位记的资产数量)，范围是 0 ~ 2^63，类型为 Amount 的变量表示锁定在合约中的资产

  * **Integer**: 介于 2^-63 ~ 2^63 之间的整数

* **boolean - 布尔类型**

  * **Boolean**: 值为 true 或 false

* **string - 字符串类型，以十六进制字节串的形式出现**

  * **String**: 普通的字符串

  * **Hash**: 哈希得到的字符串

  * **Asset**: 资产编号

  * **PublicKey**: 公钥

  * **Signature**: 私钥签名后的消息

  * **Program**: 用字节形式表示的程序码

## 表达式 (Expressions)

Equity 支持很多种表达式，这些表达式可用于 `verify` 和 `lock` 语句，也可以用在 clause 声明时的 `requires` 部分里。

本节的 _expr_ 代表一段表达式(expression)。例如 _expr1_ `+` _expr2_ 就可以看作是 `20 + 15` 或更复杂的语句。

- `-` _expr_ : 对数学表达式取负值
- `~` _expr_ : 对字节串(byte string)做按位翻转(invert the bits)

下面的每种表达式都使用数字型操作数(numeric operands)(即`Integer`或`Amount`类型)，并且返回一个 `Boolean` 型的结果：

- _expr1_ `>` _expr2_ : 检测 _expr1_ 是否大于 _expr2_
- _expr1_ `<` _expr2_ : 检测 _expr1_ 是否小于 _expr2_
- _expr1_ `>=` _expr2_ : 检测 _expr1_ 是否大于或等于 _expr2_
- _expr1_ `<=` _expr2_ : 检测 _expr1_ 是否小于或等于 _expr2_
- _expr1_ `==` _expr2_ : 检测 _expr1_ 是否等于 _expr2_
- _expr1_ `!=` _expr2_ : 检测 _expr1_ 是否不等于 _expr2_

下面的表达式用于 byte strings，且返回值也是 byte strings：

- _expr1_ `^` _expr2_ : 得到两操作数按位异或(XOR)的结果
- _expr1_ `|` _expr2_ : 得到两操作数按位或(OR)的结果
- _expr1_ `&` _expr2_ : 得到两操作数按位与(AND)的结果

下面的表达式用于数值型(numeric)操作数(`Integer`或`Amount`)，返回数值型的结果：

- _expr1_ `+` _expr2_ : 将两操作数相加
- _expr1_ `-` _expr2_ : 从 _expr1_ 中减去 _expr2_
- _expr1_ `*` _expr2_ : 将两操作数相乘
- _expr1_ `/` _expr2_ : 用 _expr1_ 除以 _expr2_
- _expr1_ `%` _expr2_ : 即 _expr1_ 对 _expr2_ 取余
- _expr1_ `<<` _expr2_ : 将 _expr1_ 按位左移 _expr2_ 位
- _expr1_ `>>` _expr2_ : 将 _expr1_ 按位右移 _expr2_ 位

还有其他的表达式类型：

- `(` _expr_ `)` : 依然表示 _expr_ 本身
- _expr_ `(` _arguments_ `)` : 表示函数调用，传入的参数 _arguments_ 是一些用逗号分隔的表达式，具体可查阅下面的 [函数 (functions)](#函数-(functions))
- 单独出现的标识符表示变量本身
- `[` _exprs_ `]` : 表示一个列表(list)，其中 _exprs_ 是一些用逗号分隔的表达式(列表参数目前仅用于`checkTxMultiSig`)

- 以 `-` 开头的一串数字序列属于 `Integer` 类型
- 单引号 `'...'` 之间的字节序列(bytes)表示一个字符串
- 前缀 `0x` 后跟 2n 个十六进制数字，则表示 n 字节长的字符串 (注：每个十六进制数为4位，每个字符为8位)

## 函数 (Functions)

Equity 内置了一些函数，可用在 `verify` 语句和其他地方：

- `abs(n)` 返回数字 n 的绝对值。
- `min(x, y)` 返回 x 和 y 两数中较小的那个数的值。
- `max(x, y)` 返回 x 和 y 两数中较大的那个数的值。
- `size(s)` 传入一个任意类型的表达式，返回其以字节为单位时的长度，返回值类型是 `Integer`。
- `concat(s1, s2)` 对两个字符串做拼接，从而生成一个新字符串。
- `concatpush(s1, s2)` 传入两个字符串 s1 和 s2，返回的结果是 s1 与一段 Bytom 虚拟机操作码(BVM opcodes) 的拼接，这段操作码的作用是将 s2 送入 Bytom 虚拟机的栈。此函数通常用于嵌套合约中。具体可参考 [BVM 操作码](https://github.com/boy-good/Docs/wiki/BTM-VM-OPS---%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6ops%E6%8C%87%E4%BB%A4%E8%AF%B4%E6%98%8E#%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%A4%84%E7%90%86%E6%93%8D%E4%BD%9C%E7%A0%81)。
- `below(height)` 传入 `Integer` 类型的值，返回 `Boolean` 型的值，判断当前区块高度是否低于参数 height，如果是则返回 true，否则返回 false。
- `above(height)` 传入 `Integer` 类型的值，返回 `Boolean` 型的值，判断当前区块高度是否高于参数 height。
- `sha3(s)` 传入 `string` 类的值，返回其 SHA3-256 的哈希值(返回值类型为`Hash`)。
- `sha256(s)` 传入 `string` 类的值，返回其 SHA-256 的哈希值(返回值类型为`Hash`)。
- `checkTxSig(key, sig)` 传入 `PublicKey` 与 `Signature`，返回 `Boolean` 型的值，以检测签名 `sig` 是否与公钥 `key` 和解锁交易匹配。
- `checkTxMultiSig([key1, key2, ...], [sig1, sig2, ...])` 分别传入列表形式的 `PublicKeys` 与 `Signatures`，返回 `Boolean` 型的值，以检测是否每个 `sig` 都与 `key` 以及解锁交易匹配。关于列表参数顺序的问题：并非所有公钥都需要有签名与其匹配，但所有签名都应有匹配的公钥，并且这些公钥/签名必须在各自的列表中按相同的顺序排列。

## Equity 合约的规则 (Rules for contracts)

只有遵循以下规则的 Equity 合约才是有效的：

- 标识符不能互相冲突。例如，clause 参数的名称不能与 contract 参数的名称相同。(但是，两个不同的 clause 可以使用相同的参数名，这不构成冲突)
- 每个 contract 参数必须至少在一个 clause 中被使用。
- 每个 clause 参数必须在 clause 内被使用。
- 每个 clause 必须使用 `lock` 或 `unlock` 语句处理合约中锁定的 value。
- 每个 clause 还必须用 `lock` 语句处理所有的支付给 clause 的 value(如果有的话)。

## 示例 (Examples)

### 单公钥锁定的合约

下面这个合约叫做 `LockWithPublicKey`，是最简单的合约之一。通过阅读本文档，可以详细了解它是如何工作的。

```
contract LockWithPublicKey(pubKey: PublicKey) locks value {
  clause spend(sig: Signature) {
    verify checkTxSig(pubKey, sig)
    unlock value
  }
}
```

这个合约的名称是 `LockWithPublicKey`。它锁定了一笔资产，被称作 `value`。在锁定 value 的交易(即创建合约的交易)中必须为 `LockWithPublicKey` 指定合约参数，即`pubKey`。

`LockWithPublicKey` 包含一个 clause，这意味着有一种(且只有一种)途径解锁 `value`：把 `Signature` 作为参数，调用 `spend` 条款函数。

`spend` 条款中的 `verify` 检查传入的 `Signature(sig)`是否与 `pubKey` 和尝试解锁 value 的交易匹配。如果检查成功，则 `value` 被解锁。

### 借贷合约

接下来是一个更具挑战性的例子：叫做 `LoanCollateral`。

```
contract LoanCollateral(assetLoaned: Asset,
                        amountLoaned: Amount,
                        repaymentHeight: Integer,
                        lender: Program,
                        borrower: Program) locks collateral {
  clause repay() requires payment: amountLoaned of assetLoaned {
    lock payment with lender
    lock collateral with borrower
  }
  clause default() {
    verify above(repaymentHeight)
    lock collateral with lender
  }
}
```

合约的名称是 `LoanCollateral`。它锁定了一些被称为 `collateral` 的 value。在创建此合约的交易中，必须指定五个合约参数：`assetLoaned`、`amountLoaned`、`repaymentHeight`、`lender` 和 `borrower`。

此合约包含两个 clause，也就是说，有两种途径来解锁 `collateral`：

- `repay()` 不需要传入数据，但需要支付与 `amountLoaned` 等量的资产 `assetLoaned`
- `default()` 既不需要传入数据，也不需要支付资产

合约的含义是：贷方 `lender` 在借方 `borrower` 已出具抵押物 `collateral` 的情况下，将与 `amountLoaned` 等量的资产 `assetLoaned` 借给借方。如果贷款被偿还给贷方，则将抵押物退还给借方。但是，如果还款截止时间已过，则贷方有权为自己索取抵押物。

`repay()` 条款函数通过两个简单的 `lock` 语句将付款发送给贷方，且将抵押物发送给借方。回想一下，向区块链参与者发送 value，其实就是将 value 锁定至允许接收者解锁的 program。

`default()` 条款函数的 `verify` 语句确保在截止区块高度已过时，可以将抵押物 `collateral` 锁定给贷方 `lender`。需要注意的是，这一过程并不是在截止时间到达的时刻自动发生的。贷方(或某个人)必须构造一个调用 `LoanCollateral` 合约的 `default()` 条款的交易，才能解锁抵押物 `collateral` 并将其重新锁定给 `lender`。