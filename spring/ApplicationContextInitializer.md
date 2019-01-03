## 概述

ApplicationContextInitializer是Spring框架原有的概念, 这个类的主要目的就是在            ConfigurableApplicationContext类型（或者子类型）的ApplicationContext做refresh之前，允许我们                   对ConfigurableApplicationContext的实例做进一步的设置或者处理。比如向容器中在注入某个组件。

### spring boot 中使用ApplicationContextInitializer

spring boot 中使用ApplicationContextInitializer地方在如下位置：

在run方法中的prepareContext方法中执行。prepareContext在容器refreshContext之前：

```java
private void prepareContext(ConfigurableApplicationContext context,
  ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
  ApplicationArguments applicationArguments, Banner printedBanner) {
 context.setEnvironment(environment);
 postProcessApplicationContext(context);
 // 执行Initializer
 applyInitializers(context);
 listeners.contextPrepared(context);
 if (this.logStartupInfo) {
  logStartupInfo(context.getParent() == null);
  logStartupProfileInfo(context);
 }
 // Add boot specific singleton beans
 ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
 beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
 if (printedBanner != null) {
  beanFactory.registerSingleton("springBootBanner", printedBanner);
 }
 if (beanFactory instanceof DefaultListableBeanFactory) {
  ((DefaultListableBeanFactory) beanFactory)
    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
 }
 // Load the sources
 Set<Object> sources = getAllSources();
 Assert.notEmpty(sources, "Sources must not be empty");
 load(context, sources.toArray(new Object[0]));
 listeners.contextLoaded(context);
}
```

在spring boot 使用ApplicationContextInitializer有三种方式：

#### 方式一

在spring.factories,配置ApplicationContextInitializer。例如：

```properties
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
```

这种方式在SpringApplication初始化的时候，通过：

```java
setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
```

给SpringApplication的initializers赋值。

#### 方式二

在springApplication实例中添加：

```java
public static void main(String[] args) {
  SpringApplication springBootApplication = new SpringApplication(Application.class);
  springBootApplication.addInitializers(new ConfigFileApplicationContextInitializer());
  springBootApplication.run(args);
  //SpringApplication.run(Application.class, args);
}
```

直接给SpringApplication的initializers增加。



#### 方式三

在配置文件配置：

```properties
context.initializer.classes=org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
```

这种方式，通过第一步注入的DelegatingApplicationContextInitializer，查找配置文件中的context.initializer.classes属性对应ApplicationContextInitializer。然后执行相应的Initializer的initialize方法。

下面看下DelegatingApplicationContextInitializer核心代码：

```java
private static final String PROPERTY_NAME = "context.initializer.classes";

@Override
public void initialize(ConfigurableApplicationContext context) {
 ConfigurableEnvironment environment = context.getEnvironment();
 List<Class<?>> initializerClasses = getInitializerClasses(environment);
 if (!initializerClasses.isEmpty()) {
  applyInitializerClasses(context, initializerClasses);
 }
}

private List<Class<?>> getInitializerClasses(ConfigurableEnvironment env) {
 String classNames = env.getProperty(PROPERTY_NAME);
 List<Class<?>> classes = new ArrayList<Class<?>>();
 if (StringUtils.hasLength(classNames)) {
  for (String className : StringUtils.tokenizeToStringArray(classNames, ",")) {
   classes.add(getInitializerClass(className));
  }
 }
 return classes;
}

private Class<?> getInitializerClass(String className) throws LinkageError {
 try {
  Class<?> initializerClass = ClassUtils.forName(className,
    ClassUtils.getDefaultClassLoader());
  Assert.isAssignable(ApplicationContextInitializer.class, initializerClass);
  return initializerClass;
 }
 catch (ClassNotFoundException ex) {
  throw new ApplicationContextException(
    "Failed to load context initializer class [" + className + "]", ex);
 }
}

private void applyInitializerClasses(ConfigurableApplicationContext context,
  List<Class<?>> initializerClasses) {
 Class<?> contextClass = context.getClass();
 List<ApplicationContextInitializer<?>> initializers = new ArrayList<ApplicationContextInitializer<?>>();
 for (Class<?> initializerClass : initializerClasses) {
  initializers.add(instantiateInitializer(contextClass, initializerClass));
 }
 applyInitializers(context, initializers);
}
```



### spring mvc 中使用ApplicationContextInitializer

在FrameworkServlet中的configureAndRefreshWebApplicationContext方法，这个方法是配置并且刷新WebApplicationContext.

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			if (this.contextId != null) {
				wac.setId(this.contextId);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
			}
		}

		wac.setServletContext(getServletContext());
		wac.setServletConfig(getServletConfig());
		wac.setNamespace(getNamespace());
		wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
		}

		postProcessWebApplicationContext(wac);
        // 执行Initializers，
		applyInitializers(wac);
		wac.refresh();
	}
```

在容器刷新之前applyInitializers(wac);执行Initializers的initialize方法。