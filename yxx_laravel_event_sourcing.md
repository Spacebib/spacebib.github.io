## 思考
[Laravel-Event-Sourcing](https://spatie.be/docs/laravel-event-sourcing/v4/introduction) 中 StoredEvent 怎么保存 而 Projector 和 Reactor 又是怎么样监听到对应的事件的？
## 案例
用户取消订单,需要触发 `BookingCancelled` 事件，我们需要在 `Projector` 监听到 `BookingCancelled` 后改变订单状态。同时需要在 `Reactor` 发送一封邮件给用户。
* 我们有一个 AggregateRoot  `App\Bookings\AggregateRoot\BookingRoot`。里面有个取消订单的方法 cancle 代码如下：
```php
class BookingRoot extends AggregateRoot
    public function cancel(): self
    {
        $this->recordThat(
            new BookingCancelled(
                $this->userId,
            )
        );
        return $this;
    }
}
```

* 我们有一个 Projector  `App\Bookings\Projector\BookingsProjector` 。他是同步的，代码如下：
```php
class BookingsProjector extends Projector
{
    public function onBookingCancelled(
        StoredEvent $storedEvent,
        BookingCancelled $event,
        string $aggregateUuid
    ) {
        $projection         = ProjectionBooking::query()
            ->where('booking_aggregate_id', $aggregateUuid)
            ->first();
        $projection->status = BookingStatus::STATUS_CANCELLED;
        $projection->cancelled = $storedEvent->created_at;
        $projection->save();
    }
}
```
* 我们有一个 Reactor `App\Bookings\Reactor\NotificationReactor`。他是异步的,代码如下：
```php
class NotificationReactor extends Reactor implements ShouldQueue
{
    public function onBookingCancelled(BookingCancelled $event, StoredEvent $storedEvent)
    {
        $user = User::query()->findOrFail($event->userId);
        Notification::send($user, new BookingCancelledNotification('订单取消'));
    }
```

## 源码分析
当我们执行如下代码：
```php
BookingRoot::retrieve($bookingAggregateId)->cancel()->persist();
```
*  ##### `Spatie\EventSourcing\AggregateRoots\AggregateRoot` 的 persist 方法（将事件保存到数据库）：
```php
    public function persist()
    {
        $storedEvents = $this->persistWithoutApplyingToEventHandlers();

        $storedEvents->each(fn (StoredEvent $storedEvent) => $storedEvent->handleForAggregateRoot());

        $this->aggregateVersionAfterReconstitution = $this->aggregateVersion;

        return $this;
    }

    protected function persistWithoutApplyingToEventHandlers(): LazyCollection
    {
        $this->ensureNoOtherEventsHaveBeenPersisted();

        $storedEvents = $this
            ->getStoredEventRepository()
            ->persistMany(
                $this->getAndClearRecordedEvents(),
                $this->uuid(),
                $this->aggregateVersion,
            );

        return $storedEvents;
    }
```
在代码 `$this->persistWithoutApplyingToEventHandlers` 会调用 `Spatie\EventSourcing\StoredEvents\Repositories\EloquentStoredEventRepository` 的 persistMany 再循环要保存的事件通过 persist 保存到 `Spatie\EventSourcing\StoredEvents\Models\EloquentStoredEvent` 最后存到数据库表 stored_events 中。`Spatie\EventSourcing\StoredEvents\Repositories\EloquentStoredEventRepository` 代码如下:
```php
class EloquentStoredEventRepository implements StoredEventRepository
{
    public function persistMany(array $events, string $uuid = null, int $aggregateVersion = null): LazyCollection
    {
        $storedEvents = [];
        foreach ($events as $event) {
            $storedEvents[] = $this->persist($event, $uuid, $aggregateVersion);
        }

        return new LazyCollection($storedEvents);
    }

    public function persist(ShouldBeStored $event, string $uuid = null, int $aggregateVersion = null): StoredEvent
    {
        /** @var EloquentStoredEvent $eloquentStoredEvent */
        $eloquentStoredEvent = new $this->storedEventModel();

        $eloquentStoredEvent->setOriginalEvent($event);

        $eloquentStoredEvent->setRawAttributes([
            'event_properties' => app(EventSerializer::class)->serialize(clone $event),
            'aggregate_uuid' => $uuid,
            'aggregate_version' => $aggregateVersion,
            'event_class' => $this->getEventClass(get_class($event)),
            'meta_data' => json_encode($event->metaData()),
            'created_at' => Carbon::now(),
        ]);

        $eloquentStoredEvent->save();

        return $eloquentStoredEvent->toStoredEvent();
    }
```
*  ##### 事件监听代码逻辑：
    ```php
$storedEvents = $this->persistWithoutApplyingToEventHandlers();
   ```
返回的是 item 为  `Spatie\EventSourcing\StoredEvents\StoredEvent` 的 `LazyCollection` 
第二步：
```php
     $storedEvents->each(fn (StoredEvent $storedEvent) => $storedEvent->handleForAggregateRoot());
```
循环调用每一个 `Spatie\EventSourcing\StoredEvents\StoredEvent` 的 `handleForAggregateRoot` 方法。
```php
namespace Spatie\EventSourcing\StoredEvents;
class StoredEvent implements Arrayable
{

    public function handleForAggregateRoot(): void
    {
        $this->handle();

        if (! config('event-sourcing.dispatch_events_from_aggregate_roots', false)) {
            return;
        }

        $this->event->firedFromAggregateRoot = true;
        event($this->event);
    }
    
    public function handle()
    {
        // Projector 和 Reactor 如果是同步 那么在 Projectionist 的 handleWithSyncEventHandlers 方法处理
        Projectionist::handleWithSyncEventHandlers($this);

        if (method_exists($this->event, 'tags')) {
            $tags = $this->event->tags();
        }

        if (! $this->shouldDispatchJob()) {
            return;
        }
        $storedEventJob = call_user_func(
            [config('event-sourcing.stored_event_job'), 'createForEvent'],
            $this,
            $tags ?? []
        );
        // Projector 和 Reactor 如果是异步就把 Spatie\EventSourcing\StoredEvents\HandleStoredEventJob 推送到队列
        dispatch($storedEventJob->onQueue($this->getQueueName()));
    }
```
上面代码中的 $this->event 为 `App\Bookings\Event\BookingCancelled`
$storedEventJob 为 `Spatie\EventSourcing\StoredEvents\HandleStoredEventJob` 属性 $storedEvent 为代码中的 $this。

*  ##### Projector 和 Reactor 同步监听逻辑
上面代码中同步调用代码：
 ```php
 Projectionist::handleWithSyncEventHandlers($this);
 ```
我们找到 `Spatie\EventSourcing\Projectionist` 的 handleWithSyncEventHandlers
 ```php
 namespace Spatie\EventSourcing;
 use Spatie\EventSourcing\EventHandlers\EventHandlerCollection;
 class Projectionist
{
    private EventHandlerCollection $projectors;

    private EventHandlerCollection $reactors;

    private bool $catchExceptions;

    private bool $replayChunkSize;

    private bool $isProjecting = false;

    private bool $isReplaying = false;

    public function __construct(array $config)
    {
        $this->projectors = new EventHandlerCollection();
        $this->reactors = new EventHandlerCollection();

        $this->catchExceptions = $config['catch_exceptions'];
        $this->replayChunkSize = $config['replay_chunk_size'];
    }
    public function handleWithSyncEventHandlers(StoredEvent $storedEvent): void
    {
        $projectors = $this->projectors
            ->forEvent($storedEvent)
            ->syncEventHandlers();

        $this->applyStoredEventToProjectors($storedEvent, $projectors);

        $reactors = $this->reactors
            ->forEvent($storedEvent)
            ->syncEventHandlers();

        $this->applyStoredEventToReactors($storedEvent, $reactors);
    }
}
 ```
上面代码中的 `handleWithSyncEventHandlers ` 方法其实是通过 `EventHandlerCollection` 筛选出同步的 Projector 和 Reactor。
如果是 Projector 执行 `$this->applyStoredEventToProjectors($storedEvent, $projectors)`，
如果是 Reactor 执行 `$this->applyStoredEventToReactors(StoredEvent $storedEvent, Collection $reactors):`
 ```php
   private function applyStoredEventToProjectors(StoredEvent $storedEvent, Collection $projectors): void
    {
        $this->isProjecting = true;

        foreach ($projectors as $projector) {
            $this->callEventHandler($projector, $storedEvent);
        }

        $this->isProjecting = false;
    }
	
    private function applyStoredEventToReactors(StoredEvent $storedEvent, Collection $reactors): void
    {
        foreach ($reactors as $reactor) {
            $this->callEventHandler($reactor, $storedEvent);
        }
    }

    private function callEventHandler(EventHandler $eventHandler, StoredEvent $storedEvent): bool
    {
        try {
            $eventHandler->handle($storedEvent);
        } catch (Exception $exception) {
            if (! $this->catchExceptions) {
                throw $exception;
            }

            $eventHandler->handleException($exception);

            event(new EventHandlerFailedHandlingEvent($eventHandler, $storedEvent, $exception));

            return false;
        }

        return true;
    }
 ```
在上面的案例中到了这一步,我们触发的是 `$this->applyStoredEventToProjectors($storedEvent, $projectors)`。
当调用 `$this->callEventHandler($projector, $storedEvent)` 时候：
$projector 就是 `App\Bookings\Projector\BookingsProjector` ，
$storedEvent 就是 `Spatie\EventSourcing\StoredEvents\StoredEvent` 它的属性 event 是 `App\Bookings\Event\BookingCancelled`。
所以 `callEventHandler` 里面的 `$eventHandler->handle($storedEvent)` 可以理解为：`$projector->handle($storedEvent)`;
其实就是调用了  `App\Bookings\Projector\BookingsProjector`  的 handle 方法。然后 handle 执行了`onBookingCancelled ` 方法。

* ####  Projector 和 Reactor 异步监听逻辑
上面代码中异步调用代码：
 ```php
 dispatch($storedEventJob->onQueue($this->getQueueName()));
 ```
Laravel 的监听最后通过 `Illuminate\Bus\Dispatcher`  执行方法 `dispatchNow($command, $handler = null)` 还是会调用  $storedEventJob 的 handler 方法。
我们找到 `Spatie\EventSourcing\StoredEvents\HandleStoredEventJob`  的相关代码：
 ```php
 namespace Spatie\EventSourcing\StoredEvents;
 class HandleStoredEventJob implements HandleDomainEventJob, ShouldQueue
{
    use InteractsWithQueue, Queueable, SerializesModels;

    public StoredEvent $storedEvent;

    public array $tags;

    public function __construct(StoredEvent $storedEvent, array $tags)
    {
        $this->storedEvent = $storedEvent;

        $this->tags = $tags;
    }

    public function handle(Projectionist $projectionist): void
    {
        $projectionist->handle($this->storedEvent);
    }
}
 ```
`handle` 继续调用了 `Spatie\EventSourcing\Projectionist` 的 `handle` 方法,代码如下：
  ```php
namespace Spatie\EventSourcing;
class Projectionist
{
    public function handle(StoredEvent $storedEvent): void
    {
        $projectors = $this->projectors
            ->forEvent($storedEvent)
            ->asyncEventHandlers();

        $this->applyStoredEventToProjectors(
            $storedEvent,
            $projectors
        );

        $reactors = $this->reactors
            ->forEvent($storedEvent)
            ->asyncEventHandlers();

        $this->applyStoredEventToReactors(
            $storedEvent,
            $reactors
        );
    }
} 
  ```
到这里其实和同步的逻辑就一样了。 `handle` 和 `handleWithSyncEventHandlers` 的区别只是前者筛选的是异步的 Projector 和 Reactor。后者筛选的是同步的。然后走到
`$this->applyStoredEventToProjectors($storedEvent, $projectors)` 以及
`$this->applyStoredEventToReactors(StoredEvent $storedEvent, Collection $reactors)`。
案例中：触发的就是`App\Bookings\Reactor\NotificationReactor` 的 `handle` 方法。然后 `handle` 执行了 `onBookingCancelled` 方法。


## 流程图
![](http://www.plantuml.com/plantuml/png/dPFVQXD15CRlzodE2_G5Sb4eKX5Ha5JmmjniaY4PsSwiixFYUXMjs8O6RRMAD2eKAfPYRK2KDD7qPNPcurjuajc4dUsc_hbPTcU_-SwPttT6KkaHqDyU9qVRgZjogloXuxj2qXhrNIPXfT4GfE5AKkPSMdzMFNu_94okIIv8VVK1lfQ9pmEAtv6bp2YizLk2toCrIJcZWVtdcilg7idikywh3c5rcBJdkA7aB5ol4k7SNLgsEeII-lag7cp7m-_W4n5CZ2t11JsCUnl9tX5KMAg_GsMJXtB5zxs8iiPjFct0T2I2lDkb5ERcgVLDbqMWDf-fmqtLm-T-t3yspSRdxzN9M-TIjpyKmwEeqN7o_5ItFuqFEhEQW9N-fOPrFZp0-PxgVW0goJh4_K4sIqIMx3-56-wZw0htF9DadazMNpBz6IRwz4NSRo60d6Lp2leg5vQHVdEclxvsCbBBUWxQx8QBSbXQjiODQQNVN81wsO4oSStxJaUVV4owkshdyw_MS3pQhJ13r7XFncCjOZLxAYplNAcIRe_KLa-tM_fTszZZiCsA1nMMbgwmR7xgomRIdV608W15jmHAQVmrGP0TSZJthaYRIsS1ZnzFvZommwsk6WxIu8eyKNA6LoyCce1d1foqa2meNjp-T0Vut5zW3rayDfpYCldOKpxsictqzsaQE9ES_Y_Gtm00)

## 注意点
[Laravel-Event-Sourcing](https://spatie.be/docs/laravel-event-sourcing/v4/introduction) 异步处理机制是判断监听当前事件的 Projector 和 Reactor 是否 `instanceof ShouldQueue`，如果比对有一个则将当前事件创建的 HandleStoredEventJob 推送到 queue。无论同步还是异步的执行。都是拿到监听这个事件的所有 Projector 和 Reactor。如案例中的 BookingCancelled，则循环执行所有  Projector 和 Reactor 的 onBookingCancelled 方法。代码如下：
```php
    private function applyStoredEventToProjectors(StoredEvent $storedEvent, Collection $projectors): void
    {
        $this->isProjecting = true;

        foreach ($projectors as $projector) {
            $this->callEventHandler($projector, $storedEvent);
        }

        $this->isProjecting = false;
    }

    private function applyStoredEventToReactors(StoredEvent $storedEvent, Collection $reactors): void
    {
        foreach ($reactors as $reactor) {
            $this->callEventHandler($reactor, $storedEvent);
        }
    }

    private function callEventHandler(EventHandler $eventHandler, StoredEvent $storedEvent): bool
    {
        try {
            $eventHandler->handle($storedEvent);
        } catch (Exception $exception) {
            if (! $this->catchExceptions) {
                throw $exception;
            }

            $eventHandler->handleException($exception);

            event(new EventHandlerFailedHandlingEvent($eventHandler, $storedEvent, $exception));

            return false;
        }

        return true;
    }
```
如果我们不希望循环执行所有  Projector 和 Reactor 时候被中断,可以在 event-sourcing.php  配置文件的 catch_exceptions 设置为 true,然后在 Projector 和 Reactor 自定义 handleException 方法处理我们的异常。