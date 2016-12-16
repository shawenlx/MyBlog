---
title: 『Hexo』使用Hexo搭建GitHub博客
date: 2016-12-01 09:15:38
tags: daily
categories: daily
---

# 『Hexo』使用Hexo搭建GitHub博客

------

### 前言：

​   谈起这个博客的由来，真的挺不容易的。一年前兴奋地买了域名，后来因为很多东西都不具备实现这个模版博客的能力，遂搁置了域名，选择了简书。有点笨，这一次也折腾了两三个晚上，总算把它搭建起来了。

------
<!--more-->

### 环境准备

- git搭建 ：[git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/)
- node.js :  [node.js download](http://nodejs.org/download/)
- git主要用于将代码同步到GitHub平台，node.js主要用到了npm这个包管理器，方便我们快速搭建Hexo环境。
- 个人的主机环境为：MacOS

------

### 注册Github

- 我们的主要目的就是把本地的静态网站托管到GitHub这个平台上。
- **Tips: 我遇到的一个坑就是，我用的是QQ邮箱注册，GitHub验证邮件被拦截了，貌似163邮箱也会拦截。一定要验证邮箱，否则即使你博客搭建好了，托管到GitHub上面，也会一直404无法打开。**

------

### Quick Start

- 安装Hexo

```sh
sudo npm install hexo-cli -g
```

Hexo官方文档：[Hexo](https://hexo.io/)

- 利用Hexo初始化一个博客文件夹

```sh
hexo init [文件夹]
```

- [`npm install`](https://docs.npmjs.com/cli/install) 命令用来安装模块到`node_modules`目录。

```sh
cd [文件夹]            //进入到博客文件夹
npm install           //利用npm install命令安装模块node_modules
```

关于npm的详细使用，参照文档，或者[npm 模块安装机制简介](http://www.ruanyifeng.com/blog/2016/01/npm-install.html)。

至此，一个博客已经搭建好了，但是我们还没办法看到它。接下来就利用Hexo命令来启动我们的博客。

```sh
hexo g              //利用Hexo 生成静态页面
hexo s              //启动本地服务
```

你一定迫不及待想看看这个东西给你带来了什么样的惊喜。打开浏览器，http://localhost:4000，你就可以看到一个模版博客已经出现在你面前了，虽然这不是你最终想要的，不过也算是让我们眼前一亮。

命令倒是执行了几个了，但是我们不知道它到底做了些什么。先来看看吧。

```
ls      //查看文件目录， 当然一些隐藏文件无法查看， 没关系，ls -al即可。
```

暂时忽略其他文件是干什么的，先看看 `_config.yml`，这是我博客的设置。

```sh
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: Shawenlx                             #博客标题
subtitle:                                   #副标题
description: I love code and share.         #个人描述
author: Liuxi                               #作者名
language: en                                #站点语言，如果是中文，把en替换成zh-CN
timezone:                                   #同步时区

# URL
url: /
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory 文件目录对应生成的文件夹目录
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:                      #代码高亮
  enable: true
  line_number: true
  auto_detect: false
  tab_replace:

# Category & Tag
default_category: uncategorized
category_map: categories
tag_map:

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination    
## Set per_page to 0 to disable pagination 设置分页
per_page: 10
pagination_dir: page

# Extensions
theme: next                 #主题模版，搜索 ./themes 文件夹下的主题，此处我用的NexT模版

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/shawenlx/shawenlx.github.io.git
  branch: master

```

**Tips**: 当我们着手修改这个文件的时候，字段名以及`:`后面，切记一定要加**一个空格！**我一开始也遇到这个问题，执行`hexo g`命令的时候，就会报错提示我`_config.yml`文件无法解析。

------

### 将博客托管到Github平台

- 创建一个GitHub仓库，[`new repository`](https://github.com/new)
- 在 *Repository name* 栏中，输入 `your user name.github.io`，譬如我的GitHub用户名是shawenlx，所以就建立工程名为：[`shawenlx.github.io`](https://github.com/shawenlx/shawenlx.github.io)
- 配置好本地的SSH Key。
- 拥有了个人的GitHub Pages后，再回到Hexo的`_config.yml`文件, 参照我的配置保存即可。

```sh
deploy:
  type: git
  repo: https://github.com/shawenlx/shawenlx.github.io.git
  branch: master
```

- 最后利用hexo命令，将博客推送到Github上。

```sh
hexo clean              #推送前，先清除下本地数据库，防止缓存导致上传不成功
hexo d -g               #利用这个组合命令，将博客上传到github上。
```

- 当你打开浏览器，输入`your_user_name.github.io`，就可以看到你托管到平台上的数据了。

------

### 关于博客主题模版

- 网上有各种各样的模版，我用的[NexT](http://theme-next.iissnan.com/), 参照官方文档进行配置就可以了。目前博客还在完善，特此记录本博客的搭建经历。

------

### 关于管理博客内容

- 参照[Hexo官方文档](https://hexo.io/)

------

### 最后

以后简书会同步更新这个博客的内容。

**复习本博客链接：[shawenlx.github.io](shawenlx.github.io)**

