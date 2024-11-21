---
title: 【说说】使用 Artitalk 新增说说功能
date: 2024-11-20 21:27:21
tags: 
  - Artitalk
  - LeanCloud
categories: 建站日志
---

{% note info flat%}
Artitalk 是基于 LeanCloud（快速搭建后端云服务）的可实时发布说说的 js
{% endnote %}

# 前提

{% note danger modern%}

1. 你需要有一个域名

2. [LeanCloud](https://console.leancloud.app/) 国际版已不支持中国大陆，需要代理才能访问到说说

3. [LeanCloud](https://console.leancloud.app/) 大陆版需要一个备案的域名，最好准备 SSL 证书

{% endnote %}



# LeanCloud 相关准备

直接按照 Artitalk.js 官方文档完成操作即可：

{% link https://artitalk.js.org/doc.html#hexo-theme-volantis,Artitalk.js,,基于的可实时发布说说的 js, %}

1. 前往 [LeanCloud 国际版](https://leancloud.app/)，注册账号。
2. 注册完成之后根据 LeanCloud 的提示绑定手机号和邮箱。
3. 绑定完成之后点击`创建应用`，应用名称随意，接着在`结构化数据`中创建 `class`，命名为 `shuoshuo`。**【结构化数据名字一定要为shuoshuo，后续Artitalk才能识别到！】**
4. 在你新建的应用中找到 `内建账户` 下的 `用户管理`。点击 `添加用户`，输入想用的用户名及密码。
5. 回到`结构化数据`中，点击 `class` 下的 `shuoshuo`。找到权限，在 `Class 访问权限`中将 `add_fields` 以及 `create` 权限设置为指定用户，输入你刚才输入的用户名会自动匹配。为了安全起见，将 `delete` 和 `update` 也设置为跟它们一样的权限。
6. 然后新建一个名为 `atComment` 的class，权限什么的使用默认的即可。**【同第三点】**
7. 点击 `class` 下的 `_User` 添加列，列名称为 `img`，默认值填上你这个账号想要用的发布说说的头像url，这一项不进行配置，说说头像会显示为默认头像 —— Artitalk 的 logo。
8. 在最菜单栏中找到设置-> 应用 keys，记下来 `AppID` 和 `AppKey` ，一会会用。
9. 最后将 `_User` 中的权限全部调为指定用户，或者数据创建者，为了保证不被篡改用户数据以达到强制发布说说。



## 大陆版域名补充

国际版国内加载不了，如果使用的是大陆版本，需要在域名绑定中绑定一个已有域名，这个域名在后面的配置中有用：

![image-20241120222700201](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241120222700201.png)

同时需要去服务商进行 DNS 配置，按照图片提示：

![image-20241120222800480](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241120222800480.png)



# ButterFly 主题安装插件和配置

## 安装插件

butterfly 主题已经整合了 Artitalk，直接使用 npm 命令安装：

```bash
npm install hexo-butterfly-artitalk
```



## 默认配置

官方文档采用直接在 `_config.yml` 文件中配置（**采用这种方法就不需要手动创建页面，只需要填写配置即可完成**）：

```yml
  # Artitalk
  # see https://artitalk.js.org/
  artitalk:
    enable: true
    appId: LeanCloud appId
    appKey: LeanCloud appKey
    path: Artitalk 的路径名称（默认为 artitalk，生成的有页面为artitalk/index.html）
    js: 更换 Artitalk 的 js CDN（默认 https://cdn.jsdelivr.net/npm/artitalk）
    option: Artitalk 可选配置
    front_matter: Artitalk front_matter 配置
```

需要注意：国内无法访问 `jsdelivr cdn`，需要更换成 `https://unpkg.com/artitalk`！



## 我的配置

考虑到官方配置方法无法关闭当前页面评论（有需要的话），我采用自己建立页面的方式。

首先新建一个页面，在博客根目录文件夹打开 CMD：

```bash
hexo new page 页面名
```

填入下面内容，采用md文件编写的方法（会暴露key，官方提供 `Artitalk_SafeMode` 工具解决安全性）：

```markdown
---
title: 动态说说
date: 2024-11-20 20:16:10
type: talking
comments: false
---

<!-- 引用 artitalk -->

<script type="text/javascript" src="https://unpkg.com/artitalk"></script>

<!-- 存放说说的容器 -->

<div id="artitalk_main"></div>
<script>
new Artitalk({
    appId: '', // Your LeanCloud appId
    appKey: '', // Your LeanCloud appKey
    serverURL: '大陆版需要配置，国际版不需要',
    pageSize: 10,
    color1: 'linear-gradient(60deg, #ffd7e4 0, #c8f1ff 93%)',  
    color2: 'linear-gradient(60deg, #ffd7e4 0, #c8f1ff 93%)',  
    color3: '#4c4948'
})
</script>
```



## 效果展示

后续在访问前面建好的页面就可以看到说说啦。

![image-20241120223851121](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241120223851121.png)

