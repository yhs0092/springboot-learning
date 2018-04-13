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
4. LiquibaseServiceLocatorApplicationListener （作用不明）

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
广播`ApplicationEnvironmentPreparedEvent`，相关的`ApplicationListener`有：
0. BootstrapApplicationListener<br/>
特定条件下，已知是Spring Cloud场景下，会从bootstrap.yaml文件加载配置。 org.springframework.cloud.bootstrap.BootstrapApplicationListener会重新拉起一个Spring Context，并且重新跑一遍整个启动过程，此过程分析见[BootstrapApplicationListener生成新的Spring Context](#BootstrapApplicationListener "BootstrapApplicationListener生成新的Spring Context")。<br/>
***TODO: 还有`bootstrapProperties`回合以及`DelegatingEnvironmentDecryptApplicationInitializer`没有分析***
1. LoggingSystemShutdownListener<br/>
<blockquote>Cleans up the logging system immediately after the bootstrap context is created on startup. Logging will go dark until the ConfigFileApplicationListener fires, but this is the price we pay for that listener being able to adjust the log levels according to what it finds in its own configuration.</blockquote>
按照`LoggingSystemShutdownListener`的注释，应该是暂时关闭了日志记录。
2. ConfigFileApplicationListener<br/>
读取配置文件，见[ConfigFileApplicationListener 处理 ApplicationEnvironmentPreparedEvent](#ConfigFileApplicationListener处理ApplicationEnvironmentPreparedEvent "ConfigFileApplicationListener 处理 ApplicationEnvironmentPreparedEvent")。
3. AnsiOutputApplicationListener
4. LoggingApplicationListener
5. ClasspathLoggingApplicationListener
6. BackgroundPreinitializer
7. DelegatingApplicationListener
8. FileEncodingApplicationListener

##### <div id="ConfigFileApplicationListener处理ApplicationEnvironmentPreparedEvent">1.2.3.2.1 ConfigFileApplicationListener 处理 ApplicationEnvironmentPreparedEvent</div>
加载`EnvironmentPostProcessor`，包含：
0. SpringApplicationJsonEnvironmentPostProcessor<br/>
从`spring.application.json`或`SPRING_APPLICATION_JSON`加载json串格式的配置项，将其增加到ConfigurableEnvironment的property sources中，新增property source的名字为`spring.application.json`，优先级高于`jndiProperties`和`systemProperties`（没有这两个时会放在最前面）。参考资料见[Spring Boot 1.4.5](https://docs.spring.io/spring-boot/docs/1.4.5.RELEASE/reference/html/boot-features-external-config.html)和[Spring Boot 2.0](https://juejin.im/entry/5a4b33e6518825258227bfbe)。
1. HostInfoEnvironmentPostProcessor<br/>
获取本机的网络相关的信息，生成一个名为`springCloudClientHostInfo`的property source加入到ConfigurableEnvironment中，property source的优先级为当前最低。
2. CloudFoundryVcapEnvironmentPostProcessor<br/>
貌似是用来处理Cloud Foundry的配置项的（存疑）。<br/>
如果当前ConfigurableEnvironment中存在名为`VCAP_APPLICATION`或`VCAP_SERVICES`的配置项则生效，将`VCAP_APPLICATION`和`VCAP_SERVICES`中的json串配置扁平化，分别加上`vcap.application.`和`vcap.services.`前缀，放入到名为`vcap`的property source中，置入ConfigurableEnvironment，该property source的优先级仅次于"commandLineArgs"。
3. ConfigFileApplicationListener<br/>
Spring Boot在这里加载配置文件，操作内容包括：
  1. 增加一个`RandomValuePropertySource`，优先级低于"systemEnvironment"（ConfigFileApplicationListener.java 210行）
  2. 加载property source，设置active profiles
    - 根据"spring.profiles.active"属性获取active profiles， active profiles 的优先级是后面的高于前面的（这与property sources相反，因为后面根据active profile加载property source时是用`addFirst()`方法加入到property sources中的）。 如果没有 active profile， 则增加一个default profile。然后增加了一个`null` profile。<br/>
    ***TODO: 这里的`default`还有`null` profile究竟是如何表现的还需要实验验证***
    - ConfigFileApplicationListener.java 377行，根据profile、文件名、路径、文件扩展名加载配置文件。加载成的property source在这个时候还不会直接加入到`ConfigurableEnvironment.propertySources`中，而是保存在当前用于加载配置文件的`PropertySourcesLoader`实例里。
    - ConfigFileApplicationListener.java 470行，读取到配置文件后还会再一次扫描active profile，如果读取到了就会将`ConfigurableEnvironment.activeProfiles`清空并set进去，然后将`ConfigFileApplicationListener.activatedProfiles`设置为`true`以保证只会设置一次profile（方法`ConfigFileApplicationListener.maybeActivateProfiles()`会检查`activatedProfiles`）。
    - 设置active profile之后会检查一次是否配置了`spring.profiles.include`属性，若配置了则将其中指定的 profile 加入到`ConfigFileApplicationListener`中，并设置到`ConfigurableEnvironment.activeProfiles`中（先清空再设置的）。
    - ConfigFileApplicationListener.java 384行，经过反复加载配置文件之后（文件位置、文件名、profile、文件扩展名这几个维度），将`PropertySourcesLoader.propertySources`中保存的property sources加入到`ConfigurableEnvironment`中。
    - ***TODO: 有条件进一步弄清楚`ConfigFileApplicationListener.loadIntoGroup()`方法的三个参数各有什么意义***<br/>
    当`ConfigFileApplicationListener.load()`方法的profile不为null时，为什么要分三个阶段调用`loadIntoGroup()`方法。<br/>
    _貌似这种反复加载是因为同一个配置文件中也可以配置多个profile的属性，待验证。参考[Multi-profile YAML documents](https://docs.spring.io/spring-boot/docs/1.4.5.RELEASE/reference/htmlsingle/#boot-features-external-config-multi-profile-yaml "Multi-profile YAML documents")。_
  3. 从System property中读取名为"spring.beaninfo.ignore"的属性，如果没有则从`ConfigurableEnvironment`中解析出对应的配置项（使用`RelaxedPropertyResolver`），默认值为`true`，并设置到System property中。<br/>
  根据查的资料，"spring.beaninfo.ignore"配置项似乎决定了是否跳过BeanInfo类加载。
  4. Bind the environment to the SpringApplication. 具体产生了什么效果还不明确。
4. SpringBootConfigMergeProcessor
5. ServoEnvironmentPostProcessor

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

1. [SpringBoot源码分析之SpringBoot的启动过程](http://fangjian0423.github.io/2017/04/30/springboot-startup-analysis/ "SpringBoot源码分析之SpringBoot的启动过程")
