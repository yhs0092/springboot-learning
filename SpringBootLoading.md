## SpringApplication.initialize()

### deduceWebEnvironment()
是否是web程序？（如果SpringApplication.WEB_ENVIRONMENT_CLASSES中的类可以加载则是web程序）

### 加载org.springframework.context.ApplicationContextInitializer

### 加载org.springframework.context.ApplicationListener

### 找出启动的Main类

## SpringApplication.run()
  > - SpringApplication.run()方法中初始化了一个SpringApplicationRunListeners对象，SpringApplication.run()方法执行到不同阶段时会调用SpringApplicationRunListeners的不同方法去触发其内部的SpringApplicationRunListener对象的对应方法，以广播不同的ApplicationEvent。
  > - 当前SpringApplicationRunListener的实现类只有EventPublishingRunListener，其内部使用SimpleApplicationEventMulticaster广播事件。
  >
  > ![](http://7x2wh6.com1.z0.glb.clouddn.com/startup2.jpg "SpringApplicationRunListeners")


### SpringApplicationRunListeners的关键时间点

- started  
  广播ApplicationStartedEvent。对应Listener为  
  1. ConfigFileApplicationListener（但是此处广播的事件类型ApplicationStartedEvent不会触发ConfigFileApplicationListener的动作）
  2. LoggingApplicationListener
  3. DelegatingApplicationListener
  4. LiquibaseServiceLocatorApplicationListener
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

## <div id="BootstrapApplicationListener">BootstrapApplicationListener生成新的Spring Context</div>

### 1. Application Source
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
0. SpringApplicationJsonEnvironmentPostProcessor<br/>从`spring.application.json`或`SPRING_APPLICATION_JSON`加载json串格式的配置项，将其增加到ConfigurableEnvironment的property sources中，新增property source的名字为`spring.application.json`，优先级高于`jndiProperties`和`systemProperties`。参考资料见[Spring Boot 1.4.5](https://docs.spring.io/spring-boot/docs/1.4.5.RELEASE/reference/html/boot-features-external-config.html)和[Spring Boot 2.0](https://juejin.im/entry/5a4b33e6518825258227bfbe)。
1. HostInfoEnvironmentPostProcessor<br/>
获取本机的网络相关的信息，生成一个名为`springCloudClientHostInfo`的property source加入到ConfigurableEnvironment中，property source的优先级当前最低。
2. CloudFoundryVcapEnvironmentPostProcessor<br/>
貌似是用来处理Cloud Foundry的配置项的（存疑）。<br/>
如果当前ConfigurableEnvironment中存在名为`VCAP_APPLICATION`或`VCAP_SERVICES`的配置项则生效，将`VCAP_APPLICATION`和`VCAP_SERVICES`中的json串配置扁平化，分别加上`vcap.application.`和`vcap.services.`前缀，放入到名为`vcap`的property source中，置入ConfigurableEnvironment，该property source的优先级当前最高。
3. ConfigFileApplicationListener<br/>
Spring Boot在这里加载配置文件，操作内容包括：
  1. 增加一个`RandomValuePropertySource`（ConfigFileApplicationListener.java 210行）
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
