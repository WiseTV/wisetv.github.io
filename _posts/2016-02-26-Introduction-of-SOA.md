---
author: 李喆
comments: true
date: 2016-02-26 9:15:10+00:00
layout: post
title: SOA服务化解决方案探索
description: Introduction of SOA
categories:
- SOA
---

#一、发现问题  
各个系统独立存在，需要获取其他系统的数据，获取方式不统一，难于维护，难于应对变化，一旦有变化处理不好可能是灾难性的。  

<!-- more -->

####实例1：  
数据系统需要获取CMS系统的数据，直接在数据系统的数据库中创建dblink访问CMS系统的数据库，程序刚写好运行的很好，没有问题，过一段时间后，CMS系统由于一些需求需要改变数据系统调用相关的表结构，由于没有通知数据系统做相应的修改，上线后直接导致数据系统的相关功能崩溃。 

对于上面的问题，CMS系统的开发人员除本系统外也不知道有谁使用了相关的表，更别说要通知谁去做相应修改。对于数据系统的开发人员根本不知道CMS系统的修改，系统功能莫名其妙的就挂掉了，责任在谁，如何避免？

####实例2：
有些功能明明在一些系统中已经存在，但由于沟通或者一些其他问题，需要使用这些功能的其他系统又重新开发了一遍相同的功能，而不是把相同或相似的功能抽象出来共同使用，造成代码上的冗余，增加了维护成本。（如 红包：手机端、IPTV端，节目单：数据组、CMS组）

####实例3：  
为了便于各系统交互，抽象出需要调用的功能，封装成开发包（库），提供给其他系统使用，各种开发包口耳相传，有一天开发包的开发者对其做了修改，开发者没有办法知道所有的开发包使用者，从而没有办法通知到所有的使用者，最终导致未更新开发包的程序出错崩溃。  

对上面这个例子，虽然将要调用的功能抽象了出来，但开发包内部的改动没有做到对调用者透明，开发包的任何改动都要求调用者不厌其烦的更新，才能保证系统运行正常。这个问题使我想到了C/S架构的软件模式，开发容易，更新和部署困难，有些C/S架构软件甚至会限制不更新就不让使用，但作为开发包，这样做显然不现实。  

综上几个例子发现我们的系统存在以下几个问题：  

	1. 接口没有统一管理  
	虽然有些功能对外提供了一些接口，但没有统一管理，时间一长，接口变得不可控。  
	2. 很多组件无法复用，即重复造轮子  
	组件无法复用，造成功能的冗余，总体维护成本加大。  
	3. 模块间职责不清，耦合过深  
	有些模块功能重叠，职责不清。直接访问数据库而不通过接口，增大了功能之间的耦合度。  
	4. 联调排查问题比较缓慢。  
	系统耦合增大，牵一发动全身，一旦出现问题排查困难。  
	5. 缺乏API规范，文档较少。  
	有了接口，还应该规范文档，让使用者需要使用的时候有据可查。  
	6. 开发前规划不足  
	种种原因导致没有做好足够的需求分析以及调研，就开始开发，设计问题导致扩展困难。  

#二、分析问题
##1、接口没有统一管理  
没有把接口管理提升到一定的高度，往往是头痛医头脚痛医脚，需要的时候再临时确定怎么调用。需要有一个统一的位置去管理、注册各种接口，统一调用方式。  

##2、很多组件无法复用，即重复造轮子  
考虑组件无法复用主要可能有由于以下几个问题产生：  

（1）根本不知道有这个已经可用的组件，直接重写开发，导致重复开发。  
  
（2）原有功能组件划分不合理，即粒度不适合，此时便面临一个棘手的问题，是重构原有代码，还是重新开发。  
  
应该有一个统一的位置可以查询相关的组件接口，如果有合适的组件直接使用，没有合适的再重新开发。加强开发前的规划和调研。  

##3、模块间职责不清，耦合过深  

分清职责，提取公共功能。  

##4、联调排查问题比较缓慢。  

加强单元测试和持续集成，避免修改后对接口层出现影响。  

##5、缺乏API规范，文档较少。  

确定API规范，统一接口，生成api文档，便于方法的查询和使用。

##6、开发前规划不足。  

加强开发前对需求整体的调研和规划。  

