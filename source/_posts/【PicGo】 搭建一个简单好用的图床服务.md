---
title: 【PicGo】 搭建一个简单好用的图床服务
date: 2024-11-16 10:40:12
tags: 
  - PicGo
  - 图床
  - COS
categories: 技术
---

# 前言

不知小伙伴是否有过这样的烦恼，在写 文章/笔记 的时候需要插入图片，但保存到本地读取即麻烦又占用内存，希望有一个管家为我们管理这些图片，并且支持云存储，只需要一个媒介即可在各个平台读取到图片~哎这就到我们今天的主题了——**图床**。

> **图床**，又称为图像托管服务，是一种在线服务，用于上传、存储和分享图片。用户可以将图片上传到图床后，获得一个链接，通过这个链接可以在其他地方（例如网站、论坛、博客或社交媒体）显示或分享这些图片。图床服务的目的是方便图片的管理与分享，特别是针对那些无法直接上传图片的平台。



本教程使用 **PicGo** + **腾讯云COS + Typora** 搭建图床服务。



# 腾讯云 COS 创建

> **COS** 通常指的是 **对象存储服务（Cloud Object Storage）**，是一种基于云计算技术的存储服务。

首先购买一个腾讯云 COS 的资源包。[COS 对象存储购买_COS 对象存储选购 - 腾讯云](https://buy.cloud.tencent.com/cos)

需要注意的是：在购买时，**存储容量包 **和 **流量包** 是不一样的，前者指的是存储的容量，而我们在访问存储的图片时产生的流量是需要额外计费的，可以通过流量包抵扣，如果没有的话，在一定程度内是免费，但是**超出限额就会产生费用**了（别被刷爆了XD）。



## 创建存储桶

在开通腾讯云 COS 后，我们来到下面这个页面。

![image-20241116132213120](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116132213120.png)

首先选择**创建存储桶**，存储桶就是真正存放我们图片的地方。

创建时**地域选择和自己接近**的即可，然后是访问**权限选择公有读私有写**，名称按照自己平常项目的命名习惯写一个即可，后续的操作全部不用配置：如下图：

![image-20241116132411062](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116132411062.png)

这样我们的一个 COS 存储桶就创建完毕了，可以选择手动往里面放数据，也可以使用后面会提到的方式，PicGo + Typora 自动存储获取 URL。

![image-20241116132700306](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116132700306.png)



# PicGo 设置

首先下载 PicGo：[PicGo](https://picgo.github.io/PicGo-Doc/)（一个用于快速上传图片并获取图片 URL 链接的工具）。

下载好后进入设置界面：

![image-20241116132900779](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116132900779.png)

COS 版本：选择 COS v5。

>- 设定 Secretld：开发者拥有的项目身份识别 ID，用于身份认证，可在 [API 密钥管理](https://console.cloud.tencent.com/capi) 页面中创建和获取。
>
>- 设定 SecretKey：开发者拥有的项目身份密钥，可在 [API 密钥管理](https://console.cloud.tencent.com/capi) 页面获取。  
>- 设定 Bucket：存储桶，COS 中用于存储数据的容器。有关存储桶的进一步说明，请参见 [存储桶概述](https://cloud.tencent.com/document/product/436/13312) 文档。
>- 设定 AppId：开发者访问 COS 服务时拥有的用户维度唯一资源标识，用以标识资源，可在 [API 密钥管理](https://console.cloud.tencent.com/capi) 页面获取。
>- 设定存储区域：存储桶所属地域信息，枚举值可参见 [可用地域](https://cloud.tencent.com/document/product/436/6224) 文档，例如 ap-beijing、ap-hongkong、eu-frankfurt 等。
>- 设定存储路径：图片存放到 COS 存储桶中的路径。
>- 设定自定义域名：可选，若您为上方的存储空间配置了自定义源站域名，则可填写。相关介绍可参见 [开启自定义源站域名](https://cloud.tencent.com/document/product/436/36638)。
>- 设定网址后缀：通过在网址后缀添加 COS 数据处理参数实现图片压缩、裁剪、格式转换等操作，相关介绍可参见 [图片处理](https://cloud.tencent.com/document/product/436/54049)。

按照上述的操作手册，进入腾讯云获取到各个参数后，填入设置即可。

如果说你平常用不到 MarkDown，那么到这一步，使用 PicGo 作为图床工具的搭建已经成功了，快去使用吧！如果有需求则继续往下看。



# Typora 设置

在 Typora 的偏好设置（*左上角文件点开下拉表中* ）处选择图像：

![image-20241116133156603](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116133156603.png)

- 插入图片时，选择**上传图片**
- 服务设定选择刚刚下载的**PicGo**

之后重启 Typora 即可使得设置生效。这下在我们粘贴图片到 Typora 时，就会自动上传图片到 COS，并返回 COS 图片链接（可以直接移植到各个平台！）。

![image-20241116133446727](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116133446727.png)
