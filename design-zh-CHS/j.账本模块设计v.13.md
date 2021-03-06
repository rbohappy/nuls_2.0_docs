# 账本模块设计文档

[TOC]

## 一、总体描述

### 1.1 模块概述

#### 1.1.1 为什么要有《账本模块》

> 账本模块是区块链的数据中枢，所有账户的余额、交易都保存在账本模块中,
  每一个全网节点上都会保存一个全网账本，保证了数据的完整、公开、透明,同时保证了数据不可篡改、可追溯

#### 1.1.2 《账本模块》要做什么

> 为组装交易提供数据支撑,主要就是记账和查账,验证交易的合法性,如:是否有充足的余额，是否重复支付(双花)

#### 1.1.3 《账本模块》在系统中的定位

> 账本模块是数据中枢,保存系统所有存在交易的结果数据,它不依赖任何业务模块,其他模块按需依赖它。
#### 1.1.4 《账本模块》中名词解释
- 交易的随机数（nonce）
  - nonce：与此地址发送的交易数量相等的标量值，或者，对于具有关联代码的帐户，表示此帐户创建的合约数量。
  - 严格地说，nonce是始发地址的一个属性（它只在发送地址的上下文中有意义）。但是，该nonce并未作为账户状态的一部分显式存储在区块链中。相反，它是根据来源于此地址的已确认交易的数量动态计算的。
  - nonce值也用于防止帐户余额的错误计算。例如，假设一个账户有10个NULS的余额，并且签署了两个交易，都花费6个NULS，分别具有nonce 1和nonce 2。这两笔交易中哪一笔有效？在区块链分布式系统中，节点可能无序地接收交易。nonce强制任何地址的交易按顺序处理，不管间隔时间如何，无论节点接收到的顺序如何。这样，所有节点都会计算相同的余额。支付6以太币的交易将被成功处理，账户余额减少到4 ether。无论什么时候收到，所有节点都认为与带有nonce 2的交易无效。如果一个节点先收到nonce 2的交易，会持有它，但在收到并处理完nonce 1的交易之前不会验证它。
  - 使用nonce确保所有节点计算相同的余额，并正确地对交易进行排序，相当于比特币中用于防止“双重支付”的机制。但是，因为以太坊跟踪账户余额并且不会单独跟踪独立的币（在比特币中称为UTXO），所以只有在账户余额计算错误时才会发生“双重支付”。nonce机制可以防止这种情况发生。
  
### 1.2 架构图
> 账本的核心还是资产管理和记账管理。

![ledger-arch.jpg](image/ledger/ledger-arch.jpg)

## 二、功能设计

### 2.1 功能架构图
![ledger-functions.png](image/ledger/ledger-functions.png)

### 2.2 模块服务
#### 2.2.1 账本模块的系统服务
![ledger-service.png](image/ledger/ledger-service.png)

> 账本模块提供的RPC的接口调用,详细接口请参照接口设计部分。

#### 2.2.2 修改系统运行参数

> 只依赖核心系统，核心系统可以对事件模块系统的启动，停止，参数修改等，

### 2.3 模块内部功能

> 模块内部工能主要包含,资产管理,获取账户地址余额和nonce,交易记录保存，验证交易。

- 资产管理
  - 账户的总资产
  - 可用资产
  - 冻结资产,对于有锁定的资产,需要单独记录以及锁定的资产信息，包含链ID,资产ID,资产金额，锁定时间，锁定高度等
  - 资产解锁流程,当用户的锁定资产时间或者高度达到解锁条件，账本会把该资产信息解锁，累计到可用余额，并删除本地数据的资产锁定记录。
  - 多资产情况，需要加入chainId.
- 获取账户地址余额和nonce
  - 获取账户地址余额
  - 获取账户地址nonce(该nonce是一个基于零的计数器，意味着第一个交易的nonce是0.)
- 交易记录保存
  - 钱包创建的待确认交易
  - 已确认的交易是否存在账本中？
  - 在交易中，还需要考虑一个from多个to合并成一笔交易。
- 验证交易
  - 双花验证(nonce机制阻止双重支付)
  - 交易创建者验证,验证交易发出者是否拥有足够的余额,验证交易创建者的nonce是否合法
  - 连续交易验证
- 功能接口管理(rpc)
  - 提供给其他模块使用的rpc接口
