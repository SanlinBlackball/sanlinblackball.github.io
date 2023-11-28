---
layout:     post   				    # 使用的布局（不需要改）
title:      My First Post 				# 标题
subtitle:   Hello World, Hello Blog #副标题
date:       2017-02-06 				# 时间
author:     sanlinblackball 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
- 生活
---

## Hey
>这是我的第一篇博客。

进入你的博客主页，新的文章将会出现在你的主页上.


解决办法：
网站图标就不说了，按上面大佬的方法替换favicon.ico文件即可，文件名不能变，大小48x48，这是规范。并且不是随便找个图标改后缀就行了，需要去生成ico格式的文件，直接谷歌百度“ico在线制作”即可。

关于apple：
可以grep一下整个项目目录，把所有文件名为apple-touch-icon.png的地方找出来：

cd 你的github.io目录
grep -rn "apple-touch-icon.png"
然后为了避免缓存问题，可以把这些名字都改了，比如改成web-app-icon.png。
接下来删除img目录中原有的apple-touch-icon.png，自己按照苹果的开发规范添加相应大小的web-app-icon图片文件：关于Apple设备私有的apple-touch-icon属性详解

关于PWA：
还有PWA相关的最好也改了，不然谷歌浏览器还是会弹出作者原来的图标：
可以直接替换pwa/icons目录下的图片，或者自己按照PWA的规范修改manifest.json
随便搜的一个PWA文档

测试：
用浏览器的无痕模式测试就行了，避免缓存问题。