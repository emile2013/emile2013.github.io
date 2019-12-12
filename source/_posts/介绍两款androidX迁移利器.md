---
title: 介绍两款androidX迁移利器
date: 2019-12-12 19:23:57
categories: 
    - Android
tags: 
    - 小功能组
---
# 背景
support迁移至androidX相关包，绝大部分团队万不得已才处理，因其可见的收益过小，反而带来更大研发和测试成本。  
考虑androidx相关包已release发布并逐步稳定，众多开源库开始基于androidX进行迭代，Android在Android Studio 3.5上默认将androidX开启，以及后续新API适配等多方因素，微店APP也开始着手迁移工作了。网上博文众多，但实际给出迁移效率和迁移质量的解决方案较少，本文给出微店迁移过程开发的两款功能插件。
<!-- more -->
# 迁移效率插件

Android官方提供的迁移方案，操作不复杂，在讲操作前，我们先关注以下官网的辅助文档：

1.Migrating to AndroidX Overview: [https://developer.android.com/jetpack/androidx/migrate](https://developer.android.com/jetpack/androidx/migrate)   
2.Migrating to AndroidX artifact-mappings：[https://developer.android.com/jetpack/androidx/migrate/artifact-mappings](https://developer.android.com/jetpack/androidx/migrate/artifact-mappings)   
3.Migrating to AndroidX class-mappings：[https://developer.android.com/jetpack/androidx/migrate/class-mappings](https://developer.android.com/jetpack/androidx/migrate/class-mappings)   

上以三个文档很重要，是迁移过程我们人工修正问题的重要依据。本文简单介绍下官方建议的旧项目源码如何迁移。

* 升级编译依赖
```
1.修改项目build.gradle中android gradle插件至3.2.0+，例如3.5.2；
2.并且将gradle-wrapper.properties 中的gradle版本改为4.6+，例如5.6.2；
3.升级app compileSdkVersion到 28+，例如29；
```

* 修改项目`gradle.properties`，用以支持androidx
```
# When configured, Gradle will run in incubating parallel mode.
# This option should only be used with decoupled projects. More details, visit
# http://www.gradle.org/docs/current/userguide/multi_project_builds.html#sec:decoupled_projects
# org.gradle.parallel=true
# AndroidX package structure to make it clearer which packages are bundled with the
# Android operating system, and which are packaged with your app's APK
# https://developer.android.com/topic/libraries/support-library/androidx-rn
android.useAndroidX=true
# Automatically convert third-party libraries to use AndroidX
android.enableJetifier=true
```
* 用Android Studio `Migrate to Androidx` 功能自动修改代码
```
操作路径：工具栏->Refactor->Migrate to Androidx...
```

经过以上两个步骤后，如果项目能编译成功并且layout布局中所有类都能找到，那恭喜你，本文下面的内容你无需查看了。我们APP尝试迁移出现了以下androidx带来的问题：
```
1. support替换androidx相对应类错误，这种错误会同时在java文件和布局文件中，例如android.support.v4.view.ViewPager没有正确替换成androidx.viewpager.widget.ViewPager，而是修改成了不存在的类androidx.core.view.ViewPager，导致迁移失败；  
2. java文件中相关support类import审明并未完全删除，导致迁移失败；  
3. 如果项目二次开发了相关support类或用到了只限制在support lib用的类，那么这一部分类会转换成androdx相关类失败，导致迁移失败；  
4. 可能的资源引用，在混淆时会报错，但这类问题不是致命问题，可以进行dontwarn处理；
```

官方迁移方案显然不符合我们团队以下需求：  
- APP项目是通过repo管理，子项目众多，逐个迁移效率太低；  
- Android Studio `Migrate to Androidx`   功能较弱；  

多方考虑，查阅`Migrate to Androidx` 功能源码后，决定自行研发迁移插件[EasyMigrateAndroidX](https://github.com/emile2013/EasyMigrateAndroidX),其使用简单执行 `./gradlew migrateAndroidX`就能较准确替换内容：
```
EasyMigrateAndroidX原理是解析migrate.xml文件(来自AS源码)，遍历所有项目(setting.gradle中include的所有项目中的类文件、资源文件以及gradle文件，并进行内容替换，能加快像repo管理或多项目迁移速度；
```

# 迁移质量插件

项目源码自动修改后，还是担心会带来线上问题，JAVA源码错误较为直观，编译过程就能给出提示，但像layout文件里如有错误类，不易察觉。考虑以上场景，研发了检测布局文件中不存在的类功能插件[ResMonitor](https://github.com/emile2013/ResMonitor)，执行`./gradlew checkReleaseRes`即可输出错误文件，输出内容如：
```
Execution failed for task ':app:checkReleaseRes'.
> java.lang.Exception: androidx.core.view.ViewPager not exist !! but declare at:
  # Referenced at /ResMonitor/sample/app/src/main/res/layout/content_main.xml:11
```
其实现原理：
```
原理是拿到layout和manifest xml导出的混淆文件,再通过javassist检测类是否存在
```
当然[ResMonitor](https://github.com/emile2013/ResMonitor)不仅限于androidX迁移检测，后续可辅助上线前质量测试。

# 结束语

androidX迁移，AGP是如何通过enableJetifier参数实现编译期类替换的，这块有时间再另起一博文再议。另外，小伙伴们，快去踩坑吧。
