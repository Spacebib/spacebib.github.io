## 注意点

### 事件 (Event)
事件对象的属性不能随意添加,如果需要添加,应该重新添加一个事件类似于 (AwaitingPaymentV2)。

### 聚合 (AggregateRoot)

所有的涉及业务的逻辑，应该在聚合处理判断，如使用次数限制等等，方便测试

### 在项目中使用了 Event-Souring 模式的领域模块如何与传统的 Crud 模块交互
* #### 遇到问题场景：
  在项目中,我们假设有个事件叫 `CarWashEndEvent`,通常我们会在对应的 `Projector` 和 `Reactor` 处理我们的业务逻辑。可是如果加了一个需求，需要我们改变下 `vehicle` 表车辆的状态，比如状态改为洗车完成。
* #### 分析问题：
  * 我们不能直接在 `Projector` 修改 `vehicle` 表的状态,这是不符合规范的，我当时想的是可以看看数据库又没有一张表叫 `projector_vehicle`,没有就新建,通过 `Projector` 把状态保存到这种表里面。后面我发现这张表涉及的业务太多，后面没考虑这个方法。
  * 后面参考 [spatie 的这篇文章](https://freek.dev/1634-mixing-event-sourcing-in-a-traditional-laravel-app) ,在 `Reactor` 再去调用 `crud` 的事件。
    