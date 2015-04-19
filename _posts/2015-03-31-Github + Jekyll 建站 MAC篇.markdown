---
layout: post
title:  "Github + Jekyll 建站 MAC篇"
date:   2015-03-31 23:34:25
categories: jekyll
tags: jekyll github
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

###配置ruby

升级MAC到最新的版本，确保ruby版本为2.0.0以上

	$ ruby -v
	ruby 2.0.0p481 (2014-05-08 revision 45883) [universal.x86_64-darwin14]

考虑到墙内的网络问题，更换gem的source为taobao

	$ gem source -l
	*** CURRENT SOURCES ***
	
	https://rubygems.org/
	$ gem source --remove https://rubygems.org/
	$ gem source -a http://ruby.taobao.org/
	$ gem source -l
	*** CURRENT SOURCES ***
	
	http://ruby.taobao.org/

###安装jekyll

	$ sudo gem install jekyll

如果提示下面失败信息，按照提示执行 `sudo xcodebuild -license`

	Building native extensions.  This could take a while...
	ERROR:  Error installing jekyll:
		ERROR: Failed to build gem native extension.
		
	    /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby extconf.rb
	creating Makefile
	
	make "DESTDIR="
	
	You have not agreed to the Xcode license agreements, please run 'xcodebuild -license' (for user-level acceptance) or 'sudo xcodebuild -license' (for system-wide acceptance) from within a Terminal window to review and agree to the Xcode license agreements.
	
	
	Gem files will remain installed in /Library/Ruby/Gems/2.0.0/gems/yajl-ruby-1.2.1 for inspection.
	Results logged to /Library/Ruby/Gems/2.0.0/gems/yajl-ruby-1.2.1/ext/yajl/gem_make.out

然后重新执行`sudo gem install jekyll`安装jekyll

###配置git


* 在Server端新建repo

注册登陆github，创建空的repo，假设你的ID为username，新创建的repo名字就为username.github.io

	# 在本地配置server信息
	$ git remote rm origin
	$ git remote add origin https://github.com/username/username.github.io.git

* 在本地初始化repo并上传

进入[这里][1]，选择你喜欢的主题并下载。

解压缩，进入目录，执行`git init`初始化project，并且提交初始版本：

	$ git add .
	$ git commit -m "init project by template"
	$ git push -u origin master

>国内服务器没有VPN访问比较痛苦，大约重试2，3次之后才执行成功

现在访问http://username.github.io，看看你自己的blog

当然，这还不算真正你自己的blog，你还需要

###修改并管理你的博客


定制blog，一般为修改根目录下的_config.yml，这里你需要修改title,description,logo等个人信息。

管理，添加新的blog

blog都存放在_posts目录下，注意文件名前面需要是yyyy-MM-dd格式，例如2015-03-31，里面参考现存的例子。修改完毕后，在根目录下执行`jekyll serve`来启动本地的web service，然后通过访问http://127.0.0.1:4000来查看你的任何修改，确认无误执行下面命令提交:

	$ git commit -am "update my blog"
	$ git push -u origin master

邀请你的小伙伴们来访问你自己打造的blog吧。

[1]: http://jekyllthemes.org/ 'jekyll themes'
