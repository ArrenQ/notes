SEO
=========================
# 模块

### spring 加载顺序， 所有工厂文件加载的对象可以通过查看工厂文件得知
1. SpringApplication 构造器通过 工厂文件 加载 ApplicationContextInitializer (spring framework 上下文初始化)
    默认实现：
        springboot-autoconfig
            org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,
            org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
        springboot 
            org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,
            org.springframework.boot.context.ContextIdApplicationContextInitializer,
            org.springframework.boot.context.config.DelegatingApplicationContextInitializer,
            org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
2. SpringApplication 构造器通过 工厂文件 加载 ApplicationListener (spring framework 监听器)
        springboot-autoconfig
            org.springframework.boot.autoconfigure.BackgroundPreinitializer
        springboot
            org.springframework.boot.ClearCachesApplicationListener,
            org.springframework.boot.builder.ParentContextCloserApplicationListener,
            org.springframework.boot.context.FileEncodingApplicationListener,
            org.springframework.boot.context.config.AnsiOutputApplicationListener,
            org.springframework.boot.context.config.ConfigFileApplicationListener,
            org.springframework.boot.context.config.DelegatingApplicationListener,
            org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,
            org.springframework.boot.context.logging.LoggingApplicationListener,
            org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
3. SpringApplication.run() 通过 工厂文件 加载 SpringApplicationRunListener (spring boot 监听器)  
    默认实现：springboot  -> EventPublishingRunListener 
    
### spring SpringApplicationRunListener 事件
| 监听方法              | 阶段说明                                                      | spring boot 起始版本  |
| starting()            | spring刚刚启动                                                |   1.0                 | 
| environmentPrepared() | ConfigurableEnvironment 准备就绪，可以对其进行调整            |   1.0                 |
| contextPrepared()     | ConfigurableApplicationContext 准备就绪，可以对其进行调整     |   1.0                 |
| contextLoaded()       | ConfigurableApplicationContext 已加载，但没有启动             |   1.0                 | 
| started()             | ConfigurableApplicationContext 已启动, spring bean加载完成    |   2.0                 |
| running()             | spring 应用已经运行                                           |   2.0                 |
| failed()              | spring 应用运行失败                                           |   2.0                 |

### 上述事件会通过 EventPublishingListener 转换为spring事件
| 监听方法              | 阶段说明                                                      | spring boot 起始版本  |
| starting()            | ApplicationStartingEvent                                      |   1.5                 | 
| environmentPrepared() | ApplicationEnvironmentPreparedEvent                           |   1.0                 |
| contextPrepared()     |                                                               |                       |
| contextLoaded()       | ApplicationPreparedEvent                                      |   1.0                 | 
| started()             | ApplicationStartedEvent                                       |   2.0                 |
| running()             | ApplicationReadyEven                                          |   2.0                 |
| failed()              | ApplicationFailedEvent                                        |   2.0                 |


### spring mvc DispatcherServlet 中使用的关键组件及其创建方式。
1. 获取 HandlerMapping  -> RequestMappingHandlerMapping，用于分析所有用户自定义的Controller与request的映射。
    该接口将根据request分析创建一个执行链对象（HandlerExecutionChain， 内涵controller方法和所有匹配的拦截器Interceptors）
2. 

@EnableMvc 会自动创建多个 HandlerMapping。
继承关系：DispatcherServlet ->  FrameworkServlet -> HttpServletBean -> HttpServlet
调用过程如下：
Tomcat -> 
    HttpServlet.init -> 
        HttpServletBean.init 重写 ->
             HttpServletBean.initServletBean ->
                FrameworkServlet.initServletBean 重写 ->
                    FrameworkServlet.initWebApplicationContext -> 
                        FrameworkServlet.onRefresh ->
                            DispatcherServlet.onRefresh 重写 -> 
                                DispatcherServlet.initStrategies ->
                                    DispatcherServlet.initMultipartResolver
                                    DispatcherServlet.initLocaleResolver
                                    DispatcherServlet.initThemeResolver
                                    DispatcherServlet.initHandlerMappings
                                    DispatcherServlet.initHandlerAdapters
                                    DispatcherServlet.initHandlerExceptionResolvers
                                    DispatcherServlet.initRequestToViewNameTranslator
                                    DispatcherServlet.initViewResolvers
                                    DispatcherServlet.initFlashMapManager
DispatcherServlet.initXXX说明：所有这些组件都会通过加载完成后的 ApplicationContext 获取Bean。
    有些组件，例如 HandlerMapping 的Bean没有定义，则会通过加载 DispatcherServlet.properties 配置来创建并加入到上下文中。
    
