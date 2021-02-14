# 环境准备

> win7系统
>  idea 201802
>  jdk openjdk-11.0.2
>  es 6.7.0
>  gradle-5.2.1

# 环境搭建步骤

## 1. 下载es源代码

es源码可以直接从github上面直接`clone`下来即可。es-github地址: [https://github.com/elastic/elasticsearch](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Felastic%2Felasticsearch)

这里直接在idea中进行操作，下载完后然后切换到对应的分支，这里是使用的es6.7所以这里切换到此分支。



```undefined
git checkout -b 6.7 origin/6.7
```

> 建议将其fork到自己github仓库，然后从自己的仓库clone到本地。

## 2.  gradle环境准备

gradle的安装这里不做说明，详细的可以参考[https://blog.csdn.net/Tomgs/article/details/73865508](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2FTomgs%2Farticle%2Fdetails%2F73865508)，这里说明一下如果不知道安装哪个版本的gradle，可以在es的源码目录下面的`gradle/wrapper/gradle-wrapper.properties`中的`distributionUrl`属性指定了es版本，所以这里直接根据这个版本进行下载安装即可，es6.7默认用的是gradle-5.2.1。

安装成功后进行如下配置修改工作方便后续步骤的进行。

- 在项目中指定gradle安装包路径
   将下载的`gradle-5.2.1-all.zip`包放到 `elasticsearch\gradle\wrapper` 目录下，
   确保和 `elasticsearch\gradle\wrapper\gradle-wrapper.properties` 在同级目录，
   然后修改 `elasticsearch\gradle\wrapper\gradle-wrapper.properties` 配置如下：



```swift
distributionUrl=gradle-5.2.1-all.zip
```

- 修改全局gradle仓库地址
   在`USER_HOME/.gradle/`下面创建新文件 `init.gradle`（没有这个文件的可以手动创建），输入下面的内容并保存。
   修改gradle的远程仓库地址为阿里云的仓库：



```ruby
allprojects {
    repositories {
        def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public/'
        all {
            ArtifactRepository repo ->
    if (repo instanceof MavenArtifactRepository) {
                def url = repo.url.toString()
                if (url.startsWith('https://repo.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
        maven {
            url REPOSITORY_URL
        }
    }
}
```

> 其中USER_HOME/.gradle/是自己的gradle安装目录，示例值：C:\Users\Administrator\.gradle，如果没有.gradle目录，可用自己创建

## 3. 编译es源码

在准备好上述环境之后，下面就可以进行相关的编译工作了，在idea的Terminal运行命令：



```undefined
./gradlew idea
```

等待几分钟，这个过程比较慢，如果成功将会看到`BUILD SUCCESSFUL`的字样。

## 4. 打包es源码

在编译好后进行对源码进行打包操作，命令如下：



```undefined
./gradlew -p distribution/archives/tar assemble --parallel
```

此命令将es源码打包成一个tar形式的压缩包`elasticsearch-6.7.0-SNAPSHOT.jar`，位于`distribution/archives/tar`目录下面，为什么这里还需要进行打包，因为es启动的时候需要加载`module`，如果使用下载的安装包里面的`module`的方式可能和下载的源码是不一致的导致不能启动成功，这里在后面的`遇到的问题`章节进行说明。

## 5. 开始调试es源码

将上面的安装包解压到一个具体的位置如：`E:\elasticsearch-6.7.0-SNAPSHOT`。
 找到es的启动类`org.elasticsearch.bootstrap.Elasticsearch`，然后运行，正常来说运行会出现错误的，这些遇到的错误在后面的`遇到的问题`章节进行说明，这里直接把正常的配置贴出来。
 在启动的时候添加如下的jvm参数



```bash
-Xms512m 
-Xmx512m 
-Des.path.home=D:\AjavaGooProject\es\config\elasticsearch-6.7.3-SNAPSHOT
-Des.path.conf=D:\AjavaGooProject\es\config\elasticsearch-6.7.3-SNAPSHOT\config
-Dlog4j2.disable.jmx=true 
-Djava.security.policy=D:\AjavaGooProject\es\config\elasticsearch-6.7.3-SNAPSHOT\config\java.policy 
```

上面`java.policy`文件的内容如下：



```cpp
// default permissions granted to all domains
grant {
    permission java.lang.RuntimePermission "createClassLoader";
    // allows anyone to listen on dynamic ports
    permission java.net.SocketPermission "localhost:0", "listen";

    // "standard" properies that can be read by anyone
    permission java.util.PropertyPermission "java.version", "read";
    permission java.util.PropertyPermission "java.vendor", "read";
    permission java.util.PropertyPermission "java.vendor.url", "read";
    permission java.util.PropertyPermission "java.class.version", "read";
    permission java.util.PropertyPermission "os.name", "read";
    permission java.util.PropertyPermission "os.version", "read";
    permission java.util.PropertyPermission "os.arch", "read";
    permission java.util.PropertyPermission "file.separator", "read";
    permission java.util.PropertyPermission "path.separator", "read";
    permission java.util.PropertyPermission "line.separator", "read";
    permission java.util.PropertyPermission
                   "java.specification.version", "read";
    permission java.util.PropertyPermission "java.specification.vendor", "read";
    permission java.util.PropertyPermission "java.specification.name", "read";
    permission java.util.PropertyPermission
                   "java.vm.specification.version", "read";
    permission java.util.PropertyPermission
                   "java.vm.specification.vendor", "read";
    permission java.util.PropertyPermission
                   "java.vm.specification.name", "read";
    permission java.util.PropertyPermission "java.vm.version", "read";
    permission java.util.PropertyPermission "java.vm.vendor", "read";
    permission java.util.PropertyPermission "java.vm.name", "read";
};
```

然后再启动，会发现启动成功没有报错，这时访问[http://localhost:9200/](https://links.jianshu.com/go?to=http%3A%2F%2Flocalhost%3A9200%2F)，出现如下结果，说明环境搭建成功了。



```json
{
  "name" : "Yqy-_KK",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "heVTcFkoRUihY-hANO3UnA",
  "version" : {
    "number" : "6.7.0",
    "build_flavor" : "unknown",
    "build_type" : "unknown",
    "build_hash" : "Unknown",
    "build_date" : "Unknown",
    "build_snapshot" : true,
    "lucene_version" : "7.7.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

> 上面为什么需要添加这些参数，可以通过报错的信息，来判断缺少什么参数，当然也可以在es的安装包中通过查看es的启动脚本`elasticsearch`的内容来确定需要那些参数。
>
> 
>
> ```bash
> exec \
>    "$JAVA" \
>    $ES_JAVA_OPTS \
>   -Des.path.home="$ES_HOME" \
>    -Des.path.conf="$ES_PATH_CONF" \
>    -Des.distribution.flavor="$ES_DISTRIBUTION_FLAVOR" \
>    -Des.distribution.type="$ES_DISTRIBUTION_TYPE" \
>    -cp "$ES_CLASSPATH" \
>    org.elasticsearch.bootstrap.Elasticsearch \
>    "$@"
> ```

# 遇到的问题

1. 编译出现`The specified initialization script 'C:\Users\***\AppData\Local\Temp\ijinit1.gradle' does not exist.`

> 这个只需要将idea的缓存清空一下再运行编译命令，`File -> Invalidate Caches/Restart`。

1. 启动出现`java.lang.IllegalStateException: path.home is not configured`

> 配置jvm参数：`-Des.path.home=E:\elasticsearch-6.7.0-SNAPSHOT`

1. 启动出现`ERROR: the system property [es.path.conf] must be set`

> 配置jvm参数：`-Des.path.home=E:\elasticsearch-6.7.0-SNAPSHOT\config`

1. 启动出现`ERROR Could not register mbeans java.security.AccessControlException: access denied ("javax.management.MBeanTrustPermission" "register")`

具体堆栈如下：



```csharp
2019-04-20 18:49:21,809 main ERROR Could not register mbeans java.security.AccessControlException: access denied ("javax.management.MBeanTrustPermission" "register")
    at java.base/java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
    at java.base/java.lang.SecurityManager.checkPermission(SecurityManager.java:358)
    at java.management/com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.checkMBeanTrustPermission(DefaultMBeanServerInterceptor.java:1805)
    at java.management/com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.registerMBean(DefaultMBeanServerInterceptor.java:318)
    at java.management/com.sun.jmx.mbeanserver.JmxMBeanServer.registerMBean(JmxMBeanServer.java:522)
    at org.apache.logging.log4j.core.jmx.Server.register(Server.java:393)
    at org.apache.logging.log4j.core.jmx.Server.reregisterMBeansAfterReconfigure(Server.java:168)
    at org.apache.logging.log4j.core.jmx.Server.reregisterMBeansAfterReconfigure(Server.java:141)
    at org.apache.logging.log4j.core.LoggerContext.setConfiguration(LoggerContext.java:558)
    at org.apache.logging.log4j.core.LoggerContext.start(LoggerContext.java:263)
    at org.apache.logging.log4j.core.impl.Log4jContextFactory.getContext(Log4jContextFactory.java:207)
    at org.apache.logging.log4j.core.config.Configurator.initialize(Configurator.java:220)
    at org.apache.logging.log4j.core.config.Configurator.initialize(Configurator.java:197)
    at org.elasticsearch.common.logging.LogConfigurator.configureStatusLogger(LogConfigurator.java:250)
    at org.elasticsearch.common.logging.LogConfigurator.configure(LogConfigurator.java:166)
    at org.elasticsearch.common.logging.LogConfigurator.configure(LogConfigurator.java:127)
    at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:302)
    at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:159)
    at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:150)
    at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86)
    at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124)
    at org.elasticsearch.cli.Command.main(Command.java:90)
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:116)
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:93)
```

> 解决办法：添加jvm参数`-Dlog4j2.disable.jmx=true`

1. 启动出现`java.lang.NoClassDefFoundError: org/elasticsearch/plugins/ExtendedPluginsClassLoader`错误



```kotlin
[2019-04-20T18:51:48,279][ERROR][o.e.b.ElasticsearchUncaughtExceptionHandler] [node-1] fatal error in thread [main], exiting
java.lang.NoClassDefFoundError: org/elasticsearch/plugins/ExtendedPluginsClassLoader
    at org.elasticsearch.plugins.PluginsService.loadBundle(PluginsService.java:545) ~[main/:?]
    at org.elasticsearch.plugins.PluginsService.loadBundles(PluginsService.java:471) ~[main/:?]
    at org.elasticsearch.plugins.PluginsService.<init>(PluginsService.java:163) ~[main/:?]
    at org.elasticsearch.node.Node.<init>(Node.java:339) ~[main/:?]
    at org.elasticsearch.node.Node.<init>(Node.java:266) ~[main/:?]
    at org.elasticsearch.bootstrap.Bootstrap$5.<init>(Bootstrap.java:212) ~[main/:?]
    at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:212) ~[main/:?]
    at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:333) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:159) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:150) ~[main/:?]
    at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86) ~[main/:?]
    at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124) ~[main/:?]
    at org.elasticsearch.cli.Command.main(Command.java:90) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:116) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:93) ~[main/:?]
