---
title: '#DASP# Time Manipulation (8)'
date: 2018-08-19 16:13:53
tags: [BlockChain, DASP]
categories: [区块链]
---

## 0x00 Info

如果攻击者拥有矿工角色，在交易打包成块时，矿工在一定范围是可以操作块的时间戳的，而恰恰合约利用时间戳生成逻辑（特别是涉及到资金交易）就会存在[Time manipulation](https://www.dasp.co/#item-8)风险。

<!-- more -->

## 0x01 Rules

一个合约需要用到“当前时间”，通常是通过block.timestamp或者now来实现，而这个值来源于矿工！矿工是可以在几秒内调整这个时间戳的，从而为自己的利益改变合同的输出。

例如一个合约使用时间戳来生成随机数（在example中有实例），矿工可以在区块被验证后30s内发布时间戳，从而利用这30s的时间增加自己获利概率。

***30-second Rule***

矿工可以改变区块的时间戳
* 仅限于被验证后的30s
* 当前块的时间戳不能小于前一块的时间戳

但是一个合约具备以下特性，是可以安全使用时间戳的：
* If the smart contract function can tolerate a 30-second time period, it’s safe to use timestamp;
* If the scale of a time-dependent event can vary by 30 seconds and maintain integrity, it’s also safe to use a block timestamp.

## 0x02 Examples

前两个实例合约主要利用block.timestamp来生成随机数，而在[#DASP# Bad Randomness](https://houugen.fun/2018/08/09/DASP-Bad-Randomness-6/#more)一文中我们详细介绍了这种随机数生成的缺陷。

### theRun

[Source Code](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code)

```js
uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y; 
    uint256 h = uint256(block.blockhash(seed)); 

    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```
theRun合约使用block.timestamp作为随机数种子，而矿工完全可以在30s内计算一个利于获胜的随机数。

### EtherLotto

[Source Code](https://etherscan.io/address/0xa11e4ed59dc94e69612f3111942626ed513cb172#code)

```js
// Compute some *almost random* value for selecting winner from current transaction.
var random = uint(sha3(block.timestamp)) % 2;
```
EtherLotto合约随机数的计算就更简单粗暴了，直接使用block.timestamp做hash。

### Governmental

[Source Code](https://etherscan.io/address/0xf45717552f12ef7cb65e95476f217ea008167ae3#code)

在[#DASP# Denial of Services](https://houugen.fun/2018/08/09/DASP-Denial-of-Services-6/)开篇中我们介绍过Governmental合约，其存在重置超长creditor列表时gas溢出导致的DOS问题（未检查send返回值）。

如果攻击者同时拥有矿工身份，那该合约同样存在时间戳篡改风险。
游戏的规则玩法我们再简单介绍下，政府在最近12小时内没有收到任何投资人捐赠，则将奖金发给最后一名投资人。

恶意的矿工用修改后的时间戳为他的交易生成了一个块，从而延迟了他的交易成为最后的交易，这样他就从合约中赢得了资金。

在[A survey of attacks on Ethereum smart contracts](https://eprint.iacr.org/2016/1007.pdf)一文中介绍了Governmental合约的三个攻击方式。
