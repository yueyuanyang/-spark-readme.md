## Spark配置

### Spark配置

Spark有以下三种方式修改配置：

- Spark properties （Spark属性）可以控制绝大多数应用程序参数，而且既可以通过 SparkConf 对象来设置，也可以通过Java系统属性来设置。
- Environment variables （环境变量）可以指定一些各个机器相关的设置，如IP地址，其设置方法是写在每台机器上的conf/spark-env.sh中。
- Logging （日志）可以通过log4j.properties配置日志。

### Spark属性

Spark属性可以控制大多数的应用程序设置，并且每个应用的设定都是分开的。这些属性可以用SparkConf 对象直接设定。SparkConf为一些常用的属性定制了专用方法（如，master URL和application name），其他属性都可以用键值对做参数，调用set()方法来设置。例如，我们可以初始化一个包含2个本地线程的Spark应用，代码如下：

注意，local[2]代表2个本地线程 – 这是最小的并发方式，可以帮助我们发现一些只有在分布式上下文才能复现的bug。

```
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
```

### 动态加载Spark属性

在某些场景下，你可能需要避免将属性值写死在 SparkConf 中。例如，你可能希望在同一个应用上使用不同的master或不同的内存总量。Spark允许你简单地创建一个空的SparkConf对象：
```
val sc = new SparkContext(new SparkConf())
```

然后在运行时设置这些属性：

```
./bin/spark-submit \
--name "My app" \
--master local[4] \
--conf spark.eventLog.enabled=false \
--conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
```

Spark shell和spark-submit工具支持两种动态加载配置的方法。第一种，通过命令行选项，如：上面提到的–master（设置master URL）。spark-submit可以在启动Spark应用时，通过–conf标志接受任何属性配置，同时有一些特殊配置参数同样可用（如，–master）。运行./bin/spark-submit –help可以展示这些选项的完整列表。

同时，bin/spark-submit 也支持从conf/spark-defaults.conf 中读取配置选项，在该文件中每行是一个键值对，并用空格分隔，如下：

```
spark.master            spark://5.6.7.8:7077
spark.executor.memory   4g
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer
```

这些通过参数或者属性配置文件传递的属性，最终都会在SparkConf 中合并。其优先级是：首先是SparkConf代码中写的属性值，其次是spark-submit或spark-shell的标志参数，最后是spark-defaults.conf文件中的属性。

有一些配置项被重命名过，这种情形下，老的名字仍然是可以接受的，只是优先级比新名字优先级低。

### 查看Spark属性

每个SparkContext都有其对应的Spark UI，所以Spark应用程序都能通过Spark UI查看其属性。默认你可以在这里看到：http://<driver>:4040，页面上的”Environment“ tab页可以查看Spark属性。如果你真的想确认一下属性设置是否正确的话，这个功能就非常有用了。注意，只有显式地通过SparkConf对象、在命令行参数、或者spark-defaults.conf设置的参数才会出现在页面上。其他属性，你可以认为都是默认值。

### 可用的属性

绝大多数属性都有合理的默认值。这里是部分常用的选项：

### 应用属性

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.app.name	| (none) |	Spark应用的名字。会在SparkUI和日志中出现。
|spark.driver.cores	| 1	| 在cluster模式下，用几个core运行驱动器（driver）进程。
|spark.driver.maxResultSize	| 1g |	Spark action算子返回的结果最大多大。至少要1M，可以设为0表示无限制。如果结果超过这一大小，Spark作业（job）会直接中断退出。但是，设得过高有可能导致驱动器OOM（out-of-memory）（取决于spark.driver.memory设置，以及驱动器JVM的内存限制）。设一个合理的值，以避免驱动器OOM。
|spark.driver.memory	| 1g	 |驱动器进程可以用的内存总量（如：1g，2g）。注意，在客户端模式下，这个配置不能在SparkConf中直接设置（因为驱动器JVM都启动完了呀！）。驱动器客户端模式下，必须要在命令行里用 –driver-memory 或者在默认属性配置文件里设置。
|spark.executor.memory	| 1g	| 单个执行器（executor）使用的内存总量（如，2g，8g）
|spark.extraListeners	|(none)	|逗号分隔的SparkListener子类的类名列表；初始化SparkContext时，这些类的实例会被创建出来，并且注册到Spark的监听总线上。如果这些类有一个接受SparkConf作为唯一参数的构造函数，那么这个构造函数会被优先调用；否则，就调用无参数的默认构造函数。如果没有构造函数，SparkContext创建的时候会抛异常。
|spark.local.dir |	/tmp	| Spark的”草稿“目录，包括map输出的临时文件，或者RDD存在磁盘上的数据。这个目录最好在本地文件系统中，这样读写速度快。这个配置可以接受一个逗号分隔的列表，通常用这种方式将文件IO分散不同的磁盘上去。注意：Spark-1.0及以后版本中，这个属性会被集群管理器所提供的环境变量覆盖：SPARK_LOCAL_DIRS（独立部署或Mesos）或者 LOCAL_DIRS（YARN）。
|spark.logConf |	false	| SparkContext启动时是否把生效的 SparkConf 属性以INFO日志打印到日志里
|spark.master	| (none)	| 集群管理器URL。参考allowed master URL’s.