Caused by: java.lang.ClassNotFoundException: org.elasticsearch.plugins.ExtendedPluginsClassLoader
    at jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:583) ~[?:?]
    at jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178) ~[?:?]
    at java.lang.ClassLoader.loadClass(ClassLoader.java:521) ~[?:?]
    ... 15 more
```

> 解决办法：在idea中的`Run/Debug Configuration`页面勾选'Include dependencies with "Provided" scope'
>
> ![img](https:////upload-images.jianshu.io/upload_images/9484885-14eacaf3b2ab00cc.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/779/format/webp)
>
> Provided

1. 启动出现`java.security.AccessControlException: access denied ("java.lang.RuntimePermission" "createClassLoader")`错误



```kotlin
[2019-04-22T19:29:52,650][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [node-1] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.security.AccessControlException: access denied ("java.lang.RuntimePermission" "createClassLoader")
    at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:163) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:150) ~[main/:?]
    at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86) ~[main/:?]
    at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124) ~[main/:?]
    at org.elasticsearch.cli.Command.main(Command.java:90) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:116) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:93) ~[main/:?]
Caused by: java.security.AccessControlException: access denied ("java.lang.RuntimePermission" "createClassLoader")
    at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472) ~[?:?]
    at java.security.AccessController.checkPermission(AccessController.java:895) ~[?:?]
    at java.lang.SecurityManager.checkPermission(SecurityManager.java:322) ~[?:?]
    at java.lang.SecurityManager.checkCreateClassLoader(SecurityManager.java:384) ~[?:?]
    at java.lang.ClassLoader.checkCreateClassLoader(ClassLoader.java:369) ~[?:?]
    at java.lang.ClassLoader.checkCreateClassLoader(ClassLoader.java:359) ~[?:?]
    at java.lang.ClassLoader.<init>(ClassLoader.java:456) ~[?:?]
    at org.elasticsearch.plugins.ExtendedPluginsClassLoader.<init>(ExtendedPluginsClassLoader.java:36) ~[main/:?]
    at org.elasticsearch.plugins.ExtendedPluginsClassLoader.lambda$create$0(ExtendedPluginsClassLoader.java:57) ~[main/:?]
    at java.security.AccessController.doPrivileged(Native Method) ~[?:?]
    at org.elasticsearch.plugins.ExtendedPluginsClassLoader.create(ExtendedPluginsClassLoader.java:56) ~[main/:?]
    at org.elasticsearch.plugins.PluginLoaderIndirection.createLoader(PluginLoaderIndirection.java:31) ~[main/:?]
    at org.elasticsearch.plugins.PluginsService.loadBundle(PluginsService.java:545) ~[main/:?]
    at org.elasticsearch.plugins.PluginsService.loadBundles(PluginsService.java:471) ~[main/:?]
    at org.elasticsearch.plugins.PluginsService.<init>(PluginsService.java:163) ~[main/:?]
    at org.elasticsearch.node.Node.<init>(Node.java:339) ~[main/:?]
    at org.elasticsearch.node.Node.<init>(Node.java:266) ~[main/:?]
    at org.elasticsearch.bootstrap.Bootstrap$5.<init>(Bootstrap.java:212) ~[main/:?]
    at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:212) ~[main/:?]
    at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:333) ~[main/:?]
    at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:159) ~[main/:?]
    ... 6 more
