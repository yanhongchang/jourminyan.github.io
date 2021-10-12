---
layout: post
title: 前端技术栈
category: 技术
date: 2020-02-14





---

更新历史：

- 2020.02 完成

------

目前前端常用的一个工程技术栈是nodejs+vuejs+Eelement，通过该组合可以搭建并实现一个前端项目，学思云管的钱后端分离的前端项目就是通过这个技术栈实现的。下面我们分别简单介绍。

### 一、Vue.js

1、简介

​		Vue.js（后续简称为Vue）官方给出的定义：是一套用于构建用户界面的渐进式框架，Vue 的核心库只关注视图层，不仅易于上手，还便于与第三方库或既有项目整合。是目前比较流行的一个前端，类似于jQuery，主要思想都是数据驱动，不过Vue相对于JQuery而言，更加容易用户操作数据，不需要直接操作dom，通过并且vue框架是中国人编写实现，技术文档也是妥妥的中文，实实在在的为推动国内前端技术做出了巨大贡献。

2、安装使用

​	Vue官方提供了常用3种使用方式

​	1）通过\<script>标签引入

​			用户可以通过下载Vue文件，然后再html中直接通过\<script>进行引入就可以使用，这时Vue就作为一个全局变量导入到html中。

```html
<script src="https://cdn.jsdelivr.net/npm/vue@2.6.11"></script>
```

​		这里下载官方提供了两个版本：

​		开发版本：包含完整的警告和调试模式

​		生产版本：删除了警告，33.30KB min+gzip

主要区别在于开发版本包含了所有的调试和警告，方便开发者调试。生产版本是压缩版本，体积更小。适用于生产环境使用。



2）通过npm安装到本地使用

​		在构建大型应用时可以通过 NPM （node pageage management）软件包管理进行安装。

```shell
npm install vue
```

​		并且npm还可以和很多现成工具配合使用，例如可以通过webpack打包工具对项目进行打包。



3）通过CLI部署工具

