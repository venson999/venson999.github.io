---
title: Saving Private ES
date: 2015/04/08 16:46:10
categories:
    - 技术
tags: [Node.js]
---

# 思想斗争
---
{% post_link 2015-03-09-nodejs-developmet-environment-with-vagrant-and-docker 上一篇文章 %}提到要拯救年迈的伙伴ES，但是如果只是搭建开发环境的话貌似并没有救活，只是保留了生的希望，要想让他重新焕发青春还得从基础入手——更新技术栈！这话说起来容易，可是仔细想想，更新技术栈都需要做什么呢？运行环境Node.js和Redis更新到最新版本，项目所有的依赖库更新到最新版本，保证更新后的程序正确运行。哎妈，这么多事情，还不能出错，没有两周估计做不完，然后就有点退缩了，脑子里反复盘算着两周做这事值不值？自己做事情往往就是这样，开始先习惯性的考虑一堆不利因素，然后把自己吓到了，再然后就没有然后了 。其实在想要做一件事的时候，先考虑不利因素不是问题，但是仅仅考虑不利因素而不考虑解决办法才是问题，而且有些问题并不像看上去那么困难。

及时跳出思维怪圈之后，开始冷静分析。首先是升级运行环境，好像在之前研究的基础上用Docker很容易。然后是升级依赖库，大概扫了一眼依赖库的更新履历，貌似变化不大。最后是保证程序的正确性，之前研究Node.js单元测试时，用ES做的实验，已经写了完整并且正确的测试代码。恩，第一印象确实不靠谱，差不多可以开始动手了。

# 先跑起来再说
---
用敏捷的思维来做这件事，先设定一个比较低的目标——跑起来，尽快验证其可行性，后面的事情顺水推舟逐步完善就可以了。于是选择了我认为最快的方式，在机器（Windows）上安装最新的Node.js 0.12.2，在Boot2Docker中启动最新的Redis 2.8.19容器，修改`package.json`如下。
before:
```
  "dependencies" : {
    "express" : "3.1.0",
    "redis" : "0.8.2",
    "underscore" : "1.4.4",
    "log4js" : "0.5.6"
  },
  "engines" : {
    "node" : "0.6.19"
  }
```
after:
```
  "dependencies" : {
    "express" : "4.12.0",
    "redis" : "0.12.1",
    "underscore" : "1.8.2",
    "log4js" : "0.6.22"
  },
  "engines" : {
    "node" : "0.12.2"
  },
```
由于各依赖库的接口变更很小，代码只是稍作了修改。然后抱着先跑跑试试有错再改的心态运行了程序，启动居然没有报错，简单试了几个接口竟然也都运行正常，幸福来得太快，万里长征一个跟头翻过了一大半。

