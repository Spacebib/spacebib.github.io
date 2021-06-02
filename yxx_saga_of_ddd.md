## 用 Saga 异步的方式保证业务一致性
### 需求
我们有一个需求，用户下单购买一件商品。我们需要生成订单，扣减库存，扣除用户余额的操作
### 实现需求
* #### 第一阶段
我们根据上面需求，可以拆分出来三个 `AggregateRoot`,分别是 `OrderRoot` ，`StockRoot` ,`WalletRoot`
逻辑如下图所示：

![Laravel](https://cdn.learnku.com/uploads/images/202106/02/43464/uOePCNnos9.png!large)

如上图所示就是一个简单的下订单扣库存，扣余额的过程在 DDD 实现，但是有一个`问题`：
假如我们走到了上图中的第五步，在第三步的时候 `deductStock` 已经分发了 `InventoryDeducted`事件,并在 `Projector` 监听了 `InventoryDeducted` 完成了数据库库存扣减逻辑,这时候发现用户余额不足抛出了异常，我们就会出现脏数据。

* #### 第二阶段
通过数据库事物保证数据一致性

![Laravel](https://cdn.learnku.com/uploads/images/202106/02/43464/zduoeuzJgt.png!large)
如上图所示，通过代码我们解决了数据一致性,在项目中我们开始也是这么解决的，但是会带来新的问题：因为我们使用的是 `event-sourcing` 本身,然后事物里面包含的肯定远比上面的逻辑更复杂，导致这个事物运行的时间很长，产生各种脏读，不可重复读，产生一些不可控的问题。

* #### 第三阶段
通过 saga 异步的方式保证一致性

![Laravel](https://cdn.learnku.com/uploads/images/202106/02/43464/pnbWIIKZRu.png!large)

saga 的方式我们是通过逻辑去保证一致性，在上图的步骤9 `restockFromRollback` 方法其实是通过逻辑将先前扣除的库存加回来，这样的好处是：
* 不用经过数据库的事物去保证一致性
* Saga 的每一步通过异步去处理，提升效率
* 在 `stored_events` 表可以追溯事件的流程，方便我们排查问题



