---
layout: post
title: "Liquid Error about regexp match when using octopress-tagcloud"
date: 2012-03-14 19:33
comments: true
categories: [octopress]
keywords: "octopress, tagcloud, liquid, regexp, encoding"
description: "Liquid Error about regexp match when using octopress-tagcloud: 'Liquid error: incompatible encoding regexp match'"
---
Octopress插件：[octopress-tagcloud](https://github.com/tokkonopapa/octopress-tagcloud)相当nice，不过在使用的时候，若是blog整体采用了非ascii码的编码格式(比如utf-8)就会出现类似这样的错误：
    Liquid error: incompatible encoding regexp match (ascii-8bit regexp with utf-8 string)
其实是由于octopress-tagcloud的插件文件：`plugins/tag_cloud.rb`文件本身是ascii编码所致:
    $ chardet tag_cloud.rb
    tag_cloud.rb: ascii (confidence: 1.00)
`tag_cloud.rb`中很多地方用到了ruby的正则表达式，而ruby的正则表达式在匹配的时候，默认是按照“代码源文件”的编码格式(在这里是ascii)进行匹配的，而若我的blog是utf-8编码的话就会出现上述错误。  

解决办法有二：

1. 将`tag_cloud.rb`转成utf-8...
1. 更改`tag_cloud.rb`中所有的正则表达式声明，加上`u`选项(根据[这里](http://www.ruby-doc.org/core-1.9.3/Regexp.html)的说明，`u`的意思就是以utf-8编码格式来进行匹配)，即，若原正则式是：`/regexp/`, 则改成：`/regexp/u`
<!-- more -->
这里是我改好的`tag_cloud.rb`：
{% gist 2036006 %}
