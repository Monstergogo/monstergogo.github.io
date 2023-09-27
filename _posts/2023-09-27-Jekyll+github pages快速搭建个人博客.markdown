---
title: Jekyll+github pages快速搭建个人博客
date: 2023-09-27 15:20:15 +0800
categories: [其他]
tags: [个人博客, jekyll, chirpy, github]
author: stephan
mermaid: true
---

## 介绍
Jekyll可以简单理解成一个转换器，预先提供了模版目录和文档，日常使用中，我们可以在本地撰写博客并以markdown形式保存在Jekyll指定目录中，然后通过Jekyll转换成一个静态网站并发布在github page上后，通过地址https://your_github_name.github.io即可访问文章，无需购买域名，也不需要购买服务器，Jekyll详情请参考官网：[Jekyll-将纯文本转换为静态博客网站](https://jekyllcn.com/)。
下面简单介绍macos下搭建步骤：

## Jekyll安装
安装Jekyll前事先需要安装ruby，mac自带了低版本的ruby, 通过命令`ruby -v`查看当前ruby版本，高版本的Jekyll需要高版本的ruby，可以通过以下步骤更新ruby版本：
```shell
brew install ruby

echo 'export PATH="/opt/homebrew/opt/ruby/bin:$PATH"' >> ~/.zshrc
export LDFLAGS="-L/opt/homebrew/opt/ruby/lib"
export CPPFLAGS="-I/opt/homebrew/opt/ruby/include"
export PKG_CONFIG_PATH="/opt/homebrew/opt/ruby/lib/pkgconfig"
```
重新打开终端通过`ruby -v`查看ruby版本是否升级成功。

安装Jekyll:
```shell
gem install jekyll
```
至此Jekyll安装完毕。

## Chirpy主题
Jekyll支持众多主题，不同的主题代表了不同的博客风格，可以根据自己喜好，选择不同的模版：[jekyll模版地址](http://jekyllthemes.org/)。chirpy主题ui简单、画面清新，是用来搭建个人博客的不二之选，以下介绍通过chirpy主题搭建个人博客步骤：
1. 打开链接：[chirpy-starter](https://github.com/cotes2020/chirpy-starter)并登陆github账号。
2. 点击按钮Use this template, 选择Create a new repository，创建一个github repository.
   > 注意！repository的名称必须符合格式：USERNAME.github.io，USERNAME表示github用户名
   {: .prompt-danger }
3. git clone刚创建的仓库到本地，进入项目根目录，执行命令：
    ```shell
    bundle
    ```
4. 待上一步完成，编辑_config.yml, 更改一些基础配置信息如：
    ```
    avatar:   # 头像，支持本地以及cors资源，更换成自己喜欢的图片就行
    timezone: # 时区,Asia/Shanghai
    lang:     # 语言，默认en,中文的话设置成zh-CN
    title:    # 博客主页名称
    tagline:  # 博客主页名称下面的副标题
    github:   # 换成自己的github地址
    ```
    其他的比如twitter地址什么的，按需配置。
5. 以上编辑保存后，执行命令：
    ```shell
    bundle exec jekyll s
    ```
    打开链接：http://127.0.0.1:4000，即可看到一个最原始的博客版本。

以上步骤参考：[Chirpy getting-started](https://chirpy.cotes.page/posts/getting-started/)

## 撰写博客
Chirpy官方有具体教程：[Writing a New Post](https://chirpy.cotes.page/posts/write-a-new-post/)
主要有以下步骤：
1. 在_posts文件夹下新建博客markdown文档，文件命名格式为：YYYY-MM-DD-TITLE.markdown, YYYY-MM-DD表示文章的年月日，TITLE表示文章题目
2. 新建文档后，文档开头配置如下：
    ```
    title: TITLE                      # 文章title
    date: YYYY-MM-DD HH:MM:SS +/-TTTT # 文章日期：如2022-09-23 12:10:10 +0800
    categories: [TOP_CATEGORIE, SUB_CATEGORIE]                    # 文章类别
    tags: [TAG]                       # 文章标签
    author: elon                      # 文章作者，需要预先在_date/authors.yml中配置
    ```
    其中author配置项需要预先在_data/authors.yml下配置，如果_data下没有authors.yml文件则先创建，配置如下：
    ```
    <author_id>:                   # author名称的唯一id如elon
      name: <full name>            # author名字
      twitter: <twitter_of_author> # 推特地址
      url: <homepage_of_author>    # 作者详情地址，如果在_tabs下编辑了about.md，设置了基础信息, 可填/about
    ```
    详细配置请参考官网。
3. 博客内容写好后，可通过命令`bundle exec jekyll s`并打开http://127.0.0.1:4000查看效果

博客撰写完成，push到github后, 通过地址https://USERNAME.github.io即可查看文章。



