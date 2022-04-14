+++
title = "DRM框架分析（五）：任务调度"
date = 2022-04-11
tags = ["kernel", "drm"]

+++

本文分析的这个DRM调度器实际上是在2018年左右由AMD的部分代码演变出来的，即由AMD私用变成DRM子系统共用的了。目前有三四个驱动使用这个DRM调度器。首先明确这个调度器调度的是什么：Hardware Run Queue，或者Hardware Ring Buffer。简而言之，就是一些硬件Command Ring类似的单元实际上是有限的资源，在同一时刻只能执行固定个数的任务，需要通过软件的手段调度这个硬件资源，让相关的任务排队，并均匀分配。

在drm_sched模块中，有以下映射关系：

* Hardware Run Queue与调度器一一对应
* 每个调度器分为多个优先级run queue
* 每个run queue中对调度实体entity进行调度
* 每个调度实体包含一个job queue

简单来说，DRM任务调度器接收调度实体，根据优先级按照轮转的方式将调度实体中的任务进行执行。这个过程发生在一个内核线程中，每一个调度器在初始化中都会创建自己的内核线程，内核线程的函数为`drm_sched_main`。从框架角度来看，框架使用`drm_sched_job`表示一个任务。

调度器由`drm_gpu_scheduler`结构提表示，上层驱动代码可以通过注册一个`drm_sched_backend_ops`特化一个调度器的行为，这个ops简单包括几个回调函数，因此调度器的实现是比较固定的。

## 队列管理

`drm_gpu_scheduler`中存在`hw_submission_limit`字段，用来设定对应hardware run queue中最大支持的任务数。在当前提交的任务数小于这个上限的时候认为这个调度器是就绪状态：

```c
static bool drm_sched_ready(struct drm_gpu_scheduler *sched)
{
        return atomic_read(&sched->hw_rq_count) <
                sched->hw_submission_limit;
}
```

从上面看到，`hw_rq_count`实际上就是已经提交给Hardware run queue的任务的数量。当任务提交之后，对应的`drm_sched_job`会从entity中卸下，然后放入`ring_mirror_list`中。当任务执行完毕之后，`drm_sched_process_job`会进行后续处理，signal对应的fence，然后将job从`ring_miror_list`中取下，并修改对应的计数器。

## 调度实体

调度器中使用`drm_sched_entity`表示一个调度实体，本质上是一个容器，用于存放执行的最小单元：`drm_sched_job`。简单来说`drm_sched_entity`中有一个队列，由`drm_sched_job`组成，调度器保证这个队列中的job按照顺序执行。

这里注意，调度实体看似只是一个简单容器，但是实际上实现了硬件负载均衡功能。简单来说，比如有两个同样功能的hardware run queue，可以同时处理同一种任务，此时需要一个负载均衡机制，才不会出现一个run queue接近满员，而另一个run queue是空的状况。这个负载均衡机制的实现比较简单，`drm_sched_entity`在创建时可以指定多个`drm_sched_rq`，注意这几个队列不是来自同一个调度器，而是多个不同的调度器。`drm_sched_entity`在这个情况下有一个概念叫“当前的”队列，每当`drm_sched_job`创建时，都会调用` drm_sched_entity_select_rq`函数设置这个当前队列。设置规则，很简单，仅仅是遍历可用队列，然后找出当前任务最少的队列即可。

也就是说，为了实现负载均衡，一个调度实体实际上可以在两个调度器里左右横跳。

## 调度策略

调度策略还是比较简单的，本质上就是一个基于优先级的轮询调度。前面看到一个调度器会根据优先级分成多个run queue，每一个run queue中有一个调度实体队列。`drm_sched_select_entity`函数负责在调度器中选择一个实体，这个选择实际上会基于run queue考虑。函数首先检索高优先级的run queue，再检索低优先级的run queue，一旦有run queue能提供一个调度实体，则返回这个调度实体。

对于run queue，`drm_sched_rq_select_entity`负责从run queue中找到可用调度实体。查找过程本质上是轮询一个循环队列，run queue会记录上一次给出的调度实体，然后从这个调度实体的下一个开始检测，依次调用`drm_sched_entity_is_ready`函数检测实体是否就绪。可以看到，实体就绪的两个必要的条件是：

* 调度实体的job队列中存在job
* 调度实体的dependency字段为NULL

当实体就绪时，则函数给出该实体作为结果。对于每一个调度实体，调度器从中取出一个job，调用调度器注册的`run_job`回调函数将其执行。而返回调度实体上的job实际上由`drm_sched_entity_pop_job`函数完成，注意返回一个job不是顺理成章直接完成的事，而是需要与dependency机制结合。

