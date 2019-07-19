---
title: 插件化之AAPT客户化
date: 2019-07-19 16:01:21
categories: 
    - Android
tags: 
    - 插件化
---
# 引言
这篇文章其实是借花献佛，微店的aapt工作，绝大部分是同事[区长](https://fucknmb.com/about/)编码和验证，文章本意在于对插件化架构中aapt相关流程的总结。
在Android编译流程中，aapt主要处理资源相关工作。插件化架构中，需要客户化aapt，让其承担宿主导出符号表、宿主导入符号表、插件导入宿主ap_文件、 修改插件package id、简化书写插件引用宿主资源等额外功能。
<!-- more -->
# 宿主导出符号表
宿主导出符号表主要用于：固定资源值；供插件调用宿主资源；aapt和aapt2实现方式不一样。
## aapt方式
通过aapt link  -P 参数实现。更多参数可通过（aapt --help）查看。
> -P  specify where to output public resource definitions  

```groovy
//DSL
android {
    aaptOptions {
        additionalParameters "-P", "${project.getBuildDir()}/public.xml"
    }
}

//伪代码
 additionalParameters.add("-P")
 additionalParameters.add(redirectFile.getAbsolutePath())
 processAndroidResourceTask.doLast {
    if (redirectFile.exists()) {
        exportSymbolFile.append("<?xml version=\"1.0\" encoding=\"utf-8\"?>\n")
        exportSymbolFile.write("<!-- AUTO-GENERATED FILE.  DO NOT MODIFY -->\n")
        exportSymbolFile.append("<resources>\n")
        def publicXMLNodes = new XmlParser().parse(redirectFile)
        publicXMLNodes.each {
                //这里可以有选择导出，比如只导出style和attr资源
                exportSymbolFile.append("    <public type=\"${it.@type}\" name=\"${it.@name}\" id=\"${it.@id}\" />\n")
        }
        exportSymbolFile.append("</resources>")
     }
}
```

导出文件类似于：

 ``` 
<resources>
   <public type="attr" name="barrierAllowsGoneWidgets" id="0x7f010004" />
   <public type="attr" name="barrierDirection" id="0x7f010005" />
</resources>
 ```


## aapt2方式

通过aapt2 link --emit-ids 参数实现。更多参数可查看[aapt2](https://developer.android.com/studio/command-line/aapt2#link)文档[1]。
>--emit-ids Emits a file at the given path with a list of names of resource 
>types and their ID mappings. It is suitable to use with --stable-ids.
>可以产生资源id文件，可以适用于--stable-ids

```
//DSL
android {
    aaptOptions {
        additionalParameters "--emit-ids", "${project.file('public.txt')}"
    }
}
//伪代码
additionalParameters.add("--emit-ids")
additionalParameters.add(redirectFile.getAbsolutePath())
processAndroidResourceTask.doLast {
    if (redirectFile.exists()) {
        Pattern filterPattern = Pattern.compile(".*?:(.*?)/.*?")
        List<String> sortedLines = new ArrayList<>()
        redirectFile.eachLine { String line ->
            Matcher matcher = filterPattern.matcher(line)
            if (matcher.matches() && matcher.groupCount() == 1) {
                //这里可以有选择导出，比如只导出style和attr资源
                exportSymbolFile.append("${line}\n")
            }
        }
    }
}
```

导出文件类似于：

 ```
com.koudai.weidian.buyer:attr/barrierAllowsGoneWidgets = 0x7f010004
com.koudai.weidian.buyer:attr/barrierDirection = 0x7f010005
 ```

# 宿主导入符号表
宿主导入符号表主要用于资源ID固定，在patch和插件化中常见。同样aapt和aapt2处理方式有所不同。

## aapt方式

aapt与aapt2相比较为简单，我们只要在aapt link(对应 processResouceTask)前（比如 mergeResourceTask或 processManifestTask）把public.xml和ids.xml放到res目录中，参与后续的aapt.link编译就行。

这里有个知识点：
>导入public.xml时，如果public.xml里有id 
>资源，但在values.xml里不存在，要额外新建一个xml文件，在文件里把id定义补上。

## aapt2方式

aapt2导入public.txt也较为简单，主要利用aapt2 --stable-ids 参数。
>--emit-ids path [Emits a file at the given path with a list of names of 
>resource types and their ID mappings. It is suitable to use with--stable-ids.]
>--stable-ids outputfilename.ext [Consumes the file generated with --emit-ids 
>containing the list of names of resource types and their assigned IDs. This 
>option allows assigned IDs to remain stable even when you delete or add new 
>resources while linking.]

但在插件化架构中，仅仅导入还不行，我们还要将导入的资源打上public flag标,才能供插件引用。打public flag标较为复杂，可参考团队同事写的[aapt2 适配之资源 id 固定](https://fucknmb.com/2017/11/15/aapt2%E9%80%82%E9%85%8D%E4%B9%8B%E8%B5%84%E6%BA%90id%E5%9B%BA%E5%AE%9A/#more)[2] ，其原理是：
> aapt2 在compile阶段就为资源打上public标，--stable-ids参数仅让资源参与link。

# 插件导入宿主ap_
插件导入宿主ap_文件，是为了能有效资源共享，在我们的插件化体系中，还有就是解决宿主合并插件Manifest后，透明主题无法动态生效等问题。
导入ap_文件,aapt和aapt2实现一致，主要利用-I参数,让ap_文件参与aapt link执行过程，参数说明如下：
>-I add an existing package to base include set

# 修改插件资源packageid

修改pacakge id，主要是指修改资源的包名id,即0xPPTTEEEE（PackageId+TypeId+EntryId）中的PP段值。在Android体系中，非系统应用，资源PP段都是0x7f。在国内ROM里，资源ID的可分配区间为[0x21,0x7f)[3],主要原因在于：
>0x1x为系统保留 0x7f为主apk使用 0x20之前的发现miui里面内部已使用

特别注意是，在联想ZUI ROM系统中，0x4f被占用。PP段固定业界实现有两种：修改二进制文件(arsc文件相关处理)和客户化aapt。二进制文件实现方式有滴滴的[VirtualAPK](https://github.com/didi/VirtualAPK/wiki),[Small](https://github.com/wequick/Small)等开源实现。我们这里仅讲解aapt实现方式。

## aapt方式

aapt修改packageid，需要我们修改aapt源码，业界实现方式大同小异,aapt各版本改动也可能不同。我们以sdk 9的版本为例，简述如何修改,大体思路如下：

>加入自定义参数-apk-module，在Main.cpp中获取其值，并在ResourceTypes.cpp中植入。

```
//Main.cpp 第一个改动点
int main(int argc, char* const argv[])
{
    ...
    if(strcmp(cp, "-apk-module") == 0){
        argc--;
        argv++;
         if (!argc) {
            fprintf(stderr, "ERROR: No argument supplied for '--apk-module' option\n");
            wantUsage = true;
            goto bail;
        }
        //取得 package id, apkModuleId可以像atlas [4]一样作为一个全局变量存在或保存在一个自定义单例中
        apkModuleId = strtol(argv[0], NULL, 16);
    }
    ...           
}    

```

```
//ResourceTypes.cpp 第二个改动点
...
DynamicRefTable::DynamicRefTable(uint8_t packageId, bool appAsLib)
    : mAssignedPackageId(packageId)
    , mAppAsLib(appAsLib)
{
    memset(mLookupTable, 0, sizeof(mLookupTable));

    // Reserved package ids
    mLookupTable[APP_PACKAGE_ID] = APP_PACKAGE_ID;
    mLookupTable[apkModuleId] = apkModuleId;
    mLookupTable[SYS_PACKAGE_ID] = SYS_PACKAGE_ID;
}
...
bool ResTable::stringToValue(Res_value* outValue, String16* outString,
                             const char16_t* s, size_t len,
                             bool preserveSpaces, bool coerceType,
                             uint32_t attrID,
                             const String16* defType,
                             const String16* defPackage,
                             Accessor* accessor,
                             void* accessorCookie,
                             uint32_t attrType,
                             bool enforcePrivate) const
{

    //只要在if (packageId != APP_PACKAGE_ID && packageId != SYS_PACKAGE_ID)的条件语句中加入自定义的packageid判断就行，如下所示
    if (packageId != apkModuleId && packageId != APP_PACKAGE_ID &&
        packageId != SYS_PACKAGE_ID) {
        ...    
    }
}    

status_t DynamicRefTable::lookupResourceId(uint32_t* resId) const {
    //与stringToValue函数处理类,相应if语句加入以下逻辑
    if ((packageId == apkModuleId || packageId == APP_PACKAGE_ID) && !mAppAsLib) {
        ...
    }
}    
```

## aapt2方式

aapt2原生就支持自定义PP段，比如现在的Android App Bundle就用到了此功能，其参数定义如下：

>--package-id package-id [Specifies the package ID to use for your app. The 
>package ID that you specify must be greater than or equal to 0x7f unless used in combination with --allow-reserved-package-id.]

但是 在sdk 6,7的aapt2只支持 0x7f-0xff区域的package id值，在8之后就我们添加 --allow-reserved-package-id参数就可以在0x02-0x7f区域。

# 简化书写引用宿主资源

插件中引用宿主资源，我们只要把宿主ap_文件参与插件编译即可。但我们在书写资源引用时，与引用android系统资源类似（@packageName:resType/resName），必须带上宿主包名，如 @xxx.xx.xx:color/white。如果我们不重写appt做相应处理，那在IDEA里会容易爆红。
解决方法：重写aapt，让业务插件不用书写包名过程，像引用plugin自身资源一样，直接找到资源。
## aapt方式
与修改packageid类似，我们加入自定义
```
//Main.cpp 第一个改动点
int main(int argc, char* const argv[])
{
    ...
    if(strcmp(cp, "-host-package") == 0) == 0){
        argc--;
        argv++;
        if (!argc) {
            fprintf(stderr, "ERROR: No argument supplied for '--host-package' option\n");
            wantUsage = true;
            goto bail;
        }
        hostPackage=std::string(argv[0]);
    }
    ...           
}    

```

```
//ResourceTypes.cpp 第二个改动点
bool ResTable::stringToValue(Res_value* outValue, String16* outString,
                             const char16_t* s, size_t len,
                             bool preserveSpaces, bool coerceType,
                             uint32_t attrID,
                             const String16* defType,
                             const String16* defPackage,
                             Accessor* accessor,
                             void* accessorCookie,
                             uint32_t attrType,
                             bool enforcePrivate) const
{

    ...
    if (*s == '#') {
         //末尾
        if (hostPackage.length() > 0 &&
            *defPackage != String16(hostPackage.c_str())) {
            String16 hostPackageName = String16(hostPackage.c_str());
            return stringToValue(outValue, outString, s, len, preserveSpaces, coerceType, attrID, defType,
                                 &hostPackageName,
                                 accessor, accessorCookie, attrType, enforcePrivate);
        } else {
            if (accessor != NULL) {
                accessor->reportError(accessorCookie, "No resource found that matches the given name");
            }
        }
        return false;
    }


    if (*s == '?') {
         //末尾
        if (hostPackage.length() > 0 &&
            *defPackage != String16(hostPackage.c_str())) {
            String16 hostPackageName = String16(hostPackage.c_str());
            return stringToValue(outValue, outString, s, len, preserveSpaces, coerceType, attrID, defType,
                                 &hostPackageName,
                                 accessor, accessorCookie, attrType, enforcePrivate);
        } else {
            if (accessor != NULL) {
                accessor->reportError(accessorCookie, "No resource found that matches the given name");
            }
        }
        return false;
    }

}      
```

## aapt2方式

与aapt类似，ResourceTypes.cpp和aapt是同个文件，不过入口函数没在Main.cpp里，而是Link.cpp。
```
//Link.cpp
int Link(const std::vector<StringPiece>& args, IDiagnostics* diagnostics) {
    ...
    Flags flags =
      Flags()
          .OptionalFlag("--host-package",
                        "Specific host packageName for plugin refer resources from host\n",
                        &host_package)
    ...
}    
```

# 参考
[1] AAPT2 USER GUIDE: [https://developer.android.com/studio/command-line/aapt2#link](https://developer.android.com/studio/command-line/aapt2#link)    
[2] aapt2 适配之资源id固定:[https://fucknmb.com/2017/11/15/aapt2%E9%80%82%E9%85%8D%E4%B9%8B%E8%B5%84%E6%BA%90id%E5%9B%BA%E5%AE%9A/#more](https://fucknmb.com/2017/11/15/aapt2%E9%80%82%E9%85%8D%E4%B9%8B%E8%B5%84%E6%BA%90id%E5%9B%BA%E5%AE%9A/#more)  
[3] Atlas 中文文档手册 :[https://www.bookstack.cn/read/atlas-zh/atlas-docs-guide-for-use-guide_for_build.md](https://www.bookstack.cn/read/atlas-zh/atlas-docs-guide-for-use-guide_for_build.md)  
[4] Atlas-aapt: [https://github.com/alibaba/atlas/blob/master/atlas-aapt/README.zh-cn.md](https://github.com/alibaba/atlas/blob/master/atlas-aapt/README.zh-cn.md)  