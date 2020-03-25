---
title: Node.js developmet environment with Vagrant and Docker
date: 2015/03/19 14:57:52
categories:
    - 技术
tags: [Node.js, Docker, Vagrant]
---

# 起因
---
* 平时用到的技术比较杂，java，node.js，python等，在一台机器上配置众多环境，导致本地环境混乱，技术切换总要瞻前顾后。
* 每一种技术的环境配置都比较繁琐，时间长了容易忘记，即使做了完善的记录，重新配置也会比较耗时，对新人不友好。
* 家里用mac，公司用windows，两个系统两套环境同样的工作，环境配置成了负担。
* 一直期望实现一种方式，无论在任何地方，只需敲几行命令，多复杂的应用都能跑起来，修改调试闲庭信步，技术切换波澜不惊，还不污染本地环境。
* Docker现在这么火，怎么能不试试。

# 选型
---
**Vagrant and VM**
挺久之前朋友推荐Vagrant，当时感觉惊为天人，竟然还可以这样，随即研究了一下，简单实现了在宿主机windows下开发node.js程序，代码通过synced_folder实时同步到虚拟机linux下运行调试。遗憾的是虚拟机linux的环境完全是通过命令手动搭建的，并没有结合Puppet自动完成。其实Vagrant是完全可以实现终极目标的，没有进行下去的原因有三：一是因为要学习Puppet和一些服务器运维知识，当时感觉有点跨领域了。二是因为复杂的多VM的应用会占用比较多的资源，电脑不给力不行。三是因为懒惰！

**Boot2docker**
本来一开始打算只用Boot2docker实现，因为不想引入额外技术，于是就开始干了。一开始都很顺利，dockerfile，image，container，link，run，应用按部就班的就运行起来了。但这样离终极目标还有一段距离，没有做到out of the box。首先build和run都需要手动执行，其中的一些参数不太好记。fig到是一个现成的解决方案，可是在windows上无法运行。其次目录同步支持的不好，无法提前配置，默认同步c:/User到/c/Users之外，如果想同步其他目录需要在boot2dockerVM里执行mount命令。整个过程需要手动干预过多，对新手不友好，不完美。

**Vagrant and Docker**
其实网上有很多基于Vagrant的方案，一开始确实不想用，无奈Boot2docker不给力。通过Vagrant主要解决了两个问题，一是可以提前配置同步目录，指定代码实时同步目录，二是可以通过Vagrant配置实现fig的功能。Vagrant和Boot2docker两者都是在VirtualBox的中介linux虚拟机上运行Docker，性能差别应该不大。

# Boot2docker探路
---
**确定目标**
要研究一项技术最好的办法是用实际的项目去验证，于是我选了一个2012年用Node.js写的项目es，线上版本最后一次修改是在2013年2月。选择这个项目是因为，长时间没有升级导致技术栈版本比较低，再修改的话环境都不好搭建，而且这个项目是当年刚开始研究Node.js时写的，也算探索的一种延续，再挽救一下这个年迈的伙伴。于是乎把这次探索的目标定为用Docker一键式搭建es的开发环境。

**任务分析**
es的技术栈是，CentOS 6.3，Node.js 0.6.19，Redis 2.4.15。从DockerHub上看，这几个项目已经找不到这么低版本的镜像了，Node.js可以通过nvm安装老版本，Redis可以通过tar包安装老版本，CentOS没办法只好选择了默认的centos6（当时版本6.6），当然也可以自己做一个6.3的镜像，太麻烦只好先忍了。后面需要做的就是，定义两个dockerfile，一个基于Node.js的app，一个基于Redis的db，构建image，运行container的时候link一下就OK了。

**说干就干**
db/dockerfile
```
FROM centos:centos6
MAINTAINER guows

RUN yum -y update
RUN yum -y install gcc tcl

ADD ./redis/redis-2.4.15.tar.gz /tmp/redis-2.4.15.tar.gz
RUN cd /tmp/redis-2.4.15.tar.gz/redis-2.4.15 && \
	make && \
	make install

EXPOSE 6379
CMD ["redis-server"]
```
构建镜像
~~~
docker build -t es-db .
~~~
运行容器
~~~
docker run --name db -d es-db
~~~

在构建db的过程中有几个需要注意的地方：`ADD`命令如果添加tar文件的话，会自动解压，这个比较令人费解。`CMD`命令执行的指令必须是前台运行的，否则container启动之后，执行完`CMD`后面的指令后会自动关闭。在运行容器的时候并没有用参数`-p 6379:6379`，因为如果用link连接容器的话，是不需要暴露端口的，这样会比较安全。

app/dockerfile
~~~
FROM centos:centos6
MAINTAINER guows

RUN yum -y update
RUN yum -y install tar gcc gcc-c++ openssl-devel

RUN curl https://raw.githubusercontent.com/creationix/nvm/v0.20.0/install.sh | bash
RUN source ~/.nvm/nvm.sh && \
	nvm install v0.6.19 && \
	nvm use v0.6.19 && \
	nvm alias default v0.6.19
