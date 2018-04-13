---
title: News not know
date: 2018-04-13 23:00:11
---
[20 Laravel Eloquent Tips and Tricks](https://laravel-news.com/eloquent-tips-tricks)
-----------------

Eloquent ORM seems like a simple mechanism, but under the hood, there’s a lot of semi-hidden functions and less-known ways to achieve more with it.

Laravel News / PovilasKorop2 小时前

[Composer v1.6.4 is Released With a Security Fix](https://laravel-news.com/composer-v1-6-4-is-released-with-a-security-fix)
-----------------



Laravel News / Eric L. Barnes2 小时前

[Drupal 远程代码执行漏洞 (CVE-2018-7600) 分析](http://blog.nsfocus.net/cve-2018-7600-analysis/)
-----------------

近日，流行的开源内容管理框架 Drupal 曝出一个远程代码执行漏洞，漏洞威胁等级为高危，攻击者可以利用该漏洞执行恶意代码，导致网站完全被控制 ... 可见，在 drupal 中，我们不需要直接写 html 表单，而是先创建一个数组，表单呈现引擎通过位于 \ drupal\core\lib\Drupal\Core\Form\FormBuilder.php 文件中的 buildForm 方法构造出一个名为 $form 表单，...

绿盟科技博客 / 高瑞强3 小时前

[腾讯 AI Lab 西雅图实验室负责人俞栋：语音识别领域的现状与进展](https://zhuanlan.zhihu.com/p/35648321)
-----------------

加入腾讯不久后，俞栋在机器之心主办的第一届全球机器智能峰会（GMIS 2017）上，发表了主题为《语音识别领域的前沿研究》的演讲，分享了语音领域的四个前沿方向，包括：更有效的序列到序列直接转换模型，鸡尾酒会问题，持续预测与适应的模型，以及前端与后端联合优化等 ... 在 90 年代初期，伯克利大学的研究人员就开始用多层感知机加上隐马尔可夫模型进行语音识别，由于模型由一个传统的生成模型 HMM 和一...

机器之心4 小时前

[ES6 实现之适配器模式 Adapter](https://juejin.im/post/5ad07d2b5188255570066f16)
-----------------

使用适配器模式之后，原本由于接口不兼容而不能工作的两个对象可以一起工作 ... 举个生活中的例子：港式插头转换器，港式的电器插头比大陆的电器插头体积要大一些 ... Adapter 类继承了 Target，重写 samll 函数，最后通过适配器，把港式 big 转成了大陆的 samll 了。

掘金 / 小小小5 小时前

[Spring Cloud OAuth2 优雅的集成短信验证码登录以及第三方登录](https://segmentfault.com/a/1190000014371789)
-----------------

基于 SpringCloud 做微服务架构分布式系统时，OAuth2.0 作为认证的业内标准，Spring Security OAuth2 也提供了全套的解决方案来支持在 Spring Cloud/Spring Boot 环境下使用 OAuth2.0，提供了开箱即用的组件 ... 基于上述的场景要求，如何优雅的集成短信验证码登录及第三方登录，怎么样才算是优雅集成呢 ... 在进入流程之前先进行拦截，设置集成认证...

Segmentfault / 李球5 小时前

[重谈 Url 签名](http://blog.kazaff.me/2018/04/13/%E9%87%8D%E8%B0%88url%E7%AD%BE%E5%90%8D/)
-----------------

没错，很简单不是么，一般大家都选择一种不可逆算法来签名，例如 MD5 ... 问题来了，如果签名的时候没有一个「私钥」的存在，中间人也可以自己重新签名来达到篡改的目的啊 ... 不考虑运营成本的话，短信不错，当客户端提交请求时，会收到一条短信验证码，客户端利用验证码做秘钥进行签名即可，但短信是要钱的。

kazaff's blog5 小时前

[如何绘制一个类甘特图 (附源码)](https://juejin.im/post/5ad076186fb9a028c813465b)
-----------------

我的业务场景绘制的点不算多，而且 canvas 对于事件挺难处理的 ... 图形绘制都改了，那离重画一张图也没啥区别，毕竟技术难度也算很高 ... 为了增加这一点灵活性，要给组件加太多的复杂度了，而且拍脑袋决定哪里开放绘制肯定做不好。

掘金 / 蚂蚁金服数据体验技术6 小时前

[原生 JavaScript 实现观察者模式](https://juejin.im/entry/5ad06eb0518825482e394dd8)
-----------------

在此种模式中，一个目标对象管理所有相依于它的观察者对象，并且在它本身的状态改变时主动发出通知 ... 维基的定义中涉及到了主动发出通知，按照这种方式，在 angularJS 中的事件广播更是中规中矩，但是其缺点是代码的可维护性较差 ... 如果不进行主动通知，而是在进行对象属性值设置时，调用相关的处理函数，也可达到同等效果。

掘金 / baron男爵6 小时前

[CNCERT 网络安全信息与动态周报 2018 年第 14 期](http://www.freebuf.com/articles/paper/168583.html)
-----------------

本周境内感染网络病毒的主机数量约为 28.1 万个，其中包括境内被木马或被僵尸程序控制的主机约 19.4 万以及境内感染飞客（conficker）蠕虫的主机约 8.7 万 ... 针对 CNCERT 自主监测发现以及各单位报送数据，CNCERT 积极协调域名注册机构等进行处理，同时通过 ANVA 在其官方网站上发布恶意地址黑名单 ... 本周，CNCERT 协调基础电信运营企业、域名注册服务机构、手机应用商店、各省分中心...

FreeBuf / CNCERT6 小时前