```

> - 第一种：
>    在jdk的home目录下面 `%JAVA_HOME%\conf\security`下面找到`java.policy`，然后打开将如下内容添加到grant里面。
>    `permission java.lang.RuntimePermission "createClassLoader";`
> - 第二种：
>    在es的config目录下面自己创建一个java.policy文件，然后添加如下内容：
>
> 
>
> ```bash
> grant {
>    permission java.lang.RuntimePermission "createClassLoader";
> }
> ```
>
> 然后启动es时指定jvm参数：
>
> 
>
> ```undefined
> -Djava.security.policy=E:\elasticsearch-6.7.0\config\java.policy
> ```

1. 启动出现`java.lang.NoSuchFieldError: INDEX_SOFT_DELETES_RETENTION_LEASE_PERIOD_SETTING`错误



```swift
[2019-04-22T19:59:54,021][ERROR][o.e.b.ElasticsearchUncaughtExceptionHandler] [node-1] fatal error in thread [main], exiting
java.lang.NoSuchFieldError: INDEX_SOFT_DELETES_RETENTION_LEASE_PERIOD_SETTING
    at org.elasticsearch.xpack.ccr.action.TransportResumeFollowAction.<clinit>(TransportResumeFollowAction.java:390) ~[?:?]
    at jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[?:?]
    at jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[?:?]
    at jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[?:?]
    at java.lang.reflect.Constructor.newInstance(Constructor.java:490) ~[?:?]
    at org.elasticsearch.common.inject.DefaultConstructionProxyFactory$1.newInstance(DefaultConstructionProxyFactory.java:49) ~[main/:?]
    ... 6 more
