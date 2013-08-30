---
layout: post
title: "使用'Tsukuba Help'观看国外免费在线网络课程(android篇)"
date: 2013-03-17 11:56
comments: true
categories: [life]
keywords: "在线课程, 国外, 视频, tsukuba, 筑波大学, android"
---

目前在[Coursera.org](https://www.coursera.org/), [udacity.com](https://www.udacity.com/), [edx.org](https://www.edx.org/)等网站都提供有丰富的、高质量的在线视频课程可学习；  
虽然国内也有网易公开课等视频课程项目(其课程来源仍是上述国外网站)，但数量相对较少且翻译也跟不上，而且，更重要的是，这些在线课程往往随堂提供有课后作业，如果你想同一时间与来自世界各地的同学一起交作业、一起讨论的话，用网易公开课平台是无法达到的；  
(当然我没有说网易公开课不好，在这里我要向网易致敬！)  
但由于网速、网络等各种原因，直接观看上述国外网站的在线课程有些困难，为此我推荐一种快速有效的办法，在android手机上，直接观看国外在线课程视频。  
  
<!-- more -->

以目前的android街机三星Galaxy Note2为例(Android 4.1)：

1. 打开浏览器，访问如图所示的网页: ![android 1](/images/tsukuba/android_tsukuba1.png)  
这个页面上面有许多ip地址等信息，并且每过2小时会刷新...而且越靠上的ip地址其服务质量越好...  
2. 对于android，要使用L2TP/IPSEC方式连接，因此我们找到"Support L2TP/IPSEC"这一列： ![android 2](/images/tsukuba/android_tsukuba2.png)  
注意这一列中标有"yes"字样的行，就是你的目标！  
3. 将刚才看到的"Support L2TP/IPSEC"这一列为“yes”的这一行的ip地址拷下来: ![android 2](/images/tsukuba/android_tsukuba3.png)  
4. 打开“设定”， 点击“更多设置”: ![android 2](/images/tsukuba/android_tsukuba4.png)  
5. 点开如图所示的选项后进去“新建”: ![android 2](/images/tsukuba/android_tsukuba5.png)  
6. 如下图般设置：  
![android 2](/images/tsukuba/android_tsukuba6.png)  
"名称"随便起一个；  
“类型”选“L2TP/IPSec PSK”;  
"服务器地址"就粘帖你上一步复制的ip地址;  
屏幕继续滑动往下走(要点开“高级选项”)：  
7. ![android 2](/images/tsukuba/android_tsukuba7.png)  
“IPsec预共享密钥”填：![android 2](/images/tsukuba/v.png), 全小写;  
“转发路由”填上图所示的值;  
然后其它不用管，保存吧！  
8. 点击连接你刚才建立的网络，若要求输入用户名密码，则均填: ![android 2](/images/tsukuba/v.png), 全小写：  
![android 2](/images/tsukuba/android_tsukuba8.png)  
9. 连接成功后，左上角会有一个小钥匙的图标，这时候就已经成功了！  
![android 2](/images/tsukuba/android_tsukuba9.png)  
10. 打开浏览器试试吧：  
![android 2](/images/tsukuba/android_tsukuba10.png)  

常见FAQ：  

* 这个工具由谁提供？
  * A: 这是日本筑波大学的研究生们发起的一个实验(社会实验？), 邀请全世界范围内的服务器，参与这个活动，免费为人们提供连接服务; 因此你最初用浏览器打开的那个服务器列表中的ip地址来自世界各地; 但由于日本同学普遍网络技术不好，导致原本的页面网络故障，我国国内的人们无法直接访问筑波大学的活动页面，因此我使用了一些技术手段把这些信息呈现到了国内，以方便爱学习的同学们使用
* 服务质量如何？
  * A: 很快，从可以看视频这一点就能体会到
* 这个服务能用多久？
  * A: 没有规定这个服务能用多久，如果断掉的话，重新连接或再去服务器列表中找一个来用就是了，换服务器的话, 所有配置都一样，只是覆盖一个ip地址而已，很方便；如果你关心的是这个免费服务的活动能进行多久，那我也不知道，总之我们目前能好好利用它帮助我们学习就OK了
* 除了android，其它平台能够使用这个服务么？
  * A: 当然可以，由于我中午还没吃饭，所以后面再介绍windows下如何使用该服务；linux众们我就忽略你们了吧，我知道你们自己能搞定的:)

......

有任何问题可发邮件至： ![email](/images/email.png), 不定期可能会回复(当我心血来潮查看垃圾箱时)  
若有与服务器列表页面相关的任何问题，可到此[页面](https://gitcafe.com/tsukubahelp/tsukubahelp/tickets)提交问题记录，会有人处理...  
