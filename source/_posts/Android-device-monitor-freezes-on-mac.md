---
title: Android device monitor 在 Mac 上无法点击的解决方案
date: 2019-10-10 15:07:30
categories: Android
tags: Tip
---
最近在 Mac 上使用 Android device monitor 多次遇到无法点击以及一打开就弹出 An error has occurred 的错误弹窗，主要的原因是升级了相关了 SDK 导致出现问题。在网上搜索解决方案绝大多数的答案是将 Java SDK 的版本降到一个指定版本，非得降级不可吗，如果不降级就没有解决方案吗？经过我各种搜索各种尝试终于在 Stackoverflow 上搜索到一个解决方案 [Android device monitor freezes on Mac OS X](https://stackoverflow.com/questions/47089757/android-device-monitor-freezes-on-mac-os-x/47090518) ，就在此简单做个记录。

<!--more-->

对于无法点击的问题，首先先下载 [swt-4.7.1a-cocoa-macosx-x86_64.zip](https://www.eclipse.org/downloads/download.php?file=/eclipse/downloads/drops4/R-4.7.1a-201710090410/swt-4.7.1a-cocoa-macosx-x86_64.zip)；然后，解压，重命名解压文件中的 swt.jar 文件名称为 Android/sdk/tools/lib/monitor-x86_64/plugins 中 org.eclipse.swt.cocoa.macosx.x86_64...jar 的名字；最后拷贝过来覆盖 SDK 中的文件就解决无法点击的相关问题。

打开 Android device monitor 会弹出 An error has occurred 的错误弹窗暂时没有找到好的解决方案，由于我并没有尝试降级 Java SDK 的方案，并不确定降级是否可以解决，那我的解决方式比较简单粗暴重新下载 Android SDK 就解决了此问题。
