---
title: Cornerstone Merge操作详解
tags:
  - iOS学习
categories:
  - iOS学习
date: 2015-09-27 09:00:00
---

Cornerstone是最好用的Mac端版本控制工具，没有之一。
尾部附下载链接，类似其他指令就不说了。<!--more-->

## 一步一步来

1、在服务器上建立新文件并上传代码（文件操作需要谨慎） 

![Paste_Image.png](http://7xqhcq.com1.z0.glb.clouddn.com/iOS-Cornerstone%20Merge-cao-zuo-xiang-jie-1.png)

2、文件拖放到本地，点击提交 

![Paste_Image.png](http://7xqhcq.com1.z0.glb.clouddn.com/iOS-Cornerstone%20Merge-cao-zuo-xiang-jie-2.png)

3、如要生成新的分支 

![Paste_Image.png](http://7xqhcq.com1.z0.glb.clouddn.com/iOS-Cornerstone%20Merge-cao-zuo-xiang-jie-3.png)

4、如果两个分支需要合并到主干，Checkout到本地，点击需要合并到的项目->Merge->Sychronize Branch选择需要从被合并的项目（merge from）合并到这里，然后提交就可以了（如果同时有两个分支，最需仍需要在分支上修改的话，先合并一个分支到主干，然后主干在合并到另一个分支，修改冲突后提交，前提是，刚开始主干和两个分支的代码一样，参考上边的步骤生成）

![QQ20151229-0.png](http://7xqhcq.com1.z0.glb.clouddn.com/iOS-Cornerstone%20Merge-cao-zuo-xiang-jie-4.png)

## 详解

```
1.在workcopying中选择目标copying，然后点击Merge，如图所示
2.选择Mergefrom的copying
3.Merge之前cornerstone会进行dry run，进行merge分析和预览
4.确认无误后Merge Changes （该操作是本地操作，注意解决冲突后在commit）
```

### 下载

> Cornerstone 2.7.10 破解版（支持正版）
链接：http://pan.baidu.com/s/1o6VRtcI  密码: p523

本站文章如无其他特殊说明，均为原创，转载请注明出处。