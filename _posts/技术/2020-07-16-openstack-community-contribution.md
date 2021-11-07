---
layout: post
title: openstack社区贡献
category: 技术
date: 2020-07-20



---

更新历史：

- 2020.07 完成

------

开源浪潮已经成为技术发展的主流方向，我们使用的openstack开源项目已经成为开源社区第二大项目，作为软件项目，肯定和普通项目一样，都会各种各样的bug在长期的使用过程中，肯定会发现一些项目中存在的bug，既然我们身处这个大的技术浪潮下，我们就不能置身事外，就不能单单只作为使用者，还要作为参与者参与其中。既作为受益者，也要作为贡献者，在开源社区中贡献出自己的一份力量，不要认为自己的贡献微不足道，开源的精神就是众人拾材火焰高，只有大家都积极贡献社区，那么开源项目才会越来越好，越来越完善，让更多的公司和个人从开源项目中受益。因此这里介绍一下如何为开源项目openstack项目提交一个bug，并且如何fixed的流程。



一、准备工作

1、注册ubuntu账号。

2、登陆launchdap。

3、登陆代码审查平台https://review.opendev.org/。

​	上传ssh密钥。

​	签署开源协议。https://review.opendev.org/#/settings/agreements





二、参与bug修复

1、上报bug

​		登陆 https://launchpad.net



2、提交review确定bug

​		

3、fixed bug并被review。

​	ssh-keygen -t rsa -C "xxx@163.com"

​	

1. 运行 `git config --global user.name "Firstname Lastname"` 命令。
2. 运行 `git config --global user.email "your_email@youremail.com"` 命令。



4、commit 

git commit

```
第一部分： 摘要

第二部分： 详细说明解决的问题

第三部分： 修复的bug ID等
```



例子：

    Wrong status change for port when restart neutron-linuxbridge-agent
    
    the port need not change when restart neutron-linuxbridge-agent on
    compute nodes, the judgment of host and conext host for ports was
    wrong here, this will trigger all compute node to refrash firewall
    and make the rabbitmq and neutron-server service will be High CPU
    load.
    
    Related-Bug: #1866743


5、review

git review

```
bogon:nova tal$ git review
remote: 
remote: Processing changes: new: 1, refs: 1 (\)
remote: Processing changes: new: 1, refs: 1 (\)
remote: Processing changes: new: 1, refs: 1 (\)
remote: Processing changes: new: 1, refs: 1 (\)
remote: Processing changes: new: 1, refs: 1, done            
remote: 
remote: New Changes:        
remote:   https://review.opendev.org/744273 add notification info when server attach or detach interface        
remote: 
To ssh://review.opendev.org:29418/openstack/nova.git
 * [new branch]            HEAD -> refs/for/master%topic=bug_1889544
```





问题记录：

1、review时提示如下：

this repository is now set up for use with git-review. You can set the

default username for future repositories with: git config --global --add gitreview.username "xxxxx"

需要本地配置gitreview.username

```
git config --global --add gitreview.username "xxxxx"
```



2、ERROR: [55b6523] missing Change-Id in commit message footer  

​	提示丢失Change-id，之前都是git review时默认加上的，现在需要在git commit前安装hook，执行下列命令后安装：

```
gitdir=$(git rev-parse --git-dir); scp -p -P 29418 yanhongchang5@review.opendev.org:hooks/commit-msg ${gitdir}/hooks/
```

安装完毕后，需要重新commit

```
 git commit --amend
```

这是提交内容里会默认加上Change-Id。

