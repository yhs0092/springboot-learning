# 1 主启动流程

## 1.1 SpringApplication.initialize()
> 启动时设置的 source 就是`TracedZuulMain.class`本身。

### 1.1.1 deduceWebEnvironment()
是否是web程序？（如果SpringApplication.WEB_ENVIRONMENT_CLASSES中的类可以加载则是web程序）

### 1.1.2 加载org.springframework.context.ApplicationContextInitializer
加载的initializer有：
0. DelegatingApplicationContextInitializer
1. ContextIdApplicationContextInitializer
2. ConfigurationWarningsApplicationContextInitializer
3. ServerPortInfoApplicationContextInitializer
4. SharedMetadataReaderFactoryContextInitializer
5. AutoConfigurationReportLoggingInitializer

### 1.1.3 加载org.springframework.context.ApplicationListener
加载的 ApplicationListener 有：
0. BootstrapApplicationListener
1. LoggingSystemShutdownListener
2. ConfigFileApplicationListener
3. AnsiOutputApplicationListener
4. LoggingApplicationListener
5. ClasspathLoggingApplicationListener
6. BackgroundPreinitializer
7. RestartListener
8. DelegatingApplicationListener
9. ParentContextCloserApplicationListener
10. ClearCachesApplicationListener
11. FileEncodingApplicationListener
12. LiquibaseServiceLocatorApplicationListener

### 1.1.4 找出启动的Main类

