## DispatcherServlet

本文源代码基于springboot 2.1.1



### spring boot 中创建ApplicationContext

在SpringApplication中的run方法中。创建ApplicationContext 方法是

createApplicationContext。在追踪代码，最终创建AnnotationConfigServletWebServerApplicationContext。

### spring boot 中创建web容器

在onRefresh中，创建createWebServer（Web容器）。

```java
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
```

```java
private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
    }
    else if (servletContext != null) {
        try {
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context",
                                                  ex);
        }
    }
    initPropertySources();
}
```

```java
@Override
protected void finishRefresh() {
    super.finishRefresh();
    WebServer webServer = startWebServer();
    if (webServer != null) {
        publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }
}
```

在ApplicationContext容器refresh完成后startWebServer，开启webserver。

### spring boot 中创建DispatcherServlet

在DispatcherServletAutoConfiguration中，注入了DispatcherServletRegistrationBean：

```java
@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public DispatcherServletRegistrationBean dispatcherServletRegistration(
    DispatcherServlet dispatcherServlet) {
    DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(
        dispatcherServlet, this.webMvcProperties.getServlet().getPath());
    registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
    registration.setLoadOnStartup(
        this.webMvcProperties.getServlet().getLoadOnStartup());
    if (this.multipartConfig != null) {
        registration.setMultipartConfig(this.multipartConfig);
    }
    return registration;
}
```

TomcatStarter实现ServletContainerInitializer，在web容器启动时候，容器去调用TomcatStarter的onStartup方法。进而调用ServletRegistrationBean的onStartup的方法。将DispatcherServlet注入到容器中。



