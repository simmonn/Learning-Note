# JRF&JMC&JCMD

## JFR

### 什么是JFR

> JFR 是 [Java](https://so.csdn.net/so/search?q=Java&spm=1001.2101.3001.7020) Flight Record （Java飞行记录） 的缩写，是 JVM 内置的基于事件的JDK监控记录框架。这个起名就是参考了黑匣子对于飞机的作用，将Java进程比喻成飞机飞行。顾名思义，这是一个收集运行的JAVA应用的**诊断信息**和**分析数据**。**JDK11**开始集成进JVM，几乎不会造成性能开销，因此即使在生产环境中也可以使用。
>
> 如果是利用默认配置启动这个记录，性能非常高效，对于业务影响很小，因为这个框架本来就是用来长期在线上部署的框架。这个记录可以输出成二进制文件，用户可以指定最大记录时间，或者最大记录大小，供用户在需要的时候输出成文件进行事后分析。

### 为什么使用JFR

> 有些问题开发测试阶段很难发现，需要在生产环境才会出现。JFR使用在生产环境，可以更好的定位问题。官方说，目标是开启 JFR 监控（默认配置），对性能的影响在1%之内，对JVM Runtime 和 GC，OS 以及 Java 库进行全方位的监控。

**注意**：生产环境配置不建议长期使用profile，否则会增加很大的性能开销。settings的详细配置在**$JAVA_HOME/lib/jfr/**，default.jfr和profile.jfr。

#### 有哪些关键特性

- 低开销（在配置正确的情况下），可在生产环境核心业务进程中始终在线运行。当然，也可以随时开启与关闭。
- 可以查看出问题时间段内进行分析，可以分析 Java 应用程序，JVM 内部以及当前Java进程运行环境等多因素。
- JFR基于事件采集，可以分析非常底层的信息，例如对象分配，方法采样与热点方法定位与调用堆栈，安全点分析与锁占用时长与堆栈分析，GC 相关分析以及 JIT 编译器相关分析（例如 CodeCache ） 
-  完善的 API 定义，用户可以自定义事件生产与消费。

### JFR的核心-事件Event

在JFR中，任何JVM的行为都是一个Event，例如类加载（Class Load Event），CPU负载变动（CPULoad），垃圾回收（GarbageCollection）等。记录、丢失Event都属于一个Event。

#### Event按照采集方式分为三种：

1. Instant Event：顾名思义，这种 Event 在发生时就立刻采集。例如：Throw Exception Event 还有 Thread Start Event，类似于这种在某一时刻发生的 Event
2. Duration Event：这种 Event 需要耗费一些时间，在完成的时候会记录。对于这种类型的 Event，可以设置一个时间限制，超过这个时间限制的才会记录。例如 GC Event，Thread Sleep Event。
3. Sample Event（或者是Requestable Event）：按照一定的频率采集，这个频率是可以配置的。例如 Thread Dump Event，Method Sampling Event

Event会被写入到.jfr二进制文件中，可通过可视化工具JMC去查看。

### 如何开启JFR

官方文档：[Running Java Flight Recorder](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/run.htm#JFRUH164)

可以通过启动参数在 JVM 进程启动的时候就启动 JFR，或者是利用 jcmd 工具，动态启用或者关闭 JFR。

#### JVM参数启动

启动命令：

```
java -XX:StartFlightRecording=disk=true,dumponexit=true,filename=recording.jfr,maxsize=1024m,maxage=1d,settings=profile,path-to-gc-roots=true test.Main

```

在 JDK 11 版本之后，启动参数被简化了很多很多；目前JFR涉及的参数仅仅只有两个，一个负责启动（`-XX:StartFlightRecording`），一个负责配置（`-XX:FlightRecorderOptions`）。JDK 8中的`-XX:+FlightRecorder`打开 FlightRecorder 状态。核心就是 `-XX:StartFlightRecording`，有了这个参数就会启用 JFR 记录。

##### -XX:StartFlightRecording参数说明

| 参数名 | 默认值 | 说明 |
|----|----|----|
|delay | 0 | 延迟多久后启动 JFR 记录，支持带单位配置， 例如 delay=60s（秒）， delay=20m（分钟）， delay=1h（小时）， delay=1d（天），不带单位就是秒， 0就是没有延迟直接开始记录。一般为了避免框架初始化等影响，我们会延迟 1 分钟开始记录（例如Spring cloud应用，可以看下日志中应用启动耗时，来决定下这个时间）。|
|disk | true | 是否写入磁盘，这个就是上文提到的， global buffer 满了之后，是直接丢弃还是写入磁盘文件。|
|dumponexit | false | 程序退出时，是否要dump出 .jfr文件|
|duration | 0 | JFR 记录持续时间，同样支持单位配置，不带单位就是秒，0代表不限制持续时间，一直记录。|
|filename | - | [启动目录]/hotspot-pid-26732-id-1-2020_03_12_10_07_22.jfr，pid 后面就是 pid， id 后面是第几个 JFR 记录，可以启动多个 JFR 记录。最后就是时间。dump的输出文件 |
|name | 无 | 记录名称，由于可以启动多个 JFR 记录，这个名称用于区分，否则只能看到一个记录 id，不好区分。|
|maxage | 0 | 这个参数只有在 disk 为 true 的情况下才有效。最大文件记录保存时间，就是 global buffer 满了需要刷入本地临时目录下保存，这些文件最多保留多久的。也可以通过单位配置，没有单位就是秒，默认是0，就是不限制|
|maxsize | 250MB | 这个参数只有在 disk 为 true 的情况下才有效。最大文件大小，支持单位配置， 不带单位是字节，m或者M代表MB，g或者G代表GB。设置为0代表不限制大小**。虽然官网说默认就是0，但是实际用的时候，不设置会有提示**： No limit specified, using maxsize=250MB as default. 注意，这个配置不能小于后面将会提到的 maxchunksize 这个参数。|
|path-to-gc-roots| false | 是否记录GC根节点到活动对象的路径，一般不打开这个，首先这个在我个人定位问题的时候，很难用到，只要你的编程习惯好。还有就是打开这个，性能损耗比较大，会导致FullGC一般是在怀疑有内存泄漏的时候热启动这种采集，并且通过产生对象堆栈无法定位的时候，动态打开即可。一般通过产生这个对象的堆栈就能定位，如果定位不到，怀疑有其他引用，例如 ThreadLocal 没有释放这样的，可以在 dump 的时候采集 gc roots|
|settings | default | 采集 Event 的详细配置，采集的每个 Event 都有自己的详细配置。另一个 JDK 自带的配置是 profile.jfc，位于 `$JAVA_HOME/lib/jfr/profile.jfc`。 |

##### -XX:FlightRecorderOption参数说明

| 参数名 | 默认值 | 说明 |
| ---- | ---- | ---- |
|allow_threadbuffers_to_disk | false | 是否允许 在 thread buffer 线程阻塞的时候，直接将 thread buffer 的内容写入文件。默认不启用，一般没必要开启这个参数，只要你设置的参数让 global buffer 大小合理不至于刷盘很慢，就行了。|
|globalbuffersize | 如果不设置，根据设置的 memorysize 自动计算得出 | 单个 global buffer 的大小，一般通过 memorysize 设置，不建议自己设置|
|maxchunksize | 12M | 存入磁盘的每个临时文件的大小。默认为12MB，不能小于1M。可以用单位配置，不带单位是字节，m或者M代表MB，g或者G代表GB。注意这个大小最好不要比 memorySize 小，更不能比 globalbuffersize 小，否则会导致性能下降|
|memorysize | 10M | JFR的 global buffer 占用的整体内存大小，一般通过设置这个参数，numglobalbuffers 还有 globalbuffersize 会被自动计算出。可以用单位配置，不带单位是字节，m或者M代表MB，g或者G代表GB。|
|numglobalbuffers | 如果不设置，根据设置的 memorysize 自动计算得出 | global buffer的个数，一般通过 memorysize 设置，不建议自己设置|
|old-object-queue-size | 256 | 对于Profiling中的 Old Object Sample 事件，记录多少个 Old Object，这个配置并不是越大越好。记录是怎么记录的，会在后面的各种 Event 介绍里面详细介绍。我的建议是，一般应用256就够，时间跨度大的，例如 maxage 保存了一周以上的，可以翻倍|
|repository | 等同于 -Djava.io.tmpdir 指定的目录 | JFR 保存到磁盘的临时记录的位置|
|retransform | true | 是否通过 JVMTI 转换 JFR 相关 Event 类，如果设置为 false，则只在 Event 类加载的时候添加相应的 Java Instrumentation，这个一般不用改，这点内存 metaspace 还是足够的|
|samplethreads | true | 这个是是否开启线程采集的状态位配置，只有这个配置为 true，并且在 Event 配置中开启线程相关的采集（这个后面会提到），才会采集这些事件。|
|stackdepth | 64 | 采集事件堆栈深度，有些 Event 会采集堆栈，这个堆栈采集的深度，统一由这个配置指定。注意这个值不能设置过大，如果你采集的 Event种类很多，堆栈深度大很影响性能。比如你用的是 default.jfc 配置的采集，堆栈深度64基本上就是不影响性能的极限了。你可以自定义采集某些事件，增加堆栈深度。|
|threadbuffersize | 8KB | threadBuffer 大小，最好不要修改这个，如果增大，那么随着你的线程数增多，内存占用会增大。过小的话，刷入 global buffer 的次数就会变多。8KB 就是经验中最合适的。|

#### JCMD启动

至于JCMD是什么，可以看下[JCMD](#JCMD)

1、**jcmd <pid> JFR.start**：启动 JFR 记录，参数和`-XX:StartFlightRecording`一模一样，**请参考上面的表格**。但是注意这里不再是逗号分割，而是空格 示例：

```
jcmd 21 JFR.start name=profile_online maxage=1d maxsize=1g
```

这个就代表启动一个名称为 profile_online, 最多保留一天，最大保留 1G 的本地文件记录

2、**jcmd <pid> JFR.stop**. 停止 JFR 记录，需要传入名称，例如如果要停止上面打开的，则执行：

```text
jcmd 21 JFR.stop name=profile_online copy_to_file=profile.jfr
```

**copy_to_file**:停止时同时复制到文件，指定文件输出位置

3、jcmd <pid> JFR.check，查看当前正在执行的 JFR 记录。

示例：

```text
jcmd 21 JFR.check verbose=true
```

输出：

```text
21:
Recording 1: name=profile_online maxsize=1.0GB maxage=1d (running)
```

verbose：是否查看每种Event采集详细配置

4、**jcmd <pid> JFR.configure**，如果不传入参数，则是查看当前配置，传入参数就是修改配置。配置与-XX:FlightRecorderOptions的一模一样。**请参考上面的表格** 示例：

```text
./jcmd 21 JFR.configure
```

输出：

```text
Repository path: /tmp/2020_03_18_08_41_44_21

Stack depth: 64
Global buffer count: 20
Global buffer size: 512.0 kB
Thread buffer size: 8.0 kB
Memory size: 10.0 MB
Max chunk size: 12.0 MB
Sample threads: true
```

示例：

```text
./jcmd 21 JFR.configure stackdepth=65
```

输出：

```text
21:
Stack depth: 65
```

5、**`jcmd <pid> JFR.dump`** :dump性能日志

|参数 |默认值| 描述|
|----|----|----|
|name | 无 | 指定要查看的 JFR 记录名称|
|filename | 无 | 指定文件输出位置|
|maxage | 0 | dump最多的时间范围的文件，可以通过单位配置，没有单位就是秒，默认是0，就是不限制|
|maxsize | 0 | dump最大文件大小，支持单位配置， 不带单位是字节，m或者M代表MB，g或者G代表GB。设置为0代表不限制大小|
|begin | 无 | dump开始位置， 可以这么配置：09:00, 21:35:00, 2018-06-03T18:12:56.827Z, 2018-06-03T20:13:46.832, -10m, -3h, or -1d|
|end : | 无| dump结束位置，可以这么配置： 09:00, 21:35:00, 2018-06-03T18:12:56.827Z, 2018-06-03T20:13:46.832, -10m, -3h, or -1d (STRING, no default value)|
|path-to-gc-roots| false | 是否记录GC根节点到活动对象的路径，一般不记录，dump 的时候打开这个肯定会触发一次 fullGC，对线上应用有影响。最好参考之前对于 JFR 启动记录参数的这个参数的描述，考虑是否有必要|

# JMC

### 什么是JMC

JMC（JAVA MISSION CONTROL）是和JFR结合使用的，由于JFR生成的报告是二进制文件，所以需要有一个可视化界面去分析，那么JMC就是这个可视化工具。可以查看应用（线程、内存、IO、异常等）、JVM（GC、GC配置、类加载等）、环境（环境变量、系统变量等）的详细信息。

**可以通过连接JVM或者读取jfr文件读取相应信息**

具体使用不再做详细赘述，感兴趣的可以看下这篇博客:[JMC使用说明](https://blog.csdn.net/yue530tomtom/article/details/80805412)



![JMC](https://tva1.sinaimg.cn/large/e6c9d24egy1gzxwskxqo2j21ga0u0jvl.jpg)



## JCMD

> 发送诊断命令请求到正在运行的Java虚拟机（JVM）。它必须和JVM运行在同一台机器上，并且与启动JVM用户具有相同的组权限。

可通过jcmd <PID> help 查看命令，下面列了一些常用

### 常用命令

|命令|    描述|
|----|----|
|jcmd PID VM.uptime |查看 JVM 的启动时长|
|jcmd PID GC.class_histogram    |查看 JVM 的类信息，这个可以查看每个类的实例数量和占用空间大小。|
|jcmd PID Thread.print  |查看 JVM 的Thread Dump|
|jcmd PID GC.heap_dump FILE_NAME    |查看 JVM 的Heap Dump,注意，如果只指定文件名，默认会生成在启动 JVM 的目录里。|
|jcmd PID VM.system_properties  |查看 JVM 的属性信息|
|jcmd PID VM.flags  |查看 JVM 的启动参数,注意，可以看到 -X 和 -XX 的参数信息|
|jcmd PID VM.command_line   |查看 JVM 的启动命令行|
|jcmd PID GC.run_finalization   |对 JVM 执行 java.lang.System.runFinalization(),尽量b别去调用这个对象的finalize方法。|
|jcmd PID GC.run    |对 JVM 执行 java.lang.System.gc()，告诉垃圾收集器打算进行垃圾收集，而垃圾收集器进不进行收集是不确定的|
|jcmd PID PerfCounter.print |查看 JVM 的性能|