除了这些以外，以下还有很多可用的参数配置，在某些特定情形下，可能会用到：

### 运行时环境

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.driver.extraClassPath | (none) | 额外的classpath，将插入到到驱动器的classpath开头。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf 在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-class-path）或者在 conf/spark-defaults.conf 配置。
|spark.driver.extraJavaOptions | (none) | 驱动器额外的JVM选项。如：GC设置或其他日志参数。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-java-options）或者conf/spark-defaults.conf 配置。
|spark.driver.extraLibraryPath | (none) | 启动驱动器JVM时候指定的依赖库路径。注意：驱动器如果运行客户端模式下，这个配置不能通过SparkConf在程序里配置，因为这时候程序已经启动呀！而是应该用命令行参数（–driver-library-path）或者conf/spark-defaults.conf 配置。
|spark.driver.userClassPathFirst | false | (试验性的：即未来不一定会支持该配置) 驱动器是否首选使用用户指定的jars，而不是spark自身的。这个特性可以用来处理用户依赖和spark本身依赖项之间的冲突。目前还是试验性的，并且只能用在集群模式下。
|spark.executor.extraClassPath | (none) | 添加到执行器（executor）classpath开头的classpath。主要为了向后兼容老的spark版本，不推荐使用。
|spark.executor.extraJavaOptions | (none) | 传给执行器的额外JVM参数。如：GC设置或其他日志设置等。注意，不能用这个来设置Spark属性或者堆内存大小。Spark属性应该用SparkConf对象，或者spark-defaults.conf文件（会在spark-submit脚本中使用）来配置。执行器堆内存大小应该用 spark.executor.memory配置。
|spark.executor.extraLibraryPath | (none) | 启动执行器JVM时使用的额外依赖库路径。
|spark.executor.logs.rolling.maxRetainedFiles | (none) | Sets the number of latest rolling log files that are going to be retained by the system. Older log files will be deleted. Disabled by default.设置日志文件最大保留个数。老日志文件将被干掉。默认禁用的。
|spark.executor.logs.rolling.maxSize | (none) | 设置执行器日志文件大小上限。默认禁用的。需要自动删日志请参考 spark.executor.logs.rolling.maxRetainedFiles.
|spark.executor.logs.rolling.strategy | (none) | 执行器日志滚动策略。默认禁用。可接受的值有”time”（基于时间滚动） 或者 “size”（基于文件大小滚动）。time：将使用 spark.executor.logs.rolling.time.interval设置滚动时间间隔size：将使用 spark.executor.logs.rolling.size.maxBytes设置文件大小上限
|spark.executor.logs.rolling.time.interval | daily | 设置执行器日志滚动时间间隔。日志滚动默认是禁用的。可用的值有 “daily”, “hourly”, “minutely”，也可设为数字（则单位为秒）。关于日志自动清理，请参考 spark.executor.logs.rolling.maxRetainedFiles
|spark.executor.userClassPathFirst | false | （试验性的）与 spark.driver.userClassPathFirst类似，只不过这个参数将应用于执行器
|spark.executorEnv.[EnvironmentVariableName] | (none) | 向执行器进程增加名为EnvironmentVariableName的环境变量。用户可以指定多个来设置不同的环境变量。
|spark.python.profile | false | 对Python worker启用性能分析，性能分析结果会在sc.show_profile()中，或者在驱动器退出前展示。也可以用sc.dump_profiles(path)输出到磁盘上。如果部分分析结果被手动展示过，那么驱动器退出前就不再自动展示了。默认会使用pyspark.profiler.BasicProfiler，也可以自己传一个profiler 类参数给SparkContext构造函数。
|spark.python.profile.dump | (none) | 这个目录是用来在驱动器退出前，dump性能分析结果。性能分析结果会按RDD分别dump。同时可以使用ptats.Stats()来装载。如果制定了这个，那么分析结果就不再自动展示。
|spark.python.worker.memory | 512m | 聚合时每个python worker使用的内存总量，和JVM的内存字符串格式相同（如，512m，2g）。如果聚合时使用的内存超过这个量，就将数据溢出到磁盘上。
|spark.python.worker.reuse | true | 是否复用Python worker。如果是，则每个任务会启动固定数量的Python worker，并且不需要fork() python进程。如果需要广播的数据量很大，设为true能大大减少广播数据量，因为需要广播的进程数减少了。


