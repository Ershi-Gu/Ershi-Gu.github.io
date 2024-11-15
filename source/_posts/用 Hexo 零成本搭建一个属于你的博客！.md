---
layout: hexo
title: 用 Hexo 零成本搭建一个属于你的博客！
date: 2024-11-15 20:31:28
tags: 博客
---

# 1. 前言

本系列记录从 0 开始搭建一个博客的过程，低代码、零成本，是一款适合小白选择的方案。

首先介绍一下本次要用到的框架（*使用框架避免我们从头开始写代码，同时提供了更方便的功能* ）：**Hexo**，一款小巧高效的 **静态博客** 框架，可使用 Markdown 解析文章，并且有非常多漂亮的主题可供选择。

> 静态博客：指所有的页面内容都预先生成并存储为静态文件（HTML、CSS、JS等），服务器只负责直接返回这些文件给用户，没有后端数据库和动态内容生成。
>
> - 优点：高性能、低成本、易于维护
> - 缺点：缺乏用户之间的互动、不够灵活
>
> 动态博客：指依赖后台数据库和服务器端技术（如 PHP、Node.js、Java、Python 等）来动态生成页面内容，页面内容是实时从数据库中获取并渲染的。
>
> - 优点：高灵活性、高互动性、动态更新、丰富的用户交互
> - 缺点：代码量大、维护成本高、安全风险

看完静态博客和动态博客的区别，会不会有朋友问为什么不选择动态博客呢，想CSDN、博客园那样的论坛多酷。哥们，静态的好维护啊，都不需要折腾服务器环境，所以综上我们还是选择静态的方法来搭建个人博客。



本套方案只需要几行命令就能生成一个博客框架，并且基于 github 部署即可。无需关心网站代码实现细节，只需要专心写文章！

**第一部分的教程分为下面两个部分：**

- （1）Hexo 的基本搭建以及 github page 部署
- （2）Hexo 的基本配置以及主题更换



# 2. Hexo 搭建

## 2.1 安装 Node.js 和 Git

Hexo 基于 Node.js 环境，所以需要先进行一下安装。

这里就不赘述了，贴一下菜鸟教程：[Node.js 安装配置 | 菜鸟教程](https://www.runoob.com/nodejs/nodejs-install-setup.html)。

*这里建议安装完 Node.js 后同时去配置一下国内镜像源，不然后面一些安装步骤的时候可能会太慢了（朋友自己百度一下：)* 



Git 安装：Git 是一个版本控制工具，可以用来拉去别人的代码。

