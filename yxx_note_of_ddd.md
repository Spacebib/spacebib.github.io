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

### Event-Souring 保存的时候出现 aggregate_version 重复
* #### 分析出现问题的原因
  调用 persist 会将聚合的所有事件保存数据表 stored_events,而每一个事件保存为一行数据的时候有个字段 aggregate_version。
  这个字段是怎么样去生成的了，可以看看下面这个代码
  ```php
      private function apply(ShouldBeStored $event): void
      {
          $classBaseName = class_basename($event);
  
          $camelCasedBaseName = ucfirst(Str::camel($classBaseName));
  
          $applyingMethodName = "apply{$camelCasedBaseName}";
  
          $reflectionClass = new ReflectionClass($this);
  
          $applyMethodExists = $reflectionClass->hasMethod($applyingMethodName);
          $applyMethodIsPublic = $applyMethodExists && $reflectionClass->getMethod($applyingMethodName)->isPublic();
  
          if ($applyMethodExists && $applyMethodIsPublic) {
              try {
                  app()->call([$this, $applyingMethodName], ['event' => $event]);
              } catch (BindingResolutionException $exception) {
                  $this->$applyingMethodName($event);
              }
          } elseif ($applyMethodExists) {
              $this->$applyingMethodName($event);
          }
  
          $this->appliedEvents[] = $event;
  
          $this->aggregateVersion++;
      }
  ```
  aggregateVersion 是把这个 aggregate_uuid 所有事件读出来,比如他有 100 个事件,因为版本号从 1 开始,在 101 个事件的时候版本号就是 101。
  。同时为了保证数据的最后一个版本和事件汇总的数量一致，Event-Souring 的包中有这么一段代码。
  ```php
    protected function ensureNoOtherEventsHaveBeenPersisted(): void
    {
        if (static::$allowConcurrency) {
            return;
        }

        $latestPersistedVersionId = $this->getStoredEventRepository()->getLatestAggregateVersion($this->uuid);

        if ($this->aggregateVersionAfterReconstitution !== $latestPersistedVersionId) {
            throw CouldNotPersistAggregate::unexpectedVersionAlreadyPersisted(
                $this,
                $this->uuid,
                $this->aggregateVersionAfterReconstitution,
                $latestPersistedVersionId,
            );
        }
    }
  ```
  这段代码的作用就是：查询当前 aggregate_uuid 的最后一个事件，和在 Event-Souring 读出来的所有事件相加做一个比对，
  验证版本号是否正确。

  在一般情况下，使用上面代码是没有问题。但是上面忽略了一个问题，就是这个 aggregate_uuid 如果同时有2个业务逻辑同时发生。
  同一个时间点调用了 `retrieve` 方法。那么对于这两个发生的事件,因为还没保存。在他们的 aggregate 里面 aggregateVersion 同时
  都是 100，因为没保存数据库。调用 `ensureNoOtherEventsHaveBeenPersisted` 去查这个 aggregate_uuid 的最后一个版本
  也是对的上的。那么这两个事件保存的时候，版本号就会都是 101。

* #### 解决方法
  在 `Spatie\EventSourcing\StoredEvents\Repositories\EloquentStoredEventRepository` 找到如下代码
  ```php
  public function retrieveAllAfterVersion(int $version, string $uuid): LazyCollection
  {
      /** @var \Illuminate\Database\Query\Builder $query */
      $query = $this->storedEventModel::query()
          ->uuid($uuid)
          ->afterVersion($version);

      return $query
          ->orderBy('id')
          ->cursor()
          ->map(fn (EloquentStoredEvent $storedEvent) => $storedEvent->toStoredEvent());
  }
  ```

  修改为：
  
   ```php
  public function retrieveAllAfterVersion(int $version, string $uuid): LazyCollection
  {
      /** @var \Illuminate\Database\Query\Builder $query */
      $query = $this->storedEventModel::query()
          ->uuid($uuid)
          ->afterVersion($version)->lockForUpdate();

      return $query
          ->orderBy('id')
          ->cursor()
          ->map(fn (EloquentStoredEvent $storedEvent) => $storedEvent->toStoredEvent());
  }
  ```
  加锁去避免这一情况
  
