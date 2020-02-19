---
title: hello world
date: 2019-09-07 09:25:00
author: bao ge
categories: hexo
tags:
  - Typora
  - Markdown
---



Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post（在博客根目录下）

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/deployment.html)

**

### 常见命令

```java
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #部署到GitHub
hexo help  # 查看帮助
hexo version  #查看Hexo的版本

缩写：

hexo n == hexo new
hexo g == hexo generate
hexo s == hexo server
hexo d == hexo deploy

组合命令：
hexo s -g #生成并本地预览
hexo d -g #生成并上传
```



### hexo 备份方式：

​	1、github的hexo仓库创建hexo分支，将源文件上传至这个分支；

​	2、在另外电脑也可同步操作这个博客库，git clone 博客库，在hexo分支开发，先执行npm install......