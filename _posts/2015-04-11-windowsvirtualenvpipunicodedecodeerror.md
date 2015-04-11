---
layout: post
title: "解决Windows环境下，virtualenv用pip安装程序报UnicodeDecodeError的问题"
description: "解决Windows环境下，virtualenv用pip安装程序报UnicodeDecodeError的问题"
category: "other" 
tags: ["Python UnicodeDecodeError"]
---
{% include JB/setup %}

Python在安装时，默认的编码是ascii，当程序中出现非ascii编码时，Python的处理常常会报这样的错UnicodeDecodeError

获取当前Python环境下的编码
------------------------

	import sys
	sys.getdefaultencoding()

>一般获取到的编码是ascii

方法一：修改有问题的文件
---------------------

>在有问题的文件头部添加

	import sys
	reload(sys)
	sys.setdefaultencoding('gb18030')

>Windows系统下，经测试使用gb18030编码是可以的，**对于Linux可以尝试使用utf8**

方法二：统一修改
--------------

>在Python安装目录下找到site.py，在头部添加，然后重启Python环境

	import sys
	reload(sys)
	sys.setdefaultencoding('gb18030')