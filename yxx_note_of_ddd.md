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
  public function retrieveAll(string $uuid = null): LazyCollection
  {
      /** @var \Illuminate\Database\Query\Builder $query */
      $query = $this->storedEventModel::query();

      if ($uuid) {
          $query->uuid($uuid);
      }

      return $query->orderBy('id')->cursor()->map(fn (EloquentStoredEvent $storedEvent) => $storedEvent->toStoredEvent());
  }
  ```

  修改为：
  ```php
  public function retrieveAll(string $uuid = null): LazyCollection
  {
      /** @var \Illuminate\Database\Query\Builder $query */
      $query = $this->storedEventModel::query();

      if ($uuid) {
          $query->uuid($uuid)->lockForUpdate();
      }

      return $query->orderBy('id')->cursor()->map(fn (EloquentStoredEvent $storedEvent) => $storedEvent->toStoredEvent());
  }
  ```
  加锁去避免这一情况

  