### 2.4 账本流程
### 2.4.1 转账交易流程

  - 用户输入转账的地址和转入的地址和转出的金额
  - 系统通过转出的地址的私钥对转账信息进行签名（用于证明这笔交易确实有本人发起）
  - 系统对交易信息进行验证
    - 余额验证
    - 手续费验证
    - nonce连续性验证
    - 签名与input账户验证
  - 把这笔交易入到本地的TxPool中（就是账本未确认交易池）
  - 把交易信息广播给其它节点
  - 打包区块,验证区块
  - 确认交易
    - 更新相关(转入或者转出)的所有账户的余额
    - 更新账户资产对应的nonce
    - 更新资金锁定记录？这里应该不更新

### 2.4.2 普通交易流程(参考实例)

![eth-transaction-flow.png](image/ledger/eth-transaction-flow.png)

### 2.4.3 交易验证流程

![trx-validate-flow.png](image/ledger/trx-validate-flow.png)

## 三、接口设计

### 3.1 模块接口
#### 3.1.1 根据资产id获取资产信息
> cmd: getAsset

##### 参数说明 (request)

| 字段      |      是否可选  | 数据类型 |  描述信息 |
|----------|:-------------:|--------:|--------:|
| chainId |  Y | String |链ID |
| assetId |  Y | String |资产ID |

```json
{
  "cmd": "getAsset",
  "minVersion": "1.0",
  "params":["chainId","assetId"]
}
```
##### 返回值说明 (response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result":{
    "chainId": "mainChainId",
    "asset_id": "xxxxxx",
    "balance" : {
      "available": 10000000000,
      "freeze": 200000000,
      "total": 12000000000
    }
  }
}
```

| 字段   |      数据类型      |  描述信息 |
|----------|:-------------:|------:|
| chainId |  String | 链ID |
| asset_id |  String | 资产ID |
| balance.available |  BigInteger | 可用余额 |
| balance.freeze |  BigInteger | 冻结余额 |
| balance.total |  BigInteger | 总资产余额 total = available+freeze  |


#### 3.1.2 查询资产列表
> cmd: getAssets

##### 参数说明 (request)

```json
{
  "cmd": "getAssets",
  "minVersion": "1.0",
  "params":[]
}
```
##### 返回值说明 (response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result": [
      {  
        "chainId": "mainChainId",
        "asset_id": "xxxxxx",
        "balance" : {
          "available": 10000000000,
          "freeze": 200000000,
          "total": 12000000000
        }
      },
      {  
        "chainId": "friendChainId",
        "asset_id": "xxxxxxyyyyyy",
        "balance" : {
          "available": 10000000000,
          "freeze": 200000000,
          "total": 12000000000
        }
    }
  ]
}
```

| 字段   |      数据类型      |  描述信息 |
|----------|:-------------:|------:|
| chainId |  String | 链ID |
| asset_id |  String | 资产ID |
| balance.available |  BigInteger | 可用余额 |
| balance.freeze |  BigInteger | 冻结余额 |
| balance.total |  BigInteger | 总资产余额 total = available+freeze  |


#### 3.1.3 查询用户余额
> cmd: getBalance

##### 参数说明 (request)

| 字段      |      是否可选  | 数据类型 |  描述信息 |
|----------|:-------------:|--------:|--------:|
| chainId |  Y | String |链ID |
| assetId |  Y | String |资产ID |
| address |  Y | String |要查询余额的地址 |

```json
{
  "cmd": "getBalance",
  "minVersion": "1.0",
  "params":[
    "chainId",
    "assetId",
    "0x407d73d8a49eeb85d32cf465507dd71d507100c1"
  ]
}
```
##### 返回值说明 (response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result": {
  "balance" : {
       "available": 10000000000,
       "freeze": 200000000,
       "total": 12000000000
     }
  }
}
```

> 说明: 1NULS=10^8Na

| 字段   |      数据类型      |  描述信息 |
|----------|:-------------:|------:|
| balance.available |  BigInteger | 可用余额 |
| balance.freeze |  BigInteger | 冻结余额 |
| balance.total |  BigInteger | 总资产余额 total = available+freeze  |

#### 3.1.4 获取账户coinData
> cmd: getCoinData

##### 参数说明 (request)

| 字段      |      是否可选  | 数据类型 |  描述信息 |
|----------|:-------------:|--------:|--------:|
| chainId |  Y | String |链ID |
| assetId |  Y | String |资产ID |
| address |  Y | String |要查询余额的地址 |

```json
{
"cmd": "getCoinData",
"minVersion": "1.0",
"params":[
  "chainId",
  "assetId",
  "0x407d73d8a49eeb85d32cf465507dd71d507100c1"
  ]
}
```

##### 返回值说明：(response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result": {
    "available": 10000000000,
    "nonce": 40
  }
}
```

