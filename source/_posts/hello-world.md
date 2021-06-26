---
title: Hello World
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

<!-- more -->

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Clean older history

```bash
$ hexo clean
```

### Run server

``` bash
$ hexo server or hexo s
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate or hexo g
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy or hexo d
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

### 添加分类和标签
```xml
categories: 分类名
tags:
- tag1
- tag2
```
### 部署打包方式
```xml
hexo clean && hexo g && hexo d
```
### 博客的同步与发布
```xml
github分支存在两个：
* 默认分支为hexo,用于使用git管理内容
* main分支为发布分支
```

### 分割符
```xml
<!-- more -->
用于文章内容分割，more下面的内容默认不显示，通过点击阅读全文显示
```