---
title: 在Docker上自动构建中文的Ghost镜像
permalink: dockerzi-dong-gou-jian
id: 3
updated: '2015-09-11 10:35:36'
tags:
  - Docker
  - Ghost
categories: Docker
abbrlink: 58299
date: 2015-08-05 15:39:36
---

{% asset_img http://7xkv17.com1.z0.glb.clouddn.com/image/1/cd/f77c151ddfafb60093de14b907eee.jpg  %}

> 这是一篇记录创建Ghost镜像过程的文章。

> Docker的镜像使用起来非常方便，可以省去很多环境的配置，甚至可以说是口袋中的系统，很方便就能部署或者迁移到一台新的服务器上。

> 当然也有一些缺点，如Mac和Window下是使用了一个Linux的虚拟机实现的。又比如Win10下无法使用这个坑爹的问题，（至少我的那台Win10系统的安装后连不上虚拟机。）这也使测试Build Dockerfile时略微不便利。

回到主题
--------

最重要的有2件事：

-	[下载中文Ghost](http://www.ghostchina.com/download/)

-	创建Dockerfile

创建这个文件时由于Docker官方已经有Ghost的支持，所以我先从官方Github上找到了Ghost对应的Dockerfile，当然这是一个图省事的方式。

```
FROM node:0.10-slim

RUN groupadd user && useradd --create-home --home-dir /home/user -g user user

RUN set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# grab gosu for easy step-down from root
RUN gpg --keyserver pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4
RUN arch="$(dpkg --print-architecture)" \
    && set -x \
    && curl -o /usr/local/bin/gosu -SL "https://github.com/tianon/gosu/releases/download/1.2/gosu-$arch" \
    && curl -o /usr/local/bin/gosu.asc -SL "https://github.com/tianon/gosu/releases/download/1.2/gosu-$arch.asc" \
    && gpg --verify /usr/local/bin/gosu.asc \
    && rm /usr/local/bin/gosu.asc \
    && chmod +x /usr/local/bin/gosu

ENV GHOST_SOURCE /usr/src/ghost
WORKDIR $GHOST_SOURCE

ENV GHOST_VERSION 0.6.4

RUN buildDeps=' \
        gcc \
        make \
        python \
        unzip \
    ' \
    && set -x \
    && apt-get update && apt-get install -y $buildDeps --no-install-recommends && rm -rf /var/lib/apt/lists/* \
    && curl -sSL "https://ghost.org/archives/ghost-${GHOST_VERSION}.zip" -o ghost.zip \
    && unzip ghost.zip \
    && npm install --production \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false -o APT::AutoRemove::SuggestsImportant=false $buildDeps \
    && rm ghost.zip \
    && npm cache clean \
    && rm -rf /tmp/npm*

ENV GHOST_CONTENT /var/lib/ghost
RUN mkdir -p "$GHOST_CONTENT" && chown -R user:user "$GHOST_CONTENT"
VOLUME $GHOST_CONTENT

COPY docker-entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 2368
CMD ["npm", "start"]
```

`具体Dockerfile语法详见[官网](https://docs.docker.com/reference/builder/)说明。`

这个Dockerfile首先用了Node 0.10这是GhostChina推荐Ghost使用的Node版本。还安装了gosu用来代替sudo命令，官方的Dockerfile中都加了这个。

这边官方是用工具更新这个Dockerfile的，使用了`GHOST_VERSION`的环境变量，官方那边版本更新只需要修改这边就可以了。下面一大段RUN的代码块，主要做了更新插件，从官网下载对应版本的Ghost、解压缩、安装、清理的工作。 接下去就是绑定内容的路径，以及拷贝执行脚本的过程。脚本中主要做了下内容路径里面文件初始化的过程。最后制定端口，然后运行。

接下来就得开始做事了
--------------------

首先必须要在Github中建立一个项目，以存放Dockerfile以及sh脚本等Dockerfile中需要使用的文件。然后再Docker官网上绑定Github账号，创建自动构建的容器，选中Github上的项目。初步的工作就完成了，每次`Trigger a Build`都会自动clone这个项目，然后执行build的过程。

我最初只是修改了其中ghost文件地址，指向到ghostchina得下载地址上去。结果在Docker那边开始build时候发现无法连接到[GhostChina](http://www.ghostchina.com/)无法下载文件。国外翻到墙内也成了一个问题。

这时我想到了Github。于是把下载下的代码新建了个项目并且传上了Github。 而Dockerfile中把下载地址指向了Github的tag的source地址。理论上的确是没问题，不过我当时没注意到source的zip文件解压出来是一个文件夹，而不是直接就是源码。这个问题坑了我一断时间，不过最后也让build正常了。生成了一个稳定的镜像。

接着我又想到
------------

为何不直接把Dockerfile也一起放到Ghost源码的项目中，可以直接对这个项目的Dockerfile Build。

这改动其实也就只是多一句`COPY ./ ${GHOST_SOURCE}/`，然后去除了下载解压一级删除ghost.zip文件的步骤的部分。 当然这种情况下GHOST_VERSION就完全没用了，因为在自动构建的时候，Docker那边会自动checkout到相应的Tag，然后Build。

最后在Docker官网上配置下Build的branch、Tag。`Trigger a Build`一下就可以慢慢等着那边完成了。有错误的话也会报error，进去也可以看到日志。

EOF
---

> 最后
>
> 感谢[Ghost中文网](http://www.ghostchina.com/)汉化以及给Ghost添加了一些适用于国内插件。
>
> 附上这个项目的[Github地址](https://github.com/Luck9Star/Ghost-Zh)
>
> 以及Docker镜像 luck9star/ghost-zh
>
> 顺便吐槽下，用七牛存来图片的话，服务器在墙外的话会各种传输失败。最后还是只能用本地文件了。
