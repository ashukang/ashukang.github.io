---
layout: post
title: "Android声控CubieBoard播放音乐（客户端）"
description: "本文以Android客户端声控Cubieboard播放音乐为例，简述了离线语音识别的应用，以及Android端与Cubieboard端 Socket通信的过程，旨在建立一个Android端声控CubieBoard端的框架。"
category: "smart"
tags: ["Android", "pocketsphinx", "cubieboard", "语音识别", "socket"]
---
{% include JB/setup %}

CubieBoard是一块优秀的开发板，小巧节能，能运行Linux系统，一般拿它来做BT下载机，或者小型的服务器，搭载如git、samba等服务。平常都是外接显示器，键鼠操作它，或者直接SSH远程登录。考虑到家里现在都会有的局域网，以及手头开放性强的Android智能手机，可以做一个交互界面更加友好的简单操控器，本文还借助了PocketSphinx库，利用它强大的离线语音识别能力，实现声控的功能，并且以播放音乐为例，讲述整个声控框架所涉及到的主要地方。

 - 项目源代码[ github ][1]
 - 示例视频

<div align="center"><embed src="http://player.youku.com/player.php/sid/XODQ3ODU3OTAw/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed></div>

使用PocketSphinx实现离线语音识别
--------------------------------

 - 下载[ PocketSphinx Android Demo ][2]

>这里我下载的是PocketSphinx Android Demo的Eclipse项目版本，如果你使用的是Google提供的最新版本ADT Bundle IDE，那可能需要先解决下ant缺失的问题，因为PocketSphinx Android Demo里面有一个ant builder，用来处理语音素材。如果是使用原生Eclipse IDE + ADT Plugin则可以直接导入这个Demo。直接运行这个Demo，根据界面上面的提示，读出指定的英文则可以进行下一步，我测试了下，渣渣的口音都可以识别出来，真心点赞！

 - 尝试修改配置，新增识别项

>这个官方的Demo只包含了英文的字典，所以识别的都是英文单词，因为我想要实现的控制器，多以命令的模式表示，所以可以把控制命令简化为如play pause next等简单的口令，就没有选择寻找中文字典，如果想更加接地气，很明显中文语音识别是必须，[ 这里 ][3]有大神小结了相关的知识点。

>Demo中采用来AsyncTask的方式来加载数据，可以推测这里的IO操作应该非常的繁重，果不其然，所有的与语音识别相关的数据，包括字典、目标词汇等都在Assert文件夹中，数量不少而且个别文件的大小还不小。在加载文件的方法中，可以看出来Demo展示来3种注册目标词汇的方法。

<div align="center"><img src="http://i3.tietuku.com/add48a70171d3d4c.png" alpngt="setup recognizer"/></div>

>第一种方式是注册单个单词或者短语，甚至是一个句子。如Demo的第一个页面所示，想要触发到这里的识别，需要读准整个短语。

>第二种方式是注册一个词组，如Demo的第二个页面所示，触发这里的识别，只要读准某一个词就可以了。找到Assert文件夹中对应的词组描述文件：

<div align="center"><img src="http://i3.tietuku.com/1ca46cca9745fea6.png" alpngt="digits gram"/></div>

>这个词组描述文件有一定的文件格式，可以猜测到gammer后面对应的digits是整个词组文件的KEY，用于注册到PocketSphinx服务中，下面的digit是目标词汇，以竖线分隔，最后以分号表示结束。

>第三种是带语义的识别？？没有验证到，只看到load到内存中的文件可能是一个类db的文件，所以推测是功能更加强大的语义识别，后续如果需要应用到更加智能的AI，或许需要对这里有所学习。目前看来，仅使用第一、第二种识别方式也是可以的。

>尝试修改digits文件，在目标词汇中添加如“play”、“pause”、“next”等单词，再次运行Demo，发现确实可以识别到我们新添加的目标词汇，故可以以这种方式添加或者修改目标词汇来满足我们的需求。

 - 学习PocketSphinx的API

>回调API

<div align="center"><img src="http://i3.tietuku.com/0318809b53dc1855.png" alpngt="callback api"/></div>

>onBeginningOfSpeech 是在PocketSphinx开始录音的时候会回调，在这里我们可以在UI上有所展示，比如显示“正在录音”的tips。

>onEndOfSpeech 是在PocketSphinx结束录音的时候会回调，这与onBeginningOfSpeech是成对出现的，这里可以将“正在录音”的tips去掉，或者展示“正在识别”的tips。

>onPartialResult 当识别到目标词汇时，PocketSphinx会回调这个接口，并且返回值非空，一定是目标词语，或者是目标词组中的某个。这里可以根据场景判断是发送口令到CubieBoard服务器，还是执行本定操作。

>onResult 在测试中发现，仅使用上诉第一、第二种方式时，可以忽略这个接口。从Demo上看这个接口会返回空值。

>注册与反注册回调

	SpeechRecognizer::addListener(RecognitionListener listener);
	SpeechRecognizer::removeListener(RecognitionListener listener);

>开始监听与结束监听，这里这里传给PocketSphinx的是目标词汇的KEY，而不是目标词汇本身。

	SpeechRecognizer::startListening(String key);
	SpeechRecognizer::stop();

>自此，如何在Android上使用PocketSphinx来识别英文词汇，已经变得非常的明朗，把它整合起来就变成来这个离线语音控制器的一个核心部分。

使用Socket将识别到的口令发送到Cubieboard服务器上
------------------------------------------------

>Android上对Socket的封装还是非常的简便，这里做以个简单的示例，考虑到是局域网，以及口令的发送可能不会出现特别多，特别频繁的场景，所以每次发送完口令时，都将Socket关闭，当然这里可以根据具体的使用场景再做优化。

<div align="center"><img src="http://i3.tietuku.com/921c5a35241548e9.png" alpngt="Android Socket"/></div>

UI整合
------

>这里考虑使用的便利性，在唤醒控制器的场景使用的即时监听语音的方式，如Demo本身所展示的方式。但是在音乐播放控制页面，在采用了“按住说话”的方式监听的方式，是因为可能此时有音乐在播放，对语音识别有一定的干扰，所以加了个“按住”操作，同时支持正常的按钮操作。运行截图如下，具体代码实现在[ 这里 ][1]。

 - 启动界面，等待唤醒。

<div align="center"><img src="http://i3.tietuku.com/65db0128e5752134.png" alpngt="UI_1" width="300"/></div>

 - 唤醒后，等待识别具体的操作。

<div align="center"><img src="http://i3.tietuku.com/40513b7bb4654ff3.png" alpngt="UI_2" width="300"/></div>

 - 进入音乐播放控制界面。

<div align="center"><img src="http://i3.tietuku.com/31814ef89d477173.png" alpngt="UI_3" width="300"/></div>

  [1]: https://github.com/ashukang/ARobot/tree/master/V1.0/Client_Android
  [2]: http://sourceforge.net/projects/cmusphinx/files/pocketsphinx/5prealpha/
  [3]: http://blog.csdn.net/zouxy09/article/details/7941585
