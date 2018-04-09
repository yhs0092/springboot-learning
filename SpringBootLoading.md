## SpringApplication.initialize()

### deduceWebEnvironment()
是否是web程序？（如果SpringApplication.WEB_ENVIRONMENT_CLASSES中的类可以加载则是web程序）

### 加载org.springframework.context.ApplicationContextInitializer

### 加载org.springframework.context.ApplicationListener

### 找出启动的Main类

## SpringApplication.run()
  > - SpringApplication.run()方法中初始化了一个SpringApplicationRunListeners对象，SpringApplication.run()方法执行到不同阶段时会调用SpringApplicationRunListeners的不同方法去触发其内部的SpringApplicationRunListener对象的对应方法，以广播不同的ApplicationEvent。
  > - 当前SpringApplicationRunListener的实现类只有EventPublishingRunListener，其内部使用SimpleApplicationEventMulticaster广播事件。
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
    0. BootstrapApplicationListener （特定条件下，已知是Spring Cloud场景下，会从bootstrap.yaml文件加载配置。 org.springframework.cloud.bootstrap.BootstrapApplicationListener会重新拉起一个Spring Context，并且重新跑一遍整个启动过程）
      - BootstrapApplicationListener新生成的SpringApplication里面包含了4个source:  
        0. org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration
        1. org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration
        2. org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration
        3. org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration
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