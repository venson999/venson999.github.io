---
title: Node.js之HelloWorld背后的大坑2
date: 2015/11/06 15:34:01
categories:
    - 技术
tags: [Node.js]
---

# 填坑
---

{% post_link 2015-08-21-the-big-pit-behind-nodejs-hello-world 上一篇文章 %}留了个扣子，本来不想继续写的，因为这样写实在是又累又耗时间。而且中间Node.js还发布了v4.0.0版本，一下子没动力了有木有，人家v4.0.0都发布了，我还分析v0.12.7干吗？于是便在这段时间写了两篇自我感觉良好的小文，奈何文笔太差，完全没人看。倍受打击之余，默默的在多看上买了一套《诺贝尔文学奖作品典藏书系（共28册）》，至今还躺在书架上供每日瞻仰使用。还是多写技术文章吧，于是便回来继续填坑。

新版本出了并不代表老版本就没有价值了，我想只有充分了解了老版本才能更好的体会新版本，于是乎欣然继续分析v0.12.7。温习一下git命令，大神略过。

```
% git checkout v0.12.7
```

# 想说Hello不容易
---
上回书说到，Node.js已经摆好姿势，“'Hello World”已经含在嘴里，伺机待发，但是有句歌词是这么唱的“让我怎么说出口……”，充分体现了这种状态，真是不简单。看了代码之后，充分被Node.js秀了一把继承和异步，一个乱字已然无法形容，所以必须先做几点温馨提示。

* 首先最好先重新温习一下网络方面的知识，百度一下TCP/IP、HTTP、Socket、Stream，随便翻翻前几页，有个大概了解就行，要不有些东西真的理解不了。
* 其次要了解一下相关类的继承关系，撇开多继承，这种程度的继承关系在Java里不算复杂，在IDE里也非常容易查看，但是放到JavaScript里只能用一坨来形容，不梳理出来一定会陷入其中。

![Stream Inheritance](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-11-06-the-big-pit-behind-nodejs-hello-world-2/270064-db74aab5708e6432.png)

* 然后一定要按照之前提到的方法，把V8和Debug日志全部打开，对比日志**逐行分析**代码，这样分析起来会事半功倍。
* 最后就是去触发一下程序，毕竟人家已经listen了这么久。这里不推荐浏览器，因为在地址栏里直接输入请求地址会有个问题，浏览器会默认先请求`favicon.ico`，这样日志里会出现大量的干扰，不利于分析。我在这里被坑了半天之后，不由哼出了周董的旋律，“快使用命令行！轻松！愉快！”

```
% curl http://127.0.0.1:1337/
```

# SayHello
---
#### 连接过程

![SayHello1](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-11-06-the-big-pit-behind-nodejs-hello-world-2/270064-9977ef7f964dca99.png)

图中有一个细节需要注意，左上角的第一个调用是空心箭头`->`，这代表了异步调用，我们真正开始接触异步代码了。等等，为什么`EventEmitter.emit()`是普通的实心箭头？没错，他确实只是一个同步调用。我一直以为他是一个Node.js经典的异步操作，OMG，我居然被自己骗了这么长时间。以我目前阅读的代码和日志来看，好像只有`libuv`和`process.nextTick()`发起的调用是异步的，特此说明。

代码是如何从`libuv`到C++再到Node.js的呢？
* `tcp_wrap.cc`中有如下代码

```
void TCPWrap::Initialize(Handle<Object> target,
                         Handle<Value> unused,
                         Handle<Context> context) {
...
  t->InstanceTemplate()->Set(String::NewFromUtf8(env->isolate(),
                                                 "onconnection"),
                             Null(env->isolate()));
...
}
```
```
void TCPWrap::Listen(const FunctionCallbackInfo<Value>& args) {
...
  int err = uv_listen(reinterpret_cast<uv_stream_t*>(&wrap->handle_),
                      backlog,
                      OnConnection);
  args.GetReturnValue().Set(err);
}
```
```
void TCPWrap::OnConnection(uv_stream_t* handle, int status) {
  ...
  tcp_wrap->MakeCallback(env->onconnection_string(), ARRAY_SIZE(argv), argv);
}
```

