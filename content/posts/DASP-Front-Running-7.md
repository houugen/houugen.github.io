---
title: "#DASP# Front Running (7)"
date: 2018-08-16 14:58:48
tags: ["BlockChain", "DASP"]
categories: ["区块链"]
---

## 0x00 Info

本篇介绍DASP第七种漏洞类型--[前置执行](https://www.dasp.co/#item-7)。
可以理解为因为区块链是公开，其他用户或者合约的交易信息均透明可查，恶意用户可以利用这些已知信息制造有利于自己的交易，并通过提高手续费的方式抢先执行获利。

当然，在[front-running-griefing-and-the-perils-of-virtual-settlement](https://blog.0xproject.com/front-running-griefing-and-the-perils-of-virtual-settlement-part-1-8554ab283e97)中描述，如果这个恶意用户本身又是矿工，可以任意安排交易并审查其他人交易，使自己利益最大化。

<!-- more -->

## 0x01 实例

### LastIsMe

[Contract Source Code](https://etherscan.io/address/0x5d9b8fa00c16bcafae47deed872e919c8f6535bf#code)

LastIsMe是一个lottery游戏合约，在一个游戏回合（一定的区块数内）参与者购买一张票认领最后一个座位，回合结束时最后一个就坐的玩家会获得头奖。

攻击者可以在回合快结束时观察其他玩家的交易池，通过提高gas来挤掉其他玩家的交易从而获得奖金。

### ICO

[Contract Source Code](https://rinkeby.etherscan.io/address/0xd80cc3550Da18313aF09fbd35571084913Cd5246#code)

ICO本身是一道CTF题，它包含一个DAPP网站和与其交互的两个合约（HaCoin & ICO），部署在rinkeby测试链上。

HaCoin是一个Token合约，比赛的目标也是从该合约中获取超过31337枚HackCoin，而这个合约本身并没有问题，需要配合Web网站存在的XSS漏洞来提权，才能进行转币操作。很有意思，具体writeup参见[ZeroNights ICO Hacking Contest Writeup](https://blog.positive.com/zeronights-ico-hacking-contest-writeup-63afb996f1e3)。

而另一个合约ICO是一个lottery合约，游戏规则也很简单，5个块为一回合，回合内参与者猜一个数字，回合结束时机器人会随机抛出一个数字，如果谁猜中谁就获胜。而该合约存在前置执行漏洞。

```solidity
function spinLottery(uint number) public {
    if (msg.sender != robotAddress) {
        playerNumber[msg.sender] = number;
        players.push(msg.sender);
        NewLotteryBet(msg.sender);
    } else {
        require(block.number - lotteryBlock > 5);
        lotteryBlock = block.number;

        for (uint i = 0; i < players.length; i++) {
            if (playerNumber[players[i]] == number) {
                desires[players[i]].active = true;
                desires[players[i]].email = "*Use changeEmail func to set your email.*";
                Proposal(players[i], desires[players[i]].email);
            }
        }
        delete players; // flushing round
        NewLotteryRound(lotteryBlock);
    }
}
```
机器人发布的随机数明文提交到交易池，如果攻击者能够足够快的检索pending交易并找到该随机数，然后使用该随机数参与游戏，保证两笔交易在同一个块中，并提高gas在机器人发布随机数的交易之前执行，就能获胜。

### BancorFormula

[Contract Source Code](https://github.com/bogatyy/bancor/blob/master/solidity/BancorFormula.sol)

详细分析：[Implementing Ethereum trading front-runs on the Bancor exchange in Python](https://hackernoon.com/front-running-bancor-in-150-lines-of-python-with-ethereum-api-d5e2bfd0d798)


