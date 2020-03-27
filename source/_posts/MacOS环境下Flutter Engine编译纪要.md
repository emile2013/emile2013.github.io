---
title: MacOS环境下Flutter Engine编译纪要
date: 2020-03-27 09:29:48
categories: 
    - Flutter
---
## 引言
最近工作之外，时间稍有富余，便尝试在macOS环境下编译[futter engine](https://github.com/flutter/engine)工程，以方便阅读engine源码和定制化engine。编译flutter不复杂，只是在国内，我们需要翻墙开放给gclient等工具下载源码。本文仅记录在参考flutter wiki [Setting-up-the-Engine-development-environment](https://github.com/flutter/flutter/wiki/Setting-up-the-Engine-development-environment)下，碰到的问题，以及给出解决方案，不对依赖工具作安装和介绍。
<!-- more -->
## 过程纪要
### gclient sync失败问题
安装依赖工具，新建engine目录并配置.gclient文件后，我们执行gclient sync操作，这时我们可能会遇到http代理和engine/src/flutter/DEPS配置文件问题，具体如下。

http代理问题：
```
RPC failed transiently. Will retry in 1m4s  {"error":"failed to send request: Post https://chrome-infra-packages.appspot.com/prpc/cipd.Repository/GetInstanceURL: EOF", "host":"chrome-infra-packages.appspot.com", "method":"GetInstanceURL", "service":"cipd.Repository", "sleepTime":"1m4s"}
```

上述错误是我们无法访问chrome-infra-packages.appspot.com站点问题，需要配置http代理到我们本地http代理端口，可在命令窗口执行以下操作(其中端口号视本机具体配置改变)：
```
export http_proxy=http://127.0.0.1:1089
export https_proxy=http://127.0.0.1:1089
```

engine/src/flutter/DEPS配置文件问题出现在重复配置了'src/third_party/dart/tools/sdks'和'src/third_party/dart/pkg/analysis_server/language_model'两项（基于engine 8d6f008fb commit）,解决此错误，我们只需删除以下重复配置即可：
```
   'src/third_party/dart/tools/sdks': {
     'packages': [
       {
         'package': 'dart/dart-sdk/${{platform}}',
         'version': 'version:2.4.0'
       }
     ],
     'dep_type': 'cipd',
   },

   'src/third_party/dart/pkg/analysis_server/language_model': {
     'packages': [
       {
        'package': 'dart/language_model',
        'version': 'EFtZ0Z5T822s4EUOOaWeiXUppRGKp5d9Z6jomJIeQYcC',
       }
     ],
     'dep_type': 'cipd',
   },
```

gclient sync过程中，我们不能强制结束gclient sync进程，可能很多小伙伴和我一样，看到以下输出认为gclient sync已结束，但其实不然，gclient还在继续工作。
```
Syncing projects: 100% (102/102), done.
```

### 导入engine源码
我们可以采用CLion IDEA来阅读engine源码，参考flutter wiki [Compiling-the-engine](https://github.com/flutter/flutter/wiki/Compiling-the-engine),执行gn命令或ninja编译后就可以导入engine源码了（建议ninja编译成功后导入，不然会因大量中间文件不存在报红）。这里以软链方式进行导入:
>建立软链

```
在src/futter目录执行
ln -s /Users/xxx/engine/src/out/compile_commands.json /compile_commands.json
```

>导入CLion

clion打开上一步软链compile_commands.json文件，导入时间可能较长，耐心等待即可。
![](https://github.com/emile2013/emile2013.github.io/blob/master/imgs/clion_engine.png?raw=true) 

### 引用locally-built engine

引用engine建议采用WIKI [The-flutter-tool] (https://github.com/flutter/flutter/wiki/The-flutter-tool) 推荐的参数形式，例如:
```
A typical invocation would be: --local-engine-src-path /path/to/engine/src --local-engine=android_debug_unopt.
```
当然也可以通过直接替换方式，把flutter SDK bin/cache/artifacts/engine目录下的相应文件进行替换，这里不做推荐。

## 参考

1.Setting-up-the-Engine-development-environment：[Setting-up-the-Engine-development-environment](https://github.com/flutter/flutter/wiki/Setting-up-the-Engine-development-environment)  
2.Compiling-the-engine: [Compiling-the-engine](https://github.com/flutter/flutter/wiki/Compiling-the-engine)
