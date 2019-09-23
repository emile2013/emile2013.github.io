---
title: 治理令人头痛的pthread_create OutOfMemoryError错误
date: 2019-09-20 17:44:10
categories: 
    - Android
tags: 
    - 应用性能
---
# 引言
我相信很多团队都面对过令人头痛的pthread_create stack内存溢出问题。在Android中,一个典型的pthread_create stack内存溢出堆栈信息如下：

```
java.lang.OutOfMemoryError
pthread_create (1040KB stack) failed: Try again
1 java.lang.Thread.nativeCreate(Native Method)
2 java.lang.Thread.start(Thread.java:733)
3 java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:975)
4 java.util.concurrent.ThreadPoolExecutor.processWorkerExit(ThreadPoolExecutor.java:1043)
5 java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1185)
6 java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
7 java.lang.Thread.run(Thread.java:764)
```

出现栈内存溢出无非两个原因：  
1、线程数达到了系统最大限制数；  
2、进程的栈内存超过了虚拟机的最大内存数。  
关于线程数达到了系统最大限制数，在国内手机厂商中，华为手机在7.0+手机上已将最大线程数修改成了300。我们APP有大量的华为用户，不得不面对华为系统限制问题。  
Android Dalvik和ART，将stack分为了java stack和native stack，本文没去具体实验，从[kongxinsun](https://blog.csdn.net/kongxinsun/article/details/78679860)的博客，我们了解到两者总量是1056KB。栈内存回收和堆内存策略不一样，比较简单，当线程结束，线程占用的栈内存也就回收了。  
<!-- more -->
接下来通过简述Android中常见的使用线程方式，尝试给出解决方案，优化线程引起的性能问题。
# 线程使用
在Android中，我们使用新建线程，无非就是想避免一些耗时操作，影响主线程响应。常见的相关类，比如Timer,ThreadHandle，AsyncTask，ThreadPoolExecutor等等都和线程直接相关。相关网络库、图片库、埋点库和其他三方SDK等都会存在大量使用线程场景。  
查看当前所有激活的线程，我们可以通过以下API获取：
```
//输出总数
File file = new File(Environment.getExternalStorageDirectory(), "threads.txt");
BufferedWriter writer = new BufferedWriter(new FileWriter(file, false));
writer.write("count:" + Thread.getAllStackTraces().size() + "\n");
for (Map.Entry<Thread, StackTraceElement[]> entry : Thread.getAllStackTraces().entrySet()) {
    writer.write(entry.getKey().getName() + ":" + "\n");
    for (StackTraceElement traceElement : entry.getValue()) {
        writer.write("\tat " + traceElement + "\n");
    }
}
```

```
//输出内容
count:166
ConnectivityThread:
    at android.os.MessageQueue.nativePollOnce(Native Method)
    at android.os.MessageQueue.next(MessageQueue.java:336)
    at android.os.Looper.loop(Looper.java:174)
    at android.os.HandlerThread.run(HandlerThread.java:67)
pool-16-thread-1:
    at sun.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2067)
    at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at java.lang.Thread.run(Thread.java:919)
BundleTaskExecutor #5:
    at sun.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
    at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
    at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
    at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at java.lang.Thread.run(Thread.java:919)
execute_task:
    at android.os.MessageQueue.nativePollOnce(Native Method)
    at android.os.MessageQueue.next(MessageQueue.java:336)
    at android.os.Looper.loop(Looper.java:174)
    at android.os.HandlerThread.run(HandlerThread.java:67)
ClassLoaderCreator #4:
    at sun.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2109)
    at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at java.lang.Thread.run(Thread.java:919)
```
这种方式我们能拿到Java层与其对接的native层thread总数，但是拿不到没有attach到java层的native thread，比如Futter engine中的native thread等。  
如果为线程分配一个可定位到代码的名称，那我们完全能对症下药。但实际很难，我们没法约束开发同学和三方SDK为每个线程起自定义名称，例如上面的pool-16-thread-1的线程，我们很难定位到是哪个类发起的调用。  
另外我们也可以直接dump进程内存，来查看内存情况，使用方式如下所示：
```
adb shell dumpsys meminfo [pacakgename]

Applications Memory Usage (in Kilobytes):
Uptime: 122226380 Realtime: 451586500
** MEMINFO in pid 19468 [pacakgename] **
                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap   109473   109384        0       62   128460   122131     6328
  Dalvik Heap    17679    17604        0       63    26514    13257    13257
 Dalvik Other     5481     5480        0        0
        Stack       88       88        0        0
       Ashmem      326      124        0        0
      Gfx dev    55952    55952        0        0
    Other dev      216        0      216        0
     .so mmap    17532     1012    13224       21
    .jar mmap     2232        0       80        0
    .apk mmap    34716      132    19520        0
    .ttf mmap      199        0      140        0
    .dex mmap    45759    44872      492        0
    .oat mmap      213        0       80        0
    .art mmap     4888     4440        8       40
   Other mmap     9466      836     6200        0
   EGL mtrack    28572    28572        0        0
    GL mtrack    18996    18996        0        0
      Unknown     3572     3564        0        5
        TOTAL   355551   291056    39960      191   154974   135388    19585
...        
```
也可以查看线程等汇总数据，使用方式如下所示：
```
adb shell
sargo:/ $ cat /proc/19468/status
Name:   xxxx
Umask:  0077
State:  S (sleeping)
Tgid:   19468
Ngid:   0
Pid:    19468
PPid:   789
TracerPid:  0
Uid:    10243   10243   10243   10243
Gid:    10243   10243   10243   10243
FDSize: 512
Groups: 3001 3002 3003 9997 20243 50243
VmPeak:  2657056 kB
VmSize:  2301416 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:    409676 kB
VmRSS:    236604 kB
RssAnon:      128356 kB
RssFile:      103380 kB
RssShmem:       4868 kB
VmData:  1470524 kB
VmStk:      8192 kB
VmExe:        20 kB
VmLib:    148356 kB
VmPTE:      1936 kB
VmPMD:        16 kB
VmSwap:     4052 kB
Threads:    149
SigQ:   0/13364
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000080001204
SigIgn: 0000000000000001
SigCgt: 0000000e400084f8
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000000000000000
CapAmb: 0000000000000000
Seccomp:    2
Speculation_Store_Bypass:   unknown
Cpus_allowed:   30
Cpus_allowed_list:  4-5
Mems_allowed:   1
Mems_allowed_list:  0
voluntary_ctxt_switches:    123554
nonvoluntary_ctxt_switches: 3887
```

# 线程优化
要优化线程数和线程栈，我们只能避免创建多余的线程，在合适的时机去结束空闲线程，来达到优化目的。接下来列出几个优化策略。
### 1.提供线程池管理库
为App提供线程池管理SDK，目的很简单，能避免业务组各自作战，从底层限制线程使用浪费。在SDK中我们可以提供IO，密集计算，单线程等API，方便上层调用同时，也能限制线程数。这里需要注意的是，我们要自定义实现一个ThreadFactory，用于为每个线程重命名，如以下代码：
```
public class VThreadFactory implements ThreadFactory {
    private AtomicInteger mThreadNumber = new AtomicInteger(1);

    @Override
    public Thread newThread(@NonNull final Runnable r) {
        Runnable wrapperRunnable;
        if (mThreadPriority == Process.THREAD_PRIORITY_DEFAULT) {
            wrapperRunnable = r;
        } else {
            wrapperRunnable = new Runnable() {
                @Override
                public void run() {
                    try {
                        Process.setThreadPriority(mThreadPriority);
                    } catch (Throwable ignore){}
                    r.run();
                }
            };
        }
        Thread t = new Thread(group, wrapperRunnable,
                String.format("%s-%s-thread", mPrefix, mThreadNumber.getAndIncrement()));
        if (t.isDaemon()) {
            t.setDaemon(false);
        }
        return t;
    }
}    
```
除了为每个线程重命名,我们还要为线程池调用以下API：
```
    /**
     * Sets the policy governing whether core threads may time out and
     * terminate if no tasks arrive within the keep-alive time, being
     * replaced if needed when new tasks arrive. When false, core
     * threads are never terminated due to lack of incoming
     * tasks. When true, the same keep-alive policy applying to
     * non-core threads applies also to core threads. To avoid
     * continual thread replacement, the keep-alive time must be
     * greater than zero when setting {@code true}. This method
     * should in general be called before the pool is actively used.
     *
     * @param value {@code true} if should time out, else {@code false}
     * @throws IllegalArgumentException if value is {@code true}
     *         and the current keep-alive time is not greater than zero
     *
     * @since 1.6
     */
    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            if (value)
                interruptIdleWorkers();
        }
   }
```
含义为在time out时，能同时结束core threads。
这里建议线程池管理SDK提供的池程数不要超过100，为其他三方SDK和系统留至少三分之二的线程额度。

### 2.少用HandlerThread
HandlerThread是Android提供的和Looper绑定的Thread辅助类，借助HandlerThread我们能在子线程中处理消息。但在实际使用过程，我们经常用静态类来避免内存泄漏，例如以下调用：
```
final static HandlerThread handlerThread;
final static Handler handler;
static {
        handlerThread = new HandlerThread("Bundle-thread");
        handlerThread.start();
        handler = new Handler(handlerThread.getLooper());
}
```
这种使用就会造成HandlerThread一直存在于内存中，thread实例不会结束。如果HandlerThread实例中还存在嵌套其他Thread或Stack内存相关函数，那就更不合理了。
### 3.优化常见的池程池
常见池程池的使用，比如AsyncTask类中用THREAD_POOL_EXECUTOR来执行异步操作，我们可以在Appliction或其他初始化代码块中执行以下代码：
```
try {
            final ThreadPoolExecutor executor = ((ThreadPoolExecutor) android.os.AsyncTask.THREAD_POOL_EXECUTOR);
            executor.allowCoreThreadTimeOut(true);
} catch (final Throwable t) {
            Log.e("OptAsyncTask", "Optimize AsyncTask executor error: allowCoreThreadTimeOut = true", t);
}
```

如果团队用的Okhttp作为网络基础库，尽量所有网络请求用同个HttpClient实例。Okhttp在Dispatcher、ConnectionPool等类中，用到了类实例或static线程池等。
### 4.为每个线程命名
如果团队在经过一系列优化后，还是避免不了pthread OOM异常，那我们就要从异常Log中能快速定位到问题代码。默认新建线程和Executors类中命名线程相关代码如下：
```
    /**
     * Allocates a new {@code Thread} object. This constructor has the same
     * effect as {@linkplain #Thread(ThreadGroup,Runnable,String) Thread}
     * {@code (null, null, gname)}, where {@code gname} is a newly generated
     * name. Automatically generated names are of the form
     * {@code "Thread-"+}<i>n</i>, where <i>n</i> is an integer.
     */
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
```
```
DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
}
```
如何我们要快速定位问题线程，采用默认方式肯定不行，所以我们要修改字节码，重新为每个线程命名，在新名字，带上使用类或其他有用信息。
滴滴团队估计也面临了相同问题，在开源[booster](https://github.com/didi/booster)中利用ASM对Java字节码修改,实现了上述我们要为每个线程命名的需求。目前为止，booster在 [ThreadTransformer](https://github.com/didi/booster/blob/master/booster-transform-thread/src/main/kotlin/com/didiglobal/booster/transform/thread/ThreadTransformer.kt) 存在修改节码缺陷,以及[ShadowExecutors.newOptimizedFixedThreadPool](https://github.com/didi/booster/blob/aa3f74eedb70a47cd657e1dfc23361ffed988aa4/booster-android-instrument-thread/src/main/java/com/didiglobal/booster/instrument/ShadowExecutors.java)方法中错误的使用了LinkedBlockingQueue队列等缺陷，建议大家如果采用booster开源实现，尽量多作一些测试以及代码修复。
# 结束语
当APP功能逐渐庞大时，带来的不仅是包的大小，也同时带来了各种性能问题。pthread oom没有根治办法，我们只能减少发生量，尽量在性能测试中，提前发现问题，推动问题修复。

# 参考
1.android java process stack OOM：[https://blog.csdn.net/kongxinsun/article/details/78679860](https://blog.csdn.net/kongxinsun/article/details/78679860)  
2.dumpsys:[https://developer.android.com/studio/command-line/dumpsys](https://developer.android.com/studio/command-line/dumpsys)  
3.booster:[https://github.com/didi/booster](https://github.com/didi/booster)
