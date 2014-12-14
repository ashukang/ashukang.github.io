---
layout: post
title: "Android声控Cubieboard播放音乐（服务器）"
description: "本文以Android客户端声控Cubieboard播放音乐为例，简述了离线语音识别的应用，以及Android端与Cubieboard端 Socket通信的过程，旨在建立一个Android端声控CubieBoard端的框架>。"
category: "smart"
tags: ["cubieboard", "socket", "python", "mplayer slave"]
---
{% include JB/setup %}

在Cubieboard上面通过Python开启Socket监听，接受来自Android平台的请求。当Socket连接接通之后，根据协定好的协议，处理Android客户端发起的命令。这里使用了Mplayer来播放音乐，通过Mplayer的slave模式，可以轻松地控制音乐的播放过程，同时引入了python的subprocess模块，用于多进程编程。

 - 项目源代码 [ github ][1]

Python的Socket编程
------------------

在Android平台上，通过Android SDK可以非常方便的创建Socket请求，现在需要先在Cubieboard上建立Socket监听才可以响应Android端的请求。

 - Python对Socket编程的封装也是非常的成熟，首先使用socket模块创建一个socket对象。

 - 调用bind方法，将刚创建的socket对象绑定到对应的IP地址与端口上，再通过listen方法限定最大的接受请求数。

 - 准备就绪后可以调用accept方法，阻塞地等待客户端的请求，如果有客户端请求，则会返回一个connection对象。通过调用connection的send和recv方法，可以实现与客户端的通信。

 - 最后调用close方法，将资源释放掉。

>下面的代码简单地示例了Python的Socket编程：

	import socket

	if __name__ == "__main__":

		sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
		sock.bind(('192.168.4.10', 7556))
		sock.listen(1)
		while True:
			try:
				print 'do accept socket'
				connection,address = sock.accept()
				print 'client ip is ',address
				connection.settimeout(120)
				while True:
					try:
						buf = connection.recv(1024)
						print 'get cmd:',buf
					except Exception, ex:
						print 'recv error:',ex
						try:
							connection.close()
						except Exception,ex:
							print 'try close connection error:',ex 
						break
			except Exception,ex:
				print 'accept error:',ex
				break
		sock.close()

通信协议
--------

Cubieboard在接收到Android口令时，需要识别出来具体的操作，所以需要同客户端执行同一个协议，这就涉及到了通信协议的设定。通常的通信协议都由一个定长字节数的包头加可变长的包体组成，包头包含协议的版本号、口令号、包体大小、校验信息等。

不过作为一个偏实验性质的局域网应用，可以使用较为直观的字符串和普通的分隔符来表示或者间隔数据类型。本文所实现的功能是播放音乐，大概有的操作是play、pause、stop、next等，但是这里旨在搭建一个通用的控制模块，假如以后拓展了控制视频播放的功能，那么单纯的play、pause、stop、next就无法满足要求，因此还需要引入一个操作类别，在一个大的范围内限定操作的对象。

>这里可以使用“\n”来表示一条口令的结束，以“_”来分隔一个口令内类型、操作与具体数据。

	类型_操作_数据

>视相关操作，决定是否有数据，有些操作并不需要携带相关数据，比如pause，则数据项为空即可。

工厂模式搭建Cubieboard响应框架
------------------------------

设计模式是在面向对象编程的过程中实践总结出来的编码范式，它能帮助我们更加优雅地解决问题或者拓展模块。以上诉协议框架为例，当识别到具体的类型时，接下来的工作就是将操作指令以及相关数据交与对应的模块去执行，非常直观地编码这一个过程，可能是使用if else，或者是switch。当类型的数量少的时候，这段派发操作的代码还是可阅读的，但是当类型模块越来越大时，这里的代码长度将会变得越来越大，而且都是些重复性质的代码，显得不优雅。这个时候就可以使用简单工厂模式来解决这个问题，把所有的类型模块用它对应的类型做KEY，存放在一个Map中，当识别到操作类型时，只需要在这个Map中检索对应的类型模块即可，这样在派发类型操作的模块不需要在新增类型模块时被修改，而且整体的设计也是层次鲜明。

 - 对于如何使用Python来实现设计模式的问题，可以参看这里[ 这里 ][2]
 
 - 首先我们需要抽象出来各种操作的统一接口，明显地这个接口需要2种参数，一个操作口令，一个是相关数据。

>这里将口令和数据合并为一个参数，让具体的操作决定是否需要分割为操作口令和相关数据

	class CmdHandler:
	        def handle(self, cmd):
       	        	pass

 - 根据具体的操作类型，继承这个接口，并实现相应的功能。