```

> 这个的原因是下载的发行版的安装包和你git下来的代码不一致导致的，因为在指定es的home路径时会从种加载module模块相关的jar包，所以建议是用源码的方式编译打包然后指定到自己打包的环境，虽然说是安装的版本和源码版本是一样的但是实际结果就是不一样，尴尬。
>  这里也是看到网上很多博客都是下载的和源码版本一样的安装包，就不自己打包了，这里自己打包和源码包对比一下就会发现还是不一样的。这个自己可以把安装包的对应的类的jar包和自己打包的jar包对比一下就知道了，这也是为什么我参考网上一些作者的资料没有运行成功的原因。

1. 启动出现：`initial heap size [201326592] not equal to maximum heap size [3202351104]; this can cause resize pauses and prevents mlockall from locking the entire heap`
    错误如下：



```css
[2019-04-23T11:16:07,203][INFO ][o.e.b.BootstrapChecks    ] [node-1] bound or publishing to a non-loopback address, enforcing bootstrap checks
ERROR: [1] bootstrap checks failed
[1]: initial heap size [201326592] not equal to maximum heap size [3202351104]; this can cause resize pauses and prevents mlockall from locking the entire heap
```

> 这个错误的原因是因为我们启动的时候是直接用的idea命令运行的所以启动的jvm堆的参数都是使用的默认的，所以需要我们自己指定或者指定es的jvm.options文件路径（这个相对麻烦）这里就直接手动指定。
>  `-Xms512m -Xmx512m`

这些就是我在搭建es的源码编译环境所遇到的问题，这里作一下记录。

# elasticsearch源码运行调试的问题

 2019年7月23日 60次阅读 来源: [jinchaolv](https://www.jianshu.com/p/41d949746e62)

由于官方文档对es原理方面的讲解非常少，所以如果时间允许可以自己下载源码在本地调试。

首先在`github`上可以下载到各个版本的源码，写作本文时最新的是`6.4.2`版本：https://github.com/elastic/elasticsearch/tree/v6.4.2

下载的源码是`gradle`项目，所以需要安装`gradle`，对版本有要求，我用的是`4.10.2`版本。

对`java`的版本也有要求，我用的是`jdk11`，`jdk9`以上应该就可以了。

用`idea`打开需要在源码根目录下执行`gradle idea`命令。

然后尝试运行`server`模块下`org.elasticsearch.bootstrap.Elasticsearch`主类，可能会遇到下列问题：

### 1. 配置文件路径设置问题

```
ERROR: the system property [es.path.conf] must be set
```

解决方法：新建任意目录，`idea`在`vm option`中添加设置，如`-Des.path.conf=F:\middleware\elasticsearch-data\config`。

这是放配置文件的地方，如果路径下没有配置文件，将使用默认配置，可将`elasticsearch.yml`文件放在此路径下。

### 2. home路径设置问题

```
Exception in thread "main" java.lang.IllegalStateException: path.home is not configured
```

解决方法：添加`es.path.home`的路径设置，可随意设置，如`-Des.path.home=F:\middleware\elasticsearch-data`。

这是es运行的`home`路径。`plugins`，`modules`，`lib`等都会在此路径下相应的子路径加载。如果不指定，默认的`data`，`log`，`config`路径也会创建在此`home`路径下。但其实因为代码的问题`config`路径必须指定，否则会报错。

### 3. java security权限设置问题

```
ERROR Could not register mbeans java.security.AccessControlException: access denied ("javax.management.MBeanTrustPermission" "register")
```

解决方法：添加设置，`-Dlog4j2.disable.jmx=true`。
由于本人对`java security`了解甚少，此处存疑，这是网上找到的办法，不知道是真正解决了问题，还是仅仅抑制了报错。
经试验也可以在`server`模块找到`src\main\resources\org\elasticsearch\bootstrap\security.policy`文件，在`grant{}`中添加上相应的`permission`，比如这里可以加上

```
permission javax.management.MBeanTrustPermission "register";
```

后面遇到其它类似的问题，加上相应的`permission`即可。

### 4. 日志配置问题

```
ERROR: no log4j2.properties found; tried [F:\middleware\elasticsearch-data\config] and its subdirectories
```

解决方法：在上面配置的`es.path.conf`路径下添加`log4j2.properties`文件，此文件可以在`distribution`模块的`src\config\`路径下找到，建议下载一个相同版本的es发行版（因为后面也要用到），用发行版里面的配置文件。

### 5. modules加载问题

```
org.elasticsearch.bootstrap.StartupException: java.lang.IllegalStateException: modules directory [F:\middleware\elasticsearch-data\modules] not found
```

解决方法：下载一个相同版本的es发行版，将里面的`modules`文件夹复制到上面配置的`es.path.home`路径下。

### 6. libs模块的类加载问题

```
java.lang.NoClassDefFoundError: org/elasticsearch/plugins/ExtendedPluginsClassLoader
```

解决方法：这个类在`libs`模块，`server`模块中原来的`gradle`配置是

```
compileOnly project(':libs:plugin-classloader')
```

`compileOnly`改为`compile`即可。

至此，应该已经能够正常运行es源码了。

### 7. transport通信断点调试

es集群节点间通信都是通过`transport`模块来进行的，想要调试节点间的transport请求，在`TransportRequestHandler`接口的默认方法`messageReceived`方法处打个断点，节点间交互时都会进来。

```
public interface TransportRequestHandler<T extends TransportRequest> {