综上问题看出，应该有一个类似总线的系统，各个系统对外发布的功能以规范的方式发布到这个总线上，同理，当需要从这个总线上获取数据或调用功能时，也能够以一个统一的规范的方式使用这些数据或功能。这个系统应该能够记录每个系统对外发布的接口信息，并提供不间断的服务，以备使用者随时能够使用。作为一个多个系统都依赖的总线系统，这个总线系统还应该是高可用的，多机热备，避免因为单个机器产生的问题造成所有系统的瘫痪。最好能够支持负载均衡，根据实际情况动态决定由负载较小的server提供服务。

#三、解决问题  
既然分析了问题所在，我们就应该看看到底有没有一个类似这样的一个总线系统适合我们。  

通过了解，我们发现SOA思想与我们想要的这个总线的思路很贴合。  
##1、SOA(Service-Oriented Architecture)  
面向服务的体系结构是一个组件模型，它将应用程序的不同功能单元（称为服务）通过这些服务之间定义良好的接口和契约联系起来。接口独立于实现服务的硬件平台、操作系统和编程语言。使构建在各种各样的系统中的服务可以使用一种统一和通用的方式进行交互。   

![](http://i12.tietuku.com/a4d8a4f7c21aa8e7.jpg)  

上图为一个简单的SOA的模型，实线为实际的调用，虚线为为实际调用服务的准备工作，也是SOA框架需要完成的内容，可见SOA框架首先要解决的是如何实现应用之间的远程调用（RPC）以及如何优化这种调用达到负载均衡。  
###（1）基于HTTP协议的RPC  
RPC不局限于HTTP协议，也可以基于其他协议实现如TCP协议，但基于TCP协议实现RPC，程序员要考虑多线程并发、锁、I/O、等复杂的底层实现细节，实现起来较为复杂。在大流量高并发压力下，任意一个细小的错误，都会被无限放大、最终导致程序宕机。而对于基于HTTP协议的实现来说，很多成熟的开源Web容器已经帮其处理好了这些事情，开发人员可以将更多的经历集中在业务的实现上，而非处理底层细节。  

###（2）服务器的路由和负载均衡  
上图中通过虚线部分交互以及路由最终将实际调用Server的信息返回给调用者，由调用者发起请求。  
负载均衡算法：
####1)	轮询法  
顺序轮询每一台服务器。
####2)	随机法
随机访问每一台服务器，等效于轮序。  
####3)	加权轮询
给配置负载低的机器更大的权重，从而能够使其处理更多的请求。
####4)	加权随机
与加权轮询法类似，不同的是按照权重来随机选择服务器。
####5)	最小连接数
根据后台服务器当前链接情况，动态地选取其中当前积压连接数最少的一台服务器来处理当前请求，尽可能地提高后端服务器的利用效率，将负载合理地分流到每一台机器。  
###（3）统计、限流、授权、报警
##2、DUBBO
DUBBO是一个分布式服务框架，提供高性能和透明化的RPC远程服务调用方案，是阿里巴巴SOA服务化治理方案的核心框架。
实现透明化的远程方法调用，就像调用本地方法一样调用远程方法，只需简单配置，没有任何API侵入。软负载均衡及容错机制，可在内网替代F5等硬件负载均衡器，降低成本。服务自动注册与发现，不再需要写死服务提供方地址，注册中心基于接口名查询服务提供者的IP地址，并且能够平滑添加或删除服务提供者。  
###1、节点和角色
![](http://dubbo.io/dubbo-architecture.jpg-version=1&modificationDate=1330892870000.jpg)  

* Provider: 暴露服务的服务提供方  
* Consumer: 调用远程服务的服务消费方  
* Registry: 服务注册与发现的注册中心  
* Monitor: 统计服务的调用次调和调用时间的监控中心  
* Container: 服务运行容器  

###2、调用关系说明  
（0） 服务容器负责启动，加载，运行服务提供者。  
（1） 服务提供者在启动时，向注册中心注册自己提供的服务。  
（2） 服务消费者在启动时，向注册中心订阅自己所需的服务。  
（3） 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。  
（4） 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。  
（5） 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。  
  
###3、关于注册中心zookeeper
官方推荐使用zookeeper作为注册中心。  

zookeeper其实是一个树形的目录服务器，引用官方的说法：“Zookeeper是一个高性能，分布式的，开源分布式应用协调服务。它提供了简单原始的功能，分布式应用可以基于它实现更高级 的服务，比如同步，配置管理，集群管理，命名空间。它被设计为易于编程，使用文件系统目录树作为数据模型。服务端跑在java上，提供java和C的客户端 API”。  

##1、总体结构
Zookeeper服务自身组成一个集群(2n+1个服务允许n个失效)。Zookeeper服务有两个角色，一个是leader，负责写服务和数据同步，剩下的是follower，提供读服务，leader失效后会在follower中重新选举新的leader。  

![](http://static.oschina.net/uploads/img/201212/25151024_hgTU.jpg)  

* 客户端可以连接到每个server，每个server的数据完全相同。
* 每个follower都和leader有连接，接受leader的数据更新操作。  
* Server记录事务日志和快照到持久存储。  
* 大多数server可用，整体服务就可用。  

##2、dubbo应用下的zookeeper数据模型  

![](http://dubbo.io/zookeeper.jpg-version=1&modificationDate=1323255359000.jpg)  

流程说明：  

	服务提供者启动时  

	向/dubbo/com.foo.BarService/providers目录下写入自己的URL地址。  

	服务消费者启动时  

	订阅/dubbo/com.foo.BarService/providers目录下的提供者URL地址。  

	并向/dubbo/com.foo.BarService/consumers目录下写入自己的URL地址。  

	监控中心启动时  

	订阅/dubbo/com.foo.BarService目录下的所有提供者和消费者URL地址。 

##四、安装
###1.注册中心（zookeeper） 

拷贝zookeeper-3.4.6.tar.gz到合适的目录中,如 /usr/local/zookeeper  

安装：  

	tar zxvf zookeeper-3.4.6.tar.gz  
	cd zookeeper-3.4.6  
	cp conf/zoo_sample.cfg conf/zoo.cfg //将配置模板文件复制出来重命名为正式配置文件  

配置：  

	vi conf/zoo.cfg  

非集群情况下（主要设置dataDir为实际的输出目录，我这里设置为/usr/local/zookeeper/zookeeper-3.4.6/data）   

	tickTime=2000
	initLimit=10
	syncLimit=5
	dataDir=/usr/local/zookeeper/zookeeper-3.4.6/data
	clientPort=2181

集群情况下可以设置为   

	tickTime=2000  
	initLimit=10  
	syncLimit=5  
	dataDir=/usr/local/zookeeper/zookeeper-3.4.6/data  
	clientPort=2181  
	server.1=192.168.17.53:2555:3555  
	server.2=192.168.17.54:2555:3555 //假设第二台机器ip为192.168.17.54		


并在data目录下放置myid文件：(上面zoo.cfg中的dataDir)  

	mkdir data
	vi myid

myid指明自己的id，对应上面zoo.cfg中server.后的数字，第一台的内容为1，第二台的内容为2，内容如下：  

	1

启动：  
进入/usr/local/zookeeper/zookeeper-3.4.6/bin  

	./zkServer.sh  start

停止：  
进入/usr/local/zookeeper/zookeeper-3.4.6/bin  

	./bin/zkServer.sh  stop  

###2、简易监控中心（dubbo-monitor-simple）  

拷贝dubbo-monitor-simple-2.5.3-assembly.tar.gz到合适的目录中,如 /usr/local/dubbo

安装：  

	tar zxvf dubbo-monitor-simple-2.5.3-assembly.tar.gz
	cd dubbo-monitor-simple-2.5.3  

配置：  

	vi conf/dubbo.properties

启动：  

	./bin/start.sh

停止：  

	./bin/stop.sh

重启：  

	./bin/restart.sh

###3、管理控制台（dubbo-admin）  

管理控制台其实就是一个web应用，需要容器支持，如tomcat  

如果没有tomcat需要先安装tomcat  

我这里安装了apache-tomcat-7.0.65.tar.gz  

拷贝apache-tomcat-7.0.65.tar.gz到合适的目录中,如 /usr/local/tomcat  

安装：  

	tar zxvf apache-tomcat-6.0.65.tar.gz
	cd apache-tomcat-7.0.65/webapps/  

创建目录便于存放dubbo-admin程序  

	mkdir dubboadmin  

拷贝dubbo-admin-2.5.3.war到合适的目录中,如 /usr/local/dubbo  

解压war包到webapps下合适的目录，上面创建的目录dubboadmin  

	unzip dubbo-admin-2.5.3.war -d /usr/local/tomcat/apache-tomcat-7.0.65/webapps/dubboadmin

配置：  

	vi webapps/dubboadmin/WEB-INF/dubbo.properties  

	dubbo.registry.address=zookeeper://127.0.0.1:2181
此处可设置默认两个用户的密码  

用户名root 密码root  

	dubbo.admin.root.password=root 
用户名guest密码guest  

	dubbo.admin.guest.password=guest

设置完成后启动tomcat  

	cd /usr/local/tomcat/apache-tomcat-7.0.65/
	./bin/startup.sh

访问地址http:// 192.168.17.53:8080/dubboadmin

输入用户名 root 密码 root  
显示如下界面说明dubboadmin安装成功  

![](http://i11.tietuku.com/0b78a7fc74c25a77.png)

##五、配置和使用  
###1、maven中引入开发包  

		<!-- dubbo -->
		<dependency>
		    <groupId>com.alibaba</groupId>
		    <artifactId>dubbo</artifactId>
		    <version>2.5.3</version>
		    <scope>compile</scope>
		    <exclusions>
			    <exclusion>
			       <artifactId>spring</artifactId>
			       <groupId>org.springframework</groupId>
			    </exclusion>
		    </exclusions>
	    </dependency>

		<!-- zkclient  -->
		<dependency>
		  <groupId>com.github.sgroschupf</groupId>
		  <artifactId>zkclient</artifactId>
		  <version>0.1</version>
		</dependency>

		<!--  zookeeper -->
		<dependency>
		  <groupId>org.apache.zookeeper</groupId>
		  <artifactId>zookeeper</artifactId>
		  <version>3.3.6</version>
		</dependency>

		<!-- 自定义接口包  -->       	
		<dependency>
			<groupId>com.viewstar</groupId>
			<artifactId>viewstar-api</artifactId>
			<version>${viewstar-api.version}</version>
		</dependency>

###2、服务提供者  

定义服务接口: (该接口需单独打包，在服务提供方和消费方共享)  

	package com.alibaba.dubbo.demo;
 
	public interface DemoService {
 
    	String sayHello(String name);
 
	}

在服务提供方实现接口：(对服务消费方隐藏实现)

	package com.alibaba.dubbo.demo.provider;
 
	import com.alibaba.dubbo.demo.DemoService;
 
	public class DemoServiceImpl implements DemoService {
 
    	public String sayHello(String name) {
        	return "Hello " + name;
    	}
	}

用Spring配置声明暴露服务：  

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
    	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    	xsi:schemaLocation="http://www.springframework.org/schema/beans   
		http://www.springframework.org/schema/beans/spring-beans.xsd
		http://code.alibabatech.com/schema/dubbo
		http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
	    <!-- 提供方应用信息，用于计算依赖关系 -->
	    <dubbo:application name="hello-world-app"  />
	 
	    <!-- 使用zookeeper作为注册中心 -->
	    <dubbo:registry address="zookeeper://192.168.17.53:2181" />
	 
	    <!-- 用dubbo协议在20880端口暴露服务 -->
	    <dubbo:protocol name="dubbo" port="20880" />
	 
	    <!-- 声明需要暴露的服务接口 -->
	    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService" />
	 
	    <!-- 和本地bean一样实现服务 -->
	    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl" />
 
	</beans> 

加载Spring配置：

	import org.springframework.context.support.ClassPathXmlApplicationContext;
 
	public class Provider {
 
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"applicationContext.xml"});
        context.start();
			
		// 按任意键退出
        System.in.read();    
		}
	}  

###3、服务消费者  

通过Spring配置引用远程服务：  
 
	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
    	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    	xsi:schemaLocation="http://www.springframework.org/schema/beans  
		http://www.springframework.org/schema/beans/spring-beans.xsd  
		http://code.alibabatech.com/schema/dubbo  
		http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
	    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
	    <dubbo:application name="consumer-of-helloworld-app"  />
	 
	    <!-- 从注册中心订阅服务 -->
	    <dubbo:registry address="zookeeper://192.168.17.53:2181" />
	 
	    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
	    <dubbo:reference id="demoService" interface="com.alibaba.dubbo.demo.DemoService" />
 
	</beans>

加载Spring配置，并调用远程服务：  


	import org.springframework.context.support.ClassPathXmlApplicationContext;
	import com.alibaba.dubbo.demo.DemoService;
 
	public class Consumer {
 
    	public static void main(String[] args) throws Exception {
        	ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"applicationContext.xml"});
        	context.start();  

			// 获取远程服务代理
        	DemoService demoService = (DemoService)context.getBean("demoService");

			// 执行远程方法
        	String hello = demoService.sayHello("world");

			// 显示调用结果
			System.out.println( hello );    
		}
 
	}

（完）



