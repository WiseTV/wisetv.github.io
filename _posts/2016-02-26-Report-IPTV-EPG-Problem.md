---
author: 谷悦
comments: true
date: 2016-02-26 9:15:10+00:00
layout: post
title: 2016年2月13日：数据库负载压力问题说明
description: Report IPTV EPG Problem
categories:
- CMS
---
2016年2月13日至2月14日期间，IPTV EPG平台在晚间高锋出现页面无响应故障。以下是本次故障的详细报告
<!-- more -->
###事件描述###
2016年2月13日晚20:30左右，EPG现网首页、详细页点播播放及我的IPTV出现了长时间无响应的故障。我们接到反馈后紧急调查问题，先后查看了前端动态服务器、数据库的负载及运行情况并联系运维部同事排除了网络环境故障的可能性。继续尝试切换数据库节点，重启动态服务，并临时修改了日志信息中反馈的bug。至22:30左右，系统恢复正常，此时没有找到真实原因，并不能断定此次系统是通过修改bug恢复，还是高峰期过后负载下降自动恢复。

2016年2月14日晚上20:50左右，现网再次出现无响应故障。我们启动预案临时关闭了记忆播放功能，使用户详细页播放及首页刷新不受影响。此时进一步调查分析问题，通过对mongodb状态参数的监控发现数据锁很高，在加上对nginx日志的分析基本上断定了是由于节后活跃用户的增加导致数据库性能支撑不住。23:00左右，系统全部恢复正常。

2016年2月15日，经数据库和动态服务的升级，未出现无响应故障。

故障修复之后，我们立即召集所有相关工程师进行了问题总结和技术讨论，评估可行措施来防止类似问题再次发生。

###故障时间###
	
- 2016年2月13日下午 8:30 至 23:00
- 2016年2月14日下午 8:50 至 23:00

###故障现象###

- EPG首页，详细页播放，我的IPTV等页面无响应
- 全部六台nodejs服务器负载正常，nodejs服务部分接口也正常响应。nginx反向代理大量报错，集中表象为播放记录相关查询请求响应超时。
- 数据库服务器负载在2-6左右，mongostat监控结果中，数据库锁占比超过10%高锋时达到50%。(平时正常为5-7%)
- 高峰期数据库主节点cms.playhistories reads秒并发为1000+,writes  秒并发为800+。（平时正常为reads:700+,writes:500+）其余集合读写并发均正常。
- 高峰期间数据库日志查询响应由平均200ms增加至3000ms以上。

###影响范围###

- 联通分发域中兴平台
- 联通分发域华为平台
 
###故障原因###

服务器方面：

	为应对春节期间用户访问高锋，解决之前高峰期nodejs动态服务负载压力过高的性能瓶颈。
	EPG项目组进行了nodejs动态服务器的扩容，新增加了4台nodejs服务器，但是没有扩容升级mongodb数据库。
	这使得数据库服务成为了整个系统新的性能瓶颈，事实上13-14日的访问无响应故障，就是大量读写操作导致数据库不堪重负引发的。

数据库方面：

	EPG mongodb现网数据版本为2.0.6，该版本的锁机制为全局锁，当数据插入、更新时，整个数据库的其他操作都处于等待状态。

	EPG mongodb数据库故障前采用的是双节点副本集，并没有在nodejs服务端进行读写分离的处理，所有服务读取压力都在主节点。
	这为访问高峰期系统的稳定性留下了隐患。

	EPG mongodb数据库索引新建形式采用的是前台索引方式，当数据插入时即刻建立索引。
	这会导致插入操作时间延长，在高并发插入时，数据库锁定时间居高不下，导致所有其他请求时间也响应延长。

nodejs服务方面：

	故障前动态服务器nodejs版本为0.6.19，nodejs服务采用的mongoose为2.7.2。
	旧版本mongoose不支持也没开启从节点读取功能，导致所有读写压力都集中在数据库主节点，从节点则一直处于空闲状态。

EPG业务方面：

	观察统计EPG动态服务各接口高峰期调用量，会发现所有压力都集中在点播播放记录的更新和读取中。
	这是因为EPG详细页、我的IPTV、首页均存在相关接口的调用。而13-14号晚高峰期间，也正是用户使用点播业务高峰期。

###紧急处理###
我们于2月14日开始调查分析故障原因，一方面开始服务升级的准备工作，一方面制定了应急预案，包括在发生故障时临时去掉首页及详细页获取播放记录的代码，临时关闭播放记录插入接口，使故障不影响用户首页刷新及基本的点播播放功能。

###后续措施###
针对故障原因，我们于2月15日对node服务及数据库进行了多项升级调整。

- 为nodejs服务添加从节点读取功能，使数据库从节点分担主节点的访问压力，提高数据库查询性能。包括升级服务器node版本升级至5.2。
mongoose版本升级至4.4.3，并在新版node上重新适配测试多个现网node服务等。
- 为mongodb增加一台新的从节点服务器，提升数据库查询性能。
- 优化播放记录数据库索引，将索引建立时机改为后台建立，减少数据插入耗时，减小数据库锁定的时间占比，提升数据库读写性能。

###事故反思与如何避免###
                    
- 完善监控机制，监控服务返回值的正确性，并确定一系列安全阀值，例如当数据锁达到百分之多少的时候发出警报等。
- 完善风险预案，通过一系列限流和降级的原则，做到服务异常对用户无感知。
- 注意加强软件版本迭代意识，关注新版本，新特性以及对功能优化，避免一味开发新功能而不注重现网优化工作。

最后，我们再次对受影响的用户表示道歉。我们会尽一切努力，保证IPTV EPG平台的稳定运行，不辜负大家对我们的信任！
