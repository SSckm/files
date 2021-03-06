## 简介

```
继上篇文章调试源码环境以后，本文主要讲解Elasticsearch在启动中要做的那些事。
```

## 环境介绍

 ```
 系统：Mac os
 Elasticsearch6.1.3
 ```

## 类图

基本上重要的就先看这四个类。启动的过程基本上都在这四个类里面了。

![](https://img.soaer.com/blog/29.png)



## 方法执行流程

```
这个基本上就是启动所经过的类的防范。下面会分开介绍每个流程中的关键点。
1.org.elasticsearch.bootstrap.Elasticsearch:main(String[] args)
2.org.elasticsearch.cli.Command:main(String[] args, Terminal terminal)
3.org.elasticsearch.cli.Command:mainWithoutErrorHandling(String[] args, Terminal terminal)
4.org.elasticsearch.cli.EnvironmentAwareCommand:execute(Terminal terminal, OptionSet options)
5.org.elasticsearch.bootstrap.Elasticsearch:execute(Terminal terminal, OptionSet options, Environment env)
6.org.elasticsearch.bootstrap.Bootstrap:init(final boolean foreground, final Path pidFile, final boolean quiet, final Environment initialEnv)
7.org.elasticsearch.bootstrap.Bootstrap:setUp(boolean addShutdownHook, Environment environment)
8.org.elasticsearch.bootstrap.Bootstrap:start()
```

## 启动类

```
core:java.org.elasticsearch.bootstrap.Elasticsearch.java
public static void main(final String[] args) throws Exception {
    System.setSecurityManager(new SecurityManager() {
        @Override
        public void checkPermission(Permission perm) {
        }
    });
    LogConfigurator.registerErrorListener();
    ......
}
```

##设置SecurityManager

```
由于java的SecurityManager默认是关闭的，打开的方式之一就是System.setSecurityManager(sm),这里的第一步就是打开SecurityManager，控制应用程序所能访问的资源，这里新建了一个SecurityManager，并且重写了checkPermission方法，方法无方法体，安全管理器通过抛出异常来提供阻止操作完成的机会,说明如果使用这个SecurityManager将不会抛出异常，允许操作。ES在后面后重新设置安全管理器。我觉得在这里就是先把权限放开，启动完成后会再次设置安全管理器。后面会有介绍。
```

## Bootstrap.init

```
由于前面几步基本上都是环境变量或者配置参数的问题，在这就不在详细讲述了。
```

## 初始化Bootstrap

```
在这里其实就是申请了一个非守护线程，并把该线程的执行放到了当jvm退出时执行。
```

## keystore文件

```
没读源码之前根本就不知道这个东西的存在。这个文件可以配置在$ES_HOME/config下面。文件名称为：elasticsearch.keystore，在启动的过程中会读取这个文件如果有的话。这个文件的主要作用为有些设置是敏感的，仅仅依靠系统文件权限是不够的，所以可以加密处理。
详情请看：https://www.elastic.co/guide/en/elasticsearch/reference/6.1/secure-settings.html#creating-keystore   官网有很详细的解释和使用步骤。以及怎么创建keystore文件，这里需要注意的是要使用elasticsearch提供的命令进行生成这个文件，而不是直接拿keytool生成，那样会报错。在bin/目录下面会有这个命令，感兴趣的同学可以查看下。
```

## 检查Lucene版本

```
private static void checkLucene() {
	....
	....
}
如果Lucene不对的话也会报错。
```

## setUp方法

### 扫描插件

```
基本上就是读取$ES_HOME/plugins/下面产插件目录的plugin-descriptor.properties文件，然后读取信息，查看插件是针对ES的那个版本等等操作。从而也会判断该插件是否需要程序启动，如果需要启动则会放入到Process里面可以执行外部的程序。
如下：
if (!info.hasNativeController()) {
     final String message = String.format(Locale.ROOT,
                            "plugin [%s] does not have permission to fork native controller",
                            plugin.getFileName());
                    throw new IllegalArgumentException(message);
     }
     final Process process = spawnNativePluginController(spawnPath, environment.tmpFile());
     processes.add(process);
```

## 禁止root执行程序

```
if (Natives.definitelyRunningAsRoot()) {
    throw new RuntimeException("can not run elasticsearch as root");
}
```

## Installs a system call 

```
Natives.tryInstallSystemCallFilter(tmpFile);安装不成功会报错。跟系统调用有关系
```

## Swapping

```
配置文件里面bootstrap.memory_lock: true这个选项。看到这感觉配置文件好像默认的是true, 但是代码里面默认的为false.
代码如下：
Setting.boolSetting("bootstrap.memory_lock", false, Property.NodeScope);
官方说法是发生系统swapping的时候ES节点的性能会非常差，也会影响节点的稳定性。所以要避免swapping。swapping会导致Java GC的周期延迟从毫秒级恶化到分钟，更严重的是会引起节点响应延迟甚至脱离集群，但是如果不设置的话默认节点都会是false，所以在部署集群的时候要注意。默认值为false,重要的事情要多说几遍。
```

## 设置安全管理器

```
Security.configure(environment, BootstrapSettings.SECURITY_FILTER_BAD_DEFAULTS_SETTING.get(settings));
在这里可以看到，在执行或者检查了一系列系统操作或者可能会设计到系统调用后重新设置了安全管理器。重新设置了Policy，基本上都是基于路径、模板或者插件的权限控制。
```

```
static void configure(Environment environment, boolean filterBadDefaults) throws IOException, NoSuchAlgorithmException {
  Policy.setPolicy(new ESPolicy(createPermissions(environment), getPluginPermissions(environment), filterBadDefaults));
 final String[] classesThatCanExit = new String[]{ElasticsearchUncaughtExceptionHandler.class.getName(), Command.class.getName()};
  System.setSecurityManager(new SecureSM(classesThatCanExit));
  selfTest();
}
```

## Node的初始化

### Environment

```
从上面的类图里面可以看到Environment类主要存放的是数据目录各种file的存放路径。在这里要注意一点就是：node.local_storage如果在配置文件中设置的是false,那么这个节点的node.master和node.data也必须设置为false。不然会报以下错误：
throw new IllegalArgumentException("storage can not be disabled for master and data nodes");
主要作用是，从配置文件或者启动参数中设置：
dataFiles，dataWithClusterFiles，configFile，pluginsFile，modulesFile，sharedDataFile，binFile，libFile，logsFile，pidFile。

这里要说下dataWithClusterFiles，代码里面有说明如下：
/**
 * The data location with the cluster name as a sub directory.
 *
 * @deprecated Used to upgrade old data paths to new ones that do not include the cluster name, should not be used to write files to and
 * will be removed in ES 6.0
*/
 @Deprecated
 public Path[] dataWithClusterFiles() {
    return dataWithClusterFiles;
 }
```

### NodeEnvironment类

#### 目录加锁

```
在Node初始换工程中会创建一个NodeEnvironment， 就是节点环境，其中会检查该节点的数据目录是否已经有节点占用，如果有节点占用则会跑出异常：
下面这个代码获
locks[dirIndex] = luceneDir.obtainLock(NODE_LOCK_FILENAME);，如果上个节点使用了这个目录，那么重新启动节点会获
取不到锁，抛出异常

failed to obtain node locks, tried ................
如果有兴趣可以查看下java中的LockFactory的obtainLock的实现类。
```

#### 生成节点名称

```
如果没有配置节点名称，会随机生成，生成节点名称在NodeMetaData里面生成，生成过程中用到了java的SecureRandom类。具体的实现可以看下org.elasticsearch.common.RandomBasedUUIDGenerator里面的getBase64UUID方法。

public String getBase64UUID(Random random) {
	final byte[] randomBytes = new byte[16];
	random.nextBytes(randomBytes);
	/* Set the version to version 4 (see http://www.ietf.org/rfc/rfc4122.txt)
	* The randomly or pseudo-randomly generated version.
	* The version number is in the most significant 4 bits of the time
	* stamp (bits 4 through 7 of the time_hi_and_version field).*/
	randomBytes[6] &= 0x0f;  /* clear the 4 most significant bits for the version  */
	randomBytes[6] |= 0x40;  /* set the version to 0100 / 0x40 */

	/* Set the variant: 
	 * The high field of th clock sequence multiplexed with the variant.
	* We set only the MSB of the variant*/
	randomBytes[8] &= 0x3f;  /* clear the 2 most significant bits */
	randomBytes[8] |= 0x80;  /* set the variant (MSB is set)*/
	这里为什么会对第六个和第八个字节做特殊处理呢，随机数是根据当前时间的，看这个注释应该是这四个bit对时间的影响比较大，所以做这个操作。
	有兴趣的可以看下http://www.ietf.org/rfc/rfc4122.txt这里地址里面有随机的说明。
```

#### 扫描Module

```
static Set<Bundle> getModuleBundles(Path modulesDirectory) throws IOException {
    if (Files.notExists(modulesDirectory)) {
        return Collections.emptySet();
    }
    Set<Bundle> bundles = new LinkedHashSet<>();
    try (DirectoryStream<Path> stream = Files.newDirectoryStream(modulesDirectory)) {
        for (Path module : stream) {
            PluginInfo info = PluginInfo.readFromProperties(module);
             Set<URL> urls = new LinkedHashSet<>();
             try (DirectoryStream<Path> jarStream = Files.newDirectoryStream(module, "*.jar")) {
                 for (Path jar : jarStream) {
                     URL url = jar.toRealPath().toUri().toURL();
                     if (urls.add(url) == false) {
                         throw new IllegalStateException("duplicate codebase: " + url);
                     }
                    }
                }
                if (bundles.add(new Bundle(info, urls)) == false) {
                    throw new IllegalStateException("duplicate module: " + info);
                }
            }
        }
        return bundles;
}
可以学习一下代码里面扫描目录和加载固定后缀文件的方式。此处仅仅是加载module的属性以及jar包的文件路径。
```

#### 加载Moudle和plugin的jar包

```
加载jar包主要在PluginsService类里面的loadBundles方法。

ClassLoader loader = URLClassLoader.newInstance(bundle.urls.toArray(new URL[0]),getClass().getClassLoader());
reloadLuceneSPI(loader);
final Class<? extends Plugin> pluginClass = loadPluginClass(bundle.plugin.getClassname(), loader);
final Plugin plugin = loadPlugin(pluginClass, settings, configPath);
plugins.add(new Tuple<>(bundle.plugin, plugin));


public void reload(ClassLoader classloader) {
    Objects.requireNonNull(classloader, "classloader");
    final LinkedHashMap<String,S> services = new LinkedHashMap<>(this.services);
    final SPIClassIterator<S> loader = SPIClassIterator.get(clazz, classloader);
    while (loader.hasNext()) {
      final Class<? extends S> c = loader.next();
      try {
        final S service = c.newInstance();
        final String name = service.getName();
        // only add the first one for each name, later services will be ignored
        // this allows to place services before others in classpath to make 
        // them used instead of others
        if (!services.containsKey(name)) {
          checkServiceName(name);
          services.put(name, service);
        }
      } catch (Exception e) {
        throw new ServiceConfigurationError("Cannot instantiate SPI class: " + c.getName(), e);
      }
    }
    this.services = Collections.unmodifiableMap(services);
}

把加载的类放入到services里面。这里面部分是lucene的代码
加载完成后就是我们所看到的加载日志了如下：
....
[2018-02-02T14:04:56,206][INFO ][o.e.p.PluginsService     ] [KFsrWtW] loaded module [reindex]
[2018-02-02T14:04:56,210][INFO ][o.e.p.PluginsService     ] [KFsrWtW] loaded module [repository-url]
[2018-02-02T14:04:56,213][INFO ][o.e.p.PluginsService     ] [KFsrWtW] loaded module [transport-netty4]
[2018-02-02T14:04:56,216][INFO ][o.e.p.PluginsService     ] [KFsrWtW] loaded module [tribe]
[2018-02-02T14:05:56,828][INFO ][o.e.p.PluginsService     ] [KFsrWtW] loaded plugin [analysis-ik]
.....
```

### ThreadPool类

```
计算线程池的容量
打印日志如下可以大致看下都有哪些，以及容量和队列的大小。
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [force_merge], size [1], queue size [unbounded]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [fetch_shard_started], core [1], max [8], keep alive [5m]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [listener], size [2], queue size [unbounded]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [index], size [4], queue size [200]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [refresh], core [1], max [2], keep alive [5m]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [generic], core [4], max [128], keep alive [30s]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [warmer], core [1], max [2], keep alive [5m]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [search], size [7], queue size [1k]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [flush], core [1], max [2], keep alive [5m]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [fetch_shard_store], core [1], max [8], keep alive [5m]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [management], core [1], max [5], keep alive [5m]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [get], size [4], queue size [1k]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [bulk], size [4], queue size [200]
[o.e.t.ThreadPool] [KFsrWtW] created thread pool: name [snapshot], core [1], max [2], keep alive [5m]

--core      the minimum number of threads in the pool
--max       the maximum number of threads in the pool
--keepAlive the time that spare threads above {@code core} threads will be kept alive
--queueSize the size of the backing queue, -1 for unbounded
根据机器而定，比如多核的肯定要比这个大，我这个是四核的机器。 可以看到线程池容量最大的还是search。

注意此时ES还申请了一个叫timer守护线程，用来定时更新时间，缓存时间。
this.cachedTimeThread = new CachedTimeThread(EsExecutors.threadName(settings, "[timer]"), estimatedTimeInterval.millis());
可以设置指定的时间间隔。
```

### 加载所有的分词器

```
这里贴一下所有的（安装ik以后的，可以参考一下）算是分词器大全了吧。ik_smart和ik_max_word是ik的两个分词器，其他的都是默认自带的。看到有chinese，但是没用过。
{
    standard=org.elasticsearch.indices.analysis.AnalysisModule
    german=org.elasticsearch.indices.analysis.AnalysisModule
    pattern=org.elasticsearch.indices.analysis.AnalysisModule
    irish=org.elasticsearch.indices.analysis.AnalysisModule
    sorani=org.elasticsearch.indices.analysis.AnalysisModule
    simple=org.elasticsearch.indices.analysis.AnalysisModule
    standard_html_strip=org.elasticsearch.indices.analysis.AnalysisModule
    hungarian=org.elasticsearch.indices.analysis.AnalysisModule
    norwegian=org.elasticsearch.indices.analysis.AnalysisModule
    dutch=org.elasticsearch.indices.analysis.AnalysisModule
    chinese=org.elasticsearch.indices.analysis.AnalysisModule
    default=org.elasticsearch.indices.analysis.AnalysisModule
    arabic=org.elasticsearch.indices.analysis.AnalysisModule
    ik_smart=org.elasticsearch.plugin.analysis.ik.AnalysisIkPlugin
    bengali=org.elasticsearch.indices.analysis.AnalysisModule
    english=org.elasticsearch.indices.analysis.AnalysisModule
    fingerprint=org.elasticsearch.indices.analysis.AnalysisModule
    portuguese=org.elasticsearch.indices.analysis.AnalysisModule
    keyword=org.elasticsearch.indices.analysis.AnalysisModule
    romanian=org.elasticsearch.indices.analysis.AnalysisModule
    french=org.elasticsearch.indices.analysis.AnalysisModule
    czech=org.elasticsearch.indices.analysis.AnalysisModule
    greek=org.elasticsearch.indices.analysis.AnalysisModule
    indonesian=org.elasticsearch.indices.analysis.AnalysisModule
    swedish=org.elasticsearch.indices.analysis.AnalysisModule
    spanish=org.elasticsearch.indices.analysis.AnalysisModule
    danish=org.elasticsearch.indices.analysis.AnalysisModule
    russian=org.elasticsearch.indices.analysis.AnalysisModule
    cjk=org.elasticsearch.indices.analysis.AnalysisModule
    armenian=org.elasticsearch.indices.analysis.AnalysisModule
    basque=org.elasticsearch.indices.analysis.AnalysisModule
    italian=org.elasticsearch.indices.analysis.AnalysisModule
    lithuanian=org.elasticsearch.indices.analysis.AnalysisModule
    thai=org.elasticsearch.indices.analysis.AnalysisModule
    persian=org.elasticsearch.indices.analysis.AnalysisModule
    catalan=org.elasticsearch.indices.analysis.AnalysisModule
    finnish=org.elasticsearch.indices.analysis.AnalysisModule
    stop=org.elasticsearch.indices.analysis.AnalysisModule
    brazilian=org.elasticsearch.indices.analysis.AnalysisModule
    turkish=org.elasticsearch.indices.analysis.AnalysisModule
    hindi=org.elasticsearch.indices.analysis.AnalysisModule
    snowball=org.elasticsearch.indices.analysis.AnalysisModule
    bulgarian=org.elasticsearch.indices.analysis.AnalysisModule
    ik_max_word=org.elasticsearch.plugin.analysis.ik.AnalysisIkPlugin
    whitespace=org.elasticsearch.indices.analysis.AnalysisModule
    galician=org.elasticsearch.indices.analysis.AnalysisModule
    latvian=org.elasticsearch.indices.analysis.AnalysisModule
}
```

### 加载所有的Module

```
b.bind(Node.class).toInstance(this);
b.bind(NodeService.class).toInstance(nodeService);
b.bind(NamedXContentRegistry.class).toInstance(xContentRegistry);
b.bind(PluginsService.class).toInstance(pluginsService);
b.bind(Client.class).toInstance(client);
b.bind(NodeClient.class).toInstance(client);
b.bind(Environment.class).toInstance(this.environment);
b.bind(ThreadPool.class).toInstance(threadPool);
b.bind(ResourceWatcherService.class).toInstance(resourceWatcherService);
......
```

