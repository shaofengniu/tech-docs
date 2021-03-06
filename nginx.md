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

*nginx的这种处理方式使其线程模型，事件逻辑和应用逻辑完全解耦合。*

接下来，我们以`ngx_epoll_module.c`为例，分析一下底层的event engine的工
作机制。

# event/modules/ngx_epoll_module.c

在`ngx_event_module_t ngx_epoll_module_ctx`的定义中，我们看到底层的
event engine需要提供的一些通用接口，如添加、删除、处理事件等。

*似乎每个module中都会提供一些commands，如ngx_epoll_commands，暂时不了
 解其作用。*
 
在分析epoll_module中的各个接口之前，先顺便看一眼`struct
ngx_event_s`（event/ngx_event.h)的定义：

* `void *data`存放该事件相关的private data
* `unsigned write:1, accept:1`事件类型，但是为啥没有read?
* `unsigned active:1, disable:1`事件状态
* `ngx_event_handler_pt handler`处理函数指针

其他field我们暂时跳过，之后涉及到的时候再做分析。

## ngx_epoll_add_event

函数签名为

```c
ngx_epoll_add_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t
flags)
```

其中`ev`为外部创建好的事件结构，event用来区分是读还是写，flags则用来控
制一些额外的属性。

这个函数的流程如下：

1. `ngx_connection_t *c = ev->data`，取出该event相关联的connection
2. 获取与该connection相关联的上一个不同类型事件，如果该事件为active，
则与当前添加的时间一并作为新的时间类型
3. 调用`epoll_ctl`进行设置

## ngx_epoll_process_events

此函数处理流程如下：

1. 使用`epoll_wait`获取当前active的事件
2. 根据`flags & NGX_POST_EVENTS`来判断是直接调用事件对应的handler进行
处理，还是将该事件放入`ngx_posted_accepted_events`或
`ngx_posted_events`中稍后进行处理。

这里提一下在获取event关联的两个变量的时候的一个trick：

```c
c = event_list[i].data.ptr;
instance = (uintptr_t) c & 1;
c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
```

根据`ngx_event_s`中的定义，`instance`是一个1bit的field，而
`ngx_connection_t *c`的低3位一定为0（指针以4或8bytes对齐），所以在存放
的时候就可以如下：

```c
ee.data.ptr = (void *) ((uintptr_t) c | ev->instance);
```

至于`instance`的具体作用，现在还不知道……

现在我们回过头来看一下几种event的具体处理流程是怎样的。首先是accept
event，从文件名来看应该是定义在`event/ngx_event_accept.c`中的
`ngx_event_accept`，然后根据这个符号出现的位置找到
`event/ngx_event.c`中的`ngx_event_process_init`，然后这个函数作为
`ngx_module_t ngx_event_core_module`的一部分，将在ngx初始化的时候被调
用。

## ngx_event_process_init

我们暂时不关心这个函数中的大部分初始化工作，他会对
`cycle->listening.elts`中的每一个需要监听的socket创建一个connection，
然后将其handler设置为`ngx_event_accept`，然后将其加入event engine中。

## ngx_event_accept

accept的流程如下：

1. 判断该时间是由io触发还是timeout触发，如果是timeout触发，则调用
`ngx_enable_accept_events`，接受accept事件
2. 设定`ev->available`的数值，即该次事件处理需要连续accept多少个连接
3. accept socket
4. 处理accept异常返回，如果errno为`EMFILE`或者`ENFILE`的话，说明当前连
接数过多，需要屏蔽accept事件一段时间
5. 如果accept成功，则从连接池中创建一个新连接
6. 调用与这个listening socket相关联的handler处理该连接：
`ls->handler(c)`
7. 如果`ev->available`大于0，则跳转到第3步

我们现在来关注一下6中与`ngx_listening_t`相关联的handler。

首先，`ngx_listening_t`定义在`ngx_connection.h`中，然后在该文件中，我
们找到了一个与`ngx_listening_t`相关的函数：`ngx_create_listening`。但
是这个函数中只是对`ngx_listening_t`进行了一些最基本的初始化，并没有对
`handler`进行赋值，所以我们需要查看一下nginx都在哪里调用了这个函数。

结果是在`http/ngx_http.c/ngx_http_add_listening`和
`src/mail/ngx_mail.c/ngx_mail_optimize_servers`两个地方分别将
`handler`赋值为`ngx_http_init_connection`和`ngx_mail_init_connection`。
由此可见，该`handler`的工作是进行一些与具体协议相关的初始化工作。这里
需要注意，是由这个`handler`根据具体协议的需求，将这个connection放入
event loop中。









