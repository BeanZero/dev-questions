#  开发人员遇到的问题与建议



本文是根据笔者多年观察一线程序员在实际工作中遇到的问题结合自身的经验提出的一些建议。

类似编码规范、版本控制等属于部门岗位相关要求，本文中不会涉及。

文中内容主要是开发人员在平时工作中常常疏漏的地方与建议。

目前[**文档**](https://github.com/BeanZero/dev-questions)内容仍在不断完善中。



* [开发人员遇到的问题与建议](#%E5%BC%80%E5%8F%91%E4%BA%BA%E5%91%98%E9%81%87%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98%E4%B8%8E%E5%BB%BA%E8%AE%AE)
  * [开发篇](#%E5%BC%80%E5%8F%91%E7%AF%87)
    * [池化技术](#%E6%B1%A0%E5%8C%96%E6%8A%80%E6%9C%AF)
      * [线程池](#%E7%BA%BF%E7%A8%8B%E6%B1%A0)
      * [连接池](#%E8%BF%9E%E6%8E%A5%E6%B1%A0)
      * [对象池](#%E5%AF%B9%E8%B1%A1%E6%B1%A0)
    * [IO](#io)
      * [常见问题](#%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)
      * [阻塞与非阻塞](#%E9%98%BB%E5%A1%9E%E4%B8%8E%E9%9D%9E%E9%98%BB%E5%A1%9E)
      * [同步与异步](#%E5%90%8C%E6%AD%A5%E4%B8%8E%E5%BC%82%E6%AD%A5)
      * [BIO/NIO/AIO](#bionioaio)
      * [Netty](#netty)
    * [线程安全](#%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8)
      * [并发问题](#%E5%B9%B6%E5%8F%91%E9%97%AE%E9%A2%98)
      * [ThreadLocal](#threadlocal)
    * [性能优化](#%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
      * [时间复杂度](#%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A6)
      * [空间换时间](#%E7%A9%BA%E9%97%B4%E6%8D%A2%E6%97%B6%E9%97%B4)
      * [查询优化](#%E6%9F%A5%E8%AF%A2%E4%BC%98%E5%8C%96)
      * [JVM优化](#jvm%E4%BC%98%E5%8C%96)
  * [运维篇](#%E8%BF%90%E7%BB%B4%E7%AF%87)
    * [生产环境优化](#%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E4%BC%98%E5%8C%96)
      * [Tomcat优化](#tomcat%E4%BC%98%E5%8C%96)
      * [JVM优化](#jvm%E4%BC%98%E5%8C%96-1)
    * [生产环境监控](#%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E7%9B%91%E6%8E%A7)
      * [保存环境信息(堆栈、CPU、内存、IO等)](#%E4%BF%9D%E5%AD%98%E7%8E%AF%E5%A2%83%E4%BF%A1%E6%81%AF%E5%A0%86%E6%A0%88cpu%E5%86%85%E5%AD%98io%E7%AD%89)
  * [工具篇](#%E5%B7%A5%E5%85%B7%E7%AF%87)
    * [tcpdump抓包](#tcpdump%E6%8A%93%E5%8C%85)
    * [wireshark抓包](#wireshark%E6%8A%93%E5%8C%85)
    * [MAT内存分析工具](#mat%E5%86%85%E5%AD%98%E5%88%86%E6%9E%90%E5%B7%A5%E5%85%B7)



## 开发篇

### 池化技术

#### 线程池

如果你阅读过《阿里巴巴 Java 开发手册》又或者使用其提供的扫描插件就会知道，阿里建议直接创建线程而是通过线程池来管理线程，增加资源利用率。那么——

1. 线程池有哪些常用属性？

   > - corePoolSize：表示核心线程数。
   > - maximumPoolSize：表示最大线程数。
   > - workQueue：工作队列，如果当前线程数超过corePoolSize，那么往该队列中插入任务。
   > - keepAliveTime：空闲时间，如果线程数超出 corePoolSize，并且有些线程的空闲时间超过了这个值，会执行关闭这些线程的操作
   > - rejectedExecutionHandler：拒绝策略，用于处理当线程池不能执行此任务时的情况，默认有抛出 RejectedExecutionException 异常、忽略任务、使用提交任务的线程来执行此任务和将队列中等待最久的任务删除，然后提交此任务这四种策略，默认为抛出异常。

2. 线程池如何创建线程呢？

   > - 如果当前线程数少于 corePoolSize，那么提交任务的时候创建一个新的线程，并由这个线程执行这个任务。
   > - 如果当前线程数已经达到 corePoolSize，那么将提交的任务添加到队列中，等待线程池中的线程去队列中取任务。
   > - 如果队列已满，那么创建新的线程来执行任务，需要保证池中的线程数不会超过 maximumPoolSize，如果此时线程数超过了 maximumPoolSize，那么执行拒绝策略。

3. 尽量重用线程池，而不是每次创建

   ```java
   /**
    * 使用一个静态字段来存放线程池的引用，返回线程池的代码直接返回这个静态字段。
    */
   public class ThreadPoolHelper {
   
       private static final int CORE_THREAD_SIZE = 5;
   
       private static final int MAX_THREAD_SIZE = 10;
   
       private static final String THREAD_NAME = "THREAD_POOL_";
   
       private static ExecutorService service;
   
       static {
           service = new ThreadPoolExecutor(
                   CORE_THREAD_SIZE, MAX_THREAD_SIZE, 0L, TimeUnit.MILLISECONDS,
                   new LinkedBlockingQueue<>(1024),
                   new ThreadFactoryBuilder().setNameFormat(THREAD_NAME + "%d").build(),
                   new ThreadPoolExecutor.AbortPolicy());
       }
   
       public static ExecutorService getThreadPool() {
           return service;
       }
   }
   ```

4. 连接池属性设置多大合适？

   > 需要根据实际的场景、并发情况来评估线程池的核心参数，包括核心线程数、最大线程数、线程回收策略、工作队列的类型，以及拒绝策略，一般而言需要设置有界的工作队列。
   >
   > - 对于执行耗时较长、数量较少的任务，或许要考虑更多的线程数，而不需要太大的队列。
   > - 而对于吞吐量较大的计算型任务，线程数量不宜过多，可以是 CPU 核数或核数 *2（理由是，线程一定调度到某个 CPU 进行执行，如果任务本身是 CPU 绑定的任务，那么过多的线程只会增加线程切换的开销，并不能提升吞吐量），但可能需要较长的队列来做缓冲。
   >
   > <font color=red>`注意：例如LinkedBlockingQueue虽为有界队列，但是如果没有设置大小，缺省为Integer.MAX_VALUE这样就跟无界队列没有区别了。`</font>


#### 连接池

连接池用于对外提供获取连接、归还连接的接口给客户端使用，并暴露**最小空闲连接数**、**最大连接数**等可配置参数，在内部则实现连接建立、连接心跳保持、连接管理、空闲连接回收、连接可用性检测等功能。

常用的连接池有**数据库连接池**、**Redis连接池**、**HTTP连接池**等等

1. Tomcat默认数据库连接池变化

   > Tomcat8之前缺省连接池使用的是DBCP，而在Tomcat8及之后的版本中连接池则换成了DBCP2。二者在参数上有所修改。例如maxActive变成maxTotal，需要特别留意！
   >
   > 
   >
   > Tomcat7.X中的conf/context.xml配置——
   >
   > ```xml
   > <Resource name="xxxxxx" auth="Container"
   >               type="javax.sql.DataSource"
   >               driverClassName="oracle.jdbc.OracleDriver"
   >               url="jdbc:oracle:thin:@xxx.xxx.xx.xxx:1521:orcl"
   >               username="xxxxxxx"
   >               password="xxxxxxx"
   >               maxActive="200"
   >               maxIdle="60"
   >               maxWait="-1"
   >               testOnBorrow="true"
   >               validationQuery="select 1 from dual"/>
   > ```
   >
   > Tomcat8.5.X中的conf/context.xml配置——
   >
   > ```xml
   > <Resource name="xxxxxx" auth="Container"
   >               type="javax.sql.DataSource"
   >               driverClassName="oracle.jdbc.OracleDriver"
   >               url="jdbc:oracle:thin:@xxx.xxx.xx.xxx:1521:orcl"
   >               username="xxxxxxx"
   >               password="xxxxxxx"
   >               initialSize="20"
   >               minIdle="20"
   >               maxIdle="20"
   >               maxTotal="30"
   >               maxWaitMillis="10000"
   >               testWhileIdle="true"
   >               testOnBorrow="true"
   >               validationQuery="select 1 from dual"/>
   > ```
   >
   > 参考资料:[common-dbcp2数据库连接池参数说明](https://www.iteye.com/blog/bsr1983-2092467)

2. 连接池是不是越大越好？

   > 显然连接池并非越大越好。如果设置得太大，不仅仅是客户端需要耗费过多的资源维护连接，更重要的是由于服务端对应的是多个客户端，每一个客户端都保持大量的连接，会给服务端带来更大的压力。这个压力又不仅仅是内存压力，可以想一下如果服务端的网络模型是一个 TCP 连接一个线程，那么几千个连接意味着几千个线程，如此多的线程会造成大量的线程切换开销。

3. 如何确定合适的连接池最大连接数？

   > 建议使用 JMeter, LoadRunner等测试工具，根据生产环境实际的使用情况的峰值(建议峰值*1.25)进行压测，配合JConsole、Jvisualvm监控连接池数与等待线程的变化。在保持正常吞吐量不变的情况下连接数尽量越少越好，其次就是需要监控和报警机制，根据容量规划及时调整参数配置。


#### 对象池

1. 什么是对象池? 为什么要使用对象池？

   > 创建对象的性能开支远比单纯调用方法要高得多。为了最大限度的节省服务器资源提高效率复用对象是常见的解决方案。

2. 对象池有哪些具体实现

   > Apache的common-pool则为我们提供是使用对象池的基础。在很多我们熟悉的JAR包中都是使用对象池来实现，例如：dbcp、dbcp2，jedis等等
   >
   > `注：tomcat默认的dbcp是自身提供的，并非common-dbcp.jar的请留意`

3. 对象池参数实例

   > 1. 实现PooledObjectFactory接口或者继承BasePooledObjectFactory
   >
   >    ```java
   >    /**
   >     * GenericPooledObjectFactory 仍然是一个通用型的PooledObjectFactory，通过泛型抽象
   >     */
   >    public class GenericPooledObjectFactory<T> extends BasePooledObjectFactory<T> {
   >    
   >        private volatile Class clazz;
   >    
   >        public GenericPooledObjectFactory(Class<T> clazz) {
   >            this.clazz = clazz;
   >        }
   >    
   >        @Override
   >        public T create() {
   >            try {
   >                return (T) clazz.newInstance();
   >            } catch (Exception e) {
   >                throw new RuntimeException(e);
   >            }
   >        }
   >    
   >        @Override
   >        public PooledObject<T> wrap(T obj) {
   >            return new DefaultPooledObject<>(obj);
   >        }
   >        
   >    }
   >    ```
   >
   > 2. 例如实现一个RestTemplate的线程池，继承GenericPooledObjectFactory，并重写了create方法
   >
   >    ```java
   >    /**
   >     * 继承GenericPooledObjectFactory，并重写了create方法
   >     */
   >    public class RestTemplatePooledFactory 
   >       extends GenericPooledObjectFactory<RestTemplate> {
   >    
   >        public RestTemplatePooledFactory() {
   >            super(RestTemplate.class);
   >        }
   >    
   >        @Override
   >        public RestTemplate create() {
   >            SimpleClientHttpRequestFactory requestFactory 
   >              = new SimpleClientHttpRequestFactory();
   >            requestFactory.setReadTimeout(60000);
   >            requestFactory.setConnectTimeout(10000);
   >            return new RestTemplate(requestFactory);
   >        }
   >    }
   >    ```
   >
   > 3. 可以创建一个通用对象池工具类方便调用
   >
   >    ```java
   >    /**
   >     * 通用对象池工具类
   >     *
   >     * 获取对象请使用{@link GenericPooledObjectUtils#borrowObject(Class)}
   >     * 归还对象请使用{@link GenericPooledObjectUtils#returnObject(Object)}
   >     *
   >     * 两个方法必须成对使用，用法如下：
   >     *
   >     * try {
   >     *     // ...
   >     *     GenericObjectPoolUtils.borrowObject(obj);
   >     *     // ...
   >     * } finally {
   >     *     // ...
   >     *     GenericObjectPoolUtils.returnObject(obj)
   >     * }
   >     *
   >     */
   >    public class GenericPooledObjectUtils {
   >        private static final Logger logger = Logger.getLogger(GenericPooledObjectUtils.class);
   >        private final static Map<Class, ObjectPool> CACHE = new ConcurrentHashMap<>(32);
   >    
   >    
   >        /**
   >         * 获取对象池
   >         *
   >         * @param clazz 对象类型
   >         * @param <T>   对象类型
   >         * @return 对象池
   >         */
   >        private static <T> ObjectPool<T> getObjectPool(Class<T> clazz) {
   >            if (CACHE.containsKey(clazz)) {
   >                return CACHE.get(clazz);
   >            }
   >            synchronized (clazz) {
   >                if (!CACHE.containsKey(clazz)) {
   >                    GenericObjectPool<T> pool = new GenericObjectPool<T>(getPooledObjectFactory(clazz));
   >                    pool.setConfig(getGenericObjectPoolConfig());
   >                    CACHE.put(clazz, pool);
   >                    return pool;
   >                }
   >                return CACHE.get(clazz);
   >            }
   >    
   >        }
   >    
   >        /**
   >         * 获取对象池配置信息
   >         *
   >         * @return 对象池配置信息
   >         */
   >        private static GenericObjectPoolConfig getGenericObjectPoolConfig() {
   >            GenericObjectPoolConfig config = new GenericObjectPoolConfig();
   >            config.setMinIdle(20);
   >            config.setMinIdle(50);
   >            config.setMaxTotal(200);
   >            config.setMaxWaitMillis(30 * 1000);
   >            config.setTestOnBorrow(true);
   >            config.setTestWhileIdle(true);
   >            config.setTestOnReturn(true);
   >            return config;
   >        }
   >    
   >        /**
   >         * 获取对象工厂
   >         *
   >         * @param clazz 对象类型
   >         * @param <T>   对象类型
   >         * @return 对象工厂
   >         */
   >        private static <T> PooledObjectFactory<T> getPooledObjectFactory(Class<T> clazz) {
   >            if (clazz == null) {
   >                return null;
   >            }
   >            if (clazz == RestTemplate.class) {
   >                return (PooledObjectFactory<T>) new RestTemplatePooledFactory();
   >            } else {
   >                return new GenericPooledObjectFactory<>(clazz);
   >            }
   >        }
   >    
   >        /**
   >         * 获取对象，注意：传入的参数类型必须有无参构造函数¨
   >         *
   >         * @param clazz 对象类型
   >         * @param <T>   对象类型
   >         * @return 对象实体
   >         */
   >        public static <T> T borrowObject(Class<T> clazz) {
   >            // 类型为空，返回空对象
   >            if (clazz == null) {
   >                return null;
   >            }
   >    
   >            // 类型不为空，获取对象
   >            ObjectPool<T> pool = getObjectPool(clazz);
   >            try {
   >                return pool.borrowObject();
   >            } catch (Exception e) {
   >                logger.error(e.getMessage(), e);
   >            }
   >    
   >            return null;
   >        }
   >    
   >        /**
   >         * 归还对象
   >         *
   >         * @param t   要归还的对象
   >         * @param <T> 对象类型
   >         * @return true: 归还成功，false: 归还失败
   >         */
   >        public static <T> boolean returnObject(T t) {
   >            // 对象为空不用归还
   >            if (t == null) {
   >                return true;
   >            }
   >            // 归还对象
   >            try {
   >                // 获取对象池，没有此对象的对象池不用归还
   >                ObjectPool<T> pool = (ObjectPool<T>) getObjectPool(t.getClass());
   >                if (pool == null) {
   >                    return true;
   >                }
   >                pool.returnObject(t);
   >                return true;
   >            } catch (Exception e) {
   >                logger.error(e.getMessage(), e);
   >            }
   >            return false;
   >        }
   >    
   >    }
   >    ```


### IO

#### 常见问题

#### 阻塞与非阻塞

#### 同步与异步

#### BIO/NIO/AIO

#### Netty

### 线程安全

#### 并发问题

#### ThreadLocal

### 性能优化

#### 时间复杂度

#### 空间换时间

#### 查询优化

#### JVM优化





## 运维篇

### 生产环境优化

#### Tomcat优化

1. tomcat服务需要根据生产环境实际的并发数量进行调整，从而到达一个较为合理的状态能够长期稳定的运行，提高单节点的可用性。

   > 针对tomcat自身优化主要是提升**connector连接器**的工作能力。在Tomcat8之前默认使用的是bio的连接方式，而在Tomcat8之后则改成默认为nio连接。
   >
   > 打开文件$TOMCAT_HOME/conf/server.xml，默认Connector配置如下所示——
   >
   > ```xml
   > <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443" />
   > ```
   >
   > 方案一：使用Http11NioProtocol
   >
   > ```xml
   > <Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
   >   maxHttpHeaderSize="8192"
   >   maxThreads="500"
   >   minSpareThreads="50"
   >   maxSpareThreads="200"
   >   acceptCount="200"
   >   URIEncoding="utf-8"
   >   maxPostSize="-1"
   >   connectionTimeout="30000"
   >   redirectPort="443"
   >   compression="on"
   >   compressionMinSize="2048"
   >   noCompressionUserAgents="gozilla, traviata"
   >   compressableMimeType="text/html,text/plain,text/css,application/javascript,application/json,application/x-font-ttf,application/x-font-otf,image/svg+xml,image/jpeg,image/png,image/gif,audio/mpeg,video/mp4"
   >   disableUploadTimeout="true"
   >   enableLookups="false" />
   > ```
   >
   > 方案二：connector绑定线程池
   >
   > ```xml
   > <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
   > 	maxThreads="500" minSpareThreads="50" 
   > 	maxSpareThreads="200" maxIdleTime="60000"
   >     prestartminSpareThreads = "true"
   >     maxQueueSize = "200"/>
   > 
   > <Connector port="8080" executor="tomcatThreadPool" protocol="HTTP/1.1"
   >   maxHttpHeaderSize="8192"
   >   acceptCount="200"
   >   URIEncoding="utf-8"
   >   maxPostSize="-1"
   >   connectionTimeout="30000"
   >   redirectPort="443"
   >   compression="on"
   >   compressionMinSize="2048"
   >   noCompressionUserAgents="gozilla, traviata"
   >   compressableMimeType="text/html,text/plain,text/css,application/javascript,application/json,application/x-font-ttf,application/x-font-otf,image/svg+xml,image/jpeg,image/png,image/gif,audio/mpeg,video/mp4"
   >   disableUploadTimeout="true"
   >   enableLookups="false" />
   > ```
   >
   > <font color=red>`注：具体参数需要根据生产环境实际情况进行调整，以上给出的实例仅用于演示`</font>

#### JVM优化

1. JVM优化参考哪些性能指标？

   > 与上文中提到的tomcat一样也是根据环境自身的配置进行优化，需要特别关注的指标有**CPU的核心数**、**物理内存**、**操作系统及其位数**、**JDK版本**

2. JVM优化具体的参数是哪些？

   > **堆设置——**
   > -Xms:初始堆大小
   > -Xmx:最大堆大小
   > -XX:NewSize=n:设置年轻代大小（可选）
   > -XX:NewRatio=n:设置年轻代和年老代的比值（可选）
   > -XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值
   > -XX:MaxPermSize=n:设置持久代大小（JDK1.8之前）
   > -XX:MaxMetaspaceSize=n:设置元数据区大小（JDK1.8）
   >
   > **垃圾回收统计信息——**
   > -XX:+PrintGC
   > -XX:+PrintGCDetails
   > -XX:+PrintGCTimeStamps
   > -Xloggc:filename
   >
   > **Serial收集器——**
   > -XX:+UseSerialGC:设置串行收集器
   >
   > **Parallel收集器——**
   > -XX:+UseParallelGC:设置并行收集器
   >
   > **CMS收集器——**
   > -XX:+UseParNewGC:设置CMS收集器
   > -XX:+UseConcMarkSweepGC:设置并发收集器
   >
   > **G1收集器——**
   > –XX:+UseG1GC:设置G1垃圾收集器
   >
   > 
   >
   > **以下示例仅供参考：**
   >
   > ```shell
   > -server
   > -Xmx4096M
   > -Xms4096M
   > -XX:MaxMetaspaceSize=512M
   > -XX:MetaspaceSize=512M
   > -XX:+UseG1GC
   > -XX:MaxGCPauseMillis=100
   > -XX:+ParallelRefProcEnabled
   > -XX:ErrorFile=$file_path/hs_err_pid_%p.log
   > -Xloggc:$file_path/gc_pid_%p.log
   > -XX:HeapDumpPath=$file_path/HeapDump
   > -XX:+PrintGCDetails
   > -XX:+PrintGCDateStamps
   > -XX:+HeapDumpOnOutOfMemoryError
   > ```

### 生产环境监控

#### 保存环境信息(堆栈、CPU、内存、IO等)

> 生产环境会遇到一些**内存徒增**，**CPU使用率过高**，**I/O阻塞**甚至**系统不可用**等状态，而这些问题难以从业务日志中定位问题。但是生产环境保证可用性，需要立刻重启已到达回复正常使用。则我们需要<font color=blue>**在重启生产环境之前保存实时信息，以方便后续生产故障的排查与解决。**</font>

以下分别针对Linux与window提供保存JVM环境的脚本。

1. for Linux

   ```shell
   #!/bin/bash
   
   # ***************************
   # Author	: BeanZero
   # Email		: dawnoftan@163.com
   # Date		: 2020-03-14
   # 
   # 根据端口号获取进程ID，然后通过进程ID保存JVM运行时环境数据（GC信息，栈日志，堆dump）
   # 会在当前路径下面新建名为jvm_{datetime}_port_{port}_pid_{pid}的目录
   #
   # ***************************
   
   read -p "Please enter the port : " port
   
   pid=`lsof -i:\$port | awk NR==2{print} | awk '{print \$2}'`
   
   echo pid=$pid, port=$port
   date_time=`date +%s`
   
   # 创建文件夹
   file_path="jvm_"$date_time"_port_"$port"_pid_"$pid
   mkdir -p $file_path
   
   # 线程资源占用(考虑CPU切换 保存3次 -n3)
   top -Hp $pid -b -n3 >> $file_path/top.log
   
   # GC信息
   jstat -gc $pid >> $file_path/gc.log
   
   # 栈日志
   jstack $pid >> $file_path/stack.log
   
   # 堆现场日志(live)
   jmap -dump:live,format=b,file=$file_path/heap.hprof $pid
   
   ```

   

2. for window

   ```powershell
   @echo off
   Setlocal Enabledelayedexpansion
   
   
   REM 说明：通过端口号查询pid，然后保存堆栈日志(用于window版本)
   REM 作者：BeanZero
   REM 邮箱：dawnoftan@163.com
   
   color a
   
   set/p option= "Please enter the port : "
   if "%option%"=="" goto :end
   set port=%option%
   set port_row=0.0.0.0:%port%
   
   REM For /f "tokens=5" %%i in ('netstat -ano^|findstr "%port_row%"') do (
   REM     Set /a n+=1
   REM     If !n!==1 set pid=%%i
   REM )
   
   For /f "tokens=2-5" %%a in ('netstat -ano^|findstr "%port_row%"') do (
       If %%a==%port_row% set pid=%%d
   )
   
   echo pid=%pid%, port=%port%
   
   REM zh system language
   set timestamp=%date:~0,4%%date:~5,2%%date:~8,2%%time:~0,2%%time:~3,2%%time:~6,2%
   set "timestamp=%timestamp: =0%"
   
   REM en system language
   REM set timestamp=%date:~10,4%%date:~4,2%%date:~7,2%%time:~0,2%%time:~3,2%%time:~6,2%
   REM set "timestamp=%timestamp: =0%"
   REM echo %timestamp%
   
   set file_path=save_current_%timestamp%_%pid%
   mkdir %file_path%
   
   REM GC信息
   jstat -gc %pid% >> %file_path%\gc.log
   
   REM 栈日志
   jstack %pid% >> %file_path%\stack.log
   
   REM 堆现场日志(live)
   jmap -dump:live,format=b,file=%file_path%\heap.hprof %pid%
   
   REM Pause
   ```





## 工具篇

### tcpdump抓包

1. 什么是tcpdump

   > TCPDump可以将网络中传送的数据包完全截获下来提供分析。它支持针对网络层、协议、主机、网络或端口的过滤，并提供and、or、not等逻辑语句来帮助你去掉无用的信息。

2. tcpdump怎么使用

   > 待更新

3. tcpdump实战

   > 待更新

### wireshark抓包

1. 什么是wireshark

   > Wireshark（前称Ethereal）是一个网络封包分析软件。网络封包分析软件的功能是撷取网络封包，并尽可能显示出最为详细的网络封包资料。Wireshark使用WinPCAP作为接口，直接与网卡进行数据报文交换。

2. wireshark怎么使用

   > 待更新

3. wireshark实战

   > 待更新

### MAT内存分析工具

1. 什么是MAT

   > MAT是分析Java堆内存的一个工具，全称是 The Eclipse Memory Analyzer Tool，用来帮助分析内存泄漏和减少内存消耗。使用MAT分析Java堆快照，可以快速计算出对象的保留大小（Retained Sizes），查找到阻止对象被回收的原因，MAT会自动生成一个包含内存泄漏疑点的报告。

2. MAT怎么使用

   > 待更新

3. 如何分析内存

   > 待更新