## 1.2 SpringApplication.run()
  > - `SpringApplication.run()`方法中初始化了一个 SpringApplicationRunListeners 对象，`SpringApplication.run()`方法执行到不同阶段时会调用`SpringApplicationRunListeners`的不同方法去触发其内部的`SpringApplicationRunListener`对象的对应方法，以广播不同的`ApplicationEvent`。
  > - 当前`SpringApplicationRunListener`的实现类只有`EventPublishingRunListener`，其内部使用`SimpleApplicationEventMulticaster`广播事件。
  >
  > ![](http://7x2wh6.com1.z0.glb.clouddn.com/startup2.jpg "SpringApplicationRunListeners工作逻辑")


### 1.2.1 SpringApplicationRunListeners的关键时间点

- started  

- prepareEnvironment  
  1. 构造一个ConfigurableEnvironment对象（也有可能是之前set进去的）
  2. 加载一些propertySource，包括SystemProperty、SystemEnvironment（此时还不会读取配置文件）
  3. 检查是否配置了active profile，若有配置，则记录下来
  4. 广播ApplicationEnvironmentPreparedEvent
  5. 广播涉及的Listener：
    0. BootstrapApplicationListener （特定条件下，已知是Spring Cloud场景下，会从bootstrap.yaml文件加载配置。 org.springframework.cloud.bootstrap.BootstrapApplicationListener会重新拉起一个Spring Context，并且重新跑一遍整个启动过程），见[BootstrapApplicationListener](#BootstrapApplicationListener)。
    1. LoggingSystemShutdownListener
    2. ConfigFileApplicationListener
      1. 加载EnvironmentPostProcessor
    3. AnsiOutputApplicationListener
    4. LoggingApplicationListener
    5. ClasspathLoggingApplicationListener
    6. BackgroundPreinitializer
    7. DelegatingApplicationListener
    8. FileEncodingApplicationListener
- contextPrepared
- contextLoaded
- finished

### 1.2.2 started
> 入口为 SpringApplication.java 304-305行

加载`SpringApplicationRunListeners`实例，里面的`SpringApplicationRunListener`实例只有一个，就是`org.springframework.boot.context.event.EventPublishingRunListener`。

广播ApplicationStartedEvent。对应Listener为
1. ConfigFileApplicationListener （但是此处广播的事件类型ApplicationStartedEvent不会触发ConfigFileApplicationListener的动作）
2. LoggingApplicationListener
3. DelegatingApplicationListener （DelegatingApplicationListener没有动作）
4. LiquibaseServiceLocatorApplicationListener

### 1.2.3 prepareEnvironment
> 入口为 SpringApplication.java 309行。

#### 1.2.3.1 生成及初始化ConfigurableEnvironment
初始化`ConfigurableEnvironment`，生成的是`StandardServletEnvironment`实例。

配置`PropertySources`。构造`StandardServletEnvironment`实例的时候已经在里面设置了一些property source，包括一些servlet配置、SystemProperties、SystemEnvironment。<br/>
如果这个`SpringApplication`里面设置了`defaultProperties`，则会用这些默认属性创建一个名为"defaultProperties"的`PropertySource`，加入到`PropertySources`的末尾，即最低优先级。<br/>
如果有commandLineProperties（就是启动代码`SpringApplication.run(TracedZuulMain.class, args)`中的`args`），则会向名为"commandLineArgs"的`PropertySource`中增加这些属性值，并将这个`PropertySource`放到`PropertySources`的最前面，即最高优先级。（放置动作取决于名为"commandLineArgs"的`PropertySource`是否事先存在，如果是则在原位置替换，否则放在最前面）

配置Profile。<br/>
如果在当前已加载的`PropertySources`中设置了`spring.profiles.active`属性值，则根据这个获取active Profile，将其设置到`Environment`的`activeProfiles`属性中。并且加入`SpringApplication.additionalProfiles`中保存的额外 profile。

#### 1.2.3.2 ApplicationEnvironmentPreparedEvent
广播`ApplicationEnvironmentPreparedEvent`，相关的`ApplicationListener`如下。
_目前分析的结果是，2、 4~9 都没有修改配置项的操作。_

1. BootstrapApplicationListener<br/>
特定条件下，已知是Spring Cloud场景下，会从bootstrap.yaml文件加载配置。 org.springframework.cloud.bootstrap.BootstrapApplicationListener会重新拉起一个Spring Context，并且重新跑一遍整个启动过程，此过程分析见[BootstrapApplicationListener生成新的Spring Context](#BootstrapApplicationListener "BootstrapApplicationListener生成新的Spring Context")。<br/>
**TODO: 还有`bootstrapProperties`回合以及`DelegatingEnvironmentDecryptApplicationInitializer`没有分析**

2. LoggingSystemShutdownListener<br/>
<blockquote>Cleans up the logging system immediately after the bootstrap context is created on startup. Logging will go dark until the ConfigFileApplicationListener fires, but this is the price we pay for that listener being able to adjust the log levels according to what it finds in its own configuration.</blockquote>
按照`LoggingSystemShutdownListener`的注释，应该是暂时关闭了日志记录。

3. ConfigFileApplicationListener<br/>
读取配置文件，见[ConfigFileApplicationListener 处理 ApplicationEnvironmentPreparedEvent](#ConfigFileApplicationListener处理ApplicationEnvironmentPreparedEvent "ConfigFileApplicationListener 处理 ApplicationEnvironmentPreparedEvent")。

4. AnsiOutputApplicationListener<br/>
<blockquote>An ApplicationListener that configures AnsiOutput depending on the value of the property spring.output.ansi.enabled.</blockquote>
尝试从`ConfigurableEnvironment`中读取配置项"spring.output.ansi.enabled"和"spring.output.ansi.console-available"，用来设置AnsiOutput（AnsiOutput应该指的是彩色日志打印）。

5. LoggingApplicationListener<br/>
根据配置初始化和设置日志系统。

6. ClasspathLoggingApplicationListener<br/>
<blockquote>A SmartApplicationListener that reacts to environment prepared events and to failed events by logging the classpath of the thread context class loader (TCCL) at DEBUG level.</blockquote>
当程序正常启动成功时，将classpath打印到debug日志；当程序启动失败时，将classpath打印到info日志。（[参考《Spring Boot启动流程详解》][Spring Boot启动流程详解]）

7. BackgroundPreinitializer
<blockquote>ApplicationListener to trigger early initialization in a background thread of time consuming tasks.</blockquote>
触发了一些耗时任务的预初始化，包括：
  1. MessageConverterInitializer
  2. MBeanFactoryInitializer
  3. ValidationInitializer
  4. JacksonInitializer
  5. ConversionServiceInitializer

8. DelegatingApplicationListener
<blockquote>ApplicationListener that delegates to other listeners that are specified under a context.listener.classes environment property.</blockquote>
根据`context.listener.classes`获取配置项（委托类的名字），加载`ApplicationListener`委托类，将event传给这些委托类处理。

9. FileEncodingApplicationListener
<blockquote>An ApplicationListener that halts application startup if the system file encoding does not match an expected value set in the environment. By default has no effect, but if you set spring.mandatory_file_encoding (or some camelCase or UPPERCASE variant of that) to the name of a character encoding (e.g. "UTF-8") then this initializer throws an exception when the file.encoding System property does not equal it.</blockquote>
检查System Properties中的`file.encoding`值和配置项中的`spring.mandatoryFileEncoding`值是否匹配（`spring.mandatoryFileEncoding`的属性名是按照一定规则模糊匹配的），如果不匹配则抛异常终止启动过程。

##### <div id="ConfigFileApplicationListener处理ApplicationEnvironmentPreparedEvent">1.2.3.2.1 ConfigFileApplicationListener 处理 ApplicationEnvironmentPreparedEvent</div>
加载`EnvironmentPostProcessor`，包含如下内容。其中`SpringBootConfigMergeProcessor`是实现了`Ordered`接口的最低优先级的`EnvironmentPostProcessor`。

1. SpringApplicationJsonEnvironmentPostProcessor<br/>
从`spring.application.json`或`SPRING_APPLICATION_JSON`加载json串格式的配置项，将其增加到ConfigurableEnvironment的property sources中，新增property source的名字为`spring.application.json`，优先级高于`jndiProperties`和`systemProperties`（没有这两个时会放在最前面）。参考资料见[Spring Boot 1.4.5](https://docs.spring.io/spring-boot/docs/1.4.5.RELEASE/reference/html/boot-features-external-config.html)和[Spring Boot 2.0](https://juejin.im/entry/5a4b33e6518825258227bfbe)。

2. HostInfoEnvironmentPostProcessor<br/>
获取本机的网络相关的信息，生成一个名为`springCloudClientHostInfo`的property source加入到ConfigurableEnvironment中，property source的优先级为当前最低。

3. CloudFoundryVcapEnvironmentPostProcessor<br/>
貌似是用来处理Cloud Foundry的配置项的（存疑）。<br/>
如果当前ConfigurableEnvironment中存在名为`VCAP_APPLICATION`或`VCAP_SERVICES`的配置项则生效，将`VCAP_APPLICATION`和`VCAP_SERVICES`中的json串配置扁平化，分别加上`vcap.application.`和`vcap.services.`前缀，放入到名为`vcap`的property source中，置入ConfigurableEnvironment，该property source的优先级仅次于"commandLineArgs"。

4. ConfigFileApplicationListener<br/>
Spring Boot在这里加载配置文件，操作内容包括：
  1. 增加一个`RandomValuePropertySource`，优先级低于"systemEnvironment"（ConfigFileApplicationListener.java 210行）
  2. 加载property source，设置active profiles
    - 根据"spring.profiles.active"属性获取active profiles， active profiles 的优先级是后面的高于前面的（这与property sources相反，因为后面根据active profile加载property source时是用`addFirst()`方法加入到property sources中的）。 如果没有 active profile， 则增加一个default profile。然后增加了一个`null` profile。<br/>
    ***TODO: 这里的`default`还有`null` profile究竟是如何表现的还需要实验验证***
    - ConfigFileApplicationListener.java 377行，根据profile、文件名、路径、文件扩展名加载配置文件。加载成的property source在这个时候还不会直接加入到`ConfigurableEnvironment.propertySources`中，而是保存在当前用于加载配置文件的`PropertySourcesLoader`实例里。
    - ConfigFileApplicationListener.java 470行，读取到配置文件后还会再一次扫描active profile，如果读取到了就会将`ConfigurableEnvironment.activeProfiles`清空并set进去，然后将`ConfigFileApplicationListener.activatedProfiles`设置为`true`以保证只会设置一次profile（方法`ConfigFileApplicationListener.maybeActivateProfiles()`会检查`activatedProfiles`）。
    - 设置active profile之后会检查一次是否配置了`spring.profiles.include`属性，若配置了则将其中指定的 profile 加入到`ConfigFileApplicationListener`中，并设置到`ConfigurableEnvironment.activeProfiles`中（先清空再设置的）。
    - ConfigFileApplicationListener.java 384行，经过反复加载配置文件之后（文件位置、文件名、profile、文件扩展名这几个维度），将`PropertySourcesLoader.propertySources`中保存的property sources加入到`ConfigurableEnvironment`中（ConfigFileApplicationListener.java 384行，`ConfigFileApplicationListener.Loader.load()`方法的结尾）。
    - ***TODO: 有条件进一步弄清楚`ConfigFileApplicationListener.loadIntoGroup()`方法的三个参数各有什么意义***<br/>
    当`ConfigFileApplicationListener.load()`方法的profile不为null时，为什么要分三个阶段调用`loadIntoGroup()`方法。<br/>
    _貌似这种反复加载是因为同一个配置文件中也可以配置多个profile的属性，待验证。参考[Multi-profile YAML documents](https://docs.spring.io/spring-boot/docs/1.4.5.RELEASE/reference/htmlsingle/#boot-features-external-config-multi-profile-yaml "Multi-profile YAML documents")。_
  3. 从System property中读取名为"spring.beaninfo.ignore"的属性，如果没有则从`ConfigurableEnvironment`中解析出对应的配置项（使用`RelaxedPropertyResolver`），默认值为`true`，并设置到System property中。<br/>
  根据查的资料，"spring.beaninfo.ignore"配置项似乎决定了是否跳过BeanInfo类加载。
  4. Bind the environment to the SpringApplication. 具体产生了什么效果还不明确。

5. SpringBootConfigMergeProcessor<br/>
这里取得SpringBoot的配置文件留待合入到ServiceComb中。<br/>
现在发现这个processor会触发两次，一次是`BootstrapApplicationListener`触发重新生成SpringApplication，一次是外层的SpringBoot启动。现在发现可以根据配置项`spring.config.name`或`spring.application.name`区分。
**TODO: 这里还需要考虑获取多次配置的场景下如何处理这些多的配置项**

6. ServoEnvironmentPostProcessor<br/>
尝试设置一个Netflix servo的默认配置项`spring.aop.proxyTargetClass`。<br/>
如果ConfigurableEnvironment中包含一个名为"defaultProperties"的property source，且为MapPropertySource类型，则在其中设置`spring.aop.proxyTargetClass=true`；否则新建一个名为"defaultProperties"的property source设置配置项，该property source放在最低优先级位置。<br/>
`ServoEnvironmentPostProcessor`没有实现`Ordered`接口或使用`@Order`/`@Priority`注解，所以会排在后面，这里的配置项`SpringBootConfigMergeProcessor`拿不到。

### 1.2.4 prepareApplicationContext
> 入口为 SpringApplication.java 312-314行。

#### 1.2.4.1 createApplicationContext
> Strategy method used to create the ApplicationContext. By default this method will respect any explicitly set application context or application context class before falling back to a suitable default.

创建了一个`org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext`。

#### 1.2.4.2 prepareContext
> 入口在 SpringApplication.java 314行。

##### 1.2.4.2.1 setEnvironment
新创建的`AnnotationConfigEmbeddedWebApplicationContext`中有一个默认生成的`Environment`，这里将上一步生成的`ConfigurableEnvironment`设置进去替代它，并且还会设置到`AnnotationConfigEmbeddedWebApplicationContext.reader`和`AnnotationConfigEmbeddedWebApplicationContext.scanner`中，在解析占位符、判断`@Conditional`注解时需要用到。

##### 1.2.4.2.2 postProcessApplicationContext
- 加载`beanNameGenerator`
- 在`ApplicationContext`中设置`resourceLoader`

debug过程中发现这两个都是null，所以没有设置。

##### 1.2.4.2.3 applyInitializers
设置`ApplicationContextInitializer`，**根据网上查到的资料，`ApplicationContextInitializer`是Spring容器刷新之前执行的回调函数，对`ConfigurableApplicationContext`进行操作。**<br/>
debug过程中发现加载的有：
1. BootstrapApplicationListener$AncestorInitializer
2. BootstrapApplicationListener$DelegatingEnvironmentDecryptApplicationInitializer
3. PropertySourceBootstrapConfiguration$$EnhancerBySpringCGLIB$$64c5bba1
4. EnvironmentDecryptApplicationInitializer
5. DelegatingApplicationContextInitializer
6. ContextIdApplicationContextInitializer
7. ConfigurationWarningsApplicationContextInitializer
8. ServerPortInfoApplicationContextInitializer
9. SharedMetadataReaderFactoryContextInitializer
10. AutoConfigurationReportLoggingInitializer

_逐个分析这些`ApplicationContextInitializer`_

1. BootstrapApplicationListener$AncestorInitializer<br/>
`BootstrapApplicationListener$AncestorInitializer`中保存了之前`BootstrapApplicationListener`生成的`ConfigurableApplicationContext`，放在`BootstrapApplicationListener.AncestorInitializer.parent`属性中。

  `AncestorInitializer`会拿到传入的`ConfigurableApplicationContext`的根context，
  如果根context的`Environment`中包含了名为"defaultProperties"的property source，会尝试将这个property source拆开重新放回去。（在`BootstrapApplicationListener.AncestorInitializer.reorderSources()`方法中）

  `BootstrapApplicationListener`生成的`AnnotationConfigApplicationContext`被设置为了当前`ApplicationContext`的parent；在当前`BeanFactory`中设置parent BeanFactory。并且将parent ApplicationContext的`ConfigurableEnvironment`和当前Application Context的`ConfigurableEnvironment`合并，具体操作有三条：
    - 将parent ConfigurableEnvironment中，当前 ConfigurableEnvironment中没有的property source加入进来，放在最后。
    - 将parent active profiles加入到当前 ConfigurableEnvironment的active profiles中。
    - 如果parent default profiles存在，则从当前 ConfigurableEnvironment 的default profiles中移除名为`AbstractEnvironment.RESERVED_DEFAULT_PROFILE_NAME`的profile，将parent default activeProfiles加入进来。

  最后在当前`ApplicationContext`中增加了一个`ParentContextApplicationContextInitializer.EventPublisher`。这个`ApplicationListener`似乎是用来在`ApplicationContext`刷新之后用来广播`ParentContextAvailableEvent`消息的，以告知其他listener 当前`ApplicationContext`已可用，并且有一个parent context。_ApplicationContextInitializer for setting the parent context. Also publishes ParentContextApplicationContextInitializer.ParentContextAvailableEvent when the context is refreshed to signal to other listeners that the context is available and has a parent._

2. BootstrapApplicationListener$DelegatingEnvironmentDecryptApplicationInitializer<br/>
> `DelegatingEnvironmentDecryptApplicationInitializer`: A special initializer designed to run before the property source bootstrap and decrypt any properties needed there (e.g. URL of config server).
>
> `EnvironmentDecryptApplicationInitializer`: Decrypt properties from the environment and insert them with high priority so they override the encrypted values.

  `DelegatingEnvironmentDecryptApplicationInitializer`中包含了一个`EnvironmentDecryptApplicationInitializer`，实际上`ConfigurableApplicationContext`是委托给后者处理的。具体功能参考[《SpringBoot应用配置项加密》][SpringBoot应用配置项加密]。如果存在加密的配置项被解密，则还会在parent ApplicationContext中发送一个`EnvironmentChangeEvent`消息。

3. PropertySourceBootstrapConfiguration$$EnhancerBySpringCGLIB$$64c5bba1<br/>
没看明白这是个什么对象，实际调用的方法是`PropertySourceBootstrapConfiguration.initialize()`。会调用`PropertySourceLocator`实例，加载property sources，保存到一个名为"bootstrapProperties"的`CompositePropertySource`中，并设置到`ConfigurableEnvironment`中。

4. EnvironmentDecryptApplicationInitializer<br/>
跟`BootstrapApplicationListener$DelegatingEnvironmentDecryptApplicationInitializer`的作用类似。这里也有可能在parent ApplicationContext中发送`EnvironmentChangeEvent`消息。

5. DelegatingApplicationContextInitializer<br/>
根据`context.initializer.classes`配置项配置的类名加载委托`ApplicationContextInitializer`，将`ConfigurableApplicationContext`交给委托类初始化。

6. ContextIdApplicationContextInitializer<br/>
根据某些配置项生成ApplicationContext ID。

7. ConfigurationWarningsApplicationContextInitializer<br/>
> ApplicationContextInitializer to report warnings for common misconfiguration mistakes.

  用于检查配置错误。在当前Application Context中增加一个`ConfigurationWarningsPostProcessor`。

8. ServerPortInfoApplicationContextInitializer<br/>
在`ConfigurableApplicationContext`中加入一个`ApplicationListener`监听`EmbeddedServletContainerInitializedEvent`，用于向`Environment`设置`EmbeddedServletContainer`实际监听的端口号。

9. SharedMetadataReaderFactoryContextInitializer
> ApplicationContextInitializer to create a shared CachingMetadataReaderFactory between the ConfigurationClassPostProcessor and Spring Boot.

  实际作用还没弄明白，看注释跟Spring Bean加载有关系。

10. AutoConfigurationReportLoggingInitializer<br/>
>ApplicationContextInitializer that writes the ConditionEvaluationReport to the log. Reports are logged at the DEBUG level unless there was a problem, in which case they are the INFO level is used.
This initializer is not intended to be shared across multiple application context instances.

  监听`ApplicationEvent`，打印auto configuration相关的信息。

##### 1.2.4.2.4 contextPrepared

`EventPublishingRunListener.contextPrepared()`方法中没有实现逻辑。

##### 1.2.4.2.5 loadSource
入口在 SpringApplication.java 365-367行。

根据source加载bean。

##### 1.2.4.2.6 contextLoaded
> Called once the application context has been loaded but before it has been refreshed.

涉及如下`ApplicationListener`：
1. BootstrapApplicationListener
2. LoggingSystemShutdownListener
3. ConfigFileApplicationListener
4. AnsiOutputApplicationListener
5. LoggingApplicationListener
6. ClasspathLoggingApplicationListener
7. BackgroundPreinitializer
8. RestartListener
9. DelegatingApplicationListener
10. ParentContextCloserApplicationListener（ApplicationContextAware）
11. ClearCachesApplicationListener
12. FileEncodingApplicationListener
13. LiquibaseServiceLocatorApplicationListener

以上`ApplicationListener`都会加入到当前`ApplicationContext`中，`ApplicationContextAware`类型的Listener还会被调用`setApplicationContext()`方法。

设置完毕后，会发送`ApplicationPreparedEvent`事件。能够接收该事件的Listener是：
1. ConfigFileApplicationListener<br/>
向 ApplicationContext 中增加一个`PropertySourceOrderingPostProcessor`，这个processor可能会重排列property sources。

2. LoggingApplicationListener<br/>
检查 ApplicationContext 中有没有一个名为"springBootLoggingSystem"的bean，如果没有则将自己的`loggingSystem`属性注册进去作为"springBootLoggingSystem"。

3. RestartListener<br/>
将当前 ApplicationContext 的引用保存下来。

4. DelegatingApplicationListener<br/>
将`ApplicationPreparedEvent`发送给代理 Listener 处理。debug过程中发现没有代理，所以没触发动作。

### 1.2.5 refreshContext

入口在 SpringApplication.java 316行。

> Load or refresh the persistent representation of the configuration, which might an XML file, properties file, or relational database schema.

#### 1.2.5.1 prepareRefresh

- initPropertySources<br/>
清空 ApplicationContext 中 `scanner` 的缓存。
初始化`Environment`中占位的property source。（AbstractApplicationContext.java 586行）例如，_Replace Servlet-based stub property sources with actual instances populated with the given servletContext and servletConfig objects._
- validateRequiredProperties<br/>
_Validate that each of the properties specified by setRequiredProperties is present and resolves to a non-null value._
- _Allow for the collection of early ApplicationEvents, to be published once the multicaster is available..._ 初始化`earlyApplicationEvents`用来保存`ApplicationEvent`，以便之后再将之前接收到的，并且没有发出去的事件消息发送出去。

#### 1.2.5.2 obtainFreshBeanFactory
> Tell the subclass to refresh the internal bean factory.

主要操作是`refreshBeanFactory()`方法<br/>
> Subclasses must implement this method to perform the actual configuration load. The method is invoked by refresh() before any other initialization work.
>
> A subclass will either create a new bean factory and hold a reference to it, or return a single BeanFactory instance that it holds. In the latter case, it will usually throw an IllegalStateException if refreshing the context more than once.

刷新`BeanFactory`并返回它的引用。

#### 1.2.5.3 prepareBeanFactory
> Configure the factory's standard context characteristics, such as the context's ClassLoader and post-processors.

这个方法是配置 Bean 对象加载的。

#### 1.2.5.4 postProcessBeanFactory
> Allows post-processing of the bean factory in context subclasses.

这里注册了`ServletContextAwareProcessor`。

#### 1.2.5.5 invokeBeanFactoryPostProcessors
> Instantiate and invoke all registered BeanFactoryPostProcessor beans, respecting explicit order if given.
>
>Must be called before singleton instantiation.

从传入的参数`beanFactoryPostProcessors`中挑出`BeanDefinitionRegistryPostProcessor`类型的processor，调用`postProcessBeanDefinitionRegistry()`方法，此方法的注释： _"Modify the application context's internal bean definition registry after its standard initialization. All regular bean definitions will have been loaded, but no beans will have been instantiated yet. This allows for adding further bean definitions before the next post-processing phase kicks in."_

按照`PriorityOrdered`,`Ordered`,其他，的顺序排列`BeanDefinitionRegistryPostProcessor`，调用`postProcessBeanDefinitionRegistry()`方法。

_这里没看明白为什么上面分两批调用`postProcessBeanDefinitionRegistry()`方法。_

PostProcessorRegistrationDelegate.java 126行，触发了`ConfigFileApplicationListener`的`postProcessBeanFactory()`方法，该方法调用`reorderSources()`，将`ConfigurableEnvironment`中的property source做了处理，把名为"applicationConfigurationProperties"的property source中包含的子property source展开插入到`ConfigurableEnvironment`中，位置和优先级保持不变，并将"applicationConfigurationProperties" property source本身移除。<br/>
即，原先保存在"applicationConfigurationProperties"中的加载自各个配置文件的property source，现在直接存放于`ConfigurableEnvironment`中。<br/>
最后，将名为"defaultProperties"的property source挪到了`ConfigurableEnvironment`的最低优先级位置。

之前触发的都是`BeanDefinitionRegistryPostProcessor`类型的BeanFactoryPostProcessor，接下来触发的是其他的BeanFactoryPostProcessor（注释里面称之为"regular BeanPostProcessors"），也是按照`PriorityOrdered`,`Ordered`,其他，的顺序排列调用。关键的`BeanFactoryPostProcessor`包括：

- **`ConfigurationSpringInitializer`，取出ServiceComb的配置回合到Spring中。**

  参考`PropertySourcesPlaceholderConfigurer`，我们这里也应该可以继承`EnvironmentAware`来事先拿到`ConfigurableEnvironment`。

- `PropertySourcesPlaceholderConfigurer`，解析bean 实例定义中的占位符，尝试用配置项去替换。这个类具有`PriorityOrdered`中的最低优先级。

  根据其`postProcessBeanFactory()`方法的注释来看，Spring自己应该是不管之后再增加的property source的。

  > Merge, convert and process properties against the given bean factory.
  Processing occurs by replacing ${...} placeholders in bean definitions by resolving each against this configurer's set of PropertySources, which includes:
  > - all environment property sources, if an Environment is present
  > - merged local properties, if any have been specified
  > - any property sources set by calling setPropertySources
  >
  > If setPropertySources is called, environment and local properties will be ignored. This method is designed to give the user fine-grained control over property sources, and once set, the configurer makes no assumptions about adding additional sources.

- `LastPropertyPlaceholderConfigurer`，实现了`Ordered`接口，最低优先级，检查bean定义中是否还存在需要解析的占位符，若存在则抛出异常。

- `ConfigurationPropertiesBindingPostProcessor`作为"nonOrderedPostProcessor"，在这之后被调用，其作用为 _"BeanPostProcessor to bind PropertySources to beans annotated with ConfigurationProperties"_ ，将配置值设置到`@ConfigurationProperties`注解标记的bean实例配置中。参考[《Guide to @ConfigurationProperties in Spring Boot》][configuration-properties-in-spring-boot]。

#### 1.2.5.6 registerBeanPostProcessors
> 入口在`AbstractApplicationContext.registerBeanPostProcessors()`方法。
>
> Instantiate and invoke all registered BeanPostProcessor beans, respecting explicit order if given.
>
> Must be called before any instantiation of application beans.

调用`BeanFactory`的`addBeanPostProcessor()`方法，将`BeanPostProcessor`注册到`BeanFactory`中，注册顺序为`PriorityOrdered` -> `Ordered` -> 其他 -> 重注册"internalPostProcessors"（筛选出类型为`MergedBeanDefinitionPostProcessor`的processor组成） -> 增加一个`ApplicationListenerDetector`。

#### 1.2.5.7 onRefresh
**在这里创建了`EmbeddedServletContainer`**，入口在`EmbeddedWebApplicationContext.createEmbeddedServletContainer()`方法中。

在`EmbeddedWebApplicationContext.createEmbeddedServletContainer()`方法的结尾调用`initPropertySources()`方法,
根据内部调用方法的注释， _"Replace Servlet-based stub property sources with actual instances populated with the given servletContext and servletConfig objects.
This method is idempotent with respect to the fact it may be called any number of times but will perform replacement of stub property sources with their corresponding actual property sources once and only once."_ 这里会尝试将"servletContextInitParams"和"servletConfigInitParams" property source 桩替换为实际的property source。

#### 1.2.5.8 registerListeners
此时，当前`ApplicationContext`中的`applicationEventMulticaster`已设置，向其中注册`ApplicationListener`，并将前面初始化的`earlyApplicationEvents`保存的事件消息通过`ApplicationContext.applicationEventMulticaster`发送出去。

#### 1.2.5.9 finishBeanFactoryInitialization
> Finish the initialization of this context's bean factory, initializing all remaining singleton beans.

该方法中调用BeanFactory的`freezeConfiguration()`方法，根据这个方法的注释， _"Freeze all bean definitions, signalling that the registered bean definitions will not be modified or post-processed any further. This allows the factory to aggressively cache bean definition metadata."_ ，之后不会再有bean 对象配置的更改了。

**然后调用`beanFactory.preInstantiateSingletons()`，将所有非懒加载的singleton bean对象实例化出来。**<br/>
**此时也触发了`ArchaiusAutoConfiguration`的初始化，将Spring的配置合入到了 Archaius 中。** 具体位置在`ArchaiusAutoConfiguration.addArchaiusConfiguration()`方法。<br/>
如果引入了`ConfigurationSpringInitializer`类，则会在更早的地方触发`DynamicProperty`:<br/>
在`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`方法中会实例化`ConfigurationSpringInitializer`对象，其构造方法会调用`ConfigUtil.installDynamicConfig()`方法，触发`ConfigurationManager.install(dynamicConfig)`，在这个时候SpringBoot的配置就已经合入到Archiaus中了。
<br/>
**TODO: 还需要检查配置项优先级是否符合预期**

#### 1.2.5.10 finishRefresh
> Finish the refresh of this context, invoking the LifecycleProcessor's onRefresh() method and publishing the ContextRefreshedEvent.

1. initLifecycleProcessor();
2. getLifecycleProcessor().onRefresh();
3. publishEvent(new ContextRefreshedEvent(this));
4. LiveBeansView.registerApplicationContext(this);
5. publishEvent(new EmbeddedServletContainerInitializedEvent(this, localContainer));

内部做的事情主要是：
- 在当前`ApplicationContext`中设置一个`LifecycleProcessor`实例，如果没有现成的`LifecycleProcessor`实例就创建一个`DefaultLifecycleProcessor`设置进去，其作用参考[《Spring中Bean的生命周期》][Spring中Bean的生命周期]。
- 在 BeanFactory 中查找实现了`SmartLifecycle`接口的singleton bean，将这些 bean 以 phase 的维度组织成 group ，并启动（`DefaultLifecycleProcessor.startBeans()`方法）。
- 发送一个`ContextRefreshedEvent`，在自己（`AnnotationConfigEmbeddedWebApplicationContext`）和parent（`AnnotationConfigApplicationContext`）中都发送一遍（此后发送的消息都会在parent context里面也发一遍）。
- 调用`LiveBeansView.registerApplicationContext(this)`，作用暂时不明。
- _Starts the embedded servlet container. Calling this method on an already started container has no effect._ 启动了内嵌的 web 容器（开始监听端口）。
- 发布`EmbeddedServletContainerInitializedEvent`消息。

#### 1.2.5.11 registerShutdownHook
> Register a shutdown hook with the JVM runtime, closing this context on JVM shutdown unless it has already been closed at that time.
This method can be called multiple times. Only one shutdown hook (at max) will be registered for each context instance.

在当前`ConfigurableApplicationContext`中注册一个 shutdown hook。

### 1.2.6 listeners.finished
_unfinished_

# 附加说明

## <div id="BootstrapApplicationListener">BootstrapApplicationListener生成新的Spring Context</div>

### <div id="BootstrapApplicationListener.applicationSource">1. Application Source</div>
BootstrapApplicationListener新生成的SpringApplication里面包含了4个source:
  0. org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration
  1. org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration
  2. org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration
  3. org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration

### <div id="BootstrapApplicationListener.applicationInitialize">2. initialize()</div>

第一次生成SpringApplication时，直到`SpringApplication.prepareEnvironment()`方法中调用`getOrCreateEnvironment()`方法才会初始化一个`StandardServletEnvironment`。而这一次初始化在`BootstrapApplicationListener.bootstrapServiceContext()`方法中就set了一个`StandardEnvironment`进去，这个StandardEnvironment初始化时获取的所有property source都会被移除，然后首先加入一个名为"bootstrap"的property source，里面描述bootstrap配置文件的文件名和存放位置，再将父级`ConfigurableEnvironment`中的property source依次加入进去。

### 3. run()

#### 3.1. started事件
能够监听ApplicationStartedEvent的Listener仍然是以下四个：
0. ConfigFileApplicationListener（started事件仍然不会触发动作）
1. LoggingApplicationListener
2. DelegatingApplicationListener
3. LiquibaseServiceLocatorApplicationListener

#### 3.2. prepareEnvironment

##### 3.2.1 ConfigurableEnvironment设置

ConfigurableEnvironment已经set进去了，不需要再构造，[其property source为一个名为"bootstrap"的property source加上父级ConfigurableEnvironment的property source](#BootstrapApplicationListener.applicationInitialize)，但仍然会检查active profile。<br/>
***BootstrapApplicationListener新生成的SpringApplication读取的配置文件名为bootstrap。***

广播ApplicationEnvironmentPreparedEvent，监听的Listener有：
0. BootstrapApplicationListener（这一次由于检查到environment的property sources中有一个名为bootstrap的，所以会跳过去）
1. LoggingSystemShutdownListener
2. ConfigFileApplicationListener
3. AnsiOutputApplicationListener（目前看到的，4~9里没有写配置）
4. LoggingApplicationListener
5. ClasspathLoggingApplicationListener
6. BackgroundPreinitializer<br/>触发了一些耗时任务的预初始化，包括：
  1. MessageConverterInitializer
  1. MBeanFactoryInitializer
  1. ValidationInitializer
  1. JacksonInitializer
  1. ConversionServiceInitializer
7. DelegatingApplicationListener<br/>
根据`context.listener.classes`获取配置项（委托类的名字），加载`ApplicationListener`委托类，将event传给这些委托类处理。
8. FileEncodingApplicationListener<br/>
检查System Properties中的`file.encoding`值和配置项中的`mandatoryFileEncoding`值（`mandatoryFileEncoding`的属性名是模糊匹配的）是否匹配，如果不匹配则抛异常终止启动过程。

##### 3.2.2 ConfigFileApplicationListener
`ConfigFileApplicationListener`加载了`EnvironmentPostProcessor`，包括：
0. SpringApplicationJsonEnvironmentPostProcessor
1. HostInfoEnvironmentPostProcessor
2. CloudFoundryVcapEnvironmentPostProcessor
3. ConfigFileApplicationListener<br/>
Spring Boot在这里加载配置文件，操作内容包括：
  1. 增加一个`RandomValuePropertySource`，优先级低于"systemEnvironment"（ConfigFileApplicationListener.java 210行）
  2. 加载名为"bootstrap"的配置文件，包括active profile（ConfigFileApplicationListener.java 212行）
  3. 设置`spring.beaninfo.ignore`配置项，决定是否跳过`BeanInfo`类加载（ConfigFileApplicationListener.java 182行）
  还不知道是做什么的。
  4. 将`ConfigurableEnvironment`绑定到`SpringApplication`（ConfigFileApplicationListener.java 183行）<br/>
  但是看代码看不出绑定的意思，真实效果不明。
4. SpringBootConfigMergeProcessor<br/>
这里取得SpringBoot的配置文件留待合入到ServiceComb中。<br/>
现在发现这个processor会触发两次，一次是`BootstrapApplicationListener`触发重新生成SpringApplication，一次是外层的SpringBoot启动。现在发现可以根据配置项`spring.config.name`或`spring.application.name`区分。
5. ServoEnvironmentPostProcessor<br/>
尝试设置一个Netflix servo的默认配置项`spring.aop.proxyTargetClass`。<br/>
如果ConfigurableEnvironment中包含一个名为`defaultProperties`的property source，且为MapPropertySource类型，则在其中设置`spring.aop.proxyTargetClass=true`。<br/>
`ServoEnvironmentPostProcessor`没有实现`Ordered`接口或使用`@Order`/`@Priority`注解，所以会排在后面，这里的配置项`SpringBootConfigMergeProcessor`拿不到。

`SpringBootConfigMergeProcessor`是实现了`Ordered`接口的最低优先级的`EnvironmentPostProcessor`。

#### 3.3. prepareContext

位于SpringApplication.java 312-314行。

##### 3.3.1 createApplicationContext

- SpringApplication.java 312行，创建了一个`org.springframework.context.annotation.AnnotationConfigApplicationContext`。`ApplicationContext`中默认初始化的`Environment`会被覆盖设置为上一步中的`ConfigurableEnvironment`。

- SpringApplication.java 349行，postProcessApplicationContext。加载`beanNameGenerator`，在`ApplicationContext`中设置`resourceLoader`，debug过程中发现这两个都是null，所以没有设置。

- SpringApplication.java 350行，applyInitializers。设置`ApplicationContextInitializer`，***根据网上查到的资料，`ApplicationContextInitializer`是Spring容器刷新之前执行的回调函数，对`ConfigurableApplicationContext`进行操作。***<br/>
debug过程中发现加载的有：
  0. DelegatingApplicationContextInitializer<br/>
  根据`context.initializer.classes`配置项配置的类名加载委托`ApplicationContextInitializer`，将`ConfigurableApplicationContext`交给委托类初始化。
  1. ContextIdApplicationContextInitializer<br/>
  根据某些配置项生成ApplicationContext ID。
  2. ConfigurationWarningsApplicationContextInitializer<br/>
  ApplicationContextInitializer to report warnings for common misconfiguration mistakes. 用于检查配置错误。
  3. ServerPortInfoApplicationContextInitializer<br/>
  在`ConfigurableApplicationContext`中加入一个`ApplicationListener`监听`EmbeddedServletContainerInitializedEvent`，用于向`Environment`设置`EmbeddedServletContainer`实际监听的端口号。
  4. SharedMetadataReaderFactoryContextInitializer（实际作用还没弄明白）
  5. AutoConfigurationReportLoggingInitializer<br/>
监听`ApplicationEvent`，打印auto configuration相关的信息。

- SpringApplication.java 351行，调用`listeners.contextPrepared()`，但是`EventPublishingRunListener.contextPrepared()`方法没有实现内容，所以没有触发任何操作。

##### 3.3.2 load source
> 这里的source是[`BootstrapApplicationListener`构建`SpringApplication`时设置进去的source](#BootstrapApplicationListener.applicationSource)。

入口位于SpringApplication.java 367行。

貌似是加载和注册Bean的，但是里面的代码没完全看明白。<br/>
大致上做的事情是扫描设置到`SpringApplication`中的四个source，这四个source都有`@Configuration`注解，于是将这些source注册到一个`AnnotatedBeanDefinitionReader`实例中。

##### 3.3.3 contextLoaded

入口位于SpringApplication.java 368行。

将当前`SpringApplication`中的`ApplicationListener`设置到当前`ConfigurableApplicationContext`中。（EventPublishingRunListener.java `contextLoaded()`方法的for循环）

发送`ApplicationPreparedEvent`，对应的Listener有：
0. ConfigFileApplicationListener<br/>
在当前`ConfigurableApplicationContext`中加入了一个`PropertySourceOrderingPostProcessor`
1. LoggingApplicationListener
2. RestartListener<br/>
A listener that stores enough information about an application as it starts, to be able to restart it later if needed. 在响应这个事件的操作中只做了一些设置。
3. DelegatingApplicationListener<br/>
前文已经提到`DelegatingApplicationListener`会加载一些委托`ApplicationListener`，这里会将事件传给委托类（如果存在委托类的话）。

#### 3.4. refreshContext

> 入口在SpringApplication.java 316行。

##### 3.4.1 prepareRefresh

0. `ClassPathBeanDefinitionScanner`清空cache。
1. initPropertySources
2. Environment.validateRequiredProperties

##### 3.4.2 obtainFreshBeanFactory

0. refreshBeanFactory

#### 3.5. afterRefresh

#### 3.6. listeners.finished

------------------------------
后面的先略过去，先把主流程跑完一遍。
------------------------------

# References

1. [SpringBoot源码分析之SpringBoot的启动过程][SpringBoot源码分析之SpringBoot的启动过程]
2. [Spring Boot启动流程详解][Spring Boot启动流程详解]
3. [SpringBoot应用配置项加密][SpringBoot应用配置项加密]
4. [什么是FactoryBean][what-s-a-factorybean]
5. [ConfigurationProperties注解使用][configuration-properties-in-spring-boot]
6. [Spring中Bean的生命周期][Spring中Bean的生命周期]

[SpringBoot源码分析之SpringBoot的启动过程]: http://fangjian0423.github.io/2017/04/30/springboot-startup-analysis/ "SpringBoot源码分析之SpringBoot的启动过程"
[Spring Boot启动流程详解]: http://zhaox.github.io/java/2016/03/22/spring-boot-start-flow "Spring Boot启动流程详解"
[SpringBoot应用配置项加密]: http://isouth.org/archives/364.html "SpringBoot应用配置项加密"
[what-s-a-factorybean]: https://spring.io/blog/2011/08/09/what-s-a-factorybean "What's a FactoryBean?"
[configuration-properties-in-spring-boot]: http://www.baeldung.com/configuration-properties-in-spring-boot "Guide to @ConfigurationProperties in Spring Boot"
[Spring中Bean的生命周期]: https://blog.csdn.net/ethanwhite/article/details/51533299 "Spring核心技术（六）——Spring中Bean的生命周期 - CSDN博客"
