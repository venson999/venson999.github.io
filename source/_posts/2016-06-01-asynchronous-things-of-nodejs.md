---
title: Node.js之异步那些事
date: 2016/06/01 13:27:39
categories:
    - 技术
tags: [Node.js, libuv, Async]
---

![nodejs](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-d48f635dbaf4e931.png)

>Node.js® is a JavaScript runtime built on [Chrome's V8 JavaScript engine](https://developers.google.com/v8/). Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient. 

Node.js官网上的介绍，其中事件驱动非阻塞I/O模型是被大家所津津乐道的，但是有多少人真正了解其究竟呢？有人可能会想到libuv，没错，libuv确实是其幕后英雄。那么问题又来了，到底是怎么用libuv实现的呢？下面我们来一探究竟。

![libuv](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-18b05afb49048c3a.png)


libuv当初主要就是为Node.js开发的，提供跨平台的事件驱动异步I/O能力，当然现在肯定不仅限于Node.js使用。我们先来看一下libuv的[Design overview](http://docs.libuv.org/en/v1.x/design.html)。

![architecture](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-304a3934c4de4dd1.png)

从架构图上看，libuv是对多个平台上的事件驱动异步I/O库进行了封装，如Linux下的epoll、FreeBSD下的kqueue、Solaris下的event ports、Windows下的IOCP。

![loop_iteration](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-703176efbc31b437.png)

上图所描述的事件循环是libuv中最重要的概念，其中的`Poll for I/O`就是事件驱动异步I/O能力的核心。到这里我们有必要先了解一些基础知识，[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)，否则后面的东西就不是特别好理解了。

# 正题
---
经过前面的学习，应该对libuv有了一个整体的印象，总结一下， libuv其实就是把各种`handle`和`io_watcher`放到事件循环里，然后每一次循环都去检查一下是否有他们关心的事件需要处理，有则调用相应的`callback`，没有则继续循环。要想弄清楚Node.js之异步那些事，我们需要关心的是，Node.js如何运行事件循环，何时把`handle`和`io_watcher`放入事件循环，以及如何调用相应的`callback`。

开始之前，本次分析的代码版本为Node.js v0.12.6，Linux平台。

#### Run
`node.cc`中`Start`方法运行事件循环，精华部分如下。唯一有些特别的地方就是，在一个`while`循环中包了两个`uv_run`，模式分别是`UV_RUN_ONCE`和`UV_RUN_NOWAIT`，其原因在中间的两行注释中已经说得很明白了。
```
...
    bool more;
    do {
      more = uv_run(env->event_loop(), UV_RUN_ONCE);
      if (more == false) {
        EmitBeforeExit(env);

        // Emit `beforeExit` if the loop became alive either after emitting
        // event, or after running some callbacks.
        more = uv_loop_alive(env->event_loop());
        if (uv_run(env->event_loop(), UV_RUN_NOWAIT) != 0)
          more = true;
      }
    } while (more == true);
...
```

然后我们可以看看`core.c`中`uv_run`方法的代码，跟上面事件循环的流程图是可以一一对应的。

#### Data Structure
继续看代码之前，有必要先了解一下重要的数据结构和相互的关系，以便更好的理解。

![Data Structure](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-88dbebb2ab072484.png)

#### io_watcher
接着我之前文章{% post_link 2015-08-21-the-big-pit-behind-nodejs-hello-world Node.js之HelloWorld背后的大坑 %}的思路，还拿Hello World举例子，跟libuv有关的代码都在`tcp_warp.cc`里面了。

* `TCPWrap::New`

![New](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-43f353f0c57de9a8.png)

`stream.c`中`uv__stream_init`方法有如下代码，将`io_watcher`的`cb`设置为`uv__stream_io`，`fd`设置为`-1`，这里只是在stream层面做的初始化设置，后面到tcp层面还会有相应的改变。

```
  uv__io_init(&stream->io_watcher, uv__stream_io, -1);
```

* `TCPWrap::Bind`

![Bind](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-710303f09d543a78.png)

`tcp.c`的`maybe_new_socket`方法中，`uv__socket`方法生成了新的`fd`，`uv__stream_open`方法将其设置到`io_watcher`的`fd`。

* `TCPWrap::Listen`

![Listen](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-7c55118e2fbf803f.png)

`tcp.c`的`uv_tcp_listen`方法中有如下代码，将`io_watcher`的`cb`设置为`uv__server_io`，`uv__server_io`里面会调用`connection_cb`，`connection_cb`已经被设置为`cb`，而这个`cb`正是`tcp_wrap.cc`中的`TCPWrap::OnConnection`方法。

```
...
  tcp->connection_cb = cb;

  /* Start listening for connections. */
  tcp->io_watcher.cb = uv__server_io;
  uv__io_start(tcp->loop, &tcp->io_watcher, UV__POLLIN);
...
```

`core.c`中`uv__io_start`方法有如下代码，利用`void* watcher_queue[2]`变量将`io_watcher`加入到`uv_loop_t`的队列中去，具体操作详见`queue.h`。将`uv_loop_t`的`uv__io_t** watchers`当做数组使用，`fd`为下标，`io_watcher`为对应的值。

```
...

  if (QUEUE_EMPTY(&w->watcher_queue))
    QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);

  if (loop->watchers[w->fd] == NULL) {
    loop->watchers[w->fd] = w;
    loop->nfds++;
  }
...
```

#### uv__io_poll
`linux-core.c`中的`uv__io_poll`方法，一行一行的读就可以了，前面的铺垫已经做得很充分了，只要读懂谜底便可揭晓。

# 未完
---
* 接下来我们来说说`process.nextTick(callback)`的事，在`node.js`中定义如下，把`callback`放到了`nextTickQueue`队列中，那么Node.js是在什么时候消费这个队列的呢？

```
    function nextTick(callback) {
      // on the way out, don't bother. it won't get fired anyway.
      if (process._exiting)
        return;

      var obj = {
        callback: callback,
        domain: process.domain || null
      };

      nextTickQueue.push(obj);
      tickInfo[kLength]++;
    }
```

* `tcp_wrap.cc`中`TCPWrap::OnConnection`方法有如下代码，`MakeCallback`方法的出处如下图。

```
  tcp_wrap->MakeCallback(env->onconnection_string(), ARRAY_SIZE(argv), argv);
```

![MakeCallback](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2016-06-01-asynchronous-things-of-nodejs/270064-676397be1a87bc96.png)

* `async-wrap.cc`中`MakeCallback`方法有如下代码。

```
  env()->tick_callback_function()->Call(process, 0, NULL);
```

* `node.cc`中`SetupNextTick`方法有如下代码，对`tick_callback_function()`进行了设定。

```
  env->set_tick_callback_function(args[1].As<Function>());
```

* `node.cc`中`SetupProcessObject`方法有如下代码，`SetupNextTick`被设定为`process`中的`_setupNextTick`方法。

```
  NODE_SET_METHOD(process, "_setupNextTick", SetupNextTick);
```

* `node.js`中`startup.processNextTick`方法有如下代码。

```
  process._setupNextTick(tickInfo, _tickCallback, _runMicrotasks);
```

* `node.js`中`_tickCallback`方法代码如下，消费`nextTickQueue`队列中的`callback`方法。

```
    function _tickCallback() {
      var callback, threw, tock;

      scheduleMicrotasks();

      while (tickInfo[kIndex] < tickInfo[kLength]) {
        tock = nextTickQueue[tickInfo[kIndex]++];
        callback = tock.callback;
        threw = true;
        try {
          callback();
          threw = false;
        } finally {
          if (threw)
            tickDone();
        }
        if (1e4 < tickInfo[kIndex])
          tickDone();
      }

      tickDone();
    }
```

省略去中间步骤，实际上是产生了如下的调用关系。

```
TCPWrap::OnConnection()
↓↓↓
_tickCallback()
```

# 总结
---
简单说，整个过程是这样的，事件循环中有相应I/O事件发生的时候，libuv调用Node.js C++部分的回调，C++部分调用JavaScript部分的回调，顺便调用nextTick设定的回调。

还是认真读代码吧，以上写的仅供参考。