### 混洗行为

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.reducer.maxSizeInFlight | 48m | map任务输出同时reduce任务获取的最大内存占用量。每个输出需要创建buffer来接收，对于每个reduce任务来说，有一个固定的内存开销上限，所以最好别设太大，除非你内存非常大。
|spark.shuffle.compress | true | 是否压缩map任务的输出文件。通常来说，压缩是个好主意。使用的压缩算法取决于 spark.io.compression.codec
|spark.shuffle.file.buffer | 32k | 每个混洗输出流的内存buffer大小。这个buffer能减少混洗文件的创建和磁盘寻址。
|spark.shuffle.io.maxRetries | 3 | （仅对netty）如果IO相关异常发生，重试次数（如果设为非0的话）。重试能是大量数据的混洗操作更加稳定，因为重试可以有效应对长GC暂停或者网络闪断。
|spark.shuffle.io.numConnectionsPerPeer | 1 | （仅netty）主机之间的连接是复用的，这样可以减少大集群中重复建立连接的次数。然而，有些集群是机器少，磁盘多，这种集群可以考虑增加这个参数值，以便充分利用所有磁盘并发性能。
|spark.shuffle.io.preferDirectBufs | true | （仅netty）堆外缓存可以有效减少垃圾回收和缓存复制。对于堆外内存紧张的用户来说，可以考虑禁用这个选项，以迫使netty所有内存都分配在堆上。
|spark.shuffle.io.retryWait | 5s | （仅netty）混洗重试获取数据的间隔时间。默认最大重试延迟是15秒，设置这个参数后，将变成maxRetries* retryWait。
|spark.shuffle.manager | sort | 混洗数据的实现方式。可用的有”sort”和”hash“。基于排序（sort）的混洗内存利用率更高，并且从1.2开始已经是默认值了。
|spark.shuffle.service.enabled | false | 启用外部混洗服务。启用外部混洗服务后，执行器生成的混洗中间文件就由该服务保留，这样执行器就可以安全的退出了。如果 spark.dynamicAllocation.enabled启用了，那么这个参数也必须启用，这样动态分配才能有外部混洗服务可用。
|spark.shuffle.service.port | 7337 | 外部混洗服务对应端口
|spark.shuffle.sort.bypassMergeThreshold | 200 | （高级）在基于排序（sort）的混洗管理器中，如果没有map端聚合的话，就会最多存在这么多个reduce分区。
|spark.shuffle.spill.compress | true | 是否在混洗阶段压缩溢出到磁盘的数据。压缩算法取决于spark.io.compression.codec


### Spark UI

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.eventLog.compress | false | 是否压缩事件日志（当然spark.eventLog.enabled必须开启）
|spark.eventLog.dir | file:///tmp/spark-events | Spark events日志的基础目录（当然spark.eventLog.enabled必须开启）。在这个目录中，spark会给每个应用创建一个单独的子目录，然后把应用的events log打到子目录里。用户可以设置一个统一的位置（比如一个HDFS目录），这样history server就可以从这里读取历史文件。
|spark.eventLog.enabled | false | 是否启用Spark事件日志。如果Spark应用结束后，仍需要在SparkUI上查看其状态，必须启用这个。
|spark.ui.killEnabled | true | 允许从SparkUI上杀掉stage以及对应的作业（job）
|spark.ui.port | 4040 | SparkUI端口，展示应用程序运行状态。
|spark.ui.retainedJobs | 1000 | SparkUI和status API最多保留多少个spark作业的数据（当然是在垃圾回收之前）
|spark.ui.retainedStages | 1000 | SparkUI和status API最多保留多少个spark步骤（stage）的数据（当然是在垃圾回收之前）
|spark.worker.ui.retainedExecutors | 1000 | SparkUI和status API最多保留多少个已结束的执行器（executor）的数据（当然是在垃圾回收之前）
|spark.worker.ui.retainedDrivers | 1000 | SparkUI和status API最多保留多少个已结束的驱动器（driver）的数据（当然是在垃圾回收之前）
|spark.sql.ui.retainedExecutions | 1000 | SparkUI和status API最多保留多少个已结束的执行计划（execution）的数据（当然是在垃圾回收之前）
|spark.streaming.ui.retainedBatches | 1000 | SparkUI和status API最多保留多少个已结束的批量（batch）的数据（当然是在垃圾回收之前）


