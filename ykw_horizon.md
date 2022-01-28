---
tags: [PHP, Laravel]
---
# Horizon

![Pasted image 20220126112926](https://user-images.githubusercontent.com/37279857/151481451-e61a6f8c-0a93-4dd4-aa16-e4ed783456b6.png)

master 通过 `proc_open(php artisan horizon:supervisor)` (Laravel\Horizon\Console\SupervisorCommand)启动 supervisor 进程。
supervisor 通过 `proc_open(php artisan horizon:work)`  启动 worker 进程。

supervisor 进程启动后，**第一时间会试图使 worker 进程的数量达到最大值**。之后每隔 cooldown 时间有一次机会调整 woker 进程的数量。
### 调度策略
- false
   将使用默认的 Laravel 行为，它按照配置中列出的顺序处理队列。
- simple
  在 worker 进程之间平均分配传入的 job。
- auto
  根据队列的当前工作量调整每个队列的 worker 进程数。
  ```PHP
  
  $queues = [
    [$queue => [
            'size' => $size,
            'time' =>  ($size * $this->metrics->runtimeForQueue($queue)),
    ]]
  ];
  
  protected function numberOfWorkersPerQueue(Supervisor $supervisor, Collection $queues)
  {
    $timeToClearAll = $queues->sum('time');

    return $queues->mapWithKeys(function ($timeToClear, $queue) use ($supervisor, $timeToClearAll) {
        if ($timeToClearAll > 0 &&
            $supervisor->options->autoScaling()) {
            return [$queue => (($timeToClear['time'] / $timeToClearAll) * $supervisor->options->maxProcesses)];
        } elseif ($timeToClearAll == 0 &&
                  $supervisor->options->autoScaling()) {
            return [
                $queue => $timeToClear['size']
                            ? $supervisor->options->maxProcesses
                            : $supervisor->options->minProcesses,
            ];
        }

        return [$queue => $supervisor->options->maxProcesses / count($supervisor->processPools)];
    })->sort();
   }
  ```
再根据当前所有的 worker 进程数量决定改增加还是减少 worker 进程数量。
变化的 woker 进程数量受配置 **balanceMaxShift** 的影响, 基本可以理解为每次最多新增或减少 balanceMaxShift 个 worker 进程。

>The balanceMaxShift and balanceCooldown configuration values to determine how quickly Horizon will scale to meet worker demand. In the example above, a maximum of one new process will be created or destroyed every three seconds. You are free to tweak these values as necessary based on your application's needs.