>这里为例是实现了MusicHandler应对播放音乐的请求，同时实现了UndefHandler，当发现有除播放音乐外的其他类型时，就可以统一使用这个类型模块，里面只是简单提示这是一个无法识别的操作类型。

	class MusicHandler(CmdHandler):
	
        	def handle(self, cmd):
                	if "play" == cmd:
                        	self.play()
                	elif "stop" == cmd:
                        	self.stop()
                	elif "next" == cmd:
                        	self.next()
                	elif "pause" == cmd:
                        	self.pause()

		def play(self):
			pass

		def stop(self):
			pass

		def next(self):
			pass

		def pause(self):
			pass


	class UndefHandler(CmdHandler):

        	def handle(self, cmd):
                	print "undefine cmd ", cmd


 - 当所有类型模块都添加完成后，还需要一个工厂类来管理这些模块。

>从设计模式的角度上说，派发操作类型的代码模块（Socket部分）是不需要知道有哪些操作类型的，它只需要通过从Android客户端拿到的类型作为KEY，向工厂查询，然后工厂返回操作的接口，派发模块再将具体操作口令、数据输入到这个操作接口即可，根据面向对象的多态性，只要工厂返回的是正确的类型操作模块，则可以认为Android客户端所请求的操作被正确地执行了。这样在就实现了派发模块与业务模块的分离。
	
	class HandlerFactory:

        	handlers = {}
        	handlers["music"] = MusicHandler()

        	def createHandler(self, type):
                	if type in self.handlers:
                        	handler = self.handlers[type]
                	else:
                        	handler = UndefHandler()
                	return handler

Mplayer的应用 
-------------

Mplayer是一款强大的播放器软件，能够解码多种格式的音视频。在命令行下使用Mplayer也是非常的简单：

	mplayer [options] [url|path/]filename

>Mplayer支持播放音乐列表，只需要把音乐路径（相对、绝对）写在一个列表中，以换行分隔即可

	mplayer -playlist music.list

>在命令行下执行Mplayer播放音乐，还可以通过键盘控制它

	Basic keys: (complete list in the man page, also check input.conf)
	 <-  or  ->       seek backward/forward 10 seconds
	 down or up       seek backward/forward  1 minute
	 pgdown or pgup   seek backward/forward 10 minutes
	 < or >           step backward/forward in playlist
	 p or SPACE       pause movie (press any key to continue)
	 q or ESC         stop playing and quit program
	 + or -           adjust audio delay by +/- 0.1 second
	 o                cycle OSD mode:  none / seekbar / seekbar + timer
	 * or /           increase or decrease PCM volume
	 x or z           adjust subtitle delay by +/- 0.1 second
	 r or t           adjust subtitle position up/down, also see -vf expand
	 double click     toggle fullscreen
	 right click      pause (press again to continue)

上述简单的操作模式，比较适合在命令行下试听的情况，因为想要在其他程序中控制Mplayer，可以使用Mplayer提供的另外一种执行模式，既slave模式，在这种模式下，Mplayer默认在后台运行，它通过接收标准输入的命令来控制播放，每个命令都是以“\n”结尾。

>可以通过下述命令查看Mplayer在slave模式下支持的命令：

	mplayer -input cmdlist

使用Mplayer的slave模式，可以方便的让我们操控它，只需要找到一种创建进程的方式，创建一个运行Mplayer的子进程，就可以通过这个子进程与Mplayer交互，从而实现控制音乐播放的功能，下一个小结将介绍如何在Python中创建一个子进程。

Python subprocess的应用
-----------------------

subprocess的目的就是启动一个新的进程并且与之通信。

>subprocess模块中定义了Popen类。可以使用Popen来创建进程，并与进程进行复杂的交互。构造函数如下：

	subprocess.Popen(args, 
		bufsize = 0, 
		executable = None, 
		stdin = None, 
		stdout = None, 
		stderr = None, 
		preexec_fn = None, 
		close_fds = False, 
		shell = False, 
		cwd = None, 
		env = None, 
		universal_newlines = False, 
		startupinfo = None, 
		creationflags=0)

>stdin, stdout, stderr分别表示程序的标准输入、输出、错误句柄。他们可以是PIPE，文件描述符或文件对象，也可以设置为None，从父进程继承。更加详细的描述可以参考[ 这里 ][3]

利用subprocess模块，可以创建一个运行Mplayer的进程，并且建立通信管道，在这个通信管道里面传输控制命令给Mplayer，从而实现了在Cubieboard上面控制音乐播放的目的。

  [1]:https://github.com/ashukang/ARobot/tree/master/V1.0/Server_Python
  [2]:http://www.cnblogs.com/wuyuegb2312/archive/2013/04/09/3008320.html
  [3]:http://blog.csdn.net/imzoer/article/details/8678029
