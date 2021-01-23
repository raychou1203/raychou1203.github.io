---
title: A note of android reverse engineering
date: '2020-02-01'
categories: 
    - Security Related
---
## 起因
凌晨在 QQ 空间看到一个同校同学发布了一款 App，正在邀请别人参加内测。正巧最近我也在帮朋友写一个小应用，便下载来研究一下。应用的主题很常见，主打校内论坛、交友功能，大概猜到是怎么回事，还是先抓包看看吧。

## 抓包
Charles 这个工具就不用说了，基本抓过包的朋友都用过。不过想要抓手机应用，前提要求手机和电脑在一个局域网络中。设置代理和安装证书这两个步骤很容易找到教程，这里不再赘述。需要注意的是 Android 7 以后按谷歌要求需要额外修改 APP 的网络安全性配置，我这里参考这篇文章^[https://mp.weixin.qq.com/s?__biz=MzU4MjgzNzk5MA==&mid=2247483695&idx=1&sn=020fe7fc09eec49ed84c7c111be105ce&chksm=fdb372a6cac4fbb09a2152a1e71da3d18ccc68e31cad0682565d7a3af3444b05269d95ae9edc&token=2000924864&lang=zh_CN#rd]得以解决。

抓到包之后看了一眼 Request 和 Response 格式，看上去不太规范。而且虽然加入了 token，但是很多接口没有鉴权，拿到接口直接请求就能修改所有用户的资料。由于是内测，我将开发者的昵称修改以提示其漏洞，同时也在内测群里反馈了，截止目前接口还没有修改。除此之外，修改密码等接口中的密码竟然是明文传输的，这让我不能不担心是否在数据库中也是明文存储密码？于是诞生了 hack 进数据库的想法。

## 调研
在处理数据库相关问题之前，先判断出其应用的技术栈。后端是拿 php 写的，而据我所知 php 框架经常爆出漏洞，在此基础之上发现是 thinkphp v5.0.24，而此版本已将漏洞修复了。此外又发现服务器上安装了宝塔面板，开发者开启了安全登录入口，默认的后台地址被修改了，这条路径也只能放弃了。

## 撞库
我之前没有接触过安全方面的技术，但是最简单的撞库的思想还是有的。在网上找了一些密码字典，一共包括5000多个常用的密码，用 hydra^[https://github.com/vanhauser-thc/thc-hydra] 暴力撞库，但是失败了。

## SQL 注入
撞库失败后联想到 SQL 注入的方式，找到了 sqlmap^[https://sqlmap.org] 这个工具。拿几个接口都测了一下，没有发现注入点这条路也只能放弃。

## 逆向工程
既然服务端没有收获，那么就尝试从客户端入手。拿来 apk 解压得到 dex 文件，果然没有加壳。用 dex2jar^[https://github.com/pxb1988/dex2jar]将其转成 jar。这里遇到一个问题^[https://sourceforge.net/p/dex2jar/tickets/237/]，在 GitHub 上最新的 release(2.1 or later) 已经解决。之后用 Java Decompiler^[http://java-decompiler.github.io/] 打开 jar 查看源码。仔细查找后，奇怪地发现所有包都是导入的依赖，没有域名对应的包，也没有开发者编写的代码。这明显不合理。后来想到用 adb 来找当前栈顶 Activity 的方法。

```bash
adb shell dumpsys activity activities | sed -En -e '/Running activities/,/Run #0/p'  
```
结果竟然是 `uni.UNIC0381B1.io.dcloud.PandoraEntryActivity`。前面的 uni 是 apk 的签名，这在我安装时还在猜测他的具体含义。后面的 dcloud 我在抓包时候看到过，是用来统计数据的。但是用来统计数据怎么会单独开一个 Activity?  

搜索一下，答案揭晓了：原来 uni-app 是 DCloud 提供的 App 跨平台开发框架...虽然我怀疑这个应用过不是用原始 Android 开发的，但是谁能想到有人拿这个东西去写 App。弄了半天，竟然被抓包时候的数据欺骗了。回到源码，既然这个应用是用这个来开发的，那么在官方文档中就能找到端倪。仔细一看，果然是用 JS 写的，将 apk 解压后源码全出来了。这时突然想起，请求都是发往服务端处理的，就算拿到了前端代码又能怎样，最多也就是找到一个 DCloud 的 key 吧。

## 最后
折腾了一天，除了把其他用户的信息改了，最后也没能确认我的密码到底是不是明文存在数据库里的。但这这都不重要了，因为虽然用户设置了密码，但是那个应用的登录操作只能通过手机验证码来进行...