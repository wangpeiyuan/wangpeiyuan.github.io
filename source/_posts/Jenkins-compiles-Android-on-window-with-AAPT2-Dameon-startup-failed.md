---
title: Jenkins 在 Windows 10 下编译 Android 项目报错：AAPT2 Dameon startup failed
date: 2020-04-02 09:57:12
categories: Jenkins
tags: Tip
---
最近在 Windows 10 下搭了 Jenkins 用来做持续性集成，之前都是用的 Mac，好几年都没碰过 Windows 了，不过前面安装过程都顺风顺水的，一路配置下来都没遇到什么问题。但最终在编译 Android 项目的时候出现了问题，报错如下：
```
org.gradle.api.tasks.TaskExecutionException: Execution failed for task ':app:mergeDebugResources'.
	at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter$3.accept(ExecuteActionsTaskExecuter.java:151)
	at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter$3.accept(ExecuteActionsTaskExecuter.java:148)
	at org.gradle.internal.Try$Failure.ifSuccessfulOrElse(Try.java:191)
  ...
Caused by: com.android.builder.internal.aapt.v2.Aapt2InternalException: AAPT2 aapt2-3.5.0-5435860-windows Daemon #0: Daemon startup failed
This should not happen under normal circumstances, please file an issue if it does.
 ...
```

<!--more-->

当然错误信息不止这些，做了精简。

资源合并失败，有点懵，Mac 上运行的好好的项目，怎么一到这就不行了。然后就在 Windows 上打开 Android Studio 编译试了一下，没问题啊。一开始以为是不是 Jenkins 上的配置哪里少了，一个个检查过去，没发现问题。重新编译一次，同样出错。

没办法了，求助 Google，搜出了一堆禁用 AAPT2 的，但仔细一看都是 AS 升级导致的，根本不适合，还有是将 gradle 的版本降级，这个还有点可能性，我就试了一下，还是不行，几乎把搜索出来的几页结果一个个翻完了，还是没有找到解决方案，这下彻底没撤了。

无奈，只能回过头来看看相关报错，使用 `build --stackTrace --info` 重新编译一次，获取更多的详细信息。然后在报错前后仔细寻找出错的蛛丝马迹了。

终于，还是让我发现了问题：
```
Transforming artifact aapt2-windows.jar (com.android.tools.build:aapt2:3.5.1-5435860) with Aapt2Extractor
Caching disabled for Aapt2Extractor: C:\Windows\System32\config\systemprofile\.gradle\caches\modules-2\files-2.1\com.android.tools.build\aapt2\3.5.1-5435860\ba4bab716d8d80e597b7baeec7df1bcf63562099\aapt2-3.5.1-5435860-windows.jar because:
  Build cache is disabled
Skipping Aapt2Extractor: C:\Windows\System32\config\systemprofile\.gradle\caches\modules-2\files-2.1\com.android.tools.build\aapt2\3.5.1-5435860\ba4bab716d8d80e597b7baeec7df1bcf63562099\aapt2-3.5.1-5435860-windows.jar as it is up-to-date.
AAPT2 aapt2-3.5.1-5435860-windows Daemon #0: starting
...
```

仔细看，会发现 gradle 的缓存竟然是在 `Windows\System32\config` 目录下，想想就觉得不可思议，怎么会跑到系统目录下去，这么一看就应该是权限问题了，搜了一下，Windows 的授权好麻烦，放弃。

如果不授权，那只能转移缓存目录了，以 Windows 修改 gradle 缓存目录为关键字搜索，果然搜索到了：配置 GRADLE_USER_HOME 环境变量就行，值的话自己新建个目录就行（最好不要在系统盘），重启 Jenkins 再次编译，成功。