# 鸟枪换炮
---
**单元测试**
2013年3月折腾Node.js单元测试，当时ES的测试依赖如下，另外用[jscoverage](https://github.com/tj/node-jscoverage)做覆盖率统计，再写个`Makefile`把相关命令封装一下，这样算是当时比较主流的做法。
```
  "devDependencies" : {
    "mocha" : "*",
    "sandboxed-module" : "*",
    "should" : "*",
    "supertest" : "*",
    "sinon" : "*"
  },
```
但是这种做法还是有一些问题的：首先对`require`的mock做的不好，当时没有特别合适的第三方库，[sandboxed-module](https://github.com/felixge/node-sandboxed-module)也是无奈之选，额外还需要写一些代码对其进行弥补。其次jscoverage不是Node.js的模块，需要单独安装，只能在linux下运行，需要`Makefile`封装其复杂的命令。于是我又在Github上翻了翻，惊喜的发现了[mockery](https://github.com/mfncooper/mockery)和[istanbul](https://github.com/gotwarlost/istanbul)可以完美解决之前的问题，而且改动非常小，于是修改依赖如下。
```
  "devDependencies" : {
    "mocha" : "2.1.0",
    "mockery" : "1.4.0",
    "should" : "3.3.2",
    "supertest" : "0.15.0",
    "sinon" : "1.12.2",
    "istanbul" : "0.3.6"
  },
```
在命令行下运行
```
set NODE_ENV=test
node_modules\.bin\istanbul cover node_modules\mocha\bin\_mocha
```
覆盖率信息生成在`coverage`目录下，通过`coverage\lcov-report\index.html`文件可直观查看。注意在Windows下istanbul和[mocha](https://github.com/mochajs/mocha)联合使用的时候，用`node_modules\.bin\_mocha`是会报错的，必须用`node_modules\mocha\bin\_mocha`，路径有少许差异，结果完全不同，因为粗心在这上浪费了些时间。单元测试和覆盖率都有了，但是运行方式很不舒服，要打那么长的命令，肯定有更好的方法。印象里好像`npm`命令能做点什么似的，于是翻阅了一下[官方文档](https://docs.npmjs.com/misc/scripts)，在`package.json`中加入如下代码。
```
  "scripts": {
    "test" : "istanbul cover node_modules/mocha/bin/_mocha"
  }
```
冗长的命令立刻简化为
```
npm test
```

**运行部署**
ES的运行部署方式一直是个问题，当时一直也没有比较好的方式，代码复制4份，用[forever](https://github.com/foreverjs/forever)分别启动做进程管理，前端用Nginx做反向代理，请求由Nginx平均分配给四个进程处理。这样做在部署的时候会非常麻烦，4份代码就意味着所有的工作都要做4次，每次项目升级都做得异常痛苦。Node.js有原生的Cluster模块，但是一直不是很成熟，所以一直还是用Nginx凑合着。Node.js v0.12对Cluster的调度算法做了修改，说是不会像以前那样出现非常不均衡的现象，应该可以一试了。[PM2](https://github.com/Unitech/PM2)早就有所了解，功能上是forever的超集，既能做进程管理又能实现Cluster，监控界面也非常漂亮，关键是不用复制4份代码了，Nginx也可以去掉了，必须试一下，修改`package.json`如下。
```
  "scripts": {
    "start" : "pm2 start app.js -i 4",
    "test" : "istanbul cover node_modules/mocha/bin/_mocha"
  }
```
启动程序只需
```
npm start
```
当初还专门为了简化forever操作写了一个shell，封装各种命令，着实费了一番功夫，现在看来还是[NPM](https://www.npmjs.com/)大法好！

程序运行起来之后想看看监控情况，于是`pm2 monit`，发现启动的4个进程在PM2的监控界面只显示一个，任务管理器里面也没有多个进程在运行的迹象，看文档也没找到相关的配置，百思不得其解。难道是Windows的问题，用Linux来试试吧，这时候再一次被Docker拯救于危难之中，直接下载Node.js的官方镜像运行就好了。结果在Linux下4个进程显示正常，哎，Windows果然不是亲生的……

# 蓦然回首
---
回顾之前的工作，预期的目标全部达成，还改进了运行部署方式，整个过程只用了2天时间，比最开始预计的两周时间快了很多。除了之前充分的单元测试做了很好的铺垫之外，最主要的还是Docker节省了大量环境搭建的时间，虽然本文对Docker只是一带而过，但确实功不可没，赞！想想这次拯救行动的前因后果，用到的东西都是零零散散学习的，但相互之间又有着种种联系，如果不学Docker就不会去研究搭建开发环境，如果不研究搭建开发环境就不会想起拯救ES，如果之前没有用ES来研究单元测就不会这么顺利的完成试验证工作，如果没有一直关注Node.js就不会想到用NPM和PM2来改善程序的运行部署方式，冥冥之中好像必然会导致今天的结果，真是有趣！

这个故事告诉我们，新东西得学啊，没准哪天就用上了，没准用上了就会改变世界，至少这次把工作方式给改变了。