| 字段   |      数据类型      |  描述信息 |
|----------|:-------------:|------:|
| available |  BigInteger | 用户可用余额 |
| nonce |  BigInteger | 有一个交易的计数为40，这意味着从0到39nonce的交易已经被确认。下一个交易的nonce将是40。 |

#### 3.1.5 查询用户交易记录

> cmd: getAccountTxs

##### 参数说明 (request)

| 字段      |      是否可选  | 数据类型 |  描述信息 |
|----------|:-------------:|--------:|--------:|
| chainId |  Y | String |链ID |
| assetId |  Y | String |资产ID |
| address |  Y | String |要查询余额的地址 |

```json
{
"cmd": "getAccountTxs",
"minVersion": "1.0",
"params":[
  "chainId",
  "assetId",
  "0x407d73d8a49eeb85d32cf465507dd71d507100c1"
  ]
}
```

##### 返回值说明：(response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result": {
      "pageNumber": 1,
      "pageSize": 10,
      "total": 31,
      "pages": 4,
      "list": [
        {
          "txHash": "0020938ee82b1b1328f4ecddfbab5ecb69cc2f2bc21b03d95b0e384b635bc2419b84",
          "blockHeight": 31244,
          "time": 1540800070000,
          "txType": 1,
          "status": 1,
          "info": "+0.02259725",
          "contractAddress": null,
          "symbol": null
        },
        {
          "txHash": "0020e812f4dc15f03ba95d7d6d7d995ac64e518552dc0d1dccc97c3a85eb691d9f20",
          "blockHeight": 31244,
          "time": 1540800060043,
          "txType": 101,
          "status": 1,
          "info": "-0.026",
          "contractAddress": null,
          "symbol": null
        }
      ]
  }
}
```

| 字段   |      数据类型      |  描述信息 |
|----------|:-------------:|------:|
| txHash |  String | 交易hash |
| blockHeight |  BigInteger | 区块高度 |
| time |  Long | 交易时间 |
| txType |  int | 交易类型 |
| status |  int | 交易类型 0：unconfirm，1:confirmed |
| contractAddress |  String | 合约地址|

#### 3.1.6 保存未确认交易
> cmd: saveTx

##### 参数说明 (request)

| 字段      |      是否可选  | 数据类型 |  描述信息 |
|----------|:-------------:|--------:|--------:|
| chainId |  Y | String | 链ID |
| txHash |  Y | String | 交易hash |
| txHexData |  Y | String | 交易数据HEX格式 |

```json
{
"cmd": "saveTx",
"minVersion": "1.0",
"params":[
  "chainId",
  "txHash",
  "txHexData"
  ]
}
```

##### 返回值说明：(response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result": {
      
  }
}
```

#### 3.1.7 删除未确认交易
> cmd: deleteTx

##### 参数说明 (request)

| 字段      |      是否可选  | 数据类型 |  描述信息 |
|----------|:-------------:|--------:|--------:|
| chainId |  Y | String | 链ID |
| txHash |  Y | String | 交易hash |

```json
{
"cmd": "deleteTx",
"minVersion": "1.0",
"params":[
  "chainId",
  "txHash"
  ]
}
```

##### 返回值说明：(response)

```json
{
  "version": "1.0",
  "code": 0,
  "msg": "response message.",
  "result": {
      "txHash":"txHash"
  }
}
```


## 四、事件说明

> 不依赖任何事件

## 五、协议

### 5.1 网络通讯协议

无

### 5.2 交易协议

## 六、模块配置
### 6.1 配置说明

### 6.2 模块依赖关系

- 内核模块
  - 模块注册
  - 模块注销
  - 模块状态上报（心跳）
  - 服务接口数据获取及定时更新

## 七、Java特有的设计

> 核心对象类定义,存储数据结构，......

## 八、补充内容

### 参考资料文献资料
- [精通以太坊-第七章 交易](https://github.com/inoutcode/ethereum_book/blob/master/%E7%AC%AC%E4%B8%83%E7%AB%A0.asciidoc)