    /**
     * Override this method if access to the Task parameter is needed
     */
    default void messageReceived(final T request, final TransportChannel channel, Task task) throws Exception {
        messageReceived(request, channel);
    }

    void messageReceived(T request, TransportChannel channel) throws Exception;
}
```

### 8. http通信断点调试

集群外部访问集群，比如`search`、`index`等操作通常是`http`方式的，调试这种可以在`RestController`类的`dispatchRequest`方法处打个断点，这样对该节点的`http`请求都会进到这里来。

```
    @Override
    public void dispatchRequest(RestRequest request, RestChannel channel, ThreadContext threadContext) {
        if (request.rawPath().equals("/favicon.ico")) {
            handleFavicon(request, channel);
            return;
        }
        try {
            tryAllHandlers(request, channel, threadContext);
        } catch (Exception e) {
            try {
                channel.sendResponse(new BytesRestResponse(channel, e));
            } catch (Exception inner) {
                inner.addSuppressed(e);
                logger.error(() ->
                    new ParameterizedMessage("failed to send failure response for uri [{}]", request.uri()), inner);
            }
        }
    }
```

  原文作者：jinchaolv
  原文地址: https://www.jianshu.com/p/41d949746e62
  本文转自网络文章，转载此文章仅为分享知识，如有侵权，请联系博主进行删除。

[ 点赞](javascript:;) [ 分享](javascript:;)