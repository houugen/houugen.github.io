---
title: '#DASP# Unchecked Return Valuse (4)'
date: 2018-08-03 10:38:03
tags: [BlockChain, DASP]
categories: [区块链]
---

## 0x00 Info

[Unchecked Return Values For Low Level Calls](https://www.dasp.co/#item-4)漏洞简单理解就是没有检查一些不安全调用函数的返回值导致。

<!-- more -->

## 0x01 CALL -> Contract

在[reentrancy](https://houugen.fun/2018/07/20/DASP-Reentrancy-2/#more)漏洞介绍中提到几种账户间转币函数，这里我们回顾并深入了解下：

| 函数 | 特性 |
| :-- | :-- |
| **address.transfer()** | 1. 失败抛出异常且回滚 <br> 2. 提供2300gas，防止reentrancy|
| **address.send()** | 1. 失败返回false <br> 2. 提供2300gass，防止reentrancy |
| **address.call.value().gas()()** | 1. 失败返回false <br> 2. 发送所有可用gas |

在这些交易函数中需要注意的是：
* `addr.transfer()`和`addr.send()`能够防止重入漏洞。但是这些方法会触发fallback函数执行，被调合约仅被提供2300gas做一些日志事件。
* `x.transfer(y)`等同于`require(x.send(y))`，transfer在发送失败时会自动revert（内置失败处理）。
* `addr.call.value(y)()`也会触发代码执行，但是会用所有提供的gas执行代码，当然这种方式不能防止reentrancy漏洞。

通过了解以上我们会提出一些疑问：
1. 2300gas由谁来提供？为何是2300gas? 2300gas能做什么？
2. 哪些行为会导致发送失败？

要回答这些问题，首先要知道一个知识点，transfer、send、call在EVM虚拟机执行时会将这些solidity编译成 **CALL** 指令，而在[以太坊wiki](https://github.com/ethereum/wiki/wiki/Subtleties)中定义了CALL的gas消耗：

{{< admonition quote >}}
* CALL has a multi-part gas cost:
   + 700 base
   + 9000 additional if the value is nonzero
   + 25000 additional if the destination account does not yet exist (note: there is a difference between zero-balance and nonexistent!)
* CALLCODE operates similarly to call, except without the potential for a 25000 gas surcharge.
* The child message of a nonzero-value CALL operation (NOT the top-level message arising from a transaction!) gains an additional 2300 gas on top of the gas supplied by the calling account; this stipend can be considered to be paid out of the 9000 mandatory additional fee for nonzero-value calls. This ensures that a call recipient will always have enough gas to log that it received funds.
{{< /admonition >}}

从wiki中可以知道交易操作至少需要9700gas（700base+9000additional），而2300gas就在其中，由接收合约提供，确保有足够gas记录其接收的资金。2300gas属于硬编码津贴，规定就是这么多。这里即回答了第一个问题。

对于第二个问题哪些行为会导致发送失败，其实在wiki中也有说明。

{{< admonition quote >}}
   * Execution running out of gas
   * An operation trying to take more slots off the stack than are available on the stack, or put more than 1024 items onto the stack
   * Jumping to a bad jump destination
   * An invalid opcode (note: the code of an account is assumed to be followed by an infinite tail of STOP instructions, so the program counter "walking off" the end of the code is not an invalid opcode exception. However, jumping outside the code is an exception, because STOP is not a valid jump destination)
   * The REVERT opcode at 0xfd (starting from Metropolis; pre-Metropolis 0xfd is simply an invalid opcode)
{{< /admonition >}}

其中gas溢出和超过调用栈限制这两点导致的发送失败很容易被忽略，但这也恰恰是漏洞发生的地方。

## 0x02 Unchecked Low Level Calls

正如DASP介绍，solidity特性中存在一些不安全的函数`call()`，`callcode()`(已经遗弃)，`delegatecall()`和`send()`。这个函数在运行错误时行为并不可逆，仅会返回false，代码流程也会继续，这就会带来很多不可预期的结果。

实例代码:
```solidity
function withdraw(uint256 _amount) public {
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] -= _amount;
    etherLeft -= _amount;
    msg.sender.send(_amount);
}
```
`msg.sender`为一个智能合约，其中未定义fallback函数，或者callstack已满，均会导致send失败，而函数未检出send返回值，最终导致`msg.sender`和`etherLeft`数量减少且状态不可逆转，最后却没有取回减少的钱。

## 0x03 KotET contract

KotET是一个抢皇位的游戏合约，合约代码见[github](https://github.com/kieranelby/KingOfTheEtherThrone/blob/v0.4.0/contracts/KingOfTheEtherThrone.sol)

游戏规则和漏洞分析可以看[Post-Mortem Investigation](https://www.kingoftheether.com/postmortem.html)这篇文章。

```solidity
 // Claim the throne for the given name by paying the currentClaimFee.
function claimThrone(string name) {

    ...
    
    uint compensation = valuePaid - wizardCommission;

    if (currentMonarch.etherAddress != wizardAddress) {
        currentMonarch.etherAddress.send(compensation);
    } else {
        // When the throne is vacant, the fee accumulates for the wizard.
    }

    ...
}
```

其主要就是因为没有判断send函数返回值，导致在接受者是一个合约时，有可能因为gas不足而导致send失败（例如Mist钱包合约对于KotET提供的gas不足以处理支付行为），从而old king没有收到补偿，new king没有消耗资金变争夺了王位。

## 0x04 lotto contract

[issues](https://github.com/etherpot/contract/issues/1)

[contract source code](https://github.com/etherpot/contract/blob/f87501ddf319559346b2983b27770ace22771ad0/app/contracts/lotto.sol)

[exp](https://gist.github.com/amiller/665cc46970f2c0684d2a)

```solidity
function cash(uint roundIndex, uint subpotIndex){

    ...

    var winner = calculateWinner(roundIndex,subpotIndex);    
    var subpot = getSubpot(roundIndex);

    winner.send(subpot);

    rounds[roundIndex].isCashed[subpotIndex] = true;
    //Mark the round as cashed
}
```

这里合约的漏洞比较简单，还是send没有判断返回值，导致在callstack超过1024时send失败，程序继续执行。
但是可以学习如何用[pyethereum](https://github.com/ethereum/pyethereum)写exp。

{{< admonition warning >}}
amiller所用的pyethereum与现在的版本已有很大区别，需要重新优化exp代码。
{{< /admonition  >}}

## 0x05 Others

其他合约也存在类似问题，例如BTC合约：
[Report: Security Audit of BTC Relay implementation](http://soc1024.ece.illinois.edu/BTCRelayAudit.pdf) [[diff](https://github.com/ethereum/btcrelay/commit/26ee2bcc4468329939a3f093023496986c357074)]

[Scanning Live Ethereum Contracts for the "Unchecked-Send" Bug](http://hackingdistributed.com/2016/06/16/scanning-live-ethereum-contracts-for-bugs/)一文中也详细描述了此类问题的并提供了修复和扫描工具原理--[Appendix A: Details on how we analyze the blockchain](https://docs.google.com/document/d/1En0DjqmSpqVVxsdGAcJs4Rkw5IjxnUeTJMUVMhWum28/edit#heading=h.ezyax39wd5z6)。

利用sand函数在虚拟机字节码中连续POP数量来判断是否做返回值处理。