### 压缩和序列化

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.broadcast.compress | true | 是否在广播变量前使用压缩。通常是个好主意。
|spark.closure.serializer | org.apache.spark.serializer.JavaSerializer | 闭包所使用的序列化类。目前只支持Java序列化。
|spark.io.compression.codec | snappy | 内部数据使用的压缩算法，如：RDD分区、广播变量、混洗输出。Spark提供了3中算法：lz4，lzf，snappy。你也可以使用全名来指定压缩算法：org.apache.spark.io.LZ4CompressionCodec,org.apache.spark.io.LZFCompressionCodec,org.apache.spark.io.SnappyCompressionCodec.
|spark.io.compression.lz4.blockSize | 32k | LZ4算法使用的块大小。当然你需要先使用LZ4压缩。减少块大小可以减少混洗时LZ4算法占用的内存量。
|spark.io.compression.snappy.blockSize | 32k | Snappy算法使用的块大小（先得使用Snappy算法）。减少块大小可以减少混洗时Snappy算法占用的内存量。
|spark.kryo.classesToRegister | (none) | 如果你使用Kryo序列化，最好指定这个以提高性能（tuning guide）。本参数接受一个逗号分隔的类名列表，这些类都会注册为Kryo可序列化类型。
|spark.kryo.referenceTracking | true (false when using Spark SQL Thrift Server) | 是否跟踪同一对象在Kryo序列化的引用。如果你的对象图中有循环护着包含统一对象的多份拷贝，那么最好启用这个。如果没有这种情况，那就禁用以提高性能。
|spark.kryo.registrationRequired | false | Kryo序列化时，是否必须事先注册。如果设为true，那么Kryo遇到没有注册过的类型，就会抛异常。如果设为false（默认）Kryo会序列化未注册类型的对象，但会有比较明显的性能影响，所以启用这个选项，可以强制必须在序列化前，注册可序列化类型。
|spark.kryo.registrator | (none) | 如果你使用Kryo序列化，用这个class来注册你的自定义类型。如果你需要自定义注册方式，这个参数很有用。否则，使用 spark.kryo.classesRegister更简单。要设置这个参数，需要用KryoRegistrator的子类。详见：tuning guide 。
|spark.kryoserializer.buffer.max | 64m | 最大允许的Kryo序列化buffer。必须必你所需要序列化的对象要大。如果你在Kryo中看到”buffer limit exceeded”这个异常，你就得增加这个值了。
|spark.kryoserializer.buffer | 64k | Kryo序列化的初始buffer大小。注意，每台worker上对应每个core会有一个buffer。buffer最大增长到 spark.kryoserializer.buffer.max
|spark.rdd.compress | false | 是否压缩序列化后RDD的分区（如：StorageLevel.MEMORY_ONLY_SER）。能节省大量空间，但多消耗一些CPU。
|spark.serializer | org.apache.spark.serializer.JavaSerializer (org.apache.spark.serializer.KryoSerializer when using Spark SQL Thrift Server) | 用于序列化对象的类，序列化后的数据将通过网络传输，或从缓存中反序列化回来。默认的Java序列化使用java的Serializable接口，但速度较慢，所以我们建议使用usingorg.apache.spark.serializer.KryoSerializer and configuring Kryo serialization如果速度需要保证的话。当然你可以自定义一个序列化器，通过继承org.apache.spark.Serializer.
|spark.serializer.objectStreamReset | 100 | 如果使用org.apache.spark.serializer.JavaSerializer做序列化器，序列化器缓存这些对象，以避免输出多余数据，然而，这个会打断垃圾回收。通过调用reset来flush序列化器，从而使老对象被回收。要禁用这一周期性reset，需要把这个参数设为-1，。默认情况下，序列化器会每过100个对象，被reset一次。

### 内存管理

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.memory.fraction | 0.75 | 堆内存中用于执行、混洗和存储（缓存）的比例。这个值越低，则执行中溢出到磁盘越频繁，同时缓存被逐出内存也更频繁。这个配置的目的，是为了留出用户自定义数据结构、内部元数据使用的内存。推荐使用默认值。请参考this description.
|spark.memory.storageFraction | 0.5 | 不会被逐出内存的总量，表示一个相对于 spark.memory.fraction的比例。这个越高，那么执行混洗等操作用的内存就越少，从而溢出磁盘就越频繁。推荐使用默认值。更详细请参考 this description.
|spark.memory.offHeap.enabled | true | 如果true，Spark会尝试使用堆外内存。启用 后，spark.memory.offHeap.size必须为正数。
|spark.memory.offHeap.size | 0 | 堆外内存分配的大小（绝对值）。这个设置不会影响堆内存的使用，所以你的执行器总内存必须适应JVM的堆内存大小。必须要设为正数。并且前提是 spark.memory.offHeap.enabled=true.
|spark.memory.useLegacyMode | false | 是否使用老式的内存管理模式（1.5以及之前）。老模式在堆内存管理上更死板，使用固定划分的区域做不同功能，潜在的会导致过多的数据溢出到磁盘（如果不小心调整性能）。必须启用本参数，以下选项才可用：spark.shuffle.memoryFractionspark.storage.memoryFractionspark.storage.unrollFraction
|spark.shuffle.memoryFraction | 0.2 | （废弃）必须先启用spark.memory.useLegacyMode这个才有用。混洗阶段用于聚合和协同分组的JVM堆内存比例。在任何指定的时间，所有用于混洗的内存总和不会超过这个上限，超出的部分会溢出到磁盘上。如果溢出台频繁，考虑增加spark.storage.memoryFraction的大小。
|spark.storage.memoryFraction | 0.6 | （废弃）必须先启用spark.memory.useLegacyMode这个才有用。Spark用于缓存数据的对内存比例。这个值不应该比JVM 老生代（old generation）对象所占用的内存大，默认是60%的堆内存，当然你可以增加这个值，同时配置你所用的老生代对象占用内存大小。
|spark.storage.unrollFraction | 0.2 | （废弃）必须先启用spark.memory.useLegacyMode这个才有用。Spark块展开的内存占用比例。如果没有足够的内存来完整展开新的块，那么老的块将被抛弃。


