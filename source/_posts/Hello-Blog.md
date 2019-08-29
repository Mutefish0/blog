---
title: Hello Blog
date: 2019-08-30 00:16:44
tags:
---

这篇文章的主题是如何快速搭建一个博客。我采用的搭建博客的方式是Github Pages + Hexo，可以快速免费的搭建一个博客。

使用[Github Pages](https://pages.github.com/)来免费托管网页源代码：
> GitHub Pages 是一种静态站点托管服务，旨在直接从 GitHub 仓库托管您的个人、组织或项目页面。

使用[Hexo](https://hexo.io/zh-cn/index.html)来生成静态页面：
> Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

## Getting Started

### 创建Github Pages仓库
我的Github用户名为`mutefish0`，然后创建一个名为`mutefish0.github.io`的仓库，后续将网页文件推送至该仓库后，访问`https://mutefish0.github.io`即可查看最新的个人网站。

### 安装Hexo
安装Hexo需要保证已经安装了[Node.js](http://nodejs.org/)和[Git](http://git-scm.com/)
`$ npm install -g hexo-cli`

### 初始化项目
`$ cd ~/workspace`
`$ hexo init blog`
`$ cd blog`

### 写一篇文章
`$ hexo new "Hello World"`
命令执行完后，会生成文件`source/_posts/Hello-World.md`，然后随意编辑该文件并保持

### Local Server
`$ hexo server`
命令执行完后，访问http://localhost:4000 即可看到主页已经刚刚发布的文章。继续编辑md文件，然后刷新浏览器，就可以看到最新的内容。

### 生成静态网页代码
`$ hexo generate`
执行完后会生成一个`public`文件夹

### 部署到个人网站
将上个步骤生成的`public`文件下的所有内容，都上传到`mutefish0.github.io`仓库。
至此已经完成博客的搭建。访问`https://mutefish0.github.io`即可查看。

## 自动化部署
安装git部署脚本
`$ npm install hexo-deployer-git --save`
修改配置文件`_confog.yml`
```
deploy:
  type: git
  repo: https://github.com/Mutefish0/Mutefish0.github.io
  branch: gh-pages
```
后续只需运行以下命令即可自动更新个人主页
`hexo clean && hexo deploy` 
