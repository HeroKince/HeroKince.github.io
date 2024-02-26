---
title: 一篇文章捋一下GAM
author: HeroKince
date: 2024-02-25 20:00:00 +0800
categories: [Ad]
tags: [Ad,Google]
---

近一年来很多人咨询GAM，我实际也了解并使用了一阵子，这篇文章来捋一下GAM是什么&如何使用。

Google Ad Manager是一个广告管理平台，适用于拥有大量直客资源的大型发行商。Google Ad Manager是谷歌在2018年推出的服务产品，整合了谷歌旗下的两项前服务功能[DoubleClick for Publishers (DFP) 与 DoubleClick Ad Exchange (AdX)]。

Google Ad Manager提供精细的流量控制功能，并支持多个广告交易平台：AdSense、Ad Exchange、第三方广告渠道(Third-party Ad Networks)、第三方交易平台(Third-party Ad Exchanges)。



#### 01 GAM的两大误解
先说说两个常见的对GAM的误解：

1、Ad Manager 会投放更优质的广告

事实并非如此。无论你使用AdSense/Admob还是Ad Manager，都可以接触到同等优质的广告客户和认证买方。因此，无论你选择哪个产品，你都有机会投放优质广告。

2、Ad Manager 是 AdSense/ Admob 的专业版

事实并非如此。Ad Manager 和 AdSense 是不同的产品。AdSense / Admob（AdSense网页变现， AdmobAPP变现）是一种广告联盟，可提供高品质的广告客户需求、实用的优化洞见以及易于实施的创收机会。Ad Manager 则是一个囊括了精细的广告资源控制系统和其他功能的一站式平台，有助于您轻松管理直接交易、第三方广告联盟以及来自桌面版网站、移动网站和应用的程序化广告需求。实际上，它们是同一个公司不同部门的产品，类似于腾讯的QQ和微信。



#### 02 如何接入GAM


接下来，让我们讨论一下GAM接入的流程：

首先，如果你想使用GAM进行变现，你可能需要通过GAM认证的合作伙伴来申请GAM账号。通常，这些合作伙伴会提供邀请链接，你可以通过链接开通GAM账号。不过，近年来国内也出现了一些GAM认证的合作伙伴，如A4G和Reklamup等。你可以选择几家合适的合作伙伴来进行申请。

其次，你可以通过任意一家GAM认证的合作伙伴的邀请链接，开通GAM账号，通过他们开通很快很顺利，不到一天可以完成。

接下来，针对APP来说，如果你已经接入了AdMob SDK，那么接入GAM不需要增加额外的SDK。但需要增加一些依赖项来调用，不同聚合需要参考官方的接入方式，比如MAX聚合有成熟的GAM接入方案，AdMob聚合则需要通过自定义事件进行接入。要注意添加app-ads.txt，由于GAM和app-ads.txt强绑定，否则可能没有相关广告填充。

对于网页变现，首先需要添加ads.txt，并让GAM认证的合作伙伴提供相应的JS代码，将其放置在他们的广告位和网页的<head></head>标签中。



#### 03 如何使用GAM并获取收益


最后，如何使用GAM并获取收益？

个人认为，你可以将GAM视为一个广告渠道来使用。由于GAM、AdMob和其他广告联盟都属于不同的广告源，它们具有不同的广告库存。通过多平台竞价，整体上可以提升广告eCPM。

在实际操作中，你可以将GAM作为一个变现渠道之一，与其他广告联盟同时使用。通过合理的瀑布流设置和竞价策略，可以最大化收益。同时，密切关注GAM的填充率和eCPM表现，根据实际情况选择保留1~2个最佳表现的合作伙伴。

总之，通过合理地使用GAM进行变现，你可以增加广告收益并优化变现效果。记得不断测试和优化，以找到最适合你的变现策略。

转载自：https://zhuanlan.zhihu.com/p/645598151