* #### 其他尝试
  在 `Spatie\EventSourcing\StoredEvents\Repositories\EloquentStoredEventRepository` 将如下代码
  ```php
  public function getLatestAggregateVersion(string $aggregateUuid): int
  {
      return $this->storedEventModel::query()
          ->uuid($aggregateUuid)
          ->max('aggregate_version') ?? 0;
  }
  ```
  修改为
  ```php
  public function getLatestAggregateVersion(string $aggregateUuid): int
  {
      return $this->storedEventModel::query()
          ->uuid($aggregateUuid)
          ->count() ?? 0;
  }
  ```
  最后还是会重复写入 aggregateVersion

* #### 总结
  只要涉及查询一条数据,然后用这条数据进行其他逻辑操作,在并发情况下必然是有问题的。必须查询的时候要加锁。可以使用数据库的悲观锁,以及自己定义的乐观锁去解决此类问题。
  我们加入 `lockForUpdate()` 去解决此类问题时候,有几个注意点：
  > * 只有在事务中才会生效。
  > * 当 sql 语句涉及到索引 , 并用索引作为查询或判断的依据时，那么 mysql 会用行级锁锁定所要修改的行，否则会使用表锁锁住整张表，因此在使用时一定要注意使用索引，否则会导致高的并发问题。
  > * 性能问题
    
### 多个 Sage 监听同一个 event 问题
![](http://www.plantuml.com/plantuml/png/dPAzJiCm58LtFyLzWIoC7IArGXK34eYYBdHnugjoHM97ZXFQcGM91IHWuGKO62eX8G6lqnyUWqQ9urO9Ah15iUztphd7Xao4S88fwXo1Guxd54R80ZLX2TU6GaguDD1JweBaEDtwkQyfHrqT7L9gje-79QkRSufuG14PmfIX553G6S-CabaSe6Pddcy5e5EP6KJA770f8jGZ-JMxMjq_ZsIYLOXfMbxXXfI4vUFxylM1CGlm_AOjw1oNWzrBldOXnnk00HzpS0fSY5DL35bma-Rv4t1sgw-k45XDzjTvKS3yusR--SPQ0Mvyfx7LqztYzcLnFOEcqd3FghZqhLlVfUFogp3CacblMW7j5bgfsuk0dL70PJagP0X5BLGkhSu1a_OUuPT5WiPOvTZNAjuiyuSO_wgZ4Q5kt9Nn1x0jG9VFtpvhzWq0)
![](http://www.plantuml.com/plantuml/png/dPCzJiD048NxFSLS81UWY9G8HHH8841Ga6RY7IKZd5rhT-qaDGqI3KX0mmKeA28X8G7NoTynY9Mi7ME34bhFxlVclRTsx4A24x9a4WA400qCqFP4Hmz5XCPnm14g1qsjhrXrRU7Zlk64p7fqoDOLn-VKuo2aHe8SOeT3TanMa9AGqWN6Jgkuto4ZBcjrBm1xnqO7PEt5SetLOkXOgmDKCYJJLelnzVpXfQsYocCmU_gOlJqAuNcrUddBJACnmELIgli3SefTX5m9fKmFh0bdEcLudJAyLXz2RcRZOaDAaglRhMKY967oUJlvUXULa58UV-ywVxwVthrERyPGhUYrybWgszb6BGz61q4SZPgZDTHdKaaknW9RcOBSrL4gnIGpiLM4cHyOLXSDEpGDXleweOq0MqRtDzl-nTc_kogvofv4tjEESD_D89kI_oN4Dm00)
在上图中,
ProcessAwaitingReservationSaga 监听完 OccupiedByBooking 调用 reserve 方法，
ProcessAwaitingWashReservationSaga 监听完 OccupiedByBooking 调用 washReserve 方法。
但是 OccupiedByBooking 一旦被调用。这两个 sage 都会被监听。但其实我只需要给其中一个 sage。   

#### 我的解决思路
  通过 ```php AggregateRoot::retrieveRoot($event->bookingAggregateId)->getAppliedEvents() ``` 获取相关对应 aggregate_uuid 的最近相关的事件。
  判断他的开始事件是什么。然后对应的 saga 执行下一步。