RUN ln -s ~/.nvm/v0.6.19/bin/node /usr/bin/node && \
	ln -s ~/.nvm/v0.6.19/bin/npm /usr/bin/npm
RUN npm config set ca=""

RUN mkdir -p /usr/local/es
ADD . /usr/local/es
WORKDIR /usr/local/es
RUN npm install

CMD ["node", "app.js"]
~~~
构建镜像
~~~
docker build -t es-app .
~~~
运行容器
~~~
docker run --name app --link db:db -p 9527:9527 -d es-app
~~~

在构建app的过程中有几个需要注意的地方：像`nvm install`这样会进行长时间的下载、编译和安装的命令，推荐单独放在一个`RUN`命令里，这样执行完成后会生成中间image进行缓存，方便dockerfile的调试，否则会浪费大量等待的时间。npm版本低，必须设置`npm config set ca=""`。app如何知道db的ip和port，由于用了link的方式连接两个容器，Docker会在app里面以`<name>_PORT_<port>_<protocol>`为前缀生成一些环境变量，这里用到的是`DB_PORT_6379_TCP_ADDR`和`DB_PORT_6379_TCP_PORT`，于是在Node.js代码里可以通过`process.env['...']`的方式取得。在宿主机的命令行里用`boot2docker ip`可以取得boot2dockerVM的ip，然后就可以通过`http://192.168.59.103:9527`来测试应用了。

**问题**
截止到目前算是有了一个阶段性成果，但是遇到了一些无法解决的问题，在前面的**选型**中已经提到。现在就好像刚爬上一座山峰，在眺望四周风景的时候发现这根本不是终点，甚至才刚刚开始，于是乎收拾收拾心情，继续爬向更高的山峰。

# Vagrant and Docker再启程
---
**确定目标**
在之前Boot2docker成果的基础上，通过Vagrant实现一键式构建，代码实时同步，开发调试轻松加愉快。

