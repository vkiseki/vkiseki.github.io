---
layout: post
title: "Octopress Setup"
date: 2015-05-21 14:48:19 +0800
comments: true
categories: [octopress, github pages]
---
在Windows系统下使用`Octopress`及`Github Pages`建立个人博客。

<!--more-->

安装Octopress
---------------------------

* 下载Octopress  
	可以使用`Git`下载，也可以直接到[Github页面](https://github.com/imathis/octopress)下载zip包。  
	下载完成后，保存在当前路径的octopress文件夹下。
	{% codeblock %}
git clone git://github.com/imathis/octopress.git octopress
	{% endcodeblock %}

* 安装Ruby  
	http://rubyinstaller.org/

* 安装DevKit
	* RubyInstaller 版本<2.4.0  
		http://rubyinstaller.org/downloads/ *（注意：DevKit的版本需要对应Ruby的版本！）*  
		下载后解压后执行以下命令完成安装：
		{% codeblock %}
cd DevKit
ruby dk.rb init
ruby dk.rb install --force
		{% endcodeblock %}
	* RubyInstaller 版本2.4.0  
		安装RubyInstaller的过程中会有三个选项，此时按Enter即可。  
		或者通过执行```rake install 1 2 3```来完成。  

* 为Ruby换源  
	由于ruby资源访问较慢，建议国内用户换成[RubyChina源](https://gems.ruby-china.org/)。  
	第二步有时会因网络原因失败，多试几次就行了。  
	{% codeblock %}
gem sources --remove https://rubygems.org/
gem sources -a https://gems.ruby-china.org/
gem sources -l
	{% endcodeblock %}

* 修改octopress\\Gemfile文件第一行为  
	{% codeblock %}
source "http://ruby.taobao.org/"
	{% endcodeblock %}

* 安装Octopress的相关依赖
	{% codeblock %}
cd octopress
gem install bundler
bundler install
	{% endcodeblock %}

* 此时如果有报错提示没有安装rdiscount  
	安装rdiscount，请自行将"2.1.8"替换成控制台提示的版本。  
	之后重试bundler install。
	{% codeblock %}
gem install rdiscount -v '2.1.8'
bundler install
	{% endcodeblock %}

* 完成Octopress的安装
	{% codeblock %}
rake install
	{% endcodeblock %}

* 此时如果有报错提示```you have already activated rake x.x.0.0, but your gemfile requires rake x.x.0.0``` 
	则需要在rake命令前添加前缀：  
	{% codeblock %}
bundler exec rake install
	{% endcodeblock %}
  
配置Octopress
---------------------------
  
* 建立代码仓库  
	在Github中建立名称为```YOURNAME.github.io```的代码仓库

* 配置站点信息  
	打开octopress\\_config.yml文件，  
	修改url为```YOURNAME.github.io```，同时填写title、author等信息。

* 部署Github Pages  
	{% codeblock %}
rake setup_github_pages
	{% endcodeblock %}

* 生成网站  
	{% codeblock %}
rake generate
	{% endcodeblock %}

* Liquid Exception  
	若出现`Liquid Exception: undefined method 'gsub' for nil:NilClass in atom.xml`，  
	则需要打开octopress\plugins\octopress_filters.rb，将第3行修改为第4行所示的代码。
	{% codeblock lang:ruby %}
def cdata_escape(input)
  #input.gsub(/<!\[CDATA\[/, '&lt;![CDATA[').gsub(/\]\]>/, ']]&gt;')
  input.to_s.gsub(/<!\[CDATA\[/, '&lt;![CDATA[').gsub(/\]\]>/, ']]&gt;')
end
{% endcodeblock %}

* 发布网站文件  
	将网站文件上传到master分支下：
	{% codeblock %}
rake deploy
	{% endcodeblock %}

* 上传网站源码  
	将网站源码上传到source分支下：
	{% codeblock %}
git add .
git commit -m 'First commit'
git push origin source
	{% endcodeblock %}

* 创建新文章  
	会在octopress\\source\\_posts下新建一个markdown文件，  
	用文本格式打开后可以对文章进行编辑。
	{% codeblock %}
rake new_post["title"]
	{% endcodeblock %}

* 预览网站  
	执行以下指令后，在浏览器中访问```localhost:4000```即可。  
	它会监听html、css和markdown文件的变动，所以每次修改完只需要刷新浏览器就行了。  
	{% codeblock %}
rake preview
	{% endcodeblock %}

* 发布修改后的网站  
	修改后需要再次执行deploy命令发布到Github，才能通过github.io访问到最新的网页。
	{% codeblock %}
rake generate
git add .
git commit -am "comments"
git push origin source
rake deploy
	{% endcodeblock %}

配置已有的Octopress
-------------------------------
在新机器上安装Github上已有的Octopress网站

* 获取网站源码，存放在octopress目录下  
	{% codeblock %}
git clone -b source git@github.com:YOURNAME/YOURNAME.github.io.git octopress
	{% endcodeblock %}

* 获取网站文件，存放在octopress/\_deploy目录下  
	{% codeblock %}
cd octopress
git clone git@github.com:YOURNAME/YOURNAME.github.com.git _deploy
	{% endcodeblock %}

* 安装部署Github Pages  
	{% codeblock %}
gem install bundler
bundle install
rake setup_github_pages
	{% endcodeblock %}

* 如果发布时提示```[rejected] master -> master (non-fast-forward)```，则需要再同步一下\_deploy目录
	{% codeblock %}
cd _deploy
git pull origin master --allow-unrelated-histories
git checkout --theirs .
git add .

cd ..
rake generate
rake deploy
	{% endcodeblock %}

其它配置
-------------------------------
* 主页只显示文章摘要  
	在文章中适当的地方插入`<!--more-->`，
	同时修改octopress\\_config.yml：
	{% codeblock %}
excerpt_link: "read more"
	{% endcodeblock %}

* 支持disqus评论  
	需要先在[Disqus.com](http://www.disqus.com)注册帐号，
	并进入`Settings -> Add Disqus To Site`。  
	新建一个站点`myblog.disqus.com`，其中**myblog**就是你的shortname。  
	然后修改octopress\\_config.yml：
	{% codeblock %}
disqus_short_name: YOUR_NAME
disqus_show_comment_count: true
	{% endcodeblock %}

* Markdown中文编码  
	若提示中文编码存在问题，将文件转换成`UTF-8无BOM`格式即可。  
	可以使用`Notepad++`进行处理：  
	{% img /images/post/post_img_20150521001.jpg %}

<br>
Last updated on March 22th, 2018