### 执行行为

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.broadcast.blockSize | 4m | TorrentBroadcastFactory每个分片大小。太大会减少广播时候的并发数（更慢了）；如果太小，BlockManager可能会给出性能提示。spark.broadcast.factory | org.apache.spark.broadcast.TorrentBroadcastFactory | 广播算法的实现。spark.cleaner.ttl | (infinite) | Spark记住任意元数据的保留时间（秒）。周期性的清理能保证比这个更老的元数据将被遗忘（删除）。这对于长期运行的Spark作业非常有用（如，一些7乘24运行）。注意，RDD持久化到内存中后，过了这么长时间以后，也会被清理掉（这。。。是不是有点坑！）。
|spark.executor.cores | YARN模式下默认1；如果是独立部署，则是worker节点上所有可用的core。 | 单个执行器可用的core数。仅针对YARN和独立部署模式。独立部署时，单个worker节点上会运行多个执行器（executor），只要worker上有足够的core。否则，每个应用在单个worker上只会启动一个执行器。
|spark.default.parallelism | 对于reduceByKey和join这样的分布式混洗（shuffle）算子，等于父RDD中最大的分区。对于parallelize这样没有父RDD的算子，则取决于集群管理器：Local mode: number of cores on the local machine — 本地模式：机器的core数Mesos fine grained mode: 8 — Mesos细粒度模式：8Others: total number of cores on all executor nodes or 2, whichever is larger — 其他：所有执行器节点上core的数量 或者 2，这两数取较大的如果用户没有在参数里指定，这个属性是默认的RDD transformation算子分区数，如：join，reduceByKey，parallelize等。
|spark.executor.heartbeatInterval | 10s | 执行器心跳间隔（报告心跳给驱动器）。心跳机制使驱动器了解哪些执行器还活着，并且可以从心跳数据中获得执行器的度量数据。
|spark.files.fetchTimeout | 60s | 获取文件的通讯超时，所获取的文件是通过在驱动器上调用SparkContext.addFile()添加的。
|spark.files.useFetchCache | true | 如果设为true（默认），则同一个spark应用的不同执行器之间，会使用一二共享缓存来拉取文件，这样可以提升同一主机上运行多个执行器时候，任务启动的性能。如果设为false，这个优化就被禁用，各个执行器将使用自己独有的缓存，他们拉取的文件也是各自有一份拷贝。如果在NFS文件系统上使用本地文件系统，可以禁用掉这个优化（参考SPARK-6313）
|spark.files.overwrite | false | SparkContext.addFile()添加的文件已经存在，且内容不匹配的情况下，是否覆盖。
|spark.hadoop.cloneConf | false | 如设为true，对每个任务复制一份Hadoop Configuration对象。启用这个可以绕过Configuration线程安全问题（SPARK-2546 ）。默认这个是禁用的，很多job并不会受这个issue的影响。
|spark.hadoop.validateOutputSpecs | true | 如设为true，在saveAsHadoopFile及其变体的时候，将会验证输出（例如，检查输出目录是否存在）。对于已经验证过或确认存在输出目录的情况，可以禁用这个。我们建议不要禁用，除非你确定需要和之前的spark版本兼容。可以简单的利用Hadoop 文件系统API手动删掉已存在的输出目录。这个设置会被Spark Streaming StreamingContext生成的job忽略，因为Streaming需要在回复检查点的时候，覆盖已有的输出目录。
|spark.storage.memoryMapThreshold | 2m | spark从磁盘上读取一个块后，映射到内存块的最小大小。这阻止了spark映射过小的内存块。通常，内存映射块是有开销的，应该比接近或小于操作系统的页大小。
|spark.externalBlockStore.blockManager | org.apache.spark.storage.TachyonBlockManager | 用于存储RDD的外部块管理器（文件系统）的实现。文件系统URL由spark.externalBlockStore.url决定。
|spark.externalBlockStore.baseDir | System.getProperty(“java.io.tmpdir”) | 外部块存储存放RDD的目录。文件系统URL由spark.externalBlockStore.url决定。也可以是逗号分隔的目录列表（Tachyon文件系统）
|spark.externalBlockStore.url | tachyon://localhost:19998 for Tachyon | 所使用的外部块存储文件系统URL。