* `env.h`中有如下代码

```
#define PER_ISOLATE_STRING_PROPERTIES(V)                                      \
...
  V(onconnection_string, "onconnection")                                      \
...
```

* `net.js`中有如下代码

```
Server.prototype._listen2 = function(address, port, addressType, backlog, fd) {
...
  self._handle.onconnection = onconnection;
...
}
```
```
function onconnection(err, clientHandle) {
...
}
```

简单说就是`tcp_wrap.cc`在用V8定义JS对象`TCP`的时候定义了`onconnection`属性，并且在调用`uv_listen`的时候设置了`OnConnection`回调，而在`OnConnection`的最后会调用`onconnection`属性所设置的方法，最后`net.js`设置了这个属性为`onconnection`方法。

#### 请求处理响应过程

![SayHello2-1](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-11-06-the-big-pit-behind-nodejs-hello-world-2/270064-50dbaa3641fdaf4c.png)

![SayHello2-2](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-11-06-the-big-pit-behind-nodejs-hello-world-2/270064-c6f25e4c0d9564a3.png)

图中小人形状代表第三方库`deps/uv`和`deps/http_parser`。图SayHello2-1和图SayHello2-2其实是一次同步顺序调用，画不开了才分成两个，也就是说图SayHello2-1右边的小人和图SayHello2-2左边的小人其实是一个人。

仔细看，`http_parser`竟然是个同步库，再一次纠正了我先入为主的认识——Node.js全身都是异步的。Node.js以异步闻名于江湖没错，但是要加上个主语——I/O，**异步I/O**才是其必杀技。回想一下跟他过招的这么长时间里，好像确实是异步多发于I/O。

看了这两张图，欣喜的发现，“Hello World”终于脱口而出了，可喜可贺~

#### 善后过程
此时并没有完，如果将时间静止于此，命令行里应该已经显示出了“Hello world”，但是程序还没有退出。后面还要干什么呢，大概跟吃完饭要刷碗，说完话嘴不能一直张着类似。再次恢复时间的流动，又会哗哗哗的输出一大堆日志，仔细查看可知，其实是在还前面的债，统统的异步回调。前面的时序图基本都忽略了异步调用，是故意的，因为实在没有找到办法和地方把他们画出来。

还好剩下的东西不多了，而且也有一些窍门。重新查看前面两个流程的代码，把C++中调用`uv_xxx`并且设置了回调的方法记下来，把JS中`process.nextTick()`的调用记下来。V8的日志中一段完整的调用总是有以下形式。

```
  1: xxx {
  2:   xxx { 
  3:     xxx {
  .
  .
  .
  3:     } -> xxx
  2:   } -> xxx
  1: } -> xxx
```

所以把前面记录下来的方法调用和V8日志里为`1:`的段落进行对比，肯定能一一对上的，这里就不详细说明了。但是V8日志我没有收集全，最后面一段莫名其妙的就没了，不知道为什么。

# 回顾
---
回想源码分析过程，异常痛苦，只是用着[*Sublime Text*](http://www.sublimetext.com/)的基本功能，翻来翻去，跳来跳去，虽然他的基本功能已经很强大了，但是还是感觉效率低下。“工欲善其事必先利其器”，还是应该把环境弄得高效趁手一些。工具放一边，其实不能随心所欲Debug源码才是硬伤，否则就不用假装有时间机器了，也许是被Java宠坏了，落差巨大。不过我想Node.js的核心开发人员不可能像我这么费劲，所以这方面还需要进一步了解。虽然重新认识了Node.js的异步，初步了解了其幕后英雄libuv和V8的机制，体系结构更加清晰明了，但是底层细节还是需要深入研究。
