---
title: 升级 macOS High Sierra 后与 Cocoapods 的兼容问题
date: 2017-12-06 11:42:36
categories: 移动技术
tags: [升级坑]
---

当你发觉自己没有问题的时候，那就尝试着重启一下，万一问题解决了呢？

最近实在厌烦苹果推送了 macOS High Sierra 更新，于是一下班前打开电脑更新系统。过程还算顺利，网络很快，大概 10左右就下载完毕，升级过程中卡死，随后就行下班回家，让其升级了一个夜晚

不过在使用 Cocoapods 的时候还是遇到了问题：

```
06-Dec-2017 09:28:38	/Users/tianyi/bamboo-agent-home/temp/IOS-PB-JOB1-382-ScriptBuildTask-1550561307052749322.sh: /usr/local/bin/pod: /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby: bad interpreter: No such file or directory
06-Dec-2017 11:24:58	env: ruby_executable_hooks: No such file or directory
06-Dec-2017 11:30:11	/Users/tianyi/bamboo-agent-home/temp/IOS-PB-JOB1-382-ScriptBuildTask-5355633338674781643.sh: line 7: pod: command not found
06-Dec-2017 11:31:27	env: ruby_executable_hooks: No such file or directory
```

看起来是 Cocoapods 依赖的 Ruby 版本问题，Google 一下，发现已经有人在 Cocoapods 的 repo 下提了这个 [issue](https://github.com/CocoaPods/CocoaPods/issues/6778)，下面也有开发者给出了解决方案：重新安装 Cocoapods. 

Pod 命令需要用到 2.0 版本的 Ruby 解释器 `/System/Library/Frameworks/Ruby.framework/Versions/2.0`，而 `macOS High Sierra` 将系统的 Ruby 解释器升级到了 `2.3` `/System/Library/Frameworks/Ruby.framework/Versions/2.3`，因此执行 pod 命令的时候由于找不到 Ruby 解释器而报错。

于是按照提示重装 Cocoapods：

```
$ sudo gem install cocoapods
```

安装完成后继续执行 pod install，又报了同样的错误。我决定继续重装一次 Cocoapods，不过这次加上 --verbose 参数，看看安装过程中做了哪些操作。log 太长我就不贴了，不过注意到最后输出的 pod 命令位置似乎跟上面执行 which pod 输出有点不一样，它是 /usr/bin/pod，而 which pod 的输出是 /usr/local/bin/pod，再看一下我的 $PATH 路径：

```
$ echo $PATH
/Users/tianyi/.rvm/gems/ruby-2.4.1/bin:/Users/tianyi/.rvm/gems/ruby-2.4.1@global/bin:/Users/tianyi/.rvm/rubies/ruby-2.4.1/bin:/Users/tianyi/.rvm/bin:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/bin:/Users/tianyi/.jenv/shims:/Users/tianyi/.jenv/bin:/opt/subversion/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Applications/Server.app/Contents/ServerRoot/usr/bin:/Applications/Server.app/Contents/ServerRoot/usr/sbin:/usr/local/go/bin:/Users/tianyi/bin:/Users/tianyi/goProject/bin:/usr/local/go/bin:/usr/local/Cellar/nginx/1.12.0/bin:/usr/local/mysql/bin
```

可以看到在我的 $PATH 环境变量里，/usr/local/bin 的优先级是高于 /usr/bin 的，因此当这两个地方都存在一个名叫 pod 的命令时，系统优先执行 /usr/local/bin/pod，于是错误就这么产生了。因此我直接删除 /usr/local/bin/pod 文件，再执行 pod install --verbose，这一次果然安装成功了。

这个问题应该是由于 Cocoapods 改变了安装路径导致的，记得 macOS 启用 System Integrity Protection 之后 Cocoapods 的安装路径也修改过，这次应该也是类似的问题吧，由于 $PATH 这个环境变量的问题，导致老版本的 pod 命令优先被执行。

事情至此还未结局

本地执行pod 指令已经没问题了，但是我们通过Bamboo 集成，使用脚本打包，却一直重复前面的错误无法自拔。

重启bamboo 服务，依然不能解决问题

尝试着使用重启治百病的手段，电脑关机重启试试看！

祈祷中...🙏


2017年12月6日午时三刻更

说了你可能不相信，重启电脑后，一切问题都好了~

更新

解决macOS Sierra下注册机无法运行的问题

很多软件都不兼容了
「安全性与隐私」设置中「任何来源」选项消失
几乎所有注册机都用不了

恢复任何来源选项
ps：如果系统升级前就已经选择了任何来源，升级后还会正常显示

打开终端(Terminal.app)
执行`sudo spctl --master-disable`

“任何来源”恢复~