结论：某些如果springmvc自带的组件如果不能满足需要，则可以通过自己@Bean创建一个对象来代替。
    至于那些可以替代，可以通过查看 DispatcherServlet.properties 文件和 DispatcherServlet.initStrategies来查看
    
注意：DispatcherServlet.initHandlerMappings 里面会对bean进行排序，每次总是把 RequestMappingHandlerMapping放在最前面，
    原因是RequestMappingHandlerMapping 通过 @EnableWebMvc 加载 DelegatingWebMvcConfiguration 配置来创建的，
    从其父类的创建了多个 HandlerMapping，而在requestMappingHandlerMapping 方法创建中可以看到，RequestMappingHandlerMapping的order被设置为0(最前面)
注意2：@EnableWebMvc 加载的 DelegatingWebMvcConfiguration 配置内部会有一个 setConfigurers 方法自动注入所有实现 WebMvcConfigurer接口的对象
    然后内部加载各个组件（如 Interceptors 时）会调用这个接口来获取组件(似乎没有加入到容器中)。这样也是为什么实现  WebMvcConfigurer 也能创建注册组件的原因。

### 关于为什么spring mvc 可以不再使用web.xml 来加载自己的DispatcherServlet。  
        因为 servlet 3.0 以后的版本通过SPI提供一个容器启动后的回调接口，spring mvc对其进行了实现。
      
### springboot 相对于 spring mvc
1、springboot 使用 WebMvcAutoConfiguration 来替代 @EnableWebMvc ，
    而值得注意的是 WebMvcAutoConfiguration 中定义了注解 @ConditionalOnMissingBean(WebMvcConfigurationSupport.class);
    换句话说，如果springboot应用中，手动填写了@EnableWebMvc 则 WebMvcAutoConfiguration 里面的设定都无效。

 
### 加载顺序， 所有的spring容器管理的对象都会创建成为一个 BeanDefinition
#### springboot 加载bean的过程
1、创建 AnnotationConfigServletWebServerApplicationContext 时，构造器会创建 
        this.reader = new AnnotatedBeanDefinitionReader(this); // 内部会调用  AnnotationConfigUtils.registerAnnotationConfigProcessors(BeanDefinitionRegistry registry)
        this.scanner = new ClassPathBeanDefinitionScanner(this); // 内部会调用 registerDefaultFilters
2、调用 AnnotationConfigUtils.registerAnnotationConfigProcessors 时会加载如下Bean,并包装成RootBean
    
    ConfigurationClassPostProcessor -> BeanPostProcessor  (BeanDefinitionRegistryPostProcessor)   Ordered.LOWEST_PRECEDENCE
    AutowiredAnnotationBeanPostProcessor -> BeanPostProcessor    Ordered.LOWEST_PRECEDENCE - 2
    CommonAnnotationBeanPostProcessor -> BeanPostProcessor       Ordered.LOWEST_PRECEDENCE - 3  目的是为了实现 JSR-250
    //如果开启了 JPA
    PersistenceAnnotationBeanPostProcessor
 
    EventListenerMethodProcessor -> BeanFactoryPostProcessor
    DefaultEventListenerFactory  
    
3、 `ClassPathBeanDefinitionScanner` 添加了两个 `AnnotationTypeFilter` 
    
    分别为 
    javax.annotation.ManagedBean  实现 JSR-250
    javax.inject.Named 实现 JSR-330
    
4、 refresh() 中执行 postProcessBeanFactory，将继续添加某些 WebApplicationContextServletContextAwareProcessor 到BeanProcessor集合中，但不在BeanDefinitionMap中
5、 invokeBeanFactoryPostProcessors
    
    内部调用 processConfigBeanDefinitions
    找到符合条件的 spring 配置进行加载 
    判断条件为 ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)
    此时会找到用户的根配置对象，
    并调用 ConfigurationClassParser.parse 来解析 @Configuration 注解的对象，解析过程如下
        a. 判断配置文件是否是为 @Component （true） ,如果为true，则查找其是否存在 memberClass(递归处理任何成员（嵌套）类)
        b. 处理所有 @PropertySource annotations
        c. 处理 @ComponentScan annotations 获取所有扫描到的Bean，
            并对所有扫描到的Bean进行 ConfigurationClassParser.parse。 这里存在递归
        d. 处理所有 @import，并对所有import进行解析，这里存在递归
        e. 处理所有 @Bean methods
        f. 处理所有 interfaces上的默认方法
        g. 处理所有 父类
    因为存在递归，且@Bean methods相对于其他加载方式是最后加载的，所有本项目中 SpringMvcConfig 里面的对象是最后加载的。        
    
    
    SpringMvcConfig(用户入口配置)
    SharedMetadataReaderFactoryContextInitializer$SharedMetadataReaderFactoryBean  -> FactoryBean