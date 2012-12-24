# core/nginx.c/main

nginx启动后会在nginx.c的`main`函数中执行一系列的初始化工作，其中最重要
的是对`ngx_cycle_t *cycle，init_cycle`的处理

nginx首先对`init_cycle`进行了部分初始化，然后调用`cycle =
ngx_init_cycle(&init_cycle)`对`cycle`进行了初始化，至于*为啥不直接对
cycle进行操作暂时不明白*

`main`的初始化中有一步`ngx_add_inherited_sockets(&init_cycle)`，从名字
猜应该是子进程继承父进程的sockets，但是*具体父子进程共享socket的机制是
怎样的呢?*

cycle是nginx用于维护各种全局状态信息的地方，例如memory pool,
connection pool等等，这个在nginx_cycle.h中有详细定义

完成初始化部分后，nginx会根据`ngx_process`是否等于
`NGX_PROCESS_SINGLE`来选择之后的处理函数，即
`ngx_single_process_cycle(cycle)`或者`ngx_master_process_cycle(cycle)`，
我们暂时先对看似比较简单的`ngx_single_process_cycle`进行分析

# os/unix/ngx_process_cycle.c/ngx_single_process_cycle

首先，对`ngx_modules`中各模块依次调用其`init_process`进行初始化。

之后进入一个循环，在每次循环中依次进行以下工作：

* 处理事件(`ngx_process_events_and_timers(cycle)`)
* 检测是否退出(`ngx_terminate || ngx_quit`)
* 检测是否需要重新初始化cycle(`ngx_reconfigure`)
* 检测是否需要重新打开文件(`ngx_reopen`)

我们接下来主要对如何处理事件进行分析

# event/ngx_event.c/ngx_process_events_and_timers

这个函数需要处理以下几种事件：

1. timer event
2. socket accept event
3. socket read/write event

处理事件的步骤如下（暂不讨论timer的问题）：

1. if (ngx\_use\_accept_mutex) then try to acquire accept mutex
2. ngx_process_events
3. if (ngx\_posted\_accept_events) then process posted accept events
4. if (ngx\_accept\_mutex_held) then release the lock
5. if (ngx\_posted\_events) then wake up workers or process posted
events itself

`ngx_process_events`的定义如下：

```
#define ngx_process_events   ngx_event_actions.process_events
```

其中`ngx_event_actions_t ngx_event_actions`为定义在`ngx_event.c`中的全
局变量，其中包含若干用于操作底层event loop的函数指针，指向底层的epool，
kequeue，select中的具体实现
（event/modules/ngx\_[epoll|kqueue|select|...]_modle.c)

从这里我们可以看出，`ngx_process_events`只与底层的event loop相关，只负
责将各种io事件收集起来，并不涉及任何应用逻辑。具体的应用逻辑应该是由
`ngx_event_process_posted`实现的。

`ngx_event_process_posted`的逻辑非常简单，只是对每一个event调用其自身
的handler进行处理(`ev->handler(ev)`)。

nginx的这种处理方式使其线程模型，事件逻辑和应用逻辑完全解耦合。




