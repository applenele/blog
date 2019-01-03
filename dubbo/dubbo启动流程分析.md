## dubbo 启动分析

### DubboComponentScan注入ServiceAnnotationBeanPostProcessor

注解@DubboComponentScan上有@Import(DubboComponentScanRegistrar.class)，DubboComponentScanRegistrar中registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);中向容器中注入了ServiceAnnotationBeanPostProcessor。



### DubboAutoConfiguration注入ServiceAnnotationBeanPostProcessor

在DubboAutoConfiguration  dubbo-springboot的自动初始化类中。在配置了dubbo.scan属性的时候注入了ServiceAnnotationBeanPostProcessor。

```java
@ConditionalOnProperty(name = BASE_PACKAGES_PROPERTY_NAME)
@ConditionalOnClass(ConfigurationPropertySources.class)
@Bean
public ServiceAnnotationBeanPostProcessor serviceAnnotationBeanPostProcessor(Environment environment) {
    Set<String> packagesToScan = environment.getProperty(BASE_PACKAGES_PROPERTY_NAME, Set.class, emptySet());
    return new ServiceAnnotationBeanPostProcessor(packagesToScan);
}

```



### ServiceAnnotationBeanPostProcessor

1. ServiceAnnotationBeanPostProcessor 类实现BeanDefinitionRegistryPostProcessor。
2. ServiceAnnotationBeanPostProcessor的postProcessBeanDefinitionRegistry方法中调用。
3. postProcessBeanDefinitionRegistry方法中调用registerServiceBeans方法。
4. 在registerServiceBeans方法中，扫描注解@Service注解的类。并将扫描结果以ServiceBean 作为rootBeanDefinition的beanDefinition注入到容器中。

```java
private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
        scanner.setBeanNameGenerator(beanNameGenerator);
       // 添加扫描的注解
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
        for (String packageToScan : packagesToScan) {

            // 扫描包
            scanner.scan(packageToScan);

          // 查找dubbo注解@service的所有beanDefinitionHolders，无论@componentscan是否扫描。
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
				// 遍历BeanDefinitionHolder
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    // 根据BeanDefinitionHolde，生成beanDefinition，注册到容器中，
                    // 这一步将ServiceBean 作为rootBeanDefinition，代表一个从配置源（XML，Java Config等）中生成的BeanDefinition
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }
            } else {

            }
        }
    }
```

```java
private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {

        Class<?> beanClass = resolveClass(beanDefinitionHolder);

        Service service = findAnnotation(beanClass, Service.class);

        Class<?> interfaceClass = resolveServiceInterfaceClass(beanClass, service);

        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();

        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, interfaceClass, annotatedServiceBeanName);

        // ServiceBean Bean name
        String beanName = generateServiceBeanName(service, interfaceClass, annotatedServiceBeanName);

        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { 
            // check duplicated candidate bean
            registry.registerBeanDefinition(beanName, serviceBeanDefinition);
        } else {
        }
    }
```

```java
private AbstractBeanDefinition buildServiceBeanDefinition(Service service, Class<?> interfaceClass,String annotatedServiceBeanName) {
		
      // 根据ServiceBean 生成rootBeanDefinition
        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

        MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();

        String[] ignoreAttributeNames = of("provider", "monitor", "application", "module", "registry", "protocol", "interface");

        propertyValues.addPropertyValues(new AnnotationPropertyValuesAdapter(service, environment, ignoreAttributeNames));

        // References "ref" property to annotated-@Service Bean
        addPropertyReference(builder, "ref", annotatedServiceBeanName);
        // Set interface
        builder.addPropertyValue("interface", interfaceClass.getName());

        /**
         * Add {@link com.alibaba.dubbo.config.ProviderConfig} Bean reference
         */
        String providerConfigBeanName = service.provider();
        if (StringUtils.hasText(providerConfigBeanName)) {
            addPropertyReference(builder, "provider", providerConfigBeanName);
        }

        /**
         * Add {@link com.alibaba.dubbo.config.MonitorConfig} Bean reference
         */
        String monitorConfigBeanName = service.monitor();
        if (StringUtils.hasText(monitorConfigBeanName)) {
            addPropertyReference(builder, "monitor", monitorConfigBeanName);
        }

        /**
         * Add {@link com.alibaba.dubbo.config.ApplicationConfig} Bean reference
         */
        String applicationConfigBeanName = service.application();
        if (StringUtils.hasText(applicationConfigBeanName)) {
            addPropertyReference(builder, "application", applicationConfigBeanName);
        }

        /**
         * Add {@link com.alibaba.dubbo.config.ModuleConfig} Bean reference
         */
        String moduleConfigBeanName = service.module();
        if (StringUtils.hasText(moduleConfigBeanName)) {
            addPropertyReference(builder, "module", moduleConfigBeanName);
        }


        /**
         * Add {@link com.alibaba.dubbo.config.RegistryConfig} Bean reference
         */
        String[] registryConfigBeanNames = service.registry();

        List<RuntimeBeanReference> registryRuntimeBeanReferences = toRuntimeBeanReferences(registryConfigBeanNames);

        if (!registryRuntimeBeanReferences.isEmpty()) {
            builder.addPropertyValue("registries", registryRuntimeBeanReferences);
        }

        /**
         * Add {@link com.alibaba.dubbo.config.ProtocolConfig} Bean reference
         */
        String[] protocolConfigBeanNames = service.protocol();

        List<RuntimeBeanReference> protocolRuntimeBeanReferences = toRuntimeBeanReferences(protocolConfigBeanNames);

        if (!protocolRuntimeBeanReferences.isEmpty()) {
            builder.addPropertyValue("protocols", protocolRuntimeBeanReferences);
        }

        return builder.getBeanDefinition();

    }
```

由此可见所有标记了@Service的类都是以ServiceBean作为RootBeanDefinition。**ServiceBean**是个十分重要的类。

重点看下ServiceBean。

### ServiceBean

首先看下这个类的定义：

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware {

}
```

1. 该类实现了InitializingBean，实现InitializingBean的接口的类，会在初始化完成之后调用afterPropertiesSet方法。

2. 实现了监听器 ApplicationListener<ContextRefreshedEvent>，在容器刷新完成的时候，执行onApplicationEvent事件。

   ```java
     @Override
       public void onApplicationEvent(ContextRefreshedEvent event) {
           if (isDelay() && !isExported() && !isUnexported()) {
               if (logger.isInfoEnabled()) {
                   logger.info("The service ready on spring started. service: " + getInterface());
               }
               // 开始暴露接口
               export();
           }
       }
   ```



由此可以看出在容器刷新完成之后，执行export()。



dubbo启动暴露端口从下面的代码开始：

```properties
com.alibaba.dubbo.config.spring.ServiceBean#onApplicationEvent
```

追踪代码可以到：

```properties
com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol
```

在doExportUrlsFor1Protocol中和很多appendParameters方法。可以往服务的url上追加参数。

如下：

```java
appendParameters(map, application);
appendParameters(map, module);
appendParameters(map, provider, Constants.DEFAULT_KEY);
appendParameters(map, protocolConfig);
appendParameters(map, this);
```

一路追代码到

```properties
com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper#export
```

- 构建url，url上带上各种参数。
