---
layout: post
title:  "搭建Jekyll+Github Pages(Windows版)"
date:   2016-11-05 10:58:19 +0800
categories: tech
author: "Benjamin"
tags:
    - Jekyll
    - Github
---

偶尔间提笔，记录下工作中的所见所想。书写博客，是一种形式，更是一种经验的积累。

>博主尝试过两次写博客的经历：伊始，在CSDN提供的免费空间，整理读书笔记；后来，学着买VPS、搭建WordPress，申请个性域名。渐渐，觉得有些累。前者在编辑上限制甚多，保存的图片文本要自己备份；后者则花费不少精力去维护站点。

今日，偶遇 [jekyll][jekyll-cn]+[Github Pages][github-pg]，托管代码也能发布博文。那我们就开始动手吧！

----

* **安装Jekyll**

  >因为[Jekyll官方文档][jekyll-win-install]并不支持在Windows上直接安装。所以，搭建本地Jekyll环境，需要更多的准备工作。

  1. 安装Windows下的包管理工具[Chocolatey][chocolatey]

     管理员身份打开`Powershell`命令行，提升脚本执行权限为`RemoteSigned`

     ```
     Set-ExecutionPolicy RemoteSigned
     ```

     然后执行安装

     ```
     iwr https://chocolatey.org/install.ps1 -UseBasicParsing | iex
     ```

     出现如下提示，`Chocolatey`安装成功

     ![choco-install-success]({{ site.baseurl }}/img/20161105/02.jpg){: .center-image }

  2. 安装[ruby][ruby-cn]

     `Jekyll`的安装是通过[gem][gems]包管理安装，所以，我们首先需要安装ruby。

     打开命令行，执行

     ```
     choco install ruby -y
     ```

     如成功提示，在执行接下来的操作时，需要重新打开命令窗口。

  3. 安装Jekyll

     终于可以安装主角Jekyll了！

     直接运行

     ```
     gem install jekyll
     ```

     会报错如下：`certificate verify failed`

     ![gem-ssl-error]({{ site.baseurl }}/img/20161105/04.jpg){: .center-image }

     解决办法：**在Gem sources中指定非HTTPS的站点来进行安装**

     移除HTTPS站点：

     ```
     gem sources --remove https://rubygems.org/
     ```

     添加非HTTPS站点：

     ```
     sudo gem sources -a http://rubygems.org/
     ```

     ![gem-sources]({{ site.baseurl }}/img/20161105/05.jpg){: .center-image }

     然后再执行jekyll的安装命令就没问题啦！

  4. 开启Jekyll站点

      * 在想要创建Jekyll站点的目录下打开命令行
      * 执行命令`jekyll new blog`
      * 进入创建的jekyll站点目录`blog`
      * `jekyll s`启动该站点
      * 访问本地地址`http://localhost:4000`

  5. 访问[Jekyll首页][jekyll]获取更多Jekyll信息

----

* **安装 [github-pages gem][github-pages gem]**

  这个Gem包，用于在Github上管理Jekyll及其依赖。

  1. 下载Ruby[DevKit][devkit]

     ```
     choco install ruby2.devkit
     ```

     该工具默认安装在目录`C:\tools\DevKit2`

  2. 配置安装

     * 在目录`C:\tools\DevKit2`中打开命令行
     * 执行`ruby dk.rb init`生成`config.yml`,该文件中会自动设置可执行文件Ruby的位置（如没有正确生成，可手动添加）
     * 执行安装`ruby dk.rb install`

  3. 安装[Nokogiri][nokogiri]

     安装依赖库

     ```
     cinst -Source "https://go.microsoft.com/fwlink/?LinkID=230477" libxml2
     ```

     ```
     cinst -Source "https://go.microsoft.com/fwlink/?LinkID=230477" libxslt
     ```

     ```
     cinst -Source "https://go.microsoft.com/fwlink/?LinkID=230477" libiconv
     ```

     通过gem安装Nokogiri `gem install nokogiri`

  4. 安装`github-pages gem`

     执行 `gem install github-pages`

----

* **申请Github账号**

    1. 博客中的所有文件都会被托管在Github上，毋容置疑，我们首先就需要一个Github账号，直接去[Github][github]上申请就可以了。

       ![github-login]({{ site.baseurl }}/img/20161105/01.jpg){: .center-image }

    2. 创建一个托管仓库，此处名称会显示在博客URL中

       ![github-repos]({{ site.baseurl }}/img/20161105/06.jpg){: .center-image }

    3. 将前述所创建的Jekyll站点提交入该仓库

    4. 在该仓库的设置项中，设置Github-pages开启的分支，图中选择了main分支，点击默认给出的链接就可以访问到Jekyll的博客站点啦！

       ![github-pages-branch]({{ site.baseurl }}/img/20161105/07.jpg){: .center-image }


至此，整个创建流程就完工啦！只需要本地写完博文，提交github发布即可，赶紧去看看吧！

----

* **参考文献**

  * [how-to-install-jekyll-and-pages-gem-on-windows-10-x46][how-to-install-jekyll-and-pages-gem-on-windows-10-x46]

  * [setting-up-your-github-pages-site-locally-with-jekyll][setting-up-your-github-pages-site-locally-with-jekyll]


[jekyll-cn]: http://jekyllcn.com/
[jekyll]: https://jekyllrb.com/docs/home/
[jekyll-win-install]: http://jekyllcn.com/docs/windows/#installation

[chocolatey]: https://chocolatey.org/install
[ruby-cn]: https://www.ruby-lang.org/zh_cn/
[gems]: https://rubygems.org/gems
[bundler]: http://bundler.io/
[github-pages gem]: https://github.com/github/pages-gem
[devkit]: http://rubyinstaller.org/add-ons/devkit/
[nokogiri]: https://rubygems.org/gems/nokogiri/

[github-pg]: https://pages.github.com/
[github]: https://github.com/
[how-to-install-jekyll-and-pages-gem-on-windows-10-x46]: https://jwillmer.de/blog/tutorial/how-to-install-jekyll-and-pages-gem-on-windows-10-x46
[setting-up-your-github-pages-site-locally-with-jekyll]: https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/
