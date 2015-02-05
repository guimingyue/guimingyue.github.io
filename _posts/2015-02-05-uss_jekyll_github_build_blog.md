---
layout: page
title: 使用jekyll和github pages搭建个人
---
弄了一个晚上，终于弄好了github pages。在这里记录下来，方便喜欢折腾的同学。一下环境都是在
Windows下搭建。

##创建github pages

首先是创建github pages的repository，按照[每个人都应该有一个Jekyll博客][1]这篇文章中"第1步：
注册github并搭建Page"这一节创建github pages。

##安装Jekyll

然后使用Jekyll来创建本地预览。Jekyll是一个类似于Wordpress的博客系统，但是它不需要用到数据库，
因为在Jekyll中，所有的文件都是静态文件，博客中都是静态文件，也就不存在Sql注入的问题了。其实使
用github pages也可以不安装Jekyll，只需要将文件Push到你的github pages中，但是用Jekyll可以
让我们在本地预览文件，github pages也是使用Jekyll来解析页面的，所以最好还是安装一个Jekyll。
关于Jekyll，可以参考[http://jekyllcn.com/][2]

    1. 安装Jekyll
    由于Jekyll是使用Ruby语言开发，所以首先需要安装Ruby环境，在[http://rubyinstaller.org/][3]
    下载ruby安装包和Devkit，官网建议使用1.9.3版本。
    2. 安装好ruby后，将其路径加入到Path中。
    3. 进入到devkit解压目录下，执行如下命令：

    ```shell
    ruby dk.rb init
    ruby dk.rb install
    ```

    4. 安装Jekyll
    安装Jekyll需要使用gem命令从网络下载安装包，但是由于众所周知的原因，直接从国外站点下载
    可能会有问题，所以这里使用ruby.taobao.org的镜像。执行如下命令

    ```shell
    gem sources --remove https://rubygems.org/
    gem sources -a https://ruby.taobao.org/
    gem sources -l
    ```

    完成之后，gem sources -l命令的结果只有一条结果，即https://ruby.taobao.org/，请确保
    只有这一条，移除的时候请确保移除的是https://rubygems.org/,接下来就执行

    ```shell
    gem install jekyll
    ```

    命令安装Jekyll

安装完Jekyll之后，选择一个目录执行`jekyll new blog`命令，新建一个目录，这个blog就是你的
站点名称，可以任意执行，接下来生成一个blog目录，在这个blog目录中，有一个_posts目录，用于
存放你的文章，Jekyll会根据_config.yml中的配置在目录_site中生成静态网页文件。

`jekyll new xxx`命令只是生成了一个目录，这个目录中有一些生成的目录和文件，可以将这些文件
拷贝到github pages所在的本地目录中，然后使用git提交到github。每次写完文章就将其放入到
_posts目录中，再提交到github，就可以在你的github pages中看到了。

##提交到github

首先将你的github pages clone到本地，执行命令

```shell
git clone https://github.com/你的用户名/你的github pages.git
```
然后将Jekyll生成的目录和文件拷贝到刚才clone的目录，再执行如下命令

```shell
git add .
git commit -a -m "comment"
git push origin master
```

这样就可以将你的更改提交到github，如果你在_posts目录中放入了文章文件，你就可以在你的github pages
中看大这篇文章了。


##Reference
创建Github Pages：[每个人都应该有一个Jekyll博客][1]
Jekyll配置：http://justcoding.iteye.com/blog/1959737
搭建Jekyll Windows环境：http://www.cnblogs.com/yevon/p/3308158.html
Jekyllcn: [Jekyllcn][2]

[1]: http://www.cellier.me/2015/01/04/jekyll%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E6%95%99%E7%A8%8B/
[2]: http://jekyllcn.com/
[3]: http://rubyinstaller.org/downloads/
