---
layout: post
title: 搭建本地离线yum源
category: 技术
date: 2019-08-08



---

更新历史：-

- 2019.08  完成

------

####

由于后期会需要对openstack一些项目进行打包，因此，需要不能使用在线yum源进行openstack docker镜像的制作，因此，需要自建本地离线yum源，方便后期替换rpm安装包。

##### 环境准备：

操作系统：centos 7.6.1810

1、关闭firewalld

```
systemctl stop firewalld
systemctl disable firewalld
```

2、安装所需工具包

```
#同步远程repo的工具
yum install reposysc -y

#创建本地所需的工具
yum isntall createrepo -y
```



##### 同步远程源

1、安装需要同步的远程repo源

​		一般我们同步都是同步centos的官方yum源，但是官方的源可能不在国内，因此同步速率特别慢。并且国内有一些公司，例如aliyun、网易等都会定期同步官方的源，因此，我们可以直接同步这些国内yum源，可以大大提高效率。推荐三个：

```
网易 mirrors.163.com
阿里云 mirrors.aliyun.com
清华 mirrors.tuna.tsinghua.edu.cn
```

我们本次搭建使用163做为我们同步的远程源。

安装163源参考：[http://mirrors.163.com/.help/centos.html](http://mirrors.163.com/.help/centos.html)

列出本地： 

```
yum repolist
```



找到想要同步的repo ，这里我们同步base

```
reposync -r base -p <yourdirpath>
```

接下来就是等待同步完成。



##### 创建和发布本地源

通过createrepo 创建本地repo源

```
createrepo <yourdirpath>
```

执行后，在目录中会生成一个repodata文件夹，里面放置了repo源所需的一些xml文件

未来使用，肯定是使用通过局域网使用源，因此我们要把yum源通过web服务器（nginx、httpd）等发布出来

这里我们选择nginx发布本地源

1、安装nginx

```
yum install nginx -y
```

2、配置/etc/nginx/nginx.conf

```
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf_bak
```

vim /etc/nginx/nginx.conf

```
server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```



修改成：

```
server {
        listen       10010 default_server;
        listen       [::]:10010 default_server;
        server_name  _;
        #root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
                root         /opt/repo_data/;
                autoindex on;
                autoindex_exact_size on; # 显示文件大小
                autoindex_localtime on; # 显示文件时间
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

```



重启nginx

```
systemctl restart nginx
```



查看是否通过web可以正常访问yum 源

![image-20190521141202426](/Users/tal/Library/Application Support/typora-user-images/image-20190521141202426.png)

可以正常访问。



##### 使用

在希望使用的节点上，配置本地离线yum源

```
touch /etc/yum.repo.d/local.repo
```

添加如下信息：

```
[base]
baseurl = http://x.x.x.x:10010/cloud/base/Packages
enabled = 1
gpgcheck = 0
name = centos base  repositor

[openstack-pike]
baseurl = http://x.x.x.x:10010/cloud/openstack-pike
enabled = 1
gpgcheck = 0
name = kolla build openstack pike  repositor

[ceph-jewel]
baseurl = http://x.x.x.x:10010/cloud/ceph-jewel
enabled = 1
gpgcheck = 0
name = kolla build ceph jewel repositor
```

重新生成缓存后，查看是否可以正常获取local.repo

```
yum clean all 
yum makecache

yum repolist
```



##### 更新

当需要增加或者更新替换local repo的某个rpm时，需要重新update一下repo

```
repocreate --update <yourrepodir>
```

