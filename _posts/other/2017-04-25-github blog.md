---
layout: post
title: 使用github搭建个人博客
categories: other
description: 使用github搭建个人博客
keywords: github，搭建，博客
---

[GitHub](https://github.com/)，是全球最大的社交编程及代码托管网站，一个拥有无数多开发者得社区。本教程使用github page搭建一个博客，开始动手搭建自己得博客吧！

## github博客概述
为啥选择github写博客呢？github本身版本控制非常方便进行博客编辑，而且还能添加社交评论，分享等功能。[Jekyll](http://jekyllcn.com/)是一个简单的免费的blog生成工具，类似wordpress。但是和WordPress又有很大的不同，原因是jekyll只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如Disqus。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。

## 搭建博客
一般的教程是在本地搭建好jekyll环境，之后一步一步的使用github搭建博客。但是，这里我没有在本地搭建环境。所以，如果想在本地搭建环境的，需要去其他地方找资料了。 步骤流程参考[Github Page](https://pages.github.com/) ，这里只是简单的介绍下！

### 新建项目
首先需要注册一个github账号。注册成功之后新建一个项目，如下图：
![新建项目](/images/posts/creategithub.png)
**项目名称必须是[用户名+github.com]或[用户名+github.io]**
之后可以设置该项目的属性，在项目的右侧的Settings，可以设置允许有wiki等，注意GitHub Pages不要选择**automatic page generator**!。

### git下载
项目建立完毕，下载git工具，
- [mingw](http://mingw.org/)
- [git](https://www.git-scm.com/download/win)
- [github](https://desktop.github.com/)

以上是git工具，自行安装下载，我用的[mingw](http://mingw.org/)，安装完毕之后把刚刚新建的github项目clone下来。

     git clone https://github.com/handexing/handexing.github.com.git
     
之后在本地会看到已经clone下来的项目。

### 设置主题
可以去jekyll下载主题，jekyll自己收集了一下[主题](https://github.com/jekyll/jekyll/wiki/Sites)。可以点开看看效果，喜欢的可以点击旁边的source链接到github里面下载代码。当然你也可以直接把我的主题下载下来直接用，我的[主题](https://github.com/handexing/handexing.github.com.git).系在到本地可以看到有好多文件夹。首先先把.git文件以外的文件拷贝到自己的项目下。

如果想关注我的项目可以在[我的博客](https://github.com/handexing/handexing.github.com.git)右上角，点击**atch Star Fork**。

博客文件夹介绍：

- _config.yml 保存了站点的配置信息。
- _includes 该目录存放可以与_layouts和_posts混合,匹配并重用的文件。 这里可以使页面的页脚、评论功能。
- _layout 存放网页布局模板
- _posts 存放发表的博文，文件命名：YEAR-MONTH-DAY-title.md
- assets 存放博客使用的资源，例如css，js等
- index.html 博客首页

对于还有一些其他的文件，自行摸索摸索吧，有问题可以到[jekyll](http://jekyllcn.com/docs/home/)官网看看。

### 设置博客
站点的配置信息在_config.yml中，打开如下图：
![配置博客](/images/posts/config.png)

把站点信息改成自己的就行了，**这个里面有用到第三方评论插件，要自己申请账号，不能用我的账号**！

### 提交
如果都修改好了，就可以提交代码了，需要使用一下几个命令：

     git add --all #添加所有文件
     git commit -m"init project" #提交写的备注
     git clone push #上传文件

### 验证博客
代码提交完就可以看看自己的博客了【https://你的用户名.github.io】，不出以外成功了，如果你没有成功，对着以上步骤在看看是不是遗漏了步骤。
我的：
![验证博客](/images/posts/blog.png)

## 结语
如果你跟着这篇不那么详尽的教程，成功搭建了自己的博客，恭喜你！剩下的就是保持热情的去写自己的文章吧。






