---
title: 分布式事务：TCC编程模型
date: 2017-06-29 13:12:44
categories: [分布式,分布式事务]
tags: [二段提交,TCC]
typora-root-url: ..
---
<Excerpt in index | 首页摘要>
本文介绍分布式事务的一种处理手段：TCC编程模型。<!-- more -->
<The rest of contents | 余下全文>
# 概念定义
TCC其实也算是一种二段提交的实现，具体如下：

- Try: 尝试执行业务

  * 完成所有业务检查（一致性）
  * 预留必须业务资源（准隔离性）

- Confirm: 确认执行业务

  * 真正执行业务
  * 不作任何业务检查
  * 只使用Try阶段预留的业务资源

- Confirm：操作满足幂等性

- Cancel: 取消执行业务

  * 释放Try阶段预留的业务资源

  * Cancel操作满足幂等性

    ​

# 例子



表设计：

| 列          | 解析              |
| ---------- | --------------- |
| userId     | 用户ID            |
| coin       | 游戏币             |
| frozenCoin | 冻结的游戏币（预充值、预扣减） |

情景1：游戏内用户使用游戏币购买道具。

```sql
# try: 冻结用户小明的100游戏币，并真实扣除小明100游戏币（这里或许需要检查游戏币是否足够100）
update user_coin set coin=coin-100,frozenCoin=frozenCoin+100 where userId='xiaoming';
# confirm: 解冻小明100游戏币
update user_coin set frozenCoin=frozenCoin-100 where userId='xiaoming';
# cancel: 假设confirm操作失败（比如游戏币不足100），则将冻结的游戏币解冻
update user_coin set coin=coin+100,frozenCoin=frozenCoin-100 where userId='xiaoming';
```

情景2：游戏内用户充值游戏币。

```sql
# try: 预充值给小明100游戏币
update user_coin set frozenCoin=frozenCoin+100 where userId='xiaoming';
# confirm: 解冻小明100游戏币，并给小明真实充值100游戏币
update user_coin set coin=coin+100,frozenCoin=frozenCoin-100 where userId='xiaoming';
# cancel: 假设confirm操作失败，则将冻结的游戏币解冻
update user_coin set frozenCoin=frozenCoin-100 where userId='xiaoming';
```

通过以上行为，显然，如果cancel失败呢？数据是否不一致？（confirm也可以采取以下方式做出处理）

- 异常重试（不容忍失败：强一致性）
- （事务）队列+定时补偿（可容忍失败：最终一致性）

# 引用