**任务分析**
这种方式网上已经有不少，这篇[Setting up a development environment using Docker and Vagrant](http://blog.zenika.com/index.php?post/2014/10/07/Setting-up-a-development-environment-using-Docker-and-Vagrant)是我参考最多的。

**照猫画虎**
tree
~~~
root   
│  DockerHostVagrantfile
│  Vagrantfile
├─app
│   │  Dockerfile
│   └─src
│          app.js
│          package.json
└─db
    │  Dockerfile
    └─redis
           redis-2.4.15.tar.gz
~~~
Vagrantfile
~~~
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'docker'
DOCKER_HOST_NAME = "dockerhost"
DOCKER_HOST_VAGRANTFILE = "./DockerHostVagrantfile"

Vagrant.configure("2") do |config|
  config.vm.define "db" do |a|
    a.vm.provider "docker" do |d|
      d.build_dir = "./db/"
      d.build_args = ["-t=es-db"]
      d.name = "es-db"
      d.vagrant_machine = "#{DOCKER_HOST_NAME}"
      d.vagrant_vagrantfile = "#{DOCKER_HOST_VAGRANTFILE}"
    end
  end

  config.vm.define "app-src" do |a|
    a.vm.provider "docker" do |d|
      d.build_dir = "./app/"
      d.build_args = ["-t=es-app"]
      d.name = "es-app-src"
      d.volumes = ["/usr/local/es-vd/app/src:/usr/local/es"]
      d.cmd = ["npm", "install"]
      d.remains_running = false
      d.vagrant_machine = "#{DOCKER_HOST_NAME}"
      d.vagrant_vagrantfile = "#{DOCKER_HOST_VAGRANTFILE}"
    end
  end

  config.vm.define "app" do |a|
    a.vm.provider "docker" do |d|
      d.image = "es-app"
      d.create_args = ["--volumes-from=es-app-src"]
      d.name = "es-app"
      d.ports = ["9527:9527"]
      d.link("es-db:db")
      d.cmd = ["node", "/usr/local/es/app.js"]
      d.vagrant_machine = "#{DOCKER_HOST_NAME}"
      d.vagrant_vagrantfile = "#{DOCKER_HOST_VAGRANTFILE}"
    end
  end
end
~~~
DockerHostVagrantfile
~~~
Vagrant.configure("2") do |config|
  config.vm.provision "docker"
  config.vm.provision "shell", inline:
    "ps aux | grep 'sshd:' | awk '{print $2}' | xargs kill"

  config.vm.define "dockerhost"
  config.vm.box = "ubuntu/trusty64"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.synced_folder ".", "/usr/local/es-vd"
  config.vm.network "forwarded_port", guest: 9527, host: 9527

  config.vm.provider :virtualbox do |vb|
    vb.name = "dockerhost"
  end
end
~~~
db/dockerfile
~~~
FROM centos:centos6
MAINTAINER guows

RUN yum -y update
RUN yum -y install gcc tcl
ADD ./redis/redis-2.4.15.tar.gz /tmp/redis-2.4.15.tar.gz
RUN cd /tmp/redis-2.4.15.tar.gz/redis-2.4.15 && \
	make && \
	make install

EXPOSE 6379
CMD ["redis-server"]
~~~
app/dockerfile
~~~
FROM centos:centos6
MAINTAINER guows

RUN yum -y update
RUN yum -y install tar gcc gcc-c++ openssl-devel
RUN curl https://raw.githubusercontent.com/creationix/nvm/v0.20.0/install.sh | bash
RUN source ~/.nvm/nvm.sh && \
	nvm install v0.6.19 && \
	nvm use v0.6.19 && \
	nvm alias default v0.6.19
RUN ln -s ~/.nvm/v0.6.19/bin/node /usr/bin/node && \
	ln -s ~/.nvm/v0.6.19/bin/npm /usr/bin/npm
RUN npm config set ca=""

RUN mkdir -p /usr/local/es
WORKDIR /usr/local/es
~~~

这里面有一个让我纠结很久的问题，image里面应不应该放代码，因为我要做的是开发环境，代码和依赖都会随时变化，把一些必然会变的东西固化到image里面让人很不爽。代码和依赖不放在image里面的话，就必须从dockerfile里面分离出来，这样就涉及到代码放在哪依赖何时安装的问题。代码还好说，用`synced_folder`和volume可以搞定。但是依赖何时安装呢？起初想把`npm install && node app.js`放在`docker run`后面，但是不成功，npm好像把后面的内容都当成了参数对待，无法安装。当然可以写一个shell，把好多东西都写在里面，但是这样又赋予了`docker run`太多的职责，而且每次启动都要重复执行一些命令，感觉与docker的思想相违背。解决这个问题的灵感来自于我之前提到的[那篇文章](http://blog.zenika.com/index.php?post/2014/10/07/Setting-up-a-development-environment-using-Docker-and-Vagrant)和[官方文档](https://docs.docker.com/userguide/dockervolumes/#creating-and-mounting-a-data-volume-container)，首先创建一个名为`es-app-src`的container专门放置代码和依赖，通过`-v`同步代码，在此之上执行`npm install`，把代码和依赖都存在了`es-app-src`的volume里，再创建一个名为`es-app`的container通过`--volumes-from`挂载`es-app-src`的volume。

在实际操作过程中遇到三个问题：一是执行`vagrant up`时需要加上`--no-parallel`，因为Vagrant默认是并行执行的，但是由于需要`--link`，也就是说必须db起来了之后才能被app来link，所以必须顺序执行。二是`npm install`报错`Error: UNKNOW , symlink '...'`，因为`npm install`需要做`symlink`，同时通过`synced_folder`同步给宿主机，宿主机windows默认情况下是不允许`symlink`的，需要管理员权限，所以需要以管理员权限启动命令行。网上还有一种修改Vagrant配置的方案，我没有成功。三是由于`es-app-src`执行完必要的任务就会关闭，所以必须配置`remains_running = false`来告诉Vagrant这是正常现象，否则会报错。

通过以上步骤，只需一条`vagrant up --no-parallel`命令，然后稍作等待（时间取决于网络条件和机器配置），应用从无到有在本地就跑起来了，so easy！

**开发调试**
应用跑起来还不算完，开发调试怎么办，修改代码需要重启应用，Docker可没有这个功能，最后这一点实现不了，前面一切都是白搭。Node.js有一些第三方的进程管理库可以解决这个问题，能够实时监控代码变化并且实时重启应用，如[forever](https://github.com/foreverjs/forever)、[PM2](https://github.com/Unitech/PM2)、[nodemon](https://github.com/remy/nodemon)，但是只有forever可以支持Node.js 0.6.19，奈何那重启速度实在让人无法直视，谁让咱版本低呢。

package.json
~~~
  "dependencies" : {
    ...
    "forever" : "0.9.2"
  },
  "scripts": {
    "start" : "forever -w app.js"
  }
~~~
Vagrantfile
~~~
  config.vm.define "app" do |a|
    a.vm.provider "docker" do |d|
      d.image = "es-app"
      d.create_args = ["--volumes-from=es-app-src"]
      d.name = "es-app"
      d.ports = ["9527:9527"]
      d.link("es-db:db")
      d.cmd = ["npm", "start"]
      d.vagrant_machine = "#{DOCKER_HOST_NAME}"
      d.vagrant_vagrantfile = "#{DOCKER_HOST_VAGRANTFILE}"
    end
  end
~~~

**问题**
现在的解决方案也不够完美，Vagrant和Docker配合起来总是有些奇怪的问题，网上资料不多，整个调试过程非常痛苦，而且和Fig相比也明显不够优雅，不过应该已经算是现阶段的最优解了，期待后续改进吧。

# 畅想未来
---
通过这次实践，对Docker的认识更进一步，算是从认知到入门了吧，着实颠覆了一下我的观念，太方便了，真的可以让开发调试轻松加愉快。而且Docker的能力绝对不止于此，开发、测试、部署、交付等等，软件生命周期的各个阶段都可以找到用武之地，想象空间无限，效率提升无限，期待把他应用到更多的工作场景中去！
