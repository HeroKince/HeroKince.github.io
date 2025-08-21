---
title: Admob无效流量问题
author: HeroKince
date: 2023-12-12 12:04:56 +0800
categories: [Admob]
tags: [Admob,广告]
---

### 引起无效流量的原因

- 广告位置不合理，造成无效请求次数过多
- 广告误点击率较高
- 由于发布商点击在自己的应用中投放的广告而产生的点击或展示
- 发布商鼓励用户点击他们的广告
- 一位或多位用户产生的广告重复点击或展示
- eCPM和CTr虚高
- 某个版本出现问题后，继续请求广告

### 解决方案汇总

- 设计合理的广告位置，避免无效请求次数过多
- 接入中介，尤其是竞价中介，不但可以分摊流量，还能提升收益
- 一直不恢复，可考虑提高广告单元的底价
- 有购买流量的计划，可适当买一些优质流量
- 限流期间，尽量保持流量的稳定，避免忽高忽低
- 有使用插页和激励广告单元，设置合理的展示频率，避免存在个别用户恶意操作导致限流
- Firebase监控单个用户行为
- 设置点击广告次数的限制
- 删除出现问题的广告位，恢复后再重新接入
- 默认不请求广告，通过Firebase逐渐开放请求

### 申诉

无效点击联系表单
https://support.google.com/adsense/contact/invalid_clicks_contact?hl=zh-Hans

无效活动
https://support.google.com/admob/contact/invalid_activity

无效流量申诉
https://support.google.com/adsense/contact/appeal_form_adsense_admob?hl=zh-Hant&rd=1

参考
- https://support.google.com/admob/thread/197606223/%E6%97%A0%E6%95%88%E6%B5%81%E9%87%8F%E9%97%AE%E9%A2%98%E6%80%8E%E4%B9%88%E8%A7%A3%E5%86%B3?hl=zh-hans
- https://support.google.com/admob/thread/160803979/admob%E7%9A%84%E8%B4%A6%E5%8F%B7%E9%87%8C%E7%9A%84app%E8%A2%AB%E9%99%90%E6%B5%81%E4%BA%86-%E7%8E%B0%E5%9C%A8%E6%B2%A1%E6%9C%89%E5%BC%84%E6%B8%85%E6%A5%9A%E5%85%B7%E4%BD%93%E5%8E%9F%E5%9B%A0%E6%98%AF%E5%9B%A0%E4%B8%BA%E4%BB%80%E4%B9%88-%E8%BF%98%E6%9C%9B%E6%8C%87%E5%AF%BC%E4%BB%A5%E4%B8%8B?hl=zh-Hans
- https://support.google.com/admob/thread/200809074?hl=zh-hans&sjid=18098667332935539575-EU
- https://support.google.com/admob/thread/98805421?hl=zh-Hans
- https://support.google.com/admob/thread/160170758/%E6%88%91%E7%9A%84%E5%B9%BF%E5%91%8A%E8%B4%A6%E6%88%B7%E9%99%90%E5%88%B6%E5%B7%B2%E7%BB%8F%E8%B6%85%E8%BF%8760%E5%A4%A9%E4%BA%86%E6%88%91%E8%AF%A5%E6%80%8E%E4%B9%88%E5%8A%9E?hl=zh-Hans