​		Vue 提供了一个[官方的 CLI](https://github.com/vuejs/vue-cli)，为单页面应用 (SPA) 快速搭建繁杂的脚手架。通过CLI我们可以快速搭建生产级别的项目。

```shell
npm install -g @vue/cli
```

​		然后我们可以通过vue的CLI工具初始化一个项目。

```shell
vue init webpack myProject
```

​		这里的webpack参数是指myProject这个项目将会在开发和完成阶段帮你自动打包代码，比如将js文件统一合成一个文件，将CSS文件统一合并压缩等。后面我们还会详细介绍如何初始化一个项目，这里先跳过。

​		具体使用参考：[CLI](https://cli.vuejs.org/)



### 二、Node.js

1、简介

​		官方定义：Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine. 通过定义可以看出nodejs（后续简称node）其实就是一个js的运行环境。类似于java的JVM虚拟机一样，nodejs就是运行JS的运行时，不管在什么操作系统平台，只要安装了node就可以运行js代码。

​		这里可能会有一个疑问，我们的项目是前后端分离的，后端是python3实现的，前端也不需要使用node编写逻辑代码，为什么要引入node呢？其实我们上边也提到了，我们是使用webpack工具来打包前端项目，而webpack就是通过node实现的，这就是我们为什么要引入node

2、安装

​		安装node的方式根据不同操作系统，下面举例centos7如何安装最新版本的node。

```shell
yum install nodejs

查看版本：
node -v
npm -v
```

​		后续可以通过npm install方式安装项目的依赖。

​	

### 三、Element UI

​	1、简介

​			Element UI是一个由外卖平台公司饿了么开发和维护的一个前端UI组件库，基于Vue 2.0开发，这个组建库提供了一系列丰富的现成通用组件供用户直接使用，开发者不需要再自己实现常用的组件，极大的提高了开发效率，缩短了开发流程。



​	2、安装

​		由于Eelement UI组件库也是基于Vue开发，因此它也提供了两种安装方式，具体如下：

​		1）通过npm自动安装

```shell
npm i element-ui -S
```

​		2) 通过\<link>和\<script>引入

```html
<!-- 引入样式 -->
<link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css">
<!-- 引入组件库 -->
<script src="https://unpkg.com/element-ui/lib/index.js"></script>
```

​		通过npm可以更好的配合webpack使用，官方推荐使用npm方式安装。



3、使用

​		element UI组件库使用起来也比较方便，由于组件太多，这里就不再一一赘述，具体可以参考官方文档使用。

​		Element UI组件官网  [https://element.eleme.cn/#/zh-CN/component/installation](https://element.eleme.cn/#/zh-CN/component/installation)



### 四、Vue demo项目

​		前面简单介绍我们常用的技术栈，那么接下来我们介绍如何通过这个技术栈来构建我们的前端项目。由于我们需要通过webpack工具，因此方便起见，我们这里就通过CLI工具构建项目。如何安装node和npm我们这里就不介绍了，下面是构建一个 Vue的demo项目流程。

1）安装Vue和CLI

```shell
npm install vue
npm install -g @vue/cli
```

​	安装以后可以通过CLI 命令行工具查看vue版本

```
vue -V
```



2）构建demo项目MyProject

```shell
vue init webpack myproject
```

​	注：上面也提到了webpack参数是node实现的一个项目打包工具。有了webpack我们就可以对Vue项目快速的打包然后部署。

构建过程中需要有一些交互，根据具体情况输入就可以。

```vue
Jourminyan:code_100tal tal$ vue init webpack myproject

? Project name myproject
? Project description test
? Author yanhongchang <yanhongchang@100tal.com>
? Vue build standalone
? Install vue-router? Yes
? Use ESLint to lint your code? Yes
? Pick an ESLint preset Standard
? Set up unit tests No
? Setup e2e tests with Nightwatch? Yes
? Should we run `npm install` for you after the project has been created? (recommended) npm

   vue-cli · Generated "myProject".

```



构建完成后会提醒如何运行项目

```
To get started:

  cd myProject
  npm run dev
```



项目跑起来后根据提示可以直接浏览器访问http://localhost:8080直接访问



五、eagle-ui项目介绍

1、目录介绍

```
eagle-ui
|-- README.md
|-- build				// 项目打包目录，该目录下一般无需修改
|   |-- build.js
|   |-- check-versions.js
|   |-- logo.png
|   |-- utils.js
|   |-- vue-loader.conf.js
|   |-- webpack.base.conf.js
|   |-- webpack.dev.conf.js
|   `-- webpack.prod.conf.js
|-- common			// 公共目录，包括css通用样式、字体和图片等
|   |-- css
|   |-- fonts
|   `-- img
|-- config			// 项目配置目录
|   |-- dev.env.js
|   |-- index.js    		// 可以在这里配置前端项目启动端口
|   |-- prod.env.js
|   `-- test.env.js
|-- module				// module目录，可以包含多个不同的module
|   `-- resource		//resource 目录是我们云管平台的资源目录，存放页面vue文件，我们开发基本都在这个目录下
|-- node_modules		// npm 安装的依赖包
|-- dist		// npm 打包后的文件目录
|-- package-lock.json				// 包依赖文件
|-- package.json						// 包依赖文件
|-- proxy.json							// 代理文件，可以指定后端的api访问地址
|-- static
|   `-- xuesi_icon.ico			// 静态图标文件
`-- test										// 测试目录
    |-- e2e
    `-- unit

```



2、module/resource 目录拆解讲解

```
module/resource/
|-- index.html 				// index.html单页面文件
`-- src
    |-- App.vue				//	vue视图文件，通过该文件结合vue-router，通过<router-view>路由配置实现组件路由
    |-- assets				
    |   |-- img					// resource资产目录，包括通用jg和img
    |   |   |-- login_01.jpg
    |   |   |-- login_9.jpg
    |   |   |-- login_bg.png
    |   |   |-- xes.png
    |   |   `-- xuesi.ico
    |   `-- js
    |       |-- common.js		// 通用的js功能，例如cookie等操作
    |       `-- messageTip.js
    |-- components					//页面组件目录，包括头部、左侧导航和右侧展示
    |   |-- header.vue				// 页面header部分vue文件
    |   |-- header_new.vue			
    |   |-- left.vue					// 左侧导航vue文件
    |   `-- right.vue					//右侧展示vue文件
    |-- http
    |   `-- http.js						// 后端请求设置文件，可以通过axios.defaults.baseURL 设置后端API访问地址。	而且还包括
    													// 请求和响应拦截器
    |-- main.js								// vue 入口文件，vue实例app就是在这个文件初始化
    |-- pages									//具体页面文件目录
    |   |-- CommonView.vue		// 通用试图文件（也不太清楚这个的作用）
    |   |-- aliyun						// 阿里云相关资源页面文件
    |   |   |-- Server.vue
    |   |   `-- Volume.vue
    |   |-- cmdb							// cmdb任务推送和定时同步页面
    |   |   |-- cmdbTask.vue
    |   |   `-- syncTask.vue
    |   |-- index.vue					// 主vue试图文件，通过该文件将header、left、right进行拼接组合
    |   |-- index_bak.vue
    |   |-- login
    |   |   `-- index.vue			// 登陆页面
    |   |-- management				// 管理员页面目录
    |   |   |-- authority			// 权限相关
    |   |   |   |-- interfaceAuth.vue				// 接口限制
    |   |   |   `-- quota.vue								// 配额
    |   |   |-- plateform			// 平台管理相关
    |   |   |   |-- cloudBackend.vue			// 云后端资源
    |   |   |   |-- cloudBackendPro.vue		// 云后端项目
    |   |   |   `-- cloudType.vue					// 云后端类型
    |   |   |-- resource									// 全局资源管理
    |   |   |   |-- allServer.vue
    |   |   |   `-- allVolume.vue
    |   |   `-- user					// 用户管理
    |   |       |-- departureUsers.vue
    |   |       |-- userGroup.vue
    |   |       `-- users.vue
    |   |-- openstack					// openstack资源目录
    |       |-- devResource		// 开发机
    |       |   `-- Server.vue
    |       |-- myResource		//我的资源
    |       |   |-- Server.vue
    |       |   `-- Volume.vue
    |       `-- opsResource		// 运维组资源
    |          |-- Server.vue
    |           `-- Volume.vue
    |-- router				
    |   `-- index.js		// 页面路由文件，配置导航按钮到对应组件
    |-- service
    |   `-- service.js	// 定义访问路径常量
    `-- store					// 	vue 本地共享存储
        |-- common
        |   `-- index.js			// 可以存储所有组件都可以使用的通用数据。例如backend等
        `-- index.js	
```



3、运行项目

​		当我们想要将vue项目跑起来，我们可以通过以下方式通过开发方式运行

```
cd eagle-ui
npm run dev --dir=resource
```

​		--dir 可以指定需要运行的module，这里我们运行resource。

​		这里需要主要的是，运行项目需要修改访问的后端，具体修改在module/resource/src/http/http.js

```
axios.defaults.baseURL = 'http://xcmp.xesv5.com:8888'
// axios.defaults.baseURL = 'api'

调整为：
// axios.defaults.baseURL = 'http://xcmp.xesv5.com:8888'
axios.defaults.baseURL = 'api'

这里的api会根据/eagle-ui/proxy.json配置的地址访问对应后端。
```

如果不调整，并且本地绑定了hosts，就直接访问到线上环境的后端eagle项目了，开发方式运行无比修改为本地开发的eagle api地址。



4、打包项目

​	如果想要部署到线上环境，需要对项目打包，打包方式也很简单

```
cd eagle-ui/
npm run build --dir=resource
```

然后就会在项目目录下多处一个目录dist，打包后的文件就放在这个目录下

```
dist
|-- resource
|   |-- index.html
|   |-- static
```

得到打包后的文件后，将index.html和static目录拷贝到生产环境的/usr/share/nginx/html/既可。不过打包时切记将http.js中的axios.defaults.baseURL修改回http://xcmp.xesv5.com:XXXX 方式，否则无法正常请求后端。





参考：

1、https://cloud.tencent.com/developer/article/1020337

2、https://cn.vuejs.org/v2/guide/installation.html

3、https://segmentfault.com/q/1010000008451764

4、https://element.eleme.cn/#/zh-CN/component/table

