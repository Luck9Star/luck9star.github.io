---
title: 阿里云ECS与Vultr跑分对比
permalink: a-li-yun-yu-vultrpao-fen-dui-bi
id: 11
updated: '2015-08-25 12:39:09'
tags:
  - 阿里云
  - Vultr
  - VPS
categories: VPS
abbrlink: 44116
date: 2015-08-25 12:13:21
---

同为1CPU 1GRAM配置，阿里云HDD，Vultr为SSD。
价格方面阿里云ECS按照1Mbps带宽月付68，年付680，还能9折。Vultr月付则是$8，年付优惠未知，按汇率，月付大致52左右。

下面是UnixBench跑分截图，都在新装的 CentOS 7 的系统下。
### 阿里云
{% asset_img http://7xkv17.com1.z0.glb.clouddn.com/ghost/0/2d/5cbb727cfdc89c90dadec8195dcc7.jpg 阿里云UnixBench跑分 %}

分数很高1417分，而且比Linode同配置高了1倍多。

### Vultr
{% asset_img http://7xkv17.com1.z0.glb.clouddn.com/ghost/4/2d/b44a7e366f433e7a76eafd454597b.jpg Vultr UnixBench跑分 %}
1331分，略逊于阿里云，差距很小。

### 总结
Vultr是基于KVM的阿里云是基于XEN的，性能阿里云占优，价格的话，Vultr略便宜。在UnixBench跑分上两者胜过Linode一倍多了，不过Linode服务和面板做得确实很好。

阿里云最坑爹的是带宽太贵了，但是不限流量，性能也好。
还有一点算不上悲剧的是，如果要使用Docker的话，Docker启动配置必须得指定bip到一个阿里云内网没有使用的IP网段上去，因为阿里云占用了Docker默认使用的内网网段。

Vultr则是限制了流量，1G配置的话日本服务器400G，其他地区2000G，超流量后按G付费。建个小网站什么的应该还是够用的了，更何况还有$5每月的最低配（流量200G/1000G），性价比还是很高的。
Vultr的CentOS 7 默认开启了 Firewalld， iptables则未安装。使用Docker需要卸载Firewalld安装iptables。