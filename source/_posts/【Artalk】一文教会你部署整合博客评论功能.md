---
title: 【Artalk】一文教会你部署整合博客评论功能
date: 2024-11-18 13:00:27
tags: 
  - Artalk
  - Hexo
  - ButterFly
  - SSL
  - Docker
categories: 技术
---

# 前言

> 注意：在看本篇文章操作之前需要先准备一台**服务器**哦！

在写博客之前我也写过一些文章，觉得笔记和博客的区别就在于后者可以有更多的和读者互动的功能，评论就是中间很重要一个互动渠道。

本文采用 **Hexo + Butterfly + Artalk + 一台服务器**实现评论功能接入博客。该方案使用的 Artalk 和 Disqus、Gitalk 等其他评论框架不同的地方在于，Artalk 采用私有部署，在国内的服务器响应会更快。效果如下：

![image-20241118182842873](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118182842873.png)

# Artalk 部署

## 介绍

Artalk 是一款简单易用但功能丰富的评论系统，你可以开箱即用地部署并置入任何博客、网站、Web 应用。[👋 Hello Friend | Artalk](https://artalk.js.org/zh/guide/intro.html)

在我看来 Artalk 是一款轻量小巧但又不缺乏功能的系统，包括但不限于：社交登录、邮件通知、多元推送、图片上传、评论审核、markdown等等（*以上来自官方文档介绍*）



## Docker 部署

Artalk 提供多种部署方式[📦 程序部署 | Artalk](https://artalk.js.org/zh/guide/deploy.html)，我觉得其中最简单就是 Docker 了，一个命令直接解决。

首先你需要在服务器上安装一个 Docker（这我就不详细介绍了，没有得小伙伴可以去网上搜搜）。

然后新建一个文件夹用于存放 Artalk 文件（`/root/artalk` )，然后执行下面的命令，只需要修改中文提示的地方：

```bash
cd artalk
docker run -d \
    --name artalk \
    -p 服务器端口:23366 \
    -v $(pwd)/data:/data \
    -e "TZ=Asia/Shanghai" \
    -e "ATK_LOCALE=zh-CN" \
    -e "ATK_SITE_DEFAULT=站点名" \
    -e "ATK_SITE_URL=站点URL" \
    --network artalk \
    artalk/artalk-go
```

> 服务器端口：需要开放该端口的防火墙，是服务器的本机实际端口
>
> 站点名：Artalk 中可以部署多个站点（可以区分不同项目），你只需要为当前的这个项目写一个站点名即可。
>
> 站点URL：连接到站点的 URL，如果有域名就写域名，没有域名就写服务器IP+端口，比如：http://12345:上面的服务器端口 或 https://hello-life.com（后面会教大家获取https证书）。

接着输入命令创建管理员账户：

```bash
docker exec -it artalk artalk admin
```

浏览器输入 `http://站点URL` 进入 Artalk 后台登录界面。

![image-20241118184907846](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118184907846.png)

至此，你就完成了 Artalk 的后端搭建，很简单吧！Artalk 会在前面指定的文件夹生成几个文件，其中 artalk.yml 用于保存配置，我们可以直接在管理页面进行修改，不用修改文件。

![image-20241118185036875](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118185036875.png)



## 配置数据库

在使用前需要配置好数据库（存放评论、用户等信息），我使用的是 MySQL，可以参考如下：

![image-20241118233843584](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118233843584.png)

注意：如果 MySQL 也是使用的 Docker 部署，只需要将其和 Artalk 加入同一个 Docker 网络，配置中直接写容器名即可。



## 建议

如果你前端和服务器**都采用 http**，没有做 https 认证，可以直接跳到 Hexo 整合部分，进行配置。



# SSL 证书配置

## 介绍

Artalk 后端默认采用 http，如果前端是 https 的话，在请求 Artalk 时就会出现混合内容出错（*现代浏览器对 HTTPS 页面加载 HTTP 资源（如图片、脚本、CSS 等）视为不安全行为*），比如报错如下：

```text
Mixed Content: The page at 'https://example.com' was loaded over HTTPS, but requested an insecure resource 'http://example.com/script.js'. This request has been blocked; the content must be served over HTTPS.
```

我们就需要为 Artalk 配置一个有效的 SSL 证书，帮助他变成 HTTPS。

在这之前你可能需要 **一个域名**。网上有很多方法获取免费域名，这里我就用我在腾讯云买的一个域名做演示。



## Let's Encrypt 申请免费证书

**Let’s Encrypt** 是一个免费、自动化、开放的证书颁发机构。简单的说，借助 Let's Encrypt 颁发的证书可以为我们的网站免费启用 HTTPS(SSL/TLS) 。

Let’s Encrypt 的证书有效期为 **90 天**，需要定期续期。



## 安装 Certbot

Certbot 是一个命令行工具，用于自动化整个 SSL 证书的管理流程。



### Linux

首先查看自己的服务器是哪种系统。

Ubuntu/Debian：

```bash
apt update
apt install certbot
```

CentOS/RHEL：

```bash
yum install epel-release
yum install certbot
```



### 其他系统

参考 Certbot 官网：[Certbot](https://certbot.eff.org/)

![image-20241118210006804](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118210006804.png)





## 单域名 SSL 证书

如果你只需要为单个域名（如 `example.com` 或 `www.example.com`）申请证书，可以使用下面命令

```bash
certbot --nginx -d example.com -d www.example.com 
```

这需要服务器首先安装好 Nginx，Certbot 获取到证书后会自动修改 Nginx 的配置将证书装在 nginx 上。



你也可以选择使用手动申请证书的方式：

```bash
sudo certbot certonly --manual -d example.com
```

Certbot 会提示使用 HTTP-01 验证，要求你在站点根目录创建一个特定的文件，用于验证域名的所有权。示例步骤：

- 提示以下内容

  ```bash
  Create a file containing the following content:
  ABC12345...
  
  And make it available on your web server at:
  http://example.com/.well-known/acme-challenge/XYZ12345...
  ```

- 在你的站点目录（根据 Nginx 域名配置的指定文件夹，如 `root /usr/share/nginx/html/域名`）下创建 `.well-known/acme-challenge` 目录。
- 完成验证之后， Certbot 就会生成证书
  - 证书文件：`/etc/letsencrypt/live/example.com/fullchain.pem`
  - 私钥文件：`/etc/letsencrypt/live/example.com/privkey.pem`



## 泛域名 SSL 证书

泛域名证书可以为同一主域名下的所有子域名提供 HTTPS 支持。例如，`*.example.com` 可以覆盖 `blog.example.com`、`api.example.com` 等子域名。

Let’s Encrypt 要求通过 **DNS-01** 验证来申请泛域名证书，运行下面的命令：

```bash
sudo certbot certonly --manual --preferred-challenges dns -d "*.example.com" -d example.com
```

运行命令后，Certbot 会生成类似下面的信息：

```yaml
Please deploy a DNS TXT record under the name:
_acme-challenge.example.com
with the following value:
AbCdEfGhIjKlMnOpQrStUvWxYz1234567890
```

这时候前往你的**域名 DNS 管理页面**，添加 TXT 记录，如下：

![image-20241118235308700](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118235308700.png)

注意主机记录名就是 `_acme-challenge` 不需要加后面的域名，Certbot 可能会连续要求两次添加，就像图片里一样，每次添加完按 `Enter` 即可。最后等待几十秒，确保 DNS 传播开来，就会颁发证书到下列位置，这是可以删除 DNS 解析记录了。

- **证书文件**：`/etc/letsencrypt/live/example.com/fullchain.pem`

- **私钥文件**：`/etc/letsencrypt/live/example.com/privkey.pem`



## Artalk SSL 配置

Artalk 管理页面提供 SSL 直接配置，如果你不需要采用 Nginx 代理，可以直接在这里配置。

![image-20241118223041622](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118223041622.png)

由于我们前面使用的是 Docker 部署的 Artalk，同样需要将外部的证书文件映射到容器内部，可以参考 **第4.2节-Docker 容器重新挂载数据卷**。



# Nginx 配置

如果你只有一个服务器，并且需要转发请求到多个项目上，可以使用 Nginx 进行反向代理转发到 Artalk。



## 配置文件

注意下面配置文件中的域名要和 Artalk 部署时的 **站点URL** 相同。

```shell
# Artalk 评论
# 配置 HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name 域名;
    return 301 https://$host$request_uri;
}

# 配置 HTTPS 服务
server {
    listen 443 ssl;
    server_name 域名;

    ssl_certificate /etc/nginx/certificates/fullchain.pem;
    ssl_certificate_key /etc/nginx/certificates/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;

	location / {
        proxy_pass http://Artalk容器名:23366/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

关于已有 Nginx 容器如何加入证书数据卷的问题看下一节。



## Docker 容器重新挂载数据卷

如果你是使用的 Docker 部署的 Nginx，由于配置文件中需要引用到证书文件，但是证书文件又在容器外面，这时候就需要重新挂载数据卷了。

Docker 不支持已有容器修改数据卷，我们需要重新开启一个容器。不用担心，容器删除，原先挂在的数据卷本地还会存在，你只需要停止、删除原先 Nginx 容器，重新挂载启动一个即可：

```bash
docker stop nginx
docker rm nginx

docker run -d \
    --name nginx \
    -p 80:80 -p 443:443 \
    -v /etc/letsencrypt/live/example.com:/etc/letsencrypt/live/example.com:ro \
    -v /etc/letsencrypt/archive/example.com:/etc/letsencrypt/archive/example.com:ro \
    -v /path/to/nginx/conf:/etc/nginx/conf.d \
    nginx
```

如果不放心的话，你可以提前备份一个当前容器的配置：

```bash
docker commit <容器名称或容器ID> nginx-backup
```

此命令会创建一个名为 `nginx-backup` 的镜像，以备恢复。



## 跨域问题踩坑

假设博客的网址为 `https://www.example.com`，在请求 Artalk `https://artalk.example.com` 时，会出现跨域问题如下：

![image-20241118230131919](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118230131919.png)

这是因为在 `https://www.example.com` 页面请求一个不同源的资源引起的。跨域问题指的是当一个网页发起的请求与当前网页所在的源（协议、域名和端口）不同而被浏览器阻止的现象。这种限制是由浏览器的**同源策略（Same-Origin Policy，SOP）**引起的，它是一种重要的安全机制，旨在防止恶意网站窃取或篡改用户的敏感数据。



在这里我们可以采用配置配置 CORS 的方法，通常会在 Nginx 中配置，但是 Artalk 后端提供设置可信域名，在文档中介绍其是通过控制 `Access-Control-Allow-Origin` 实现的，也就是会影响跨域。

![image-20241118232540636](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118232540636.png)

经实践检验，如果在 **Nginx 配置跨域，Artalk 不设置**，在请求一些类似于上传图片，发送评论的 api 时会出现重复 CORS 的情况，所以 Nginx 还是采用上面的配置，**只在 Artalk 设置跨域！！**

![image-20241118232723042](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118232723042.png)



# 置入博客

Artalk 官方对置入博客提供文档（如果没有用到下面提到的主题，直接看文档即可）：[置入博客 | Artalk](https://artalk.js.org/zh/develop/import-blog.html)。

这里采用 Hexo + ButterFly 主题 + Artalk 实现评论功能。首先打开 ButterFly 主题配置文件进行如下配置（如：`_config.butterfly.yml`）。

![image-20241118233529132](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118233529132.png)

在同文件中启用评论，并修改使用对象为 Artalk。

![image-20241118233617519](https://hello-life-1313120530.cos.ap-nanjing.myqcloud.com/blog/image-20241118233617519.png)

至此评论功能就引入成功了！



# 踩坑和解决方案

- 跨域问题？
  - 参考4.3节-跨域问题踩坑。
- 修改 Artalk 数据库配置失败，导致容器崩溃？
  - 需要先创建好数据库，Artalk 只能检测到已有数据库自动创建表，不能自动创建数据库。
- 评论显示出来了，但是提交评论或者名字为中文导致提交失败？
  - 这是因为 Artalk 在自动生成表时，虽然配置里写了utf8mb4，但是表结构却不是，不支持中文，需要手动修改 MySQL 表结构。