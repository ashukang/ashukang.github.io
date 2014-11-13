---
layout: post
title: "在Github上搭建自己的blog"
description: "一直想着要记下自己的学习过程，但却没有开始动手。其实知道github上边可以做blog也不是一天两天的事情，关键就是有点懒（=。=）。为了更正这一个陋习，将申请ashukang.github.io的过程做个纲要性小结，作为自己的第一篇文章。"
category: "other" 
tags: ["github io", "jekyll"]
---
{% include JB/setup %}
  
一直想着要记下自己的学习过程，但却没有开始动手。其实知道github上边可以做blog也不是一天两天的事情，关键就是有点懒（=。=）。为了更正这一个陋习，将申请ashukang.github.io的过程做个纲要性小结，作为自己的第一篇文章。
  
1. 安装必要的软件，包括 git、ruby、gem、jekyll、rake

        sudo apt-get install git
        sudo apt-get install ruby
        sudo apt-get install gem
        sudo apt-get install rake
        sudo apt-get install jekyll

2. 申请 github 帐号，创建一个 username.github.io 的 repository，必须注意的是用于申请 github 个人主页的 username 必须和 github 帐号是同名的，否则 github 不会自动生成个人主页。

3. 进入 username.github.io 的设置页面，配置 "Automatic Page Generator" ，选择一个主题，然后 "PUBLIS"

4. 将 username.github.io clone 到本地，然后预览。

        git clone https://github.com/ashukang/ashukang.github.io.git
        cd ashukang.github.io
        jekyll sever --server

5. 在浏览器中打开 http://localhost:4000

6. 使用 jekyll-bootstrap 框架

        cd ..
        git clone https://github.com/plusjade/jekyll-bootstrap.git
        cd jekyll-bootstrap
        rm -rf ./.git/
        cp -R * ../ashukang.github.io

7. 编辑 ashukang.github.io/_config.yml
  
   >打开自动生成，修改 Markdown 编辑器为官方推荐的版本

        auto: true
        markdown: kramdown

   >修改主页信息

        title : 一杯清茶，一本书
        tagline: Site Tagline
        author :
          name : ashukang
            email : kangsj9104@gmail.com
              github : ashukang
                #twitter : username
                #feedburner : feedname

    >关闭评论（github io不提供动态网站服务，所以评论功能是借助第三方提供，例如 Disques。因为这部分内容还有待学习，所以先把这个功能关闭）

        # Settings for comments helper
        # Set 'provider' to the comment provider you want to use.
        # Set 'provider' to false to turn commenting off globally.
        #
        comments :
            provider : false
          
8.  删除根目录下的 index.html，删除 _post 文件夹下的 core-sample 目录

9.  编辑 index.md，自定义主页内容

        ---
        layout: page
        title: 学无止境
        tagline: Supporting tagline
        ---
        {% include JB/setup %}
        
        ## 小抄
        
        <ul class="posts">
            {% for post in site.posts %}
                <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
            {% endfor %}
        </ul>
        
        ## 待续

10. 添加文章

        rake post title="Hello World"

    >编辑 _post/2014-05-31-helloworld.md，以 Markdown格式编辑博文，例如。

        ---
        layout: post
        title: "Hello World"
        description: "first post"
        category: 'test'
        tags: ['hello', 'jekyll']
        ---
        
        
        # Hello World

    >重新运行 jekyll 就可以看到这篇测试文章了

11. 将所有变更提交到github上

        git add -A .
        git commit -m "first commit"
        git push origin


