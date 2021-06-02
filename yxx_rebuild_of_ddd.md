## 为什么需要 Rebuild
### 需求
查询当前系统用户的余额是一件很容易的事情，但是我需要查询当前系统用户具体到哪一天的余额就是一件比较麻烦的事情。如果不用 `event-sourcing` 可能难以实现。
### 想用 Laravel-Event-Sourcing 自带的 Replay 解决
[Laravel-Event-Sourcing](https://spatie.be/docs/laravel-event-sourcing) 自带了 [Replaying events](https://spatie.be/docs/laravel-event-sourcing/v4/advanced-usage/replaying-events) 功能，但是它只能重播数据库所有的事件，无法进行有效筛选。当我执行：
```php
php artisan event-sourcing:replay App\\Projectors\\AccountBalanceProjector
```
它的原理是将 `stored_events` 表所有事件按 id 从小到大的顺序查询出来，然后在去 `App\\Projectors\\AccountBalanceProjector`  检查有没有 `on+事件名称` 的方法, 当方法存在便执行，`Projector` 一般是数据操作。
[Laravel-Event-Sourcing](https://spatie.be/docs/laravel-event-sourcing) 自带的 [Replaying events](https://spatie.be/docs/laravel-event-sourcing/v4/advanced-usage/replaying-events) 便存在如下几个问题：
* 无法按条件筛选，比如我要 Replay 指定日期的事件
* 事件太多的时候，比如当我事件100多万个，这个时候重播的时间可能几个小时都搞不定，而且占用内存资源。

### 自己定义 Rebuild 去解决类似的问题
针对上述两个问题，我们的解决办法是
* 在`Spatie\EventSourcing\StoredEvents\Repositories\EloquentStoredEventRepository` 基础上增加 `retrieveAllStartingFromAndUntil`方法，如下：
```php
class AppEloquentStoredEventRepository extends EloquentStoredEventRepository
{

    public function retrieveAllStartingFromAndUntil(int $startingFrom, Carbon $tillDatetime, string $uuid = null)
    {
        $query = $this->prepareEventModelQuery($startingFrom, $tillDatetime, $uuid);

        /** @var LazyCollection $lazyCollection */
        $lazyCollection = $query
            ->orderBy('id')
            ->cursor();

        return $lazyCollection->map(fn (EloquentStoredEvent $storedEvent) => $storedEvent->toStoredEvent());
    }

    private function prepareEventModelQuery(int $startingFrom, Carbon $tillDatetime, string $uuid = null): Builder
    {
        /** @var Builder $query */
        $query = $this->storedEventModel::query()->startingFrom($startingFrom)->where('created_at', "<=", $tillDatetime);

        if ($uuid) {
            $query->uuid($uuid);
        }

        return $query;
    }
}
```
这样就能查询到具体截止到哪一天的时间的事件。

* 针对事件太多，比如100万个事件，重播的时候太慢。我们的解决办法是采用异步的方式去解决。但不是每一个事件都异步，每一个事件都异步的话就会产生一个问题：
#### 问题
比如同一个 `aggregate_uuid` 有充值和消费的事件，充值和消费的事件同步被监听的话，可能消费的事件执行的时候，充值还没执行完成，这时候消费的时候去检查用户余额发现没有，导致无法消费。我们肯定不希望因为因为这种异步的先后顺序导致逻辑问题。
#### 解决方案
所以我们的解决方案是根据 `aggregate_uuid` 去异步，`stored_events`表有100万个事件，同一个`aggregate_uuid` 可能有多个充值消费等事件。
* ##### 第一步
在 `EloquentStoredEventRepository ` 基础增加  `getDistinctUuids ` 方法获取不同的 `aggregate_uuid`，数据库可能有100万事件，但是不重复 `aggregate_uuid` 可能才10万个。
 ```php
 class AppEloquentStoredEventRepository extends EloquentStoredEventRepository
{
        public function getDistinctUuids():LazyCollection
       {
            $query = $this->storedEventModel::query();
            return $query->select('aggregate_uuid')->groupBy('aggregate_uuid')->cursor();
       }
}
 ```

* ##### 第二步
将 `getDistinctUuids` 方法获取的不同`aggregate_uuid`循环丢到队列，在处理队列的逻辑里面根据 `aggregate_uuid` 和截止时间找到对应的事件进行重播。如果同时开启多个 work 监听队列，这样处理速度将会大大提升。

* 通过 `App\Support\WaitGroup` 在 php 中知道所有异步处理完成。这种处理方式主要参考的是`Go` 语言的 [WaitGroup](https://studygolang.com/static/pkgdoc/pkg/sync.htm#WaitGroup)
```php
namespace App\Support;
use Illuminate\Support\Facades\Redis;
class WaitGroup
{
    public string $workerNumKey;

    public function start()
    {
        $this->workerNumKey = "wait-group:" . md5(uniqid(mt_rand(), true));
        Redis::DEL($this->workerNumKey);
    }

    public function add()
    {
        Redis::INCR($this->workerNumKey);
    }

    public function wait()
    {
        while (Redis::GET($this->workerNumKey) != 0) {
            sleep(1);
        }
        Redis::DEL($this->workerNumKey);
    }

    public function done()
    {
        $workerNum = Redis::DECR($this->workerNumKey);
        if ($workerNum < 0) {
            throw new \Exception("worker num cannot be less than 0!");
        }
    }
}
```

### 自己定义 Rebuild  核心代码
* 执行流程图

* RebuilderBase
  当调用 `RebuilderBase` 的 `handle` 方法，将获取到不同 `aggregate_uuid`，再把不同的`aggregate_uuid` 赋值给 `RebuilderBase`自身以后，将 `RebuilderBase` 自身作为参数传递给 `RebuilderJob`队列。在队列中就可以调用`RebuilderBase` 自身的`handleStoredEvents`方法进行事件重播操作。看代码要理解的是 `RebuilderBase` 的`handleStoredEvents`方法是在队列里面执行的。
```php
abstract class RebuilderBase
{
    use RedisLockTrait;

    public $instanceHandler;

    public string $aggregateUuid;
    /**
     * @var WaitGroup
     */
    public WaitGroup $waitGroup;

    public function __construct()
    {
        $this->waitGroup = new WaitGroup();
    }

    /**
     * @return LazyCollection
     */
    public function getDistinctUuids():LazyCollection
    {
        return app(AppEloquentStoredEventRepository::class)->getDistinctUuids();
    }

    abstract public function getStoredEvents();

    public function handleStoredEvents():void
    {
        $this->getStoredEvents()
            ->filter(function (StoredEvent $storedEvent) {
                return \in_array($storedEvent->event_class, $this->instanceHandler->handles());
            })
            ->each(function (StoredEvent $storedEvent) {
                $this->instanceHandler->handle($storedEvent);
            });
        $this->waitGroup->done();
    }

    public function handle(): void
    {
        $this->instanceHandler->onStartingEventReplay();
        $this->setLockKey($this->getInstanceHandlerSnakeName())->lock();
        try {
            $this->waitGroup->start();
            $this->dispatchJob();
            $this->waitGroup->wait();
        } catch (\Exception $exception) {
            Log::error($exception->getMessage());
            throw $exception;
        } finally {
            $this->releaseLock();
        }
    }

    public function dispatchJob():void
    {
        $this->getDistinctUuids()
            ->each(function (EloquentStoredEvent $eloquentStoredEvent) {
                $this->waitGroup->add();
                $this->aggregateUuid = $eloquentStoredEvent->aggregate_uuid;
                RebuilderJob::dispatch($this)->onQueue($this->getQueueName());
            });
    }

    public function getInstanceHandlerSnakeName():string
    {
        return Str::snake((new \ReflectionClass($this->instanceHandler))->getShortName());
    }

    protected function getQueueName(): ?string
    {
        return config('queue.queue_name.longtime');
    }
}
```

* RebuilderJob
```php
class RebuilderJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public RebuilderBase $rebuilder;
 
    public function __construct(RebuilderBase $rebuilder)
    {
        $this->rebuilder = $rebuilder;
    }

    public function failed(\Throwable $exception)
    {
        $this->rebuilder->waitGroup->done();
        \Log::error($exception->getMessage());
    }
    
    public function handle()
    {
        $this->rebuilder->handleStoredEvents();
    }
}
```


