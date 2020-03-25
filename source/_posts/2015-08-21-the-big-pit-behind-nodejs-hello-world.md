---
title: Node.js之HelloWorld背后的大坑
date: 2015/08/21 17:08:07
categories:
    - 技术
tags: [Node.js]
---

# 入坑
---
先贴一段代码，再熟悉不过，她默默的待在Node.js官方首页上已经不知多长时间，迎接着初入Node.js世界的程序员们，所有人都认识她，但并非所有人都了解她，甚至很多人都没有想过要去了解她。
```
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(1337, '127.0.0.1');
console.log('Server running at http://127.0.0.1:1337/');
```
```
% node example.js
Server running at http://127.0.0.1:1337/
```
我也是很多认识但不了解她的人们的其中之一，为什么我会想要去了解她呢，其实事情是这样的。之前用PM2设置集群的时候很容易，一条命令就搞定了，当时感觉很神奇，于是便看了下官网上关于集群的文档，但是Sample代码实在非常怪异，完全不是正常的思路，便越发想了解一下Node.js的集群机制到底是如何实现的，看源码吧，遂入坑，卒！先看`cluster.js`，然后看`child_process.js`，最后看`net.js`，为什么最后是`net.js`，因为实在看不下去了，各种异步，东一榔头西一棒子，完全理不清头绪。总结一下失败原因，首先并不了解Node.js代码结构、实现方式和内部机制，不适应异步逻辑，一直以顺序的思路去看代码，导致很多地方看不明白。其次对JavaScript其实并不熟悉，尤其这种大型项目，一些高级特性的使用，所谓的对象和继承等等，相比自己之前写的那些业务代码，完全不属于一个次元。然后就是轻敌，单枪匹马杀入乱军之中，没有赵子龙那样的本事，不卒才怪。于是痛定思痛，还是从头开始学吧！

# 体系架构
---
Node.js主要分为四大部分，Node Standard Library，Node Bindings，V8，Libuv，架构图如下。
![Node.js Structure Overview](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-08-21-the-big-pit-behind-nodejs-hello-world/270064-223f568f00926ecd.png)
* Node Standard Library 是我们每天都在用的标准库，如`require('http')`，官方的API文档说的就是他。
* Node Bindings 是沟通上下层的桥梁，封装V8和Libuv的细节，向上层提供基础功能。
* V8 是Google开发的JavaScript引擎，提供JavaScript运行环境，可以说没有他就没有Node.js。
* Libuv 是专门为Node.js开发的一个封装库，提供跨平台的异步I/O能力。

# 代码结构
---
以下是代码的简易结构，已经囊括了Node.js的四大部分，对于入门来说已经足够了，并且本文分析的绝大部分代码都在lib和src下面。另外，本文是基于v0.12.7版本进行的代码分析，网上也有一些老版本的分析，好像完全说的不是一回事，由于一时没注意带来了很多困扰，特此说明。
```
node   
├─deps
│   ├─uv
│   └─v8
├─lib (Node Standard Library)
└─src (Node Bindings)
```

# 特别声明
---
后文只着重描述了看代码的思路，并没有进行过多的说明，一是表达能力有限，二是感觉任何的说明都显得苍白无力，所以光看文章不看代码是不行的！

下面以Linus Torvalds的一句名言来开启Node.js的源码之旅。

>Talk is cheap, show me the code.

Let's go！

# 起步停车
---
本来我刚开始分析的是第二句代码`http.createServer(...).listen(...);`，因为这句最长，一看就是重点嘛，但是分析完之后才发现，在这之前Node.js还做了好多好多事情，这才只是冰山一角，还是需要从真正的起跑线开始。
```
% node example.js
```
这句话有啥可分析的，一开始确实是这样想的，本来认为可以轻松越过的，谁知道刚起步就停了下来，真的没有那么简单，如下图所示。
![Node.js Startup](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-08-21-the-big-pit-behind-nodejs-hello-world/270064-35804ebf8addbe9f.png)
首先声明这不是正规的时序图，只是为了更好的理解代码才画成了时序图的样子，来描述整个代码的调用过程。上图描述了从`command line`到进入`example.js`之前的程序调用过程，属于整个HelloWorld程序的起步阶段。要想了解进入主程序之前Node.js都干了什么，细读这部分代码就可以了，尤其是`node.cc`、`node.js`和`module.js`。其中`node_Contextify.cc`中有很多关于V8的调用，暂时不在本文的讨论范围，有兴趣的可以了解一下。