### 网络

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.akka.frameSize | 128 | "control plane" 通讯中所允许的最大消息大小（MB）。通常，只应用于map输出数据的大小信息，这些信息会在执行器和驱动器之间传递。如果你的job包含几千个map和reduce任务，你可能需要增大这个设置。
|spark.akka.heartbeat.interval | 1000s | 设这么大的值，是为了禁用Akka传输失败检测器。也可以重新启用，如果你想用这个特性（但不建议）。设成较大的值可以减少网络开销，而较小的值（1秒左右）可能会对Akka的失败检测更有用。如有需要，可以调整这个值和spark.akka.heartbeat.pauses的组合。一种可能需要使用失败检测的情形是：用一个敏感的失败检测，可以快速识别并逐出不稳定的执行器。然而，在真实的spark集群中，这通常不是GC暂停或网络延迟造成的。除此之外，启用这个还会导致过多的心跳数据交换，从而造成网络洪峰。
|spark.akka.heartbeat.pauses | 6000s | 设这么大的值，是为了禁用Akka传输失败检测器。也可以重新启用，如果你想用这个特性（但不建议）。这个是可接受的Akka心跳暂停时间。这个可以用来控制对GC暂停敏感程度。如有需要，可以调整这个值和spark.akka.heartbeat.interval的组合。
|spark.akka.threads | 4 | 用于通讯的actor线程数。如果驱动器机器上有很多CPU core，你可以适当增大这个值。
|spark.akka.timeout | 100s | Spark节点之间通讯超时。
|spark.blockManager.port | (random) | 块管理器（block manager）监听端口。在驱动器和执行器上都有。
|spark.broadcast.port | (random) | 驱动器HTTP广播server监听端口。这和torrent广播没有关系。
|spark.driver.host | (local hostname) | 驱动器主机名。用于和执行器以及独立部署时集群master通讯。
|spark.driver.port | (random) | 驱动器端口。用于和执行器以及独立部署时集群master通讯。
|spark.executor.port | (random) | 执行器端口。用于和驱动器通讯。
|spark.fileserver.port | (random) | 驱动器HTTP文件server监听端口。
|spark.network.timeout | 120s | 所有网络交互的默认超时。这个配置是以下属性的默认值：
|spark.core.connection.ack.wait.timeout,
|spark.akka.timeout,
|spark.storage.blockManagerSlaveTimeoutMs,
|spark.shuffle.io.connectionTimeout,spark.rpc.askTimeout or
|spark.rpc.lookupTimeout
|spark.port.maxRetries | 16 | 绑定一个端口的最大重试次数。如果指定了一个端口（非0），每个后续重试会在之前尝试的端口基础上加1，然后再重试绑定。本质上，这确定了一个绑定端口的范围，就是 [start port, start port + maxRetries]
|spark.replClassServer.port | (random) | 驱动器HTTP class server的监听端口。只和spark shell相关。
|spark.rpc.numRetries | 3 | RPC任务最大重试次数。RPC任务最多重试这么多次。
|spark.rpc.retry.wait | 3s | RPC请求操作重试前等待时间。
|spark.rpc.askTimeout | 120s | RPC请求操作超时等待时间。
|spark.rpc.lookupTimeout | 120s | RPC远程端点查询超时。

### 调度

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.cores.max | (not set) | 如果运行在独立部署集群模式（standalone deploy cluster）或者Mesos集群粗粒度共享模式（Mesos cluster in “coarse-grained” sharing mode），这个值决定了spark应用可以使用的最大CPU总数（应用在整个集群中可用CPU总数，而不是单个机器）。如果不设置，那么独立部署时默认为spark.deploy.defaultCores，Mesos集群则默认无限制（即所有可用的CPU）。
|spark.locality.wait | 3s | 为了数据本地性最长等待时间（spark会根据数据所在位置，尽量让任务也启动于相同的节点，然而可能因为该节点上资源不足等原因，无法满足这个任务分配，spark最多等待这么多时间，然后放弃数据本地性）。数据本地性有多个级别，每一级别都是等待这么多时间（同一进程、同一节点、同一机架、任意）。你也可以为每个级别定义不同的等待时间，需要设置spark.locality.wait.node等。如果你发现任务数据本地性不佳，可以增加这个值，但通常默认值是ok的。
|spark.locality.wait.node | spark.locality.wait | 单独定义同一节点数据本地性任务等待时间。你可以设为0，表示忽略节点本地性，直接跳到下一级别，即机架本地性（如果你的集群有机架信息）。
|spark.locality.wait.process | spark.locality.wait | 单独定义同一进程数据本地性任务等待时间。这个参数影响试图访问特定执行器上的缓存数据的任务。
|spark.locality.wait.rack | spark.locality.wait | 单独定义同一机架数据本地性等待时间。
|spark.scheduler.maxRegisteredResourcesWaitingTime | 30s | 调度开始前，向集群管理器注册使用资源的最大等待时间。
|spark.scheduler.minRegisteredResourcesRatio | 0.8 for YARN mode；0.0 for standalone mode and Mesos coarse-grained mode | 调度启动前，需要注册得到资源的最小比例（注册到的资源数 / 需要资源总数）（YARN模式下，资源是执行器；独立部署和Mesos粗粒度模式下时资源是CPU core【spark.cores.max是期望得到的资源总数】）。可以设为0.0~1.0的一个浮点数。不管job是否得到了最小资源比例，最大等待时间都是由spark.scheduler.maxRegisteredResourcesWaitingTime控制的。
|spark.scheduler.mode | FIFO | 提交到同一个SparkContext上job的调度模式（scheduling mode）。另一个可接受的值是FAIR，而FIFO只是简单的把job按先来后到排队。对于多用户服务很有用。
|spark.scheduler.revive.interval | 1s | 调度器复活worker的间隔时间。
|spark.speculation | false | 如果设为true，将会启动推测执行任务。这意味着，如果stage中有任务执行较慢，他们会被重新调度到别的节点上执行。
|spark.speculation.interval | 100ms | Spark检查慢任务的时间间隔。
|spark.speculation.multiplier | 1.5 | 比任务平均执行时间慢多少倍的任务会被认为是慢任务。
|spark.speculation.quantile | 0.75 | 对于一个stage来说，完成多少百分比才开始检查慢任务，并启动推测执行任务。
|spark.task.cpus | 1 | 每个任务分配的CPU core。
|spark.task.maxFailures | 4 | 单个任务最大失败次数。应该>=1。最大重试次数 = spark.task.maxFailures – 1

