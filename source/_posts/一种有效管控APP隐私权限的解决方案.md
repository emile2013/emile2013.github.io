---
title: 一种有效管控APP隐私权限的解决方案
date: 2019-11-16 17:20:14
categories: 
    - Android
tags: 
    - 小功能组
---
# 引言
诸如读写外置存储、读取联系人、发短信等隐私权限，android在6.0系统开始进行[动态授权](https://developer.android.com/guide/topics/permissions/overview)。但在我国，仅向用户提示授权框还不够，工信部在19年11月初发布了[专项整治App八类侵权行为审明](http://www.xinhuanet.com/2019-11/05/c_1125192397.htm) ,其文明确治理以下八类问题：
>1.私自收集个人信息；  
>2.超范围收集个人信息；  
>3.私自共享给第三方用户信息；  
>4.强制用户使用定向推送功能；  
>5.不给权限不让用；  
>6.频繁申请权限；  
>7.过度索取权限；  
>8.为用户账号注销设置障碍。  

很不幸，网报通告批评：我司老版本APP中审明了隐私权限，但在隐私文档中并未进行有效说明。收到通告，团队立马对权限进行了扫描，发现APP在AndroidManifest中审明了三项隐私权限，但实际过程并未使用(有些冤大头)。我相信很多团队跟我们面临同个问题，多团队开发下，权限引入问题没有一个有效监管机制。为避免类似问题再次发生，本文给出一个简单有效的代码编译层拦截方案。  

在说方案原理之前，我们先假定检测方案是扫描APP AndroidManifest.xml文件中审明的和用户有关的隐私权限，再比对隐私文档以及实际使用场景，进行判别。面对检测方案，我们给出解决思路：
>在编译阶段processApplicationManifest task运行后，对Merged Manifest Log文件进行扫描,如果用到了新权限，抛出打包错误，直至问题解决；

<!-- more -->
# 源码简阅

Android Gradle Plugin在编译APP后，会在build/outputs/logs目录下生成名为【manifest-merger-${variantname}-report.txt】文本文件。
以AGP 3.5.0源码为例，简单分析下ProcessApplicationManifest任务是如何产生Merged Manifest Log文件的。
``` 
package com.android.build.gradle.tasks;

/** A task that processes the manifest */
@CacheableTask
public abstract class ProcessApplicationManifest extends ManifestProcessorTask {
    @Override
    @Internal
    protected boolean getIncremental() {
        return true;
    }

    @Override
    protected void doFullTaskAction() throws IOException {
        ... ...
                    MergingReport mergingReport =
                    ManifestHelperKt.mergeManifestsForApplication(
                            getMainManifest(),
                            getManifestOverlays(),
                            computeFullProviderList(compatibleScreenManifestForSplit),
                            navigationXmls,
                            getFeatureName(),
                            moduleMetadata == null
                                    ? getPackageOverride()
                                    : moduleMetadata.getApplicationId(),
                            moduleMetadata == null
                                    ? apkData.getVersionCode()
                                    : Integer.parseInt(moduleMetadata.getVersionCode()),
                            moduleMetadata == null
                                    ? apkData.getVersionName()
                                    : moduleMetadata.getVersionName(),
                            getMinSdkVersion(),
                            getTargetSdkVersion(),
                            getMaxSdkVersion(),
                            manifestOutputFile.getAbsolutePath(),
                            // no aapt friendly merged manifest file necessary for applications.
                            null /* aaptFriendlyManifestOutputFile */,
                            metadataFeatureManifestOutputFile.getAbsolutePath(),
                            bundleManifestOutputFile.getAbsolutePath(),
                            instantAppManifestOutputFile != null
                                    ? instantAppManifestOutputFile.getAbsolutePath()
                                    : null,
                            ManifestMerger2.MergeType.APPLICATION,
                            variantConfiguration.getManifestPlaceholders(),
                            getOptionalFeatures(),
                            getReportFile(),   
                            LoggerWrapper.getLogger(ProcessApplicationManifest.class));
        ... ...
    }

    public static class CreationAction
            extends AnnotationProcessingTaskCreationAction<ProcessApplicationManifest> {

        private File reportFile;

        @Override
        public void preConfigure(@NonNull String taskName) {
            super.preConfigure(taskName);
            
            //这里就【manifest-merger-${variantname}-report.txt】文件
            reportFile =
                    FileUtils.join(
                            variantScope.getGlobalScope().getOutputsDir(),
                            "logs",
                            "manifest-merger-"
                                    + variantScope.getVariantConfiguration().getBaseName()
                                    + "-report.txt");
        }
     }   
}
```

通过代码，可以发现ProcessApplicationManifest是交给ManifestHelperKt.mergeManifestsForApplication方法对所有Manifest进行合并处理的，并且Log保存在【manifest-merger-${variantname}-report.txt】文件中。
```
package com.android.build.gradle.internal.tasks.manifest

/** Invoke the Manifest Merger version 2.  */
fun mergeManifestsForApplication(
    mainManifest: File,
    manifestOverlays: List<File>,
    dependencies: List<ManifestProvider>,
    navigationFiles: List<File>,
    featureName: String?,
    packageOverride: String?,
    versionCode: Int,
    versionName: String?,
    minSdkVersion: String?,
    targetSdkVersion: String?,
    maxSdkVersion: Int?,
    outManifestLocation: String,
    outAaptSafeManifestLocation: String?,
    outMetadataFeatureManifestLocation: String?,
    outBundleManifestLocation: String?,
    outInstantAppManifestLocation: String?,
    mergeType: ManifestMerger2.MergeType,
    placeHolders: Map<String, Any>,
    optionalFeatures: Collection<ManifestMerger2.Invoker.Feature>,
    reportFile: File?,
    logger: ILogger
): MergingReport {

    try {

        //ManifestMerger2是 manifest-merger库提供的辅助类
        val manifestMergerInvoker = ManifestMerger2.newMerger(mainManifest, logger, mergeType)
            .setPlaceHolderValues(placeHolders)
            .addFlavorAndBuildTypeManifests(*manifestOverlays.toTypedArray())
            .addManifestProviders(dependencies)
            .addNavigationFiles(navigationFiles)
            .withFeatures(*optionalFeatures.toTypedArray())
            .setMergeReportFile(reportFile)
            .setFeatureName(featureName)

        if (mergeType == ManifestMerger2.MergeType.APPLICATION) {
            manifestMergerInvoker.withFeatures(ManifestMerger2.Invoker.Feature.REMOVE_TOOLS_DECLARATIONS)
        }


        if (outAaptSafeManifestLocation != null) {
            manifestMergerInvoker.withFeatures(ManifestMerger2.Invoker.Feature.MAKE_AAPT_SAFE)
        }

        setInjectableValues(
            manifestMergerInvoker,
            packageOverride, versionCode, versionName,
            minSdkVersion, targetSdkVersion, maxSdkVersion
        )
        
        //关注这里的调用
        val mergingReport = manifestMergerInvoker.merge()
        //省略其他对merge结果处理代码
        ... ...
        return mergingReport
    } catch (e: ManifestMerger2.MergeFailureException) {
        // TODO: unacceptable.
        throw RuntimeException(e)
    }
}
```

接着看manifestMergerInvoker.merge()的实现
```
package com.android.manifmerger;

/**
 * merges android manifest files, idempotent.
 */
@Immutable
public class ManifestMerger2 {
    public static class Invoker<T extends Invoker<T>>{

        @NonNull
        public MergingReport merge() throws MergeFailureException {

            // provide some free placeholders values.
            ImmutableMap<ManifestSystemProperty, Object> systemProperties = mSystemProperties.build();
            ... ...
            FileStreamProvider fileStreamProvider = mFileStreamProvider != null
                    ? mFileStreamProvider : new FileStreamProvider();
            ManifestMerger2 manifestMerger =
                    new ManifestMerger2(
                            mLogger,
                            mMainManifestFile,
                            mLibraryFilesBuilder.build(),
                            mFlavorsAndBuildTypeFiles.build(),
                            mFeaturesBuilder.build(),
                            mPlaceholders.build(),
                            new MapBasedKeyBasedValueResolver<ManifestSystemProperty>(
                                    systemProperties),
                            mMergeType,
                            mDocumentType,
                            Optional.fromNullable(mReportFile),
                            mFeatureName,
                            fileStreamProvider,
                            mNavigationFilesBuilder.build());
            //调用下面的 private MergingReport merge()方法               
            return manifestMerger.merge();
        }
    }


    /**
     * Perform high level ordering of files merging and delegates actual merging to
     * {@link XmlDocument#merge(XmlDocument, com.android.manifmerger.MergingReport.Builder)}
     *
     * @return the merging activity report.
     * @throws MergeFailureException if the merging cannot be completed (for instance, if xml
     * files cannot be loaded).
     */
    @NonNull
    private MergingReport merge() throws MergeFailureException {
        // initiate a new merging report
        MergingReport.Builder mergingReportBuilder = new MergingReport.Builder(mLogger);
        //一系列merge manifest规则处理
        ... ...
        MergingReport mergingReport = mergingReportBuilder.build();

        if (mReportFile.isPresent()) {
            writeReport(mergingReport);
        }
        return mergingReport;
    }

    //最终写入Log文件方法
    /**
     * Creates the merging report file.
     * @param mergingReport the merging activities report to serialize.
     */
    private void writeReport(@NonNull MergingReport mergingReport) {
        FileWriter fileWriter = null;
                ... ... 
                fileWriter = new FileWriter(mReportFile.get());
                mergingReport.getActions().log(fileWriter);
    } 
}        
```

到目前为止,从代码层面看到了Log文件是如何生成的。
# 方案实现

【manifest-merger-${variantname}-report.txt】文件大致内容如下：
```
-- Merging decision tree log ---
manifest
ADDED from /somepath/AndroidManifest.xml:x:x-xx:xx
MERGED from [dependencies sdk] /somepath/AndroidManifest.xml:x:x-xx:xx
INJECTED from /somepath/AndroidManifest.xml:x:x-xx:xx
...
uses-permission#android.permission.INTERNET
```

方案代码实现很简单：
>1.自定义一个Extension，列出暂禁用的权限；
>2.实现相应Plugin和Task；

Extension定义可以如下所示：
```
host{
       //明确暂禁用的权限列表
       forbiddenPermissions = ['android.permission.GET_ACCOUNTS',
                            'android.permission.SEND_SMS',
                            'android.permission.CALL_PHONE',
                            'android.permission.BLUETOOTH',
                             ... ...] 
}
```

Plugin简单示例：
```
public  class HostPlugin implements Plugin<Project> {
    @Override
    final void apply(Project project) {
        if (!project.getPlugins().hasPlugin('com.android.application') && !project.getPlugins().hasPlugin('com.android.library')) {
            throw new GradleException('apply plugin: \'com.android.application\' or apply plugin: \'com.android.library\' is required')
        }
        HostExtension hostExtension = project.getExtensions().create('host', HostExtension.class)
        
        project.afterEvaluate {
            def variants = null;
            if (project.plugins.hasPlugin('com.android.application')) {
                variants = android.getApplicationVariants()
            } else if (project.plugins.hasPlugin('com.android.library')) {
                variants = android.getLibraryVariants()
            }
            variants?.all { BaseVariant variant ->
               MergeHostManifestTask taskConfiguration=  new MergeHostManifestTask.CreationAction()
               project.getTasks().create(taskConfiguration.getName(), taskConfiguration.getType(), taskConfiguration)
            }
        }
    }   
}
```

Task简单示例：
```
import org.gradle.util.GFileUtils
import com.android.utils.FileUtils

class MergeHostManifestTask extends DefaultTask {

    List<String> forbiddenPermissions //禁用的权限列表

    VariantScope scope

    @TaskAction
    def doFullTaskAction() {

        File logFile = FileUtils.join(
                scope.getGlobalScope().getOutputsDir(),
                "logs",
                "manifest-permissions-validate-"
                        + scope.getVariantConfiguration().getBaseName()
                        + "-report.txt")
        GFileUtils.mkdirs(logFile.getParentFile())
        GFileUtils.deleteQuietly(logFile)  

        checkHostManifest(forbiddenPermissions,logFile,scope)
        if (logFile.exists() && logFile.length() > 0) {
            throw new GradleException("Has forbidden permissions in host, please check it in file ${logFile.getAbsolutePath()}")
        }              
    }    

    /**
     * 检测host manifest 是否含有禁用权限列表
     * @param forbiddenPermissions
     * @param logFile
     * @param variantScope
     */
    public static void checkHostManifest(List<String> forbiddenPermissions, File logFile, def variantScope) {
        if (forbiddenPermissions == null || forbiddenPermissions.isEmpty()) {
            return
        }

        File reportFile =
                FileUtils.join(
                        variantScope.getGlobalScope().getOutputsDir(),
                        "logs",
                        "manifest-merger-"
                                + variantScope.getVariantConfiguration().getBaseName()
                                + "-report.txt")

        if (!reportFile.exists()) {
            return
        }

        reportFile.withReader { reader ->
            String line
            while ((line = reader.readLine()) != null) {
                forbiddenPermissions.each { p ->
                    if (line.contains("uses-permission#${p.trim()}")) {
                        logFile.append("${p.trim()}\n")
                        logFile.append(reader.readLine())
                        logFile.append("\n")
                    }
                }
            }
        }
    }

    public static class CreationAction
            extends TaskConfiguration<MergeHostManifestTask> {

        BaseVariant variant

        Project project

        public CreationAction(Project project,BaseVariant variant){
            this.project= project
            this.variant=variant
        }

        @Override
        void execute(MergeHostManifestTask task) {
            ... ...
            HostExtension hostExtension = project.getExtensions().findByType(HostExtension.class)
            task.forbiddenPermissions = hostExtension.getForbiddenPermissions()
            task.scope= variant.getMetaClass().getProperty(variant, 'variantData').getScope()
            task.dependsOn getProcessManifestTask()
        }

       private Task getProcessManifestTaskCompat() {
        try {
            //>=3.3.0
            String taskName = variant.getMetaClass().getProperty(variant, 'variantData').getScope().getTaskContainer().getProcessManifestTask().getName()
            return project.getTasks().findByName(taskName)
        } catch (Exception e) {

        }
    }
}    
```

如果APP或其依赖的SDK,有引入禁用权限，则会抛出编译异常，生成的【manifest-permissions-validate-${variantname}-report.txt】文件内容类似以下所示：
```
android.permission.SEND_SMS
ADDED from /../app/src/main/AndroidManifest.xml:9:5-67
android.permission.BLUETOOTH
ADDED from /../app/src/main/AndroidManifest.xml:11:5-68
```

# 结束语
关于隐私权限列表，相关部门也未给允一个完整的列表，建议团队把所有未在隐私文档中描述的动态权限都作为禁用权限，直至隐私文档同步。

# 参考
1.Android Gradle Plugin：[https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core)  
