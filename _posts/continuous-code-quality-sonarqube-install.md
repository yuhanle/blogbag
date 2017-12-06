---
layout: post
title: '通过Sonar 初步构建代码持续审查'
date: '2017-02-15'
header-img: "img/post-bg-ios.jpg"
tags:
     - iOS
     - Sonar
     - Quality
categories:
	- 持续集成
author: '煜寒了'
---

![SonarQube](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/Benefits-of-Using-SonarQube-For-Code-Reviews.jpg)

## 介绍篇

### SonarQube 介绍

[SonarQube](https://www.sonarqube.org/) 是一款领先的持续代码质量监控平台，开源在github 上，现在已更新到6.2 版本，star 数量超过 1400+，可以轻松配置在内网服务器，实时监控代码，帮助了解提升提升团队项目代码质量。通过插件机制，SonarQube可以继承不同的测试工具，代码分析工具，以及持续集成工具。

<!-- more-->

与持续集成工具（例如 Hudson/Jenkins 等）不同，SonarQube 并不是简单地把不同的代码检查工具结果（例如 FindBugs，PMD 等）直接显示在 Web 页面上，而是通过不同的插件对这些结果进行再加工处理，通过量化的方式度量代码质量的变化，从而可以方便地对不同规模和种类的工程进行代码质量管理。

在对其他工具的支持方面，Sonar 不仅提供了对 IDE 的支持，可以在 Eclipse 和 IntelliJ IDEA 这些工具里联机查看结果；同时 SonarQube 还对大量的持续集成工具提供了接口支持，可以很方便地在持续集成中使用 SonarQube。

此外，SonarQube 的插件还可以对 Java 以外的其他编程语言提供支持，对国际化以及报告文档化也有良好的支持。

行业内提到"代码质量管理, 自动化质量管理", 一般指的都是通过Sonar来实现。本文的目标是实现在Sonar上显示出iOS项目, 先看张最终的效果图:

![SonarQube](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/main-des.png)

用Sonar能够实现什么?

* 技术债务(sonar根据"规则"扫描出不符合规则的代码)
* 覆盖率(单元测试覆盖率)
* 重复(重复的代码, 有利于提醒封装)
* 结构

问题1: "规则"指的是什么? 

     在Sonar工具中配置检测工具(规则), 然后sonar根据规则检测"质量报告文件", 得出问题数目。 比如本文配置的规则是OCLint

问题2: 技术债务的天数怎么得出?

     每个规则都有对应的处理时间, 最后:问题类型1数目 * 对应时间 + 问题类型2数目 * 对应时间 +... 得到时间。

### SonarQube 工作流程

SonarQube 并不是简单地将各种质量检测工具的结果（例如 FindBugs，PMD 等）直接展现给客户，而是通过不同的插件算法来对这些结果进行再加工，最终以量化的方式来衡量代码质量，从而方便地对不同规模和种类的工程进行相应的代码质量管理。

SonarQube 在进行代码质量管理时，会从图 1 所示的七个纬度来分析项目的质量。

![图 1](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/image001.png)

SonarQube 不是那种安装即可用的工具，他需要数据库的支持，用于存储检测项目后的分析数据，同时为了实现可持续监测，还需要项目持续集成工具（如Jenkins）的支持，在构建版本前，通过Jenkins+Sonar 插件执行项目分析指令，最终的结果会通过SonarQube 服务器的Web 页面展示。

![网络图](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/image4.png)

下面我们就通过mysql+Jenkins+SonarQube 实现项目代码质量的可持续监测

![](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/SQ55Integration.png)

## 安装篇

### SonarQube 的安装

#### 涉及到的知识点

* XCTool工具
* OClint工具
* Gcovr工具
* Git, SVN命令
* Linux 命令
* mysql 操作
* Jenkins工具
* Sonar工具
* Shell语法
* Sonar-runner工具

#### 软件及硬件的要求

SonarQube 的安装通常需要满足一定的软硬件条件，具体要求如下所示：

1. Server 要求

	Web server 最少需要 500MB 的内存空间，推荐内存空间大小 2GB。Sonar 在进行代码质量分析时，通常大约每 1 KLOC 需要存储 350KB 左右的数据，所以要尽量为 SonarQube 的 web server 提供大的内存。
	
2. Database 要求

	尽管 SonarQube 本身自带嵌入的 Derby 数据库，但是由于 Derby 比较简单，所以在生产环境中强烈推荐安装相应的企业版数据库，SonarQube 支持的数据库包括： MySQL 5.x+、Oracle10g+、PostgreSQL 9.x 和 MS SQLServer 2005 and 2008，推荐使用 MySQL。

3. Browser 要求

	SonarQube 支持大多数的浏览器，包括 Firefox、Internet Explorer 7.x and 8.x and chromed 等，推荐使用 chromed。

	~~目前官方最新的版本是6.2，但是对于部分插件会存在不兼容的问题，导致Sonar 服务启动失败，所以为了使用和演示，我采用了旧版本4.5.7，因为此版本兼容 sonar-objective-c-plugin-0.3.1 插件。~~

	**更新：**
	
	关于XCode8的兼容方案, 请看[这篇文章](https://my.oschina.net/ChenTF/blog/806565)

#### 安装步骤

1. 数据库配置
	
	进入数据库命令模式或者直接使用GUI 工具，创建Sonar-Qube 服务所需的数据库
	
	```
	mysql -u root -p
	mysql> CREATE DATABASE sonar CHARACTER SET utf8 COLLATE utf8_general_ci; 
	mysql> CREATE USER 'sonar' IDENTIFIED BY 'sonar';
	mysql> GRANT ALL ON sonar.* TO 'sonar'@'%' IDENTIFIED BY 'sonar';
	mysql> GRANT ALL ON sonar.* TO 'sonar'@'localhost' IDENTIFIED BY 'sonar';
	mysql> FLUSH PRIVILEGES;
	```
	![创建数据库](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/init-sql.png)
	
	完成以后测试一下连接是否正确

2. 安装SonarQube 与SonarQube-Runner
	
	SonarQube Runner 2.4 [下载地址](https://docs.sonarqube.org/display/SONARQUBE45/Installing+and+Configuring+SonarQube+Runner)
	
	从官网下载 SonarQube 的最新版本并解压到`/usr/local/`文件夹	
	
	![SonarQube](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/sonar-install-2.png)
	
	添加SONAR_HOME、SONAR_RUNNER_HOME 环境变量，并将SONAR_RUNNER_HOME 加入PATH

3. 修改Sonar-Qube 配置文件
	
	配置文件路径在 `./conf/sonar.properties`
	
	```
	# 设置数据库的账户密码
	sonar.jdbc.username=ua
	sonar.jdbc.password=pwd
	
	sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance

	# By default, ports will be used on all IP addresses associated with the server.
	sonar.web.host=0.0.0.0
	
	# The default value is root context (empty value).
	sonar.web.context=/
	# TCP port for incoming HTTP connections. Default value is 9000.
	sonar.web.port=9003
	```

4. 运行如下命令启动 Sonar-Qube，根据操作系统选择

	```
	Last login: Wed Feb 15 18:15:05 on ttys000
	localhost:~ tianyi$ /usr/local/sonarqube-6.2/bin/macosx-universal-64/sonar.sh start
	
	localhost:~ tianyi$ /usr/local/sonarqube-6.2/bin/macosx-universal-64/sonar.sh start
	Starting SonarQube...
	Started SonarQube.
	```

5. 创建一个简单的工程
	
	默认密码是admin：admin，登陆管理员账号以后，配置系统参数
	
	创建一个Demo 工程
	
	![创建一个工程](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/make-new1.png)
	
	Demo 工程简介
	
	![Demo 工程简介](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/make-demo.png)
	
	因为还没有进行过分析，所以Demo 暂时只能配置，在使用篇我们会详细介绍如果通过指令或自动化执行分析审查。
	
6. 安装一些必备插件，都可以从官网或者github 上搜索到
	
	Sonar支持多种插件，插件的下载地址为：http://docs.codehaus.org/display/SONAR/Plugin+Library

	将下载后的插件上传到${SONAR_HOME}extensions\plugins目录下，重新启动sonar。
	
#### 常用插件

（注意版本号兼容性问题）
	
* SonarQube 汉化包：https://github.com/SonarQubeCommunity/sonar-l10n-zh
* Objective-C 代码检查：https://github.com/octo-technology/sonar-objective-c
* JavaScript 代码检查：http://docs.codehaus.org/display/SONAR/JavaScript+Plugin
* python 代码检查：http://docs.codehaus.org/display/SONAR/Python+Plugin
* Web页面检查（HTML、JSP、JSF、Ruby、PHP等）：http://docs.codehaus.org/display/SONAR/Web+Plugin
* xml文件检查：http://docs.codehaus.org/display/SONAR/XML+Plugin
* scm源码库统计分析：http://docs.codehaus.org/display/SONAR/SCM+Stats+Plugin
* 文件度量：http://docs.codehaus.org/display/SONAR/Tab+Metrics+Plugin
* 中文语言包：http://docs.codehaus.org/display/SONAR/Chinese+Pack
* 时间表显示度量结果：http://docs.codehaus.org/display/SONAR/Timeline+Plugin
* 度量结果演进图：http://docs.codehaus.org/display/SONAR/Motion+Chart+Plugin

#### 我的资源

![](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/commonly-used1.png)

## 使用篇

### 使用SonarQube Runner分析源码

#### 预置条件

已安装SonarQube Runner且环境变量已配置，即sonar-runner命令可在任意目录下执行

如何配置环境变量，参考[这篇文章](http://www.cnblogs.com/caowei/p/mac-path_2013-08-26.html)

#### 如何分析

**1. 在项目源码的根目录下创建sonar-project.properties配置文件**

以iOS 项目为例

```
# Required metadata
sonar.projectKey=iOS::Demo
sonar.projectName=iOS::Demo
sonar.projectVersion=1.0

# Comma-separated paths to directories with sources (required)
sonar.sources=Demo

# Language
sonar.language=objectivec

# Encoding of the source files
sonar.sourceEncoding=UTF-8
```

**注意：sonar.language 和安装的代码审查插件有关，需要安装 sonar-objective-c 插件，否则运行时会提示无法找到这个语言**

**2. 执行分析**

在项目的根目录执行分析指令

```
/usr/local/sonar-runner-2.4/bin/sonar-runner
```

![执行分析指令](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/sonar-ext.png)

查看Sonar 分析的结果

![分析结果](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/sonar-ext-result.png)

以上是创建Demo 工程后，通过手动执行分析指令完成代码审查分析。


### 与Jenkins 持续集成

#### 构建前操作

在jenkins的插件管理中选择安装SonarQube-Scanner，该插件可以使项目每次构建都调用sonar进行代码度量。

进入配置页面对sonar插件进行配置，如下图：

![新增构建前操作](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/jenkins-goujian.png)

以上配置可以使项目在构建前，自动执行代码审查和分析，结果会自动保存并上传到数据库，通过Sonar 服务器展示给开发者。

#### 设置触发器

每5分钟检查一次仓库，若有上库，则自动执行代码检测

![设置触发器](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/jenkins-chufa.png)

至此，通过Sonar 初步构建代码持续审查的工作完成。

## 问题篇

### 安装中的问题

1. 资源找寻的问题
	
	文中提到的资源文件，网上谷歌百度都用上，也找了很久，包括一些插件的问题，必须安装的就是 Sonar-Qube，Runner 其实不必要安装，因为Jenkins 里有插件，可以直接使用。
	
2. 环境配置问题

	如果不常用指令的话，可以不配置，直接通过绝对路径做操作，环境配置的方法参考[这篇文章](http://www.cnblogs.com/caowei/p/mac-path_2013-08-26.html)

3. 初始化时遇到Sorry 的问题
	
	遇到这种问题可以去根目录 /log/xx.log 中查看日志，具体问题具体解决
	
	![sorry](http://7xqhcq.com1.z0.glb.clouddn.com/continuous-code-quality-sonarqube-install/qa-3.png)

4. 汉化包的问题

	如果你不习惯英文的使用，想做一做汉化的事情时，汉化包一定要对，如果有问题会导致服务启动失败。
	
	附上：[汉化包下载地址](https://github.com/SonarQubeCommunity/sonar-l10n-zh)

### SonarQube 实际使用中遇到的问题及解决方案

在使用 SonarQube6.2 分析代码质量时，可能会遇到的问题：
	
1. Xcode 8 兼容性问题
	
	原有的xctool已不支持XCode8, 改用xcodebuild + xcpretty 来替代xctool环节生成对应的产出物。
	
	关于XCode8的兼容方案, 请看[这篇文章](https://my.oschina.net/ChenTF/blog/806565)

## 结束语

代码质量管理对提高项目质量意义重大。本文介绍了 SonarQube 的工作原理，并从项目实战的角度讲解了使用 SonarQube 进行项目代码质量管理的流程和注意事项。

代码规范贵在坚持与执行力，自动审查只是做了提醒的作用，当然根据语言的规则，很多规范在使用者来讲看似不合理或者不好用，有了这个工具，参与者们可以切身感受到团队的成长。

## 参考文章

[[实践]iOS Sonar集成流程详解](https://my.oschina.net/ChenTF/blog/708646)

[[实践]Sonar Xcode8兼容](https://my.oschina.net/ChenTF/blog/806565)

[SonarQube代码质量管理平台安装与使用](http://www.uml.org.cn/rjzl/201312132.asp)

[配置sonar、jenkins进行持续审查](http://www.cnblogs.com/gao241/p/3190701.html)