前面看到调度实体`drm_sched_entity`中存在`dependency`指针指向一个`dma_fence`，这个fence实际上就是当前调度实体即将运行的job的依赖。在`drm_sched_entity_pop_job`函数中，函数会调用调度器的`dependency`函数指针，要求使用框架的上层驱动提供这个job的`dependency`指针，即一个dma_fence。如果上层驱动返回了这个fence，并经过多次检查，fence合法后，则设置调度实体的dependency指针，并注册回调函数清除这个指针，同时`drm_sched_entity_pop_job`直接返回NULL。这么做简单来讲，就是运行job之前，调用上层驱动代码注册的回调函数计算并提供job的资源依赖fence，这个fence会阻止job的运行，直到fence被触发。这样也对应上面提到的`drm_sched_entity_is_ready`的第二个条件，即`dependency`字段非NULL的情况下调度实体是没有就绪的，因为相关事件依赖没有被满足。

因此，本质上调度策略比较像内核的实时调度策略，优先执行高优先级run queue，对于每一个run queue，轮换执行每一个调度实体的一个任务，直到实体的任务全部被执行完。

## timeout机制

`drm_gpu_scheduler`中定义了一个`delayed_work`，称作`work_tdr`。它对应的callback为`drm_sched_job_timeout`。大致上，`drm_gpu_scheduler`中会顶一个一个timeout字段，如果一个job超过这个时间没有执行完毕，则会执行callback。可以看到这个`delayed_work`的作用是在job hang住时，进行一个恢复或者通知。与之相关的API是`drm_sched_start_timeout`，这个函数会将这个`delayed_work`重置。

```c
static void drm_sched_start_timeout(struct drm_gpu_scheduler *sched)
{
        if (sched->timeout != MAX_SCHEDULE_TIMEOUT &&
            !list_empty(&sched->ring_mirror_list))
                schedule_delayed_work(&sched->work_tdr, sched->timeout);
}
```

## fence机制

这里的fence基于dma-fence进行了特化，封装成了`drm_sched_fence`：

```c
struct drm_sched_fence {
        struct dma_fence                scheduled;
        struct dma_fence                finished;
        struct dma_fence                *parent;
        spinlock_t                      lock;
        void                            *owner;
};
```

可以简单看到`drm_sched_fence`是三个概念的集合：

* scheduled，这个fence在job被调度（即运行run_job回调函数之前）时触发
* finished，这个fence在真正的hardware job运行完成后触发
* parent，这个fence实际上就是run_job的返回值，这里拿着一个指针，并注册一个callback，好在hardware job完成之时触发finished fence

上面提到给parent注册回调函数实际上就是`drm_sched_process_job`，即在job完成后运行的回调函数。简单来说，`drm_gpu_scheduler`注册的`drm_sched_backend_ops->run_job`回调函数会返回的fence，这个fence会在hardware run queue执行完job后触发。得到这个fence之后，我们直接给这个fence注册`drm_sched_process_job`回调函数，然后在这个回调函数中直接触发名为`finished`的fence，并将job从`ring_mirror_list`中卸下。

## drm_sched_main

`drm_sched_main`实际上就是上面提到的内核线程的执行函数。

函数开头可以看到将这个内核线程的调度策略设置成了`SCHED_FIFO`，并将优先级设置成1。随后函数进入了任务处理循环，可以看到`drm_gpu_scheduler`中提前准备好了，一个wait queue，名为`wake_up_worker`。函数首先在这个wait queue上以如下条件进行等待：

* cleanup_job = drm_sched_get_cleanup_job(sched)) 不为NULL
* !drm_sched_blocked(sched) && (entity = drm_sched_select_entity(sched)))
* kthread_should_stop()

在上面任意一个条件满足时，内核线程即唤醒，然后继续执行任务。如果cleanup_job不为NULL，则进行如下操作：

```c
                if (cleanup_job) {
                        sched->ops->free_job(cleanup_job);
                        /* queue timeout for next job */
                        drm_sched_start_timeout(sched);
                }
```

随后函数检查entity是否为NULL，不为NULL则根据entity拿到下一个应该执行的job：

```c
                sched_job = drm_sched_entity_pop_job(entity);
                if (!sched_job)
                        continue;
```

随后函数执行以下语句：

```c
                atomic_inc(&sched->hw_rq_count);
                drm_sched_job_begin(sched_job);
```

这两个操作分别是：

* 增加`hw_rq_count`计数器，该计数器标志已经压入hardware run queue的任务的个数
* 将任务从调度实体的队列中拿下，并放到`ring_mirror_list`中，表示它正在被hardware run queue执行。同时，使能timeout定时器，防止job超时

准备工作做完后，直接上层驱动代码注册的`run_job`回调函数，执行任务：

```c
                fence = sched->ops->run_job(sched_job);
                drm_sched_fence_scheduled(s_fence);
```

注意这里的先后顺序，run_job仅仅是将任务压到hardware run queue，此时需要等待硬件执行，所以此时直接触发`scheduled`的fence而不是`finished`。注意`run_job`的返回值也是一个fence，这个fence被触发时，即表明hardware run queue上该job被硬件执行完毕。因此，函数需要在该fence上注册回调函数，当job在硬件上完成后，调用`drm_sched_process_job`函数，进行：

* 计数器更新，hw_rq_count以及num_jobs
* 触发finished fence
* 唤醒内核线程工作（有job处理完毕表示hardware run queue有新空位了）

最后的最后，函数唤醒`job_scheduled`等待队列，表示有新job推到hardware run queue上了。
