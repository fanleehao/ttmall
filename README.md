# ttmall
SSM综合分布式电子商务案例

> 分布式电子商务网站
>
> From Self-Learning. at Apr 2019. Nanjing.

### 0 项目简介

#### 概述

　　本项目来源于视频教学【[点击](<https://www.bilibili.com/video/av33269937>)】，属于Java 框架的整合实践。通过常见的SSM框架组织，使用分布式组织架构，搭建与设计项目。

　　项目是一个实现了用户权限、门户访问、商品、订单、及后台管理等多个基本模块的电商网站，提供分类、搜索等功能。在此基础上，通过分布式完成对高并发的支持。

#### 设计

本项目使用Maven进行工程搭建，使用Eclipse进行开发；项目涉及相关技术主要有：

```properties
Spring、SpringMVC、Mybatis
JSP、JSTL、jQuery、EasyUI、KindEditor（富文本编辑器）
Redis（缓存服务器，单点登录，购物车）
Solr（搜索）
dubbo（分布式服务框架）
HttpClient（HTTP 协议访问客户端）
ActiveMQ（消息队列）
Quartz（定时任务）
FastDFS（图片服务器）
FreeMarker（网页静态化）
Nginx（反向代理服务器）
MyCat（数据库中间件）
```

项目的开发环境：

```properties
Eclipse Mars.2 
Maven 3.6.0
Tomcat 7（Maven Tomcat Plugin）
JDK 1.8
Mysql 5.7
Dubbo 2.5.3
Nginx 1.8.0
Redis 3.0.0
ActiveMQ 5.13.0
Win10 OS
SVN
```



### 1 项目组织

#### 项目架构

项目弃用传统的Web开发，使用面向服务的分布式以模拟支持高并发的访问。

![img](assets/clip_image002-1555387290729.jpg)

　　系统的**功能**主要是从以下几个方面考虑：

- 后台管理系统：管理商品、订单、类目、商品规格属性、用户管理以及内容发布等功能。
- 前台系统：用户可以在前台系统中进行注册、登录、浏览商品、首页、下单等操作。
- 会员系统：用户可以在该系统中查询已下的订单、收藏的商品、我的优惠券、团购等信息。
- 订单系统：提供下单、查询订单、修改订单状态、定时处理订单。
- 搜索系统：提供商品的搜索功能。
- 单点登录系统：为多个系统之间提供用户登录凭证以及查询登录用户的信息

#### 基于服务组织

SOA：Service Oriented Architecture面向服务的架构。也就是把工程都拆分成服务层工程、表现层工程。服务层中包含业务逻辑，只需要对外提供服务即可。表现层只需要处理和页面的交互，业务逻辑都是调用服务层的服务来实现。工程都可以独立部署。

![img](assets/clip_image002-1555387469132.gif)

#### 项目结构图

　　根据不同的功能需求，使用SOA思想将不同的功能模块转为微服务的设计，使用服务的状态来对其他模块的功能进行支持，同一服务可以使用分布式集群进行管理，提供并发访问；不同的服务之间相互支持，具有较低级别的耦合度，方便系统的拓展。

![1555387623928](assets/1555387623928.png)

#### Maven工程

　　使用Maven实现Jar包等打包管理、支持同一环境、版本控制；Maven的常见打包方式：jar、war、pom。Pom工程一般都是父工程，管理jar包的版本、maven插件的版本、统一的依赖管理。聚合工程。

ttmall-parent：父工程，打包方式pom，管理jar包的版本号。

​        |  : 项目中所有工程都应该继承父工程。

​	|--ttmall-common：通用的工具类通用的pojo,util。打包方式jar

​	|--ttmall-manager：**服务层**工程。聚合工程。Pom工程

​		|--ttmall-manager-dao：打包方式jar

​		|--ttmall-manager-pojo：打包方式jar

​		|--ttmall-manager-interface：打包方式jar

​		|--ttmall-manager-service：打包方式：war  (为了发布服务的方便)

​	|--ttmall-manager-web：**表现层**工程。打包方式war

##### **依赖**

- common: common中的依赖，主要是一些工具和通用的pojo，如test/apache文件流/日期时间/日志/HttpClient等
- manager：可能只有对common通用组件的依赖（其实是其子包dao,service等对一些common包有依赖，可以在父类manager中声明）
- dao：依赖Mybatis、DB驱动、连接池等；另外还有pojo（注意其中pageHelper的版本更新为4.1.6）
- pojo：基本无其他依赖
- interface：依赖pojo；
- service：依赖spring；另依赖dao/interface
- web：依赖JSP、spring/springMVC/servlet、文件、redis等；另依赖interface(也包含pojo）；还需要配置tomcat插件

##### 启动

　　在表现层web和服务层manager都需配置一个tomcat插件；用于启动项目，发布服务，两者端口区分开；

```xml
<!-- 配置tomcat插件 -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.tomcat.maven</groupId>
            <artifactId>tomcat7-maven-plugin</artifactId>
            <configuration>
                <port>8081</port>
                <path>/</path>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**使用tomcat**插件启动

- 先安装ttmall-parent工程到本地仓库(maven install)
- 再安装ttmall-common工程、ttmall-manager到本地仓库
- 最后创建web工程（maven build…）

### 2 项目设计

#### 2.1 功能

##### 功能设计

框架：Spring+SpringMVC+Mybatis+Dubbo
前端：EasyUI
DB：MySQL

##### 系统间通信

> 商城是基于SOA的架构，表现层和服务层是不同的工程。所以要实现商品列表查询需要两个系统之间进行通信。如何实现远程通信？
> 1、使用WebService：效率不高,它是基于soap协议（http+xml）。项目中不推荐使用。
> 2、使用restful形式的服务：http+json。很多项目中应用。如果服务越来越多，服务与服务之间的调用关系复杂，调用服务的URL管理复杂，什么时候添加机器难以确定。 
> 3、使用dubbo。使用rpc协议进行远程调用，直接使用socket通信。传输效率高，并且可以统计出系统之间的调用关系、调用次数，管理服务。

![IMG_256](assets/clip_image002-1555398808881.jpg)

　　Dubbo是阿里巴巴一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，是阿里巴巴SOA服务化治理方案的核心框架，**视作资源调度和治理中心的管理工具**。

　　Dubbo 就是类似于webservice的关于系统之间通信的框架，并可以统计和管理服务之间的调用情况（包括服务被谁调用了，调用的次数是如何，以及服务的使用状况）.

##### Dubbo使用

​	![img](assets/clip_image001.jpg)

> 节点角色说明：
> Provider: 暴露服务的服务提供方。
> Consumer: 调用远程服务的服务消费方。
> Registry: 服务注册与发现的注册中心。
> Monitor: 统计服务的调用次调和调用时间的监控中心。
> Container:服务运行容器。
>
> 调用关系说明：
> 0.服务容器负责启动，加载，运行服务提供者。
> 1.服务提供者在启动时，向注册中心注册自己提供的服务。
> 2.服务消费者在启动时，向注册中心订阅自己所需的服务。
> 3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。        4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
> 5.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

- Dubbo使用只需要配置Provider、Consumer、registry、container（由spring容器托管），monitor可暂时不考虑

- Dubbo全局使用spring进行配置，集成托管；对应用没有任何API侵入，只需用Spring加载Dubbo的配置即可，Dubbo基于Spring的Schema扩展进行加载。Dubbo中将各个服务分离，不同的模块通过远程服务调用来实现，在传统的Spring中开发时情况如下：

  ```xml
  <bean id="xxxService" class="com.xxx.XxxServiceImpl" />
  <bean id="xxxAction" class="com.xxx.XxxAction">
  	<property name="xxxService" ref="xxxService" />
  </bean>
  ```

  使用Dubbo架构设计，不同的服务提供者和订阅者在不同的地方，需要通过远程接口的配置来完成；也就是分别配置provider和consumer；

  ```xml
  # 服务层
  <!-- 和本地服务一样实现远程服务 -->
  <bean id="xxxService" class="com.xxx.XxxServiceImpl" />
  <!-- 增加暴露远程服务配置 -->
  <dubbo:service interface="com.xxx.XxxService" ref="xxxService" />
  
  -------------------------------------------------------------------
  # 表现层
  <!-- 增加引用远程服务配置 -->
  <dubbo:reference id="xxxService" interface="com.xxx.XxxService" />
  <!-- 和本地服务一样使用远程服务 -->
  <bean id="xxxAction" class="com.xxx.XxxAction">
  	<property name="xxxService" ref="xxxService" />
  </bean>
  ```

##### 注册中心Zookeeper

　　在Dubbo中registry使用zookeeper实现。Apacahe Hadoop的子项目，是一个树型的目录服务，支持变更推送，适合作为Dubbo服务的注册中心，工业强度较高(稳定性好)，可用于生产环境，并推荐使用。

　　注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小。使用dubbo-2.3.3以上版本，官方建议使用zookeeper作为注册中心．

　　安装，修改参数；使用	

&nbsp;

#### 2.2 框架整合

##### 2.2.1 Dubbo依赖

加入dubbo相关的jar包。服务层、表现层都添加，分别扮演服务提供和消费方的角色；

```xml
<!-- dubbo相关 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <!-- 排除依赖 -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>spring</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.jboss.netty</groupId>
            <artifactId>netty</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
</dependency>
<dependency>
    <groupId>com.github.sgroschupf</groupId>
    <artifactId>zkclient</artifactId>
</dependency>
```



##### 



### 3 项目总结
