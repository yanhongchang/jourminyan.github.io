---
layout: post
title: kolla 制作openstack 容器镜像
category: 技术
date: 2019-11-03


---

更新历史：

- 2019.11 完成

------

为了后续更加高效快速的部署openstack云平台，减少每次部署时镜像网络获取或当时制作步骤，因此需要本地镜像仓库的方式，将openstack各种服务的容器镜像一次制作后上传的本地仓库，后续部署时通过本地仓库拉取镜像方式进行。



##### 环境准备

操作系统：centos 7.6.1810

安装pip：

```
yum install python-pip
pip install tox
```



##### Kolla源码下载

```shell
git clone https://github.com/openstack/kolla.git
git checkout -b pike origin/stable/pike
```

安装kolla 相关依赖

cd kolla

```
pip install .
```



##### 镜像制作

1、生成kolla-build.conf配置文件

cd kolla

```
pip install tox
tox -e genconfig
```

2、修改配置文件etc/kolla/kolla-build.conf

```
[DEFAULT]
# 系统版本
base_tag = 7.6.1810 
#开启debug模式
debug = true
#需要制作的镜像集合名称
profile = default
#安装类型，source表示源码安装、binary二进制yum安装
install_type = source
#镜像tag
tag = 1.0.0
# 
pull = false
```

cp etc/kolla/kolla-build.conf /etc/kolla

3、制作docker镜像

```
kolla-build --config-file /etc/kolla/kolla-build.conf 2>&1 | tee kolla-build.log
```

4、查看生成的镜像

```
docker images
```



##### 镜像进库

```shell
tag="1.0.0"
reg_url='X.X.250.66/pike_source'

for image in `docker images | grep kolla | awk '{print substr($1,7)}'`; do docker tag kolla/$image:$tag $reg_url/$image:$tag; docker push $reg_url/$image:$tag;done
```



参考：

1、https://docs.openstack.org/kolla/pike/admin/image-building.html
