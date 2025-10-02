---
title: 利用Tampermonkey实现学习通刷课
published: 2024-11-14
description: 解决水课学习刷视频一直等着浪费时间的烦恼。
#image:
tags: [教程, 脚本]
category: 学习生活
draft: false
lang: zh_CN      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---

# 前言

![image-刷课1](https://bd076fc.webp.li/2024/11/649585414300c466713026195577b655.png)

解决**水课**学习刷视频一直等着浪费时间的烦恼。

本文使用Windows内置**Edge浏览器**进行演示，建议刷课就用这个浏览器。



# 一、安装油猴(Tampermonkey)

可以直接访问 [Tampermonkey – Microsoft Edge Addons](https://microsoftedge.microsoft.com/addons/detail/tampermonkey/iikmkjmpaadaobahmlepeloendndfphd) 安装

![image-20241114010242920](https://bd076fc.webp.li/2024/11/dcfa02093cc64516521f2bc7220b0721.png)



如果使用的其它浏览器，可以进入[Tampermonkey官网](https://www.tampermonkey.net/)进行下载安装

{% note danger flat %}仅支持Chrome、Microsoft Edge、Safari、Opera Next和Firefox浏览器{% endnote %}

![image-20241114010412236](https://bd076fc.webp.li/2024/11/b2567f1814c9bebd90cbbd6666c4ccdb.png)



# 二、安装脚本

## 1. 取消隐藏Tampermonkey图标

   设置在工具栏显示，放在外面这样比较方便脚本开关。

![image-20241114010714162](https://bd076fc.webp.li/2024/11/bdf748bc9ee51905500c0108a701e06f.png)



## 2. 进入脚本网站

   脚本网站有很多，我推荐使用[Greasy Fork - 安全且实用的用户脚本站](https://greasyfork.org/zh-CN)，里面可用的脚本比较多。

![image-20241114011105044](https://bd076fc.webp.li/2024/11/eae59071a7b94083e8d629d238cb82ee.png)



## 3. 安装学习通脚本

直接搜索**学习通/超星**，看见顺眼的点进去安装脚本就行，不过有些可能已经失效了，这里我给大家推荐一个脚本（2024/11/14测试可用）

[超星学习通九九助手[一键启动][最小化运行]](https://greasyfork.org/zh-CN/scripts/469522-超星学习通九九助手-一键启动-最小化运行)

![image-20241114011519092](https://bd076fc.webp.li/2024/11/836676c57e17a414b4d87c0e03bc6266.png)

直接一直**点安装**就行

![image-20241114011614942](https://bd076fc.webp.li/2024/11/f5e866faf7b02f246713b3a9b205be8b.png)



安装完后页面会消失，打开学习通，点击油猴如果是这样显示的，那就恭喜你安装成功啦！

![刷课3](https://bd076fc.webp.li/2024/11/9d6376b46d60cb099538050d6932478f.png)




# 三、开始刷课

## 1. 打开超星学习通网站

   点去进你要刷的课程，会有弹窗出现。

![image-20241114012325293](https://bd076fc.webp.li/2024/11/af54a6b8097fdf8ed9d3226257f8eb6d.png)



{% note warning flat %}Warning 如果没有出现弹窗，那就再按下面图片的操作试试，如果出现弹窗请直接跳过{% endnote %}

![屏幕截图 2024-11-15 023745](https://bd076fc.webp.li/2024/11/79af526bbb2bfb3f4dbe58cf393b6628.png)

**必须要打开开发人员模式**，如果没开脚本不能正常运行。

如果已经打开了开发人员模式，刷新网页后还未出现弹窗，那就可以再把下面**三个框勾选上**。

![屏幕截图 2024-11-15 024020](https://bd076fc.webp.li/2024/11/acc42ff379020bf2f9cfc599ded49bda.png)

这样基本上就大功告成啦！



## 2. 可以根据自己的需求进行点击

   作业和考试板块应该是要付费的，有需要的可以自行研究一下。

   进入课程，界面大概就长这样，可以自行选择怎么去刷。

![屏幕截图 2024-11-14 012616](https://bd076fc.webp.li/2024/11/aa2f5a0e5163494dd17f79c9fb27d9f1.png)

{% note danger flat %}视频倍速播放和夜间刷课都有可能导致学习记录被清空哦。{% endnote %}


## 3. 刷课结束

   全部刷好后，叉掉此页面**回去检查**一下吧，看看进度有没有都已经完成了。



# 注意事项

   不使用刷课的时候右上角可以手动关闭插件。

   **若脚本失效可在评论区反馈，我再去寻找新的可用方案。**