廖雪峰的教程：[安装Git - Git教程 - 廖雪峰的官方网站](https://liaoxuefeng.com/books/git/install-git/index.html)



## 2.2 安装 Hexo

Hexo 官网：[Hexo](https://hexo.io/zh-cn/)。

在安装好 git 和 node.js 后，就可以安装 Hexo了。首先创建一个项目文件夹Hello-Blog（名字随意），用于存放博客文件，在该文件下打开 CMD。

（1）首先安装 Hexo：

```bash
npm install -g hexo-cli
```

（2）验证是否成功

```bash
hexo -v
```

![image-20241115231649669](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115231649669.png)

看到上面这个内容，就说明 Hexo 安装成功了。



## 2.3 初始化建站

安装 Hexo 完成后，在前面建立的文件夹下（Hello-Blog）执行下面的命令，将会进行框架的初始化建立：

```bash
hexo init 项目名 # 初始化项目
npm install # 安装必备组件
```

初始化后，项目文件夹如下：

```xml
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes	
```

接着可以通过下面的命令启动本地服务器，看到本地的效果啦：

```bash
hexo g # 生成静态网页
hexo s # 启动本地服务器
```

![image-20241115232404145](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115232404145.png)



# 3. GitHub 部署站点

上一节我们已经在本地建设博客成功了，这一节会通过 GitHub pages 实现网站的在线访问。

> **GitHub Pages** 是 GitHub 提供的一项静态网站托管服务，允许用户通过 GitHub 仓库直接托管和发布静态网页。它可以方便地将个人、项目或组织的网页部署到互联网上，而无需额外的服务器或复杂的设置。



## 3.1 创建个人仓库

在创建该项目仓库时，需要注意仓库名必须按照 **`用户名.github.io`** 的形式才能自动将该仓库的文件托管到 GitHub Pages，同时需要注意免费账户**仓库需要公开**。

![image-20241115232924106](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115232924106.png)



## 3.2 部署 Hexo 到 GitHub

这一步，就是需要将本地的 Hexo 文件和 GitHub 关联起来。

Hexo 官方提供两种部署方式：

- **一键部署：**[一键部署 | Hexo](https://hexo.io/zh-cn/docs/one-command-deployment)。通过 hexo-deployer-git 一键部署

- **GitHub Actions 手动上传**:[在 GitHub Pages 上部署 Hexo | Hexo](https://hexo.io/zh-cn/docs/github-pages)。相比于上面的一键部署，这个需要手动 git 提交，但是可以**写注释**，我更喜欢这个。



这里就用 GitHub Actions 的方式。首先将本地的 Hexo 推送到 GitHub 上：

（1）初始化本地仓库

```bash
git init
```

（2）创建本地 main 分支

```bash
git checkout -b main
```

（3）提交代码到本地仓库缓存

```bash
git add .
git commit -m "第一次提交 demo"
```

（4）连接本地和远程仓库

首先拿到远程仓库地址。

![image-20241115234642213](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115234642213.png)

接着将本地仓库和 GitHub 仓库关联。

```bash
git remote add 仓库名(自己写一个，就是用于记录远程仓库的名字的) <GitHub仓库的URL>
```

（5）推送内容

```bash
git push -u 仓库名(自己写一个，就是用于记录远程仓库的名字的) main
```



## 3.3 创建 Workflow

我们在刚刚创建的仓库中，选中设置，进入 Pages 选项，选择构建和部署方式为 GitHub Actions，创建 workflow。如下图：

![image-20241115234931318](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115234931318.png)

接着先暂停这一步，我们要去查看一下本地的 Node.js 版本。

```bash
node -v
```

![image-20241115235117817](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115235117817.png)

记住上面这个版本号。

![image-20241115235518848](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241115235518848.png)

我们在 workflow 里面将这里的版本号改成上面记录下来的版本号。以下是 workflow 文件的全文：

```tex
name: Pages

on:
  push:
    branches:
      - main # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # If your repository depends on submodule, please see: https://github.com/actions/checkout
          submodules: recursive
      - name: Use Node.js 20
        uses: actions/setup-node@v4
        with:
          # Examples: 20, 18.19, >=16.20.2, lts/Iron, lts/Hydrogen, *, latest, current, node
          # Ref: https://github.com/actions/setup-node#supported-version-syntax
          node-version: "20.12.0"
      - name: Cache NPM dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```



## 3.4 后续操作

在完成上面操作后，我们就可以通过 **`用户名.github.io`** 访问到项目啦。

后续项目有更新就只需要重复上述提交操作即可。



# 4. Hexo 基本配置

可以在根目录的 **`_config.yml`** 中修改大部分的网站配置。具体配置参考官方文档：[配置 | Hexo](https://hexo.io/zh-cn/docs/configuration)。



# 5. Hexo 主题更换

## 5.1 主题选择

接下来就到了最喜欢的部分了，谁不喜欢把自己的网站打扮的 very 好看呢。

[Themes | Hexo](https://hexo.io/themes/)：Hexo 提供了第三方的主题网站，可以在里面选择自己喜欢的主题进行稍后的部署。或者直接去 GitHub 搜索关键词 **Hexo Theme** 也可以。（*比如下面的 GitHub start 前三的主题*）

![image-20241116004201157](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116004201157.png)

这里我选择一个个人比较喜欢的 **hexo-theme-butterfly** 主题，创作者链接：[jerryc127/hexo-theme-butterfly: 🦋 A Hexo Theme: Butterfly](https://github.com/jerryc127/hexo-theme-butterfly)。



## 5.2 ButterFly 主题安装

（1）拉取主题文件

首先根据作者文档（[Butterfly 文檔(一) 快速開始 | Butterfly](https://butterfly.js.org/posts/21cfbf15/)）进行主题文件拉取。在 Hexo 项目根目录：

```bash
git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
```

（2）应用主题

修改 Hexo 根目录下的 _config.yml，把主题改为 butterfly。

```yaml
theme: butterfly
```

（3）安装插件

如果你没有 pug 以及 stylus 的渲染器，请下载安装，在项目根目录执行。

```bash
npm install hexo-renderer-pug hexo-renderer-stylus --save
```



**参考建议：**

在 hexo 的根目录创建一个文件 `_config.butterfly.yml` ，并把主题目录的 `_config.yml`  内容复制到 `_config.butterfly.yml`  去。

> 注意:
>
> 复制的是主题的 _config.yml ，而不是 hexo 的 _config.yml
>
> 不要把主题目录的 _config.yml 删掉
>
> 以后只需要在 _config.butterfly.yml 进行配置就行。如果使用了 _config.butterfly.yml， 配置主题的 _config.yml 将不会有效果。
>
> 作者: Jerry
> 連結: https://butterfly.js.org/posts/21cfbf15/#%E5%8D%87%E7%B4%9A%E5%BB%BA%E8%AD%B0
> 來源: Butterfly

Hexo 会自动合并主题中的 `_config.yml`  和 `_config.butterfly.yml`  里的

配置，如果存在同名配置，会使用 `_config.butterfly.yml` 的配置，其优先度较高。

![](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241116005253336.png)



## 5.3 常见问题

**（1）git 不了 themes 文件中的主题文件**

解答：由于上面使用的是 GitHub Actions 自动部署，在拉取完 ButterFly 文件后，记得把 themes/butterfly 中的 **.github** 文件删除，否则会干扰前面编写的工作流。