# Node Bindings
---
其实上一节还留有一个疑点，上图右边第三个`Script`是怎么来的，是如何与C++代码联系上的？接着看代码！

 * `vm.js`中有如下调用，`process.binding`干了什么？

```
var binding = process.binding('contextify');
var Script = binding.ContextifyScript;
```

 * `node.cc`的`SetupProcessObject`中有如下设置，将`process.binding`与`Binding`进行绑定，`Binding`干了什么？

```
NODE_SET_METHOD(process, "binding", Binding);
```

 * `node.cc`的`Binding`中有如下调用，对模块进行注册，`nm_context_register_func`干了什么？

```
mod->nm_context_register_func(exports, unused, env->context(), mod->nm_priv);
```

 * `node.h`中对`mod`的类型`node_module`有如下定义，往下看！

```
struct node_module {
  int nm_version;
  unsigned int nm_flags;
  void* nm_dso_handle;
  const char* nm_filename;
  node::addon_register_func nm_register_func;
  node::addon_context_register_func nm_context_register_func;
  const char* nm_modname;
  void* nm_priv;
  struct node_module* nm_link;
};
```

 * `node.h`中还有如下宏定义，接着往下看！

```
#define NODE_MODULE_CONTEXT_AWARE_X(modname, regfunc, priv, flags)    \
  extern "C" {                                                        \
    static node::node_module _module =                                \
    {                                                                 \
      NODE_MODULE_VERSION,                                            \
      flags,                                                          \
      NULL,                                                           \
      __FILE__,                                                       \
      NULL,                                                           \
      (node::addon_context_register_func) (regfunc),                  \
      NODE_STRINGIFY(modname),                                        \
      priv,                                                           \
      NULL                                                            \
    };                                                                \
    NODE_C_CTOR(_register_ ## modname) {                              \
      node_module_register(&_module);                                 \
    }                                                                 \
  }

#define NODE_MODULE_CONTEXT_AWARE_BUILTIN(modname, regfunc)           \
  NODE_MODULE_CONTEXT_AWARE_X(modname, regfunc, NULL, NM_F_BUILTIN)   \
```
 * `node_contextify.cc`中有如下宏调用，终于看清楚了！结合前面几点，实际上就是把`node_module`的`nm_context_register_func`与`node::InitContextify`进行了绑定。

```
NODE_MODULE_CONTEXT_AWARE_BUILTIN(contextify, node::InitContextify);
```

兜了这么大一个圈子，省略去中间步骤，代码对应如下，Node.js就是如此完成了Node Bindings。
```
process.binding('contextify'); 
↓↓↓
NODE_MODULE_CONTEXT_AWARE_BUILTIN(contextify, node::InitContextify);
```

# 步入正轨一
---
说了这么多终于到第一句代码了，再不到就要放弃了，赶快来看看吧。
```
var http = require('http');
```

`require`是怎么来的，为什么平白无故就能用呢，实际上都干了些什么？

* `module.js`的`_compile`中有如下代码。

```
var self = this;
...
function require(path) {
  return self.require(path);
}
...
var wrapper = Module.wrap(content);
...
var compiledWrapper = runInThisContext(wrapper, { filename: filename });
...
var args = [self.exports, require, self, filename, dirname];
return compiledWrapper.apply(self.exports, args);
```

* `Module`的`require`有如下定义。