### 动态分配

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.dynamicAllocation.enabled | false | 是否启用动态资源分配特性，启用后，执行器的个数会根据工作负载动态的调整（增加或减少）。注意，目前在YARN模式下不用。更详细信息，请参考：here该特性依赖于 spark.shuffle.service.enabled 的启用。同时还和以下配置相关：spark.dynamicAllocation.minExecutors, spark.dynamicAllocation.maxExecutors以及 spark.dynamicAllocation.initialExecutors
|spark.dynamicAllocation.executorIdleTimeout | 60s | 动态分配特性启用后，空闲时间超过该配置时间的执行器都会被移除。更详细请参考这里：description
|spark.dynamicAllocation.cachedExecutorIdleTimeout | infinity | 动态分配特性启用后，包含缓存数据的执行器如果空闲时间超过该配置设置的时间，则被移除。更详细请参考：description
|spark.dynamicAllocation.initialExecutors | spark.dynamicAllocation.minExecutors | 动态分配开启后，执行器的初始个数
|spark.dynamicAllocation.maxExecutors | infinity | 动态分配开启后，执行器个数的上限
|spark.dynamicAllocation.minExecutors | 0 | 动态分配开启后，执行器个数的下限
|spark.dynamicAllocation.schedulerBacklogTimeout | 1s | 动态分配启用后，如果有任务积压的持续时间长于该配置设置的时间，则申请新的执行器。更详细请参考：description
|spark.dynamicAllocation.sustainedSchedulerBacklogTimeout | schedulerBacklogTimeout | 和spark.dynamicAllocation.schedulerBacklogTimeout类似，只不过该配置对应于随后持续的执行器申请。更详细请参考： description

### 安全

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.acls.enable | false | 是否启用Spark acls（访问控制列表）。如果启用，那么将会检查用户是否有权限查看或修改某个作业（job）。注意，检查的前提是需要知道用户是谁，所以如果用户是null，则不会做任何检查。你可以在Spark UI上设置过滤器（Filters）来做用户认证，并设置用户名。
|spark.admin.acls | Empty | 逗号分隔的用户列表，在该列表中的用户/管理员将能够访问和修改所有的Spark作业（job）。如果你的集群是共享的，并且有集群管理员，还有需要调试的开发人员，那么这个配置会很有用。如果想让所有人都有管理员权限，只需把该配置设置为*
|spark.authenticate | false | 设置Spark是否认证集群内部连接。如果不是在YARN上运行，请参考 spark.authenticate.secret
|spark.authenticate.secret | None | 设置Spark用于内部组件认证的秘钥。如果不是在YARN上运行，且启用了 spark.authenticate，那么该配置必须设置
|spark.authenticate.enableSaslEncryption | false | 是否对Spark内部组件认证使用加密通信。该配置目前只有 block transfer service 使用。
|spark.network.sasl.serverAlwaysEncrypt | false | 是否对支持SASL认证的service禁用非加密通信。该配置目前只有 external shuffle service 支持。
|spark.core.connection.ack.wait.timeout | 60s | 网络连接等待应答信号的超时时间。为了避免由于GC等导致的意外超时，你可以设置一个较大的值。
|spark.core.connection.auth.wait.timeout | 30s | 网络连接等待认证的超时时间。
|spark.modify.acls | Empty | 逗号分隔的用户列表，在改列表中的用户可以修改Spark作业。默认情况下，只有启动该Spark作业的用户可以修改之（比如杀死该作业）。如果想要任何用户都可以修改作业，请将该配置设置为*
|spark.ui.filters | None | 逗号分隔的过滤器class列表，这些过滤器将用于Spark web UI。这里的过滤器应该是一个标准的 javax servlet Filter 。每个过滤器的参数可以通过java系统属性来设置，如下：
|spark.<class name of filer>.params=’param1=value1,param2=value2’例如：-Dspark.ui.filters=com.test.filter1 -Dspark.com.test.filter1.params=’param1=foo,param2=testing’
|spark.ui.view.acls | Empty | 逗号分隔的用户列表，在该列表中的用户可以查看Spark web UI。默认，只有启动该Spark作业的用户可以查看之。如果需要让所有用户都能查看，只需将该配置设为"*"
  
