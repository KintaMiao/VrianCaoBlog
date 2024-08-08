---
title: 在宝塔面板上利用Docker和nginx搭建Sub-Store
tags:
  - 教程
  - Docker
  - Proxy
date: 2024-08-08 17:30
categories: 教程
---
> 本教程面向小白，尽可能简化了步骤，并在更加简单的宝塔面板上操作，不涉及复杂配置和进阶玩法（如通知功能和各种脚本）

# 什么是Sub-Store / Introduction

*这里摘取官方仓库的介绍*：
>Advanced Subscription Manager for QX, Loon, Surge, Stash and Shadowrocket.

为了尽可能规避一些风险，在这里不做过多介绍，想要详细了解可以自行Google
项目当前主要维护者：[小一xream](https://github.com/xream)
项目仓库：[sub-store-org](https://github.com/sub-store-org)/[Sub-Store](https://github.com/sub-store-org/Sub-Store)

# 前期准备 / Preparations

- 一个可直连的已安装bt面板的VPS
	- 已通过面板安装Docker套件和nginx
- 最好有一个域名

# 教程 / Tutorial

### 为Sub-Store创建存储目录

>Sub-Store需要一个目录来存储你的订阅和其他文件

首先进入你面板的`文件`选项卡，在你心仪的位置新建一个文件夹作为Sub-Store的存储目录
这里以`/etc/sub-store`为例，请在任意地方记下这个目录

> 小贴士：你可以点击路径栏快速获取当前目录的绝对路径，正如Windows一样

### 使用Docker部署Sub-Store

>宝塔面板默认**不开放3001端口**，因此需先在`安全`页面，新增端口规则，如图所示：

![端口规则](https://ice.frostsky.com/2024/08/08/277e8cbf3f9074e137eac33d72a38349.png "端口规则")

接下来进入`终端`页面

以下为一个标准代码
```Shell
docker run -it -d \
--restart=always \
-e "SUB_STORE_CRON=55 23 * * *" \
-e SUB_STORE_FRONTEND_BACKEND_PATH=/XXXXXxxxxx1234567890 \
-p YOURIP:3001:3001 \
-v /etc/sub-store:/opt/app/data \
--name sub-store \
xream/sub-store
```
接下来跟我一起修改：
1. 首先将`SUB_STORE_FRONTEND_BACKEND_PATH`的/后面改为**20位随机字符串（不含特殊符号）**，尽可能随机，这与其他应用中的API Token一样重要！
2. 将`-v`后面到`:`前的目录修改为刚才你选择的、要存储Sub-Store数据的目录
3. 将`-p`后的`YOURIP`按需修改
	1. 如果你是需要本地运行，本地使用，那么请修改为`127.0.0.1`
	2. 如果是需要远程访问，那么请修改为`服务器IP`
4. 目光看向最后一行，如果你有使用`HTTP-META`的需求，请将``xream/sub-store``修改为`xream/sub-store:http-meta`，如果你不知道什么是HTTP-META，请**不要修改**

接下来按下回车，你应该会看到一串容器ID，那就说明已经运行成功了
至此，Docker部署已经完成

>你可以访问 IP:3001 来看看是否运行正常
### 使用nginx反代(无域名可跳过)

>使用nginx进行反代，使你可以通过自己的域名进行访问

#### 配置&绑定

进入`网站`页面，选择`反向代理`标签页，点击`新增反代`

配置如图所示：
![反代设置](https://ice.frostsky.com/2024/08/08/99388e5a830c15657ef41e83b8f15163.png "反代设置")
将图中`sub.example.com`改为你自己想绑定的域名，目标中的`yourip`替换为你的服务器IP
此时，你的`发送域名(host)`一栏应该显示为`$http_host`，备注显示为你的域名
确认无误后，点击添加

>记得在你的域名提供商中将你绑定的域名 使用A记录 解析至你的服务器IP

#### 配置SSL证书

找到你刚刚配置的反代规则，点击右侧的`设置`，进入`SSL`选项卡，选择`Let's Encrypt`，选择域名，申请证书，随后页面应如图所示：
![SSL证书配置](https://ice.frostsky.com/2024/08/08/ae20095683f3a8b8859ac8171cfa7c54.png "SSL证书配置")

### 初次使用

接下来，你需要访问一个相对复杂的地址，以进行后端激活和绑定
现分两种情况：
1. 如果你没有使用域名，那么请访问 http://YOURIP:3001?api=http://YOURIP:3001/BACK_END_PASSWORD
2. 如果你绑定了自己的域名，那么请访问 https://sub.example.com?api=https://sub.example.com/BACK_END_PASSWORD
其中，将`YOURIP/DOMAIN`修改为你的`服务器IP或域名`，将`BACK_END_PASSWORD`修改为你设置的后端20位访问密钥

你应该可以看到`数据刷新成功！`的提示，进入设置->后端设置，你应该可以看到类似于下图的配置：
![后端示例](https://ice.frostsky.com/2024/08/08/6c86c546f127556236099f50df6819ab.png "后端示例")

>至此你已完成所有配置，享受Sub-Store带来的便利吧！
