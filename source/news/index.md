---
title: News not know
date: 2018-02-02 11:55:59
---
[快讯 | 韩国发布新在野 Flash 0day 漏洞警告：朝鲜黑客正在利用](http://www.freebuf.com/vuls/161995.html)
-----------------

韩国计算机应急响应小组（KR-CERT）目前发出了关于新在野 Flash 0day 漏洞的警告 ... 韩国安全企业 Hauri Inc. 的安全研究员 Simon Choi 表示，朝鲜的黑客行动者已经成功利用了这个 0day 漏洞，他们其实自 2017 年 11 月中旬就开始使用这个方法进行攻击 ... Adobe 目前已经得到了报告，CVE-2018-4878 漏洞利用已经出现，朝鲜黑恶看正在应用这...

FreeBuf / Elaine_z14 小时前

[Python 无损音乐搜索引擎](http://www.freebuf.com/geek/161418.html)
-----------------

研究了一段时间酷狗音乐的接口，完美破解了其 vip 音乐下载方式，想着能更好的追求开源，故写下此篇文章，本文仅供学习参考 ... 这个接口通过传递关键字，其返回的是一段 json 数据，数据包含音乐名称、歌手、专辑、总数据量等信息，当然最重要的是数据包含音乐各个品质的 hash ... 只要将音乐的 hash+key 添加到 api_url ,get 提交过去，就能返回一段 json 数据，数据中就包括了音乐的下载链接...

FreeBuf / chinapython15 小时前

[那些工作之外的技术挣钱方式](https://juejin.im/entry/5a73baf3f265da4e853d280f)
-----------------

本文首发于 SkySeraph 博客（ skyseraph.com ），版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。

掘金 / SkySeraph16 小时前

[Android 自己开发的消息事件小项目 DBus](https://juejin.im/entry/5a73baec6fb9a0633b20dd9c)
-----------------

b、方法的返回值任意，可以是 void、int、String 等 ... 4、使用注解方式，有以下规则：a、关闭方法名限定开关：DBus.isUseMethodNameFind(false) ... 如果此方法的注解 port 值与发送处 DData 对象的 port 值一致，才能收到发送的消息。

掘金 / code小生16 小时前

[在美国政府之前，英特尔提前告知中国巨头企业关于 CPU 漏洞的信息](http://www.freebuf.com/news/161598.html)
-----------------

在告知美国政府之前，英特尔曾警告中国企业有关臭名昭著的 Meltdown 和 Spectre 处理器漏洞问题 ... 这一披露时间表提出了一种可能性，即在美国政府和公众知晓英特尔 CPU 漏洞之前，中国政府的某些部门可能已经了解了这些信息 ... 英特尔和谷歌安全安全人员以及其他发现处理器缺陷的团队致力于解决这个漏洞，当然还有 PC 制造商尤其是大型的 OEM 厂商以及云计算公司，包括联想、微软、亚马逊和 ARM。

FreeBuf / Andy16 小时前

[SpringBoot 中发送 QQ 邮件](https://segmentfault.com/a/1190000013092374)
-----------------

SMTP 协议全称为 Simple Mail Transfer Protocol，译作简单邮件传输协议，它定义了邮件客户端软件于 SMTP 服务器之间，以及 SMTP 服务器与 SMTP 服务器之间的通信规则 ... 就是说 aaa@qq.com 用户先将邮件投递到腾讯的 SMTP 服务器这个过程就使用了 SMTP 协议，然后腾讯的 SMTP 服务器将邮件投递到网易的 SMTP 服务器这个过程也依然使用了 SMTP 协议，SMTP 服务器...

Segmentfault / 江南一点雨16 小时前

[Nodejs：UDP 极简入门例子](https://juejin.im/post/5a73b4c26fb9a063421470f1)
-----------------



掘金 / 程序猿小卡_casper16 小时前

[springcloud(十二)：使用 Spring Cloud Sleuth 和 Zipkin 进行分布式链路跟踪](https://juejin.im/post/5a73b3746fb9a0635d0bec32)
-----------------

现今业界分布式服务跟踪的理论基础主要来自于 Google 的一篇论文《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》，使用最为广泛的开源实现是 Twitter 的 Zipkin，为了实现平台无关、厂商无关的分布式服务跟踪，CNCF 发布了布式服务跟踪标准 Open Tracing ... Zipkin 是一个开放源...

掘金 / ityouknow16 小时前

[疑似蔓灵花 APT 团伙钓鱼邮件攻击分析](http://www.freebuf.com/articles/paper/161429.html)
-----------------

近期，360 安全监测与响应中心协助用户处理了多起非常有针对性的邮件钓鱼攻击事件，发现了客户邮件系统中大量被投递的钓鱼邮件，被攻击的企业局限于某个重要敏感的行业 ... 360 威胁情报中心与 360 安全监测与响应中心在本文中对本次的钓鱼邮件攻击活动的过程与细节进行揭露，希望企业组织能够引起重视采取必要的应对措施 ... 从此次攻击者实施的钓鱼邮件攻击来看，攻击者显然尝试利用受害企业员工对信息安全的重视...

FreeBuf / 360天眼实验室16 小时前

[WEB 访问日志自动化分析浅谈](http://www.freebuf.com/articles/security-management/161546.html)
-----------------

正则匹配可能是 WAF 经常使用的规则，分析 WEB 访问日志时，也经常会用到，例如可执行脚本在上传目录下 (例如 / images/cmd.aspx)，那么这个文件就很有可能是 webshell, 常规的还有 attachments|images|css|uploadfiles 等，还有一些解析漏洞的格式都可以用来匹配 ... 针对威胁情报，在日志分析中，主要用来分析 IP，如果某个 IP 在一段时间内发生过情报，比如出现...

FreeBuf / 0xExploit17 小时前

