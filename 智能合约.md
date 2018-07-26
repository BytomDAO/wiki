# bytom智能合约

- [equity合约简介](#equity合约简介)
- [合约交易构造流程](#合约交易构造流程)
  - [合约参数构造](#合约参数构造)
  - [编译合约](#编译合约)
  - [锁定合约](#锁定合约)
  - [查找合约utxo](#查找合约utxo)
  - [解锁合约](#解锁合约)
- [典型合约模板解析](#典型合约模板解析)
  - [单签验证合约](#单签验证合约)
  - [多签验证合约](#多签验证合约)
  - [pubkey哈希校验和单签验证合约](#pubkey哈希校验和单签验证合约)
  - [字符串哈希校验合约](#字符串哈希校验合约)
  - [币币交易合约](#币币交易合约)
  - [第三方信任机构托管合约](#第三方信任机构托管合约)
  - [抵押贷款合约](#抵押贷款合约)
  - [看涨期权合约](#看涨期权合约)
- [区块链系统模型简介](#区块链系统模型简介)

## equity合约简介
  equity是bytom用于表达合约程序而使用的高级语言，主要用于描述bytom链上特定的资产： 
  - 区块链上的所有资产都是锁定在合约`program`中， 资产值`value`（即UTXO）一旦被一个合约解锁`unlock`，仅仅是为了被一个或多个其他合约来进行锁定`lock`
  - 合约保护资产`value`（即UTXO）的方式是采用执行虚拟机并通过`verify`指令来验证的交易要花费这个资产`value`是否达到了我的条件

### 合约组成
  contract ContractName ( parameters ) locks value { clauses }
  - `ContractName` 合约名，用户自定义
  - `parameters` 合约参数列表，其类型名必须符合合约语言的基本类型 
  - `value` 资产值（即UTXO的资产类型和对应的值）标识符，用户可以自定义
  - `clauses` 条款（即函数）列表（一个或多个）

### 条款（函数）组成
  clause ClauseName ( parameters ) { statements }
  
  或
  
  clause ClauseName ( parameters ) requires name : amount of asset { statements }

  - `ClauseName` 条款/函数名，用户自定义
  - `parameters` 条款/函数参数列表
  - `payments` 花费UTXO需要的其他限制条件。例如进行币币交易的时候需要验证交易的另一方能够提供对应的资产值，该限制条件的使用场景主要为不同资产类型的合约交易           

### 语句组成
  statements 合约语句（一条或多条），代码语句只能是verify、lock和unlock
  - `verify`语句 模式如`verify expression`，其中`expression`的结果必须是bool类型，为true才能继续往下执行
  - `unlock`语句 模式如`unlock value`，其中`value`表示对应的资产值
  - `lock`语句 模式如`lock value with program`，其中`value`表示对应的资产值，而`program`必须为Program基本类型

合约的基本类型：（`Time`类型暂已停用）
    `Amount Asset Boolean Hash Integer Program PublicKey Signature String`

内置函数：（涉及时间的内置函数`before`和`after`暂已停用，替代方法为验证区块高度的函数`below`和`above`）
  - abs(n) 返回数值n的绝对值.
  - min(x, y) 返回两个数值x和y中最小的一个.
  - max(x, y) 返回两个数值x和y中最大的一个.
  - size(s) 返回任意类型的字节大小size.
  - concat(s1, s2) 返回连接两个字符串s1和s2生成新的字符串.
  - concatpush(s1, s2) 将两个字符串类型的虚拟机执行操作码s1和s2连接起来(即将s2拼接在s1的后面），然后将他们push到栈中. 该操作函数主要用于嵌套合约中.
  - below(height) 判断当前区块高度是否低于参数height，如果是则返回true，否则返回false.
  - above(height) 判断当前区块高度是否高于参数height，如果是则返回true，否则返回false.
  - sha3(s) 返回byte类型字符串参数s的SHA3-256的哈希运算结果.
  - sha256(s) 返回byte类型字符串参数s的SHA-256的哈希运算结果.
  - checkTxSig(key, sig) 根据一个PublicKey和一个Signature验证交易的签名是否正确.
  - checkTxMultiSig([key1, key2, ...], [sig1, sig2, ...]) 根据多个PublicKey和多个Signature验证交易的多重签名是否正确.

----

## 合约交易构造流程
### 合约参数构造
  合约参数主要涉及两个方面，一个是编译合约`contract`中的参数，另一个是解锁合约`clause`中的参数。合约的基本类型已做简单描述，对应编译合约的API接口`compile`中参数类型只有如下3种：
  - `boolean` - 布尔类型的合约参数，对应的基本类型是`Boolean`.
  - `integer` - 整数类型的合约参数，对应的基本类型包括：`Integer`、`Amount`.
  - `string` - 字符串类型的合约参数，对应的基本类型包括：`String`、`Asset`、`Hash`、`Program`、`PublicKey`.

注意事项：
  - 编译合约API接口`compile`只需要使用`contract`中的参数
  - 解锁合约只需要提供`clause`中的参数
  - `Signature`类型只能在`clause`的参数列表中出现，不能出现在编译合约的API中
  - 所有`string`类型的字符串都是必须以十六进制字节的string形式出现，否则调用编译合约API的时候会报错

如果合约参数中有`Publickey`和`Signature`(配套使用)，那么获取这些参数需要调用`list-pubkeys`接口获取。`Signature`只能出现在`clause`中，表示该参数仅仅在解锁合约的时候才会被使用，由于目前对交易的签名必须通过`sign-transaction`接口才能获取签名结果，所以解锁的时候只需提供签名的参数`root_xpub`和`derivation_path`即可，需要注意的是这些参数需要跟验证签名`pubkey`匹配起来，否则合约也会执行失败。

其中`list-pubkeys`的参数如下：
- `String` - *account_id*, 账户ID.

其请求和响应的json格式如下：
```js
// Request
{
  "account_id": "0G1JIR6400A02"
}

// Result
{
  "pubkey_infos": [
    {
      "derivation_path": [
        "010100000000000000",
        "0300000000000000"
      ],
      "pubkey": "c37d5531f393bc6a3568628c0c0e17801ea452e75d604deb01403c4b161659a3"
    },
    {
      "derivation_path": [
        "010100000000000000",
        "0200000000000000"
      ],
      "pubkey": "117d12e84bb19e956451e0b1eb2bffc662ecb7aac7e63d77e524ddd467eb3617"
    },
    {
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ],
      "pubkey": "e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78"
    }
  ],
  "root_xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144"
}
```

----

### 编译合约
调用API接口`compile`编译合约。如果合约`contract`语句上有参数`contract parameters`的话，可以在调用编译合约API接口`compile`的时候加上相关参数进行实例化，这样编译后返回的结果中`program`字段会限定合约的解锁参数，否则返回的`program`仅仅是合约的执行步骤流程，在缺少合约参数的情况下，需要用户自定义相关的合约参数，否则合约锁定的资产会存在泄漏的风险。

参数：
- `String` - *contract*, 合约内容.
- `Array of Object` - *args*, 合约参数结构体（数组类型）.
  - `Boolean` - *boolean*, 布尔类型的合约参数.
  - `Integer` - *integer*, 整数类型的合约参数.
  - `String` - *string*, 字符串类型的合约参数.

以`LockWithPublicKey`为例，其请求和响应的json格式如下：
```js
// Request
{
  "contract": "contract LockWithPublicKey(publicKey: PublicKey) locks locked { clause unlockWithSig(sig: Signature) { verify checkTxSig(publicKey, sig) unlock locked }}",
  "args": [
    {
      "string": "e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78"
    }
  ]
}

// Result
{
  "name": "LockWithPublicKey",
  "source": "contract LockWithPublicKey(publicKey: PublicKey) locks locked { clause unlockWithSig(sig: Signature) { verify checkTxSig(publicKey, sig) unlock locked }}",
  "program": "20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0",
  "params": [
    {
      "name": "publicKey",
      "type": "PublicKey"
    }
  ],
  "value": "locked",
  "clause_info": [
    {
      "name": "unlockWithSig",
      "args": [
        {
          "name": "sig",
          "type": "Signature"
        }
      ],
      "value_info": [
        {
          "name": "locked"
        }
      ],
      "block_heights": [],
      "hash_calls": null
    }
  ],
  "opcodes": "0xe9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78 DEPTH 0xae7cac FALSE CHECKPREDICATE",
  "error": ""
}
```

----

### 锁定合约
lock（锁定）合约，即部署合约，其本质是调用`build-transaction`接口将资产发送到合约特定的`program`，只需将接收方`control_program`设置为指定合约即可，构造锁定合约交易的模板如下：（注意：合约交易暂时不支持接收方资产为BTM资产的交易）
```js
// Request
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0G1JIR6400A02",
      "amount": 20000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "account_id": "0G1JIR6400A02",
      "amount": 900000000,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "type": "spend_account"
    },
    {
      "amount": 900000000,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "control_program": "20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0",
      "type": "control_program"
    }
  ],
  "ttl": 0,
  "time_range": 1521625823
}

// Result
{
  "raw_transaction": "0701dfd5c8d505020161015f150ec246dc739a8c4c3f7b4083ededcb2854ca221e437a49f23ec84c7c47ea80ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014726e902e30525e01f0157f12be476c904060383b01000160015ed53c1f3388681f62ae778ac8a54c2b091bbdc91d68ec1e94b20aa2183484f8331e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280c8afa02501011600145de3c504b41019d11698d572b1a37d9a4c9118c1010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb990101160014310c2265e8e3b7057a62caf09a9f907763f369ea00013d1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280f69bf32101160014d0d18752a276c94b25f920b02a8edff251b16b7600014f1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280d293ad03012820e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c000",
  "signing_instructions": [
    {
      "position": 0,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          ],
          "signatures": null
        },
        {
          "type": "data",
          "value": "e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78"
        }
      ]
    },
    {
      "position": 1,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0700000000000000"
              ]
            }
          ],
          "signatures": null
        },
        {
          "type": "data",
          "value": "54df681ac3174a11d4456265641a204e04f64b8f860f37bf5584cf4187f54e99"
        }
      ]
    }
  ],
  "allow_additional_actions": false
}
```

构建交易成功之后，便可以对交易进行签名`sign-transaction`（签名成功的标志是返回结果`signing_instructions`中所有包含签名类型的`signatures`字段有值且个数与`quorum`的值相等），然后提交交易`submit-transaction`到交易池中，等待交易被打包上链

----

### 查找合约utxo
部署合约交易发送成功之后，接下来便需要对合约锁定的资产进行解锁，解锁合约之前需要找到合约的UTXO。

可以通过调用API接口`list-unspent-outputs`来查找，在查合约UTXO的情况下必须将`smart_contract`设置为`true`，否则会查不到，其参数如下：
- `String` - *id*, UTXO对应的outputID.
- `Boolean` - *smart_contract*, 是否展示合约的UTXO，默认不显示.

对应的输入输出结果如下：
```js
// Request
curl -X POST list-unspent-outputs -d '{"id": "413d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116", "smart_contract": true}'

// Result
{
  "account_alias": "",
  "account_id": "",
  "address": "",
  "amount": 900000000,
  "asset_alias": "GOLD",
  "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
  "change": false,
  "control_program_index": 0,
  "id": "413d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116",
  "program": "20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0",
  "source_id": "c9680e6dd5e9ae7f825fe7edab9fa35c119eb7feab0ab4e426c84a579daf4ef9",
  "source_pos": 2,
  "valid_height": 0
}
```
找到对应的合约UTXO之后，可以通过API接口`decode-program`解析合约的参数信息，用户可以根据已有的参数信息判断该合约能否解锁

----

### 解锁合约
unlock（解锁）合约，即调用合约，其本质是通过给交易添加相应的合约参数以便合约程序`program`在虚拟机中执行成功，目前合约相关的参数都可以通过`build-transaction`中的Action结构`spend_account_unspent_output`中的数组参数`arguments`进行添加，其中参数主要有两种类型：
  - `rawTxSigArgument` 签名相关的参数，主要包含主公钥`xpub`和其对应的派生路径`derivation_path`，而待验证的`publickey`是通过该主公钥和派生路径生成的子公钥生成的（这些参数可以通过API接口`list-pubkeys`获取）
    - `xpub` 主公钥
    - `derivation_path` 派生路径，为了形成子私钥和子公钥
  - `dataArgument` 其他类型的参数，其数值是[]byte类型的string格式

以合约`LockWithPublicKey`为例，其解锁合约交易的模板如下：
```js
// Request
{
  "base_transaction": null,
  "actions": [
    {
      "type": "spend_account_unspent_output",
      "output_id": "413d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116",
      "arguments": [
        {
          "type": "raw_tx_signature",
          "raw_data": {
            "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
            "derivation_path": [
              "010100000000000000",
              "0100000000000000"
            ]
          }
        }
      ]
    },
    {
      "type": "control_program",
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "amount": 900000000,
      "control_program": "0014726e902e30525e01f0157f12be476c904060383b"
    },
    {
      "type": "spend_account",
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "amount": 5000000,
      "account_id": "0G1JIR6400A02"
    }
  ],
  "ttl": 0,
  "time_range": 1521625823
}

// Result
{
  "raw_transaction": "0701dfd5c8d5050201720170c9680e6dd5e9ae7f825fe7edab9fa35c119eb7feab0ab4e426c84a579daf4ef91e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280d293ad0302012820e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c001000160015e8412e8e8c359683f1f5f3a7308b084022f1f149dab176e6e6e8daada895d0e29ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe0c6c6944f00011600141313e974d19f3d37db29a212d75b4c763e42f433010002013d1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280d293ad0301160014726e902e30525e01f0157f12be476c904060383b00013dffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffa0b095924f011600147076c737d92621e0033899a54d02fa79f362922700",
  "signing_instructions": [
    {
      "position": 0,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          ],
          "signatures": null
        }
      ]
    },
    {
      "position": 1,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0300000000000000"
              ]
            }
          ],
          "signatures": null
        },
        {
          "type": "data",
          "value": "c37d5531f393bc6a3568628c0c0e17801ea452e75d604deb01403c4b161659a3"
        }
      ]
    }
  ],
  "allow_additional_actions": false
}
```

构建交易成功之后，便可以对交易进行签名`sign-transaction`（签名成功的标志是返回结果`signing_instructions`中所有包含签名类型的`signatures`字段有值且个数与`quorum`的值相等），然后提交交易`submit-transaction`到交易池中，等待交易被打包上链

----

## 典型合约模板解析

如果合约`contract`有参数的话，在调用编译合约的API接口`compile`的时候需要加上相关参数进行实例化，编译后的返回结果中`program`字段才是需要部署的正常合约程序

### 单签验证合约
  `LockWithPublicKey`源代码如下：
  ```
  contract LockWithPublicKey(publicKey: PublicKey) locks locked {
    clause unlockWithSig(sig: Signature) {
      verify checkTxSig(publicKey, sig)
      unlock locked
    }
  }
  ```

  - 合约编译之后的字节码为：`ae7cac`
  - 合约对应的指令码为：`TXSIGHASH SWAP CHECKSIG`

  假如合约参数如下：
  - `publicKey` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78

  添加合约参数后的合约程序参考结果如下：`20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0`

  对应的解锁合约的参数如下：
  - `sig` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  其对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "913d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116",
        "arguments": [
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 100000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### 多签验证合约
  `LockWithMultiSig`源代码如下：
  ```
  contract LockWithMultiSig(publicKey1: PublicKey,
                          publicKey2: PublicKey,
                          publicKey3: PublicKey) locks value {
    clause spend(sig1: Signature, sig2: Signature) {
      verify checkTxMultiSig([publicKey1, publicKey2, publicKey3], [sig1, sig2])
      unlock value
    }
  }
  ```
  - 合约编译之后的字节码为：`537a547a526bae71557a536c7cad`
  - 合约对应的指令码为：`3 ROLL 4 ROLL 2 TOALTSTACK TXSIGHASH 2ROT 5 ROLL 3 FROMALTSTACK SWAP CHECKMULTISIG`

  假如合约参数如下：
  - `publicKey1` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
  - `publicKey2` : 1f51c25decab2168835f1292a5432707ed94b90be8f5ec0d62aca1c6daa1ec55
  - `publicKey3` : e386c85178418fc72f8182111aa818ac736f3f7f1eee75ccdd7e5a057abe8fe0

  添加合约参数后的合约程序参考结果如下：（注意添加合约参数的顺序与虚拟机入栈的顺序是相反的）`20e386c85178418fc72f8182111aa818ac736f3f7f1eee75ccdd7e5a057abe8fe0201f51c25decab2168835f1292a5432707ed94b90be8f5ec0d62aca1c6daa1ec5520e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78740e537a547a526bae71557a536c7cad00c0`

  对应的解锁合约的参数如下：（注意解锁合约参数的顺序，否则会执行VM失败）
  - `sig1` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  - `sig2` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "e0f470a91bcd99497b4804d76264a293e257ca906938a43e34a6a0e619bbf5f1606a2866eca2ceec9f93348c577b052f30bb7ef41b4b09fff28eb1004ca1f8d5",
      "derivation_path": [
        "010200000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  其对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "bf6a3999cfbd25cef24ff1b19c6f2e0988de2e8ef3d40ed7c15402f2a7327bfa",
        "arguments": [
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          },
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "e0f470a91bcd99497b4804d76264a293e257ca906938a43e34a6a0e619bbf5f1606a2866eca2ceec9f93348c577b052f30bb7ef41b4b09fff28eb1004ca1f8d5",
              "derivation_path": [
                "010200000000000000",
                "0100000000000000"
              ]
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 200000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### pubkey哈希校验和单签验证合约
  `LockWithPublicKeyHash`源代码如下：
  ```
  contract LockWithPublicKeyHash(pubKeyHash: Hash) locks value {
    clause spend(pubKey: PublicKey, sig: Signature) {
      verify sha3(pubKey) == pubKeyHash
      verify checkTxSig(pubKey, sig)
      unlock value
    }
  }
  ```
  - 合约编译之后的字节码为：`5279aa887cae7cac`
  - 合约对应的指令码为：`2 PICK SHA3 EQUALVERIFY SWAP TXSIGHASH SWAP CHECKSIG`

  假如合约参数如下：
  - `pubKeyHash` : b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89

  添加合约参数后的合约程序参考结果如下：`20b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e8974085279aa887cae7cac00c0`

  对应的解锁合约的参数如下：（注意解锁合约参数的顺序，否则会执行VM失败）
  - `pubKey` :
  ```js
  {
    "type": "data",
    "raw_data": {
      "value": "b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89"
    }
  }
  ```

  - `sig` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  其对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "f98e0aa007dec76a827e924c25678e8c04b922fb28da9f5513a92787ac53725b",
        "arguments": [
          {
            "type": "data",
            "raw_data": {
              "value": "b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89"
            }
          },
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 300000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### 字符串哈希校验合约
  `RevealPreimage`源代码如下：
  ```
  contract RevealPreimage(hash: Hash) locks value {
    clause reveal(string: String) {
      verify sha3(string) == hash
      unlock value
    }
  }
  ```
  - 合约编译之后的字节码为：`7caa87`
  - 合约对应的指令码为：`SWAP SHA3 EQUAL`

  假如合约参数如下：
  - `hash` : 22e829107201c6b975b1dc60b928117916285ceb4aa5c6d7b4b8cc48038083e0

  添加合约参数后的合约程序参考结果如下：`2022e829107201c6b975b1dc60b928117916285ceb4aa5c6d7b4b8cc48038083e074037caa8700c0`

  对应的解锁合约的参数如下：（注意解锁合约参数的顺序，否则会执行VM失败）
  - `string` :
  ```js
  {
    "type": "data",
    "raw_data": {
      "value": "737472696e67"
    }
  }
  ```

  其对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "35fe216572bea7d81effd2fed2db1bc257f977dfa492299742a0dee2a9ae1c8e",
        "arguments": [
          {
            "type": "data",
            "raw_data": {
              "value": "737472696e67"
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 400000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### 币币交易合约
  `TradeOffer`源代码如下：
  ```
  contract TradeOffer(assetRequested: Asset,
                    amountRequested: Amount,
                    seller: Program,
                    cancelKey: PublicKey) locks offered {
    clause trade() requires payment: amountRequested of assetRequested {
      lock payment with seller
      unlock offered
    }
    clause cancel(sellerSig: Signature) {
      verify checkTxSig(cancelKey, sellerSig)
      unlock offered
    }
  }
  ```
  - 合约编译之后的字节码为：`547a6413000000007b7b51547ac1631a000000547a547aae7cac`
  - 合约对应的指令码为：`4 ROLL JUMPIF:$cancel $trade FALSE ROT ROT 1 4 ROLL CHECKOUTPUT JUMP:$_end $cancel 4 ROLL 4 ROLL TXSIGHASH SWAP CHECKSIG $_end`

  假如合约参数如下：
  - `assetRequested` : 1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22
  - `amountRequested` : 99
  - `seller` : 0014c5a5b563c4623018557fb299259542b8739f6bc2
  - `cancelKey` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78

  添加合约参数后的合约程序参考结果如下：`20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78160014c5a5b563c4623018557fb299259542b8739f6bc20163201e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22741a547a6413000000007b7b51547ac1631a000000547a547aae7cac00c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause`的选择是通过合约代码位置偏移来决定的，通过调用API接口`decode-program`可以查看对应的偏移量; 此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause trade` 解锁参数 :
    - `clause_selector` : 
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00000000"
      }
    }
    ```

    - `payment` : (合约中`requires`的字段表示解锁账户需要支付对应的`amountRequested`数量的`assetRequested`资产到对应的接收对象`seller`，注意上述参数必须严格匹配，否则合约执行将失败)
    ```js
    {
      "amount": 99,
      "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
      "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
      "type": "control_program"
    },
    {
      "account_id": "0G1JJF0KG0A06",
      "amount": 99,
      "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
      "type": "spend_account"
    }
    ```

    其对应的解锁合约交易模板示例如下：
    ```js
    // clause trade
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "9f3a63d2f8352a6891dadd8a8337268873c84a852594f35b0b9815a4b9d56d86",
          "arguments": [
            {
              "type": "data",
              "raw_data": {
                "value": "00000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 99,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 99,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "type": "spend_account"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        },
        {
          "amount": 500000000,
          "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause cancel` 解锁参数 :（注意参数顺序）
    - `sellerSig` :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` :
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "13000000"
      }
    }
    ```

    其对应的解锁合约交易模板示例如下:
    ```js
    // clause cancel
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "9f3a63d2f8352a6891dadd8a8337268873c84a852594f35b0b9815a4b9d56d86",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "data",
              "raw_data": {
                "value": "13000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 500000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  ----

  ### 第三方信任机构托管合约
  `Escrow`源代码如下：
  ```
  contract Escrow(agent: PublicKey,
                sender: Program,
                recipient: Program) locks value {
    clause approve(sig: Signature) {
      verify checkTxSig(agent, sig)
      lock value with recipient
    }
    clause reject(sig: Signature) {
      verify checkTxSig(agent, sig)
      lock value with sender
    }
  }
  ```
  - 合约编译之后的字节码为：`537a641a000000537a7cae7cac6900c3c251557ac16328000000537a7cae7cac6900c3c251547ac1`
  - 合约对应的指令码为：`3 ROLL JUMPIF:$reject $approve 3 ROLL SWAP TXSIGHASH SWAP CHECKSIG VERIFY FALSE AMOUNT ASSET 1 5 ROLL CHECKOUTPUT JUMP:$_end $reject 3 ROLL SWAP TXSIGHASH SWAP CHECKSIG VERIFY FALSE AMOUNT ASSET 1 4 ROLL CHECKOUTPUT $_end`

假如合约参数如下：
  - `agent` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
  - `sender` : 0014905df16bc248790676744bab063a1ae810803bd7
  - `recipient` : 0014929ec7d92f89d74716ba9591eaea588aa1867f75

  添加合约参数后的合约程序参考结果如下：`160014929ec7d92f89d74716ba9591eaea588aa1867f75160014905df16bc248790676744bab063a1ae810803bd720e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787428537a641a000000537a7cae7cac6900c3c251557ac16328000000537a7cae7cac6900c3c251547ac100c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause`的选择是通过合约代码位置偏移来决定的，通过调用API接口`decode-program`可以查看对应的偏移量; 此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause approve` 解锁参数 :
    - `sellerSig` :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` :
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00000000"
      }
    }
    ```

    其对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`recipient`，否则执行失败）
    ```js
    // clause approve
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "15ee3368e77a0699eadaf99bbc096a48595ac975d05b10b198d902dde808e120",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "data",
              "raw_data": {
                "value": "00000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 600000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```  

  - `clause reject` 解锁参数 :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` :
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "1a000000"
      }
    }
    ```  

    其对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`sender`，否则执行失败）
    ```js
    // clause reject
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "15ee3368e77a0699eadaf99bbc096a48595ac975d05b10b198d902dde808e120",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "data",
              "raw_data": {
                "value": "1a000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 600000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```  

  ----

  ### 抵押贷款合约
  `LoanCollateral`源代码如下：
  ```
  contract LoanCollateral(assetLoaned: Asset,
                        amountLoaned: Amount,
                        blockHeight: Integer,
                        lender: Program,
                        borrower: Program) locks collateral {
    clause repay() requires payment: amountLoaned of assetLoaned {
      lock payment with lender
      lock collateral with borrower
    }
    clause default() {
      verify above(blockHeight)
      lock collateral with lender
    }
  }
  ```
  - 合约编译之后的字节码为：`557a641b000000007b7b51557ac16951c3c251557ac163260000007bcd9f6900c3c251567ac1`
  - 合约对应的指令码为：`5 ROLL JUMPIF:$default $repay FALSE ROT ROT 1 5 ROLL CHECKOUTPUT VERIFY 1 AMOUNT ASSET 1 5 ROLL CHECKOUTPUT JUMP:$_end $default ROT BLOCKHEIGHT LESSTHAN VERIFY FALSE AMOUNT ASSET 1 6 ROLL CHECKOUTPUT $_end`

假如合约参数如下：
  - `assetLoaned` : 1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22
  - `amountLoaned` : 88
  - `blockHeight` : 1920
  - `lender` : 0014905df16bc248790676744bab063a1ae810803bd7
  - `borrower` : 0014929ec7d92f89d74716ba9591eaea588aa1867f75

  添加合约参数后的合约程序参考结果如下：`160014929ec7d92f89d74716ba9591eaea588aa1867f75160014905df16bc248790676744bab063a1ae810803bd70280070158201e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c227426557a641b000000007b7b51557ac16951c3c251557ac163260000007bcd9f6900c3c251567ac100c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause`的选择是通过合约代码位置偏移来决定的，通过调用API接口`decode-program`可以查看对应的偏移量; 此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause repay` 解锁参数 :
    - `clause_selector` : 
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00000000"
      }
    }
    ```

    - `payment` : (合约中`requires`的字段表示解锁账户需要支付对应的`amountLoaned`数量的`assetLoaned`资产到对应的接收对象`lender`，注意上述参数必须严格匹配，否则合约执行将失败; 在该`clause`中解锁对象`collateral`是被接收对象`borrower`锁定的，所以该参数也需要匹配; 此外该合约使用了两个`lock`语句，其解锁模板需要按下面的方式进行顺序排列)
    ```js
    {
      "amount": 88,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
      "type": "control_program"
    },
    {
      "amount": 700000000,
      "asset_id": "2ee76aeaa110308fcdfb382fa02a4a35823d5a589ffcaddb23f11f8a1fae3302",
      "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
      "type": "control_program"
    },
    {
      "account_id": "0G1JIR6400A02",
      "amount": 88,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "type": "spend_account"
    }
    ```

    其对应的解锁合约交易模板示例如下：
    ```js
    // clause repay
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "3cf74c303558b7343cfa20b32ccddfd8e66293ae1970f7612ca6cb9a006e76bc",
          "arguments": [
            {
              "type": "data",
              "raw_data": {
                "value": "00000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 88,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "amount": 700000000,
          "asset_id": "2ee76aeaa110308fcdfb382fa02a4a35823d5a589ffcaddb23f11f8a1fae3302",
          "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 88,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "type": "spend_account"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause default` 解锁参数 :（注意参数顺序）
    - `clause_selector` :
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "1b000000"
      }
    }
    ```

    其对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`lender`，否则执行失败）
    ```js
    // clause default
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "3cf74c303558b7343cfa20b32ccddfd8e66293ae1970f7612ca6cb9a006e76bc",
          "arguments": [
            {
              "type": "data",
              "raw_data": {
                "value": "1b000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 700000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  ----

  ### 看涨期权合约
  `CallOption`源代码如下：
  ```
  contract CallOption(strikePrice: Amount,
                    strikeCurrency: Asset,
                    seller: Program,
                    buyerKey: PublicKey,
                    blockHeight: Integer) locks underlying {
    clause exercise(buyerSig: Signature) requires payment: strikePrice of strikeCurrency {
      verify below(blockHeight)
      verify checkTxSig(buyerKey, buyerSig)
      lock payment with seller
      unlock underlying
    }
    clause expire() {
      verify above(blockHeight)
      lock underlying with seller
    }
  }
  ```
  - 合约编译之后的字节码为：`557a6420000000547acda069547a547aae7cac69007c7b51547ac1632c000000547acd9f6900c3c251567ac1`
  - 合约对应的指令码为：`5 ROLL JUMPIF:$expire $exercise FALSE ROT ROT 1 5 ROLL CHECKOUTPUT VERIFY 1 AMOUNT ASSET 1 5 ROLL CHECKOUTPUT JUMP:$_end $expire ROT BLOCKHEIGHT LESSTHAN VERIFY FALSE AMOUNT ASSET 1 6 ROLL CHECKOUTPUT $_end`

假如合约参数如下：
  - `strikePrice` : 199
  - `strikeCurrency` : 1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22
  - `seller` : 0014905df16bc248790676744bab063a1ae810803bd7
  - `buyerKey` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
  - `blockHeight` : 3096

  添加合约参数后的合约程序参考结果如下：`02180c20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78160014905df16bc248790676744bab063a1ae810803bd7201e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2201c7742c557a6420000000547acda069547a547aae7cac69007c7b51547ac1632c000000547acd9f6900c3c251567ac100c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause`的选择是通过合约代码位置偏移来决定的，通过调用API接口`decode-program`可以查看对应的偏移量; 此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause exercise` 解锁参数 :
    - `buyerSig` : 
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` : 
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00000000"
      }
    }
    ```

    - `payment` : (合约中`requires`的字段表示解锁账户需要支付对应的`strikePrice`数量的`strikeCurrency`资产到对应的接收对象`seller`，注意上述参数必须严格匹配，否则合约执行将失败)
    ```js
    {
      "amount": 199,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
      "type": "control_program"
    },
    {
      "account_id": "0G1JIR6400A02",
      "amount": 199,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "type": "spend_account"
    }
    ```

    其对应的解锁合约交易模板示例如下：
    ```js
    // clause exercise
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "7afa15ad39356a7e6dc363fba823146ecb0132967f067ddfa13494a34a984518",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "data",
              "raw_data": {
                "value": "00000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 199,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 199,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "type": "spend_account"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        },
        {
          "amount": 800000000,
          "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause expire` 解锁参数 :
    - `clause_selector` : 
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "20000000"
      }
    }
    ```

    其对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`seller`，否则执行失败）
    ```js
    // clause expire
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "7afa15ad39356a7e6dc363fba823146ecb0132967f067ddfa13494a34a984518",
          "arguments": [
            {
              "type": "data",
              "raw_data": {
                "value": "20000000"
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 800000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```    

## 区块链系统模型简介

### UTXO模型
  在比特币系统中，每一笔交易花费的输出（outputs）都是之前的交易产生的，而这笔交易的输出（outputs）将会在未来的某一笔交易中花费出去，这种模型便称为UTXO模型。用户的钱包跟踪与用户拥有的所有地址相关联的未花费的交易的列表，并且钱包的余额被计算为那些未花费的交易的总和。UTXO模型跟现实生活中的现金模型相似，每个现金都可以表示为一个UTXO。比特币系统中的流向图如下：
  
  ![image](image/bitcoin.png)

### 账户模型/余额模型
  在以太坊系统中，账户保存了与它相关交易的余额的世界状态，在每一币笔交易发出之前都需要检查帐户余额以确保其大于或等于支出交易金额。账户模型跟现实生活中银行卡/支付宝/微信等账户的模型相似，所以用户理解起来也相对比较容易。

### 模型对比
  UTXO模型：
  - 可扩展性 如果一个用户有多个UTXO，那么所有的UTXO可以同时被花费掉，而且不会相互影响，所以它具有很好的并行性和扩展性
  - 隐私性 尽管比特币系统不是一个完全的匿名系统，但是UTXO本身具有很高的匿名性，因为用户在接收UTXO的时候都可以使用不同的地址，这样用户实际的钱包花费情况将不能被跟踪

  账户模型/余额模型：
  - 简便 基于账户模型的以太坊最直观的应用是智能合约，多方参与和保存状态信息的特性，使智能合约的应用更具优势。而UTXO模型的只会保存少量的交易状态信息，在这种情况下去设计复杂的合约会显得比较困难。
  - 效率高 账户模型效率高是因为每笔交易只需要验证发送账户有足够的余额来执行交易即可
  - 双花攻击多 基于账户模型的以太坊是通过自增的nonce来防止双花攻击的，每次构造交易的时候都需要修改nonce（每发一笔交易nonce都会加1），这样便可以防止交易被多次提交。但是nonce自增特性也严重影响了以太坊交易的并行程度

  总结：
    不同的区块链系统架构之间各有优点和缺陷，需要在多个方面进行权衡。