### 加密

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.ssl.enabled | false | 是否启用SSL连接（在所有所支持的协议上）。所有SSL相关配置（spark.ssl.xxx，其中xxx是一个特定的配置属性），都是全局的。如果需要在某些协议上覆盖全局设置，那么需要在该协议命名空间上进行单独配置。使用 spark.ssl.YYY.XXX 来为协议YYY覆盖全局配置XXX。目前YYY的可选值有 akka（用于基于AKKA框架的网络连接） 和 fs（用于应广播和文件服务器）
|spark.ssl.enabledAlgorithms | Empty | 逗号分隔的加密算法列表。这些加密算法必须是JVM所支持的。这里有个可用加密算法参考列表： this
|spark.ssl.keyPassword | None | 在key-store中私匙对应的密码。
|spark.ssl.keyStore | None | key-store文件路径。可以是绝对路径，或者以本组件启动的工作目录为基础的相对路径。
|spark.ssl.keyStorePassword | None | key-store的密码。
|spark.ssl.protocol | None | 协议名称。该协议必须是JVM所支持的。这里有JVM支持的协议参考列表：this
|spark.ssl.trustStore | None | trust-store文件路径。可以是绝对路径，或者以本组件启动的工作目录为基础的相对路径。
|spark.ssl.trustStorePassword | None | trust-store的密码

### Spark Streaming [流式]

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.streaming.backpressure.enabled | false | 是否启用Spark Streaming 的内部反压机制（spark 1.5以上支持）。启用后，Spark Streaming会根据当前批次的调度延迟和处理时长来控制接收速率，这样一来，系统的接收速度会和处理速度相匹配。该特性会在内部动态地设置接收速率。该速率的上限将由 spark.streaming.receiver.maxRate 和 spark.streaming.kafka.maxRatePerPartition 决定（如果它们设置了的话）。
|spark.streaming.blockInterval | 200ms | 在将数据保存到Spark之前，Spark Streaming接收器组装数据块的时间间隔。建议不少于50ms。关于Spark Streaming编程指南细节，请参考 performance tuning 这一节。
|spark.streaming.receiver.maxRate | not set | 接收速度的最大速率（每秒记录条数）。实际上，每个流每秒将消费这么多条记录。设置为0或者负数表示不限制速率。更多细节请参考： deployment guide
|spark.streaming.receiver.writeAheadLog.enable | false | 是否启用接收器预写日志。所有的输入数据都会保存到预写日志中，这样在驱动器失败后，可以基于预写日志来恢复数据。更详细请参考：deployment guide
|spark.streaming.unpersist | true | 是否强制Spark Streaming 自动从内存中清理掉所生成并持久化的RDD。同时，Spark Streaming收到的原始数据也将会被自动清理掉。如果设置为false，那么原始数据以及持久化的RDD将不会被自动清理，以便外部程序可以访问这些数据。当然，这将导致Spark消耗更多的内存。
|spark.streaming.stopGracefullyOnShutdown | false | 如果设为true，Spark将会在JVM关闭时，优雅地关停StreamingContext，而不是立即关闭之。
|spark.streaming.kafka.maxRatePerPartition | not set | 在使用Kafka direct stream API时，从每个Kafka数据分区读取数据的最大速率（每秒记录条数）。更详细请参考：Kafka Integration guide
|spark.streaming.kafka.maxRetries | 1 | 驱动器连续重试的最大次数，这个配置是为了让驱动器找出每个Kafka分区上的最大offset（默认值为1，意味着驱动器将最多尝试2次）。只对新的Kafka direct stream API有效。
|spark.streaming.ui.retainedBatches | 1000 | Spark Streaming UI 以及 status API 中保留的最大批次个数

### SparkR

| 属性名称 | 默认值 | 含义 
| - | - | - |
|spark.r.numRBackendThreads | 2 | SparkR RBackEnd处理RPC调用的后台线程数
|spark.r.command | Rscript | 集群模式下，驱动器和worker上执行的R脚本可执行文件
|spark.r.driver.command | spark.r.command | client模式的驱动器执行的R脚本。集群模式下会忽略