```
Module.prototype.require = function(path) {
  assert(path, 'missing path');
  assert(util.isString(path), 'path must be a string');
  return Module._load(path, this);
};
```

* `Module`的`wrap`有如下定义。

```
Module.wrap = NativeModule.wrap;
```

* `node.js`中`NavtiveModule`有如下定义。

```
NativeModule.wrap = function(script) {
  return NativeModule.wrapper[0] + script + NativeModule.wrapper[1];
};

NativeModule.wrapper = [
  '(function (exports, require, module, __filename, __dirname) { ',
  '\n});'
];
```

不用多解释了，代码已经说明了一切。

# 步入正轨二
---
正餐开始，不过感觉前面的开胃菜似乎有点多……
```
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(1337, '127.0.0.1');
```

一如既往，看图说话。

* 首先了解一下HTTP Server的继承关系，有利于更好的理解代码。

![HTTP Server  Inheritance](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-08-21-the-big-pit-behind-nodejs-hello-world/270064-edbf9b53812f0433.png)

* 然后就是HTTP Server的工作流程。

![HTTP Server Workflow](http://venson-blog-images.oss-cn-beijing.aliyuncs.com/2015-08-21-the-big-pit-behind-nodejs-hello-world/270064-9170d92558870ee2.png)

通过上图可以看出，绝大部分逻辑都在`net.js`中，细读这部分代码可以更好的了解其工作原理。其中`tcp_warp.cc`中有很多关于Libuv的调用，暂时不在本文的讨论范围，有兴趣的可以了解一下。

# 步入正轨三
---
最后一句了，挺住！
```
console.log('Server running at http://127.0.0.1:1337/');
```

`console`和`require`还不一样，不是以参数的形式传进来的，这就要说到`global`对象了，Node.js的顶层对象。官方文档已经有了相关的说明，在这就不多做解释，重点看看他是怎么来的。

* `node.js`中有如下定义，这个`this`到底是谁？

```
this.global = this;
...
startup.globalVariables = function() {
  global.process = process;
  global.global = global;
  global.GLOBAL = global;
  global.root = global;
  global.Buffer = NativeModule.require('buffer').Buffer;
  process.domain = null;
  process._exiting = false;
};
...
startup.globalConsole = function() {
  global.__defineGetter__('console', function() {
    return NativeModule.require('console');
  });
};
```

* `node.cc`中的`LoadEnvironment`有如下定义，`f`代表`node.js`所形成的方法，`Call`跟JavaScript中的`Function.prototype.call`是一个意思，也就是说`f`中的`this`指向的就是`global`。

```
Local<Object> global = env->context()->Global();
Local<Value> arg = env->process_object();
f->Call(global, 1, &arg);
```

这样`console`作为全局变量的身份也就真相大白了。

# 大功告成？
---
所有代码都分析完了，“Hello World”这两个字竟然还没有出现！？
这是段服务端程序，没有请求，哪来的应答！？
哎，你要是长这样该多好……
```
console.log('Hello World');
```

即使没完，也准备告一段落了。给有缘人一个探索的空间？No！No！No！累了，需要恢复元气！
如果确实非常想知道后事如何，那便在此留下一些线索，以供参考。

其实在看代码的过程中，Server响应请求的过程更加令人匪夷所思，卡了好久，还好找到了比较好的办法才算弄清楚，那就是看！日！志！其实看日志根本不算什么办法，地球人都知道，但是怎么让日志打出来，还真费了半天功夫，反正百度上是没找到。

* V8日志

```
% node --trace example.js
```
* 源码Debug日志

```
% NODE_DEBUG=HTTP,STREAM,MODULE,NET node example.js
```

关于V8的一些参数可以通过`node --v8-options`查看，`--trace`的作用是输出方法调用过程。源码Debug日志的原理可以查看`util.js`的`debuglog`方法。这些日志都比较长，最好输出到文件中以便反复查看。

# 感想
---
别总以为什么都知道，其实可能连最基本的都不知道！
知道的越多，就越觉得无知！
唯有学习！
