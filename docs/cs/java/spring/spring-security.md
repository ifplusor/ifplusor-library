---
tags: Java Spring SpringSecurity
---

# Spring Security 笔记

Spring Security 是 Java Web 技术中，管理**认证**和**授权**的**安全框架**。

## Spring Security 原理（Servlet）

在 Spring MVC 项目中，Spring Security 在 Web 容器中插入一个 **Filter**，从而在 Http 请求进入 **DispatcherServlet** 前对**认证**（Authentication）和**授权**（Authorization）进行校验。

> 关于 Spring Security 的更多架构原理请参看 [Servlet 架构](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-architecture)

## 关键结构

```txt
                     springSecurityFilterrChain
                                  :
                                  :
DelegatingFilterProxy --> FilterChainProxy --> SecurityFilterChain
                                  ^                     ^
                                  |                     |
                             WebSecurity           HttpSecurity
```

Spring Security 框架应用了职责链模式，通过多级 Filter 的对请求的层层拦截实现了对不同职责关注点的解耦。

### 1. SecurityFilterChain 和 FilterChainProxy

Spring Security 框架中表示职责链的接口是 SecurityFilterChain，具有 *matches*() 和 *getFilters*() 两个方法。因为 Spring Security 可以配置多个 SecurityFilterChain，所以需要 *matches*() 方法根据具体的请求完成选择。*getFilters*() 方法则是用来获取职责链中的 Filter 的有序列表。

```java
package org.springframework.security.web;
// ...
public interface SecurityFilterChain {

    boolean matches(HttpServletRequest request);

    List<Filter> getFilters();
}
```

FilterChainProxy 是 SecurityFilterChain 集合的代理，它将多个 SecurityFilterChain 包装起来，对外暴露统一的入口。

```java
package org.springframework.security.web;
// ...
public class FilterChainProxy extends GenericFilterBean {
    // ...
    private List<SecurityFilterChain> filterChains;
    // ...
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;
        if (clearContext) {
            try {
                request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
                doFilterInternal(request, response, chain);
            }
            finally {
                SecurityContextHolder.clearContext();
                request.removeAttribute(FILTER_APPLIED);
            }
        }
        else {
            doFilterInternal(request, response, chain);
        }
    }

    private void doFilterInternal(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {

        FirewalledRequest fwRequest = firewall
                .getFirewalledRequest((HttpServletRequest) request);
        HttpServletResponse fwResponse = firewall
                .getFirewalledResponse((HttpServletResponse) response);

        List<Filter> filters = getFilters(fwRequest);

        if (filters == null || filters.size() == 0) {
            if (logger.isDebugEnabled()) {
                logger.debug(UrlUtils.buildRequestUrl(fwRequest)
                        + (filters == null ? " has no matching filters"
                                : " has an empty filter list"));
            }

            fwRequest.reset();

            chain.doFilter(fwRequest, fwResponse);

            return;
        }

        VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
        vfc.doFilter(fwRequest, fwResponse);
    }

    private List<Filter> getFilters(HttpServletRequest request) {
        for (SecurityFilterChain chain : filterChains) {
            if (chain.matches(request)) {
                return chain.getFilters();
            }
        }

        return null;
    }
    // ...
}
```

### 2. DelegatingFilterProxy

DelegatingFilterProxy 是 FilterChainProxy 的代理，是 Spring Security 框架与 Web 容器关联的 Filter 对象，其 *initDelegate*() 方法根据 **targetBeanName** 来获取真正作为 Filter 的 Bean。

事实上，代理 FilterChainProxy 对象的 DelegatingFilterProxy 对象的 targetBeanName 成员的值是 **springSecurityFilterChain**，即 DelegatingFilterProxy 对象代理了名为 **springSecurityFilterChain** 的 Bean。

```java
package org.springframework.web.filter;
// ...
public class DelegatingFilterProxy extends GenericFilterBean {
    // ...
    @Nullable
    private volatile Filter delegate;
    // ...
    public DelegatingFilterProxy(String targetBeanName, @Nullable WebApplicationContext wac) {
        Assert.hasText(targetBeanName, "Target Filter bean name must not be null or empty");
        this.setTargetBeanName(targetBeanName);
        this.webApplicationContext = wac;
        if (wac != null) {
            this.setEnvironment(wac.getEnvironment());
        }
    }
    // ...
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        // Lazily initialize the delegate if necessary.
        Filter delegateToUse = this.delegate;
        if (delegateToUse == null) {
            synchronized (this.delegateMonitor) {
                delegateToUse = this.delegate;
                if (delegateToUse == null) {
                    WebApplicationContext wac = findWebApplicationContext();
                    if (wac == null) {
                        throw new IllegalStateException("No WebApplicationContext found: " +
                                "no ContextLoaderListener or DispatcherServlet registered?");
                    }
                    delegateToUse = initDelegate(wac);
                }
                this.delegate = delegateToUse;
            }
        }

        // Let the delegate perform the actual doFilter operation.
        invokeDelegate(delegateToUse, request, response, filterChain);
    }
    // ...
    protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
        String targetBeanName = getTargetBeanName();
        Assert.state(targetBeanName != null, "No target bean name set");
        Filter delegate = wac.getBean(targetBeanName, Filter.class);
        if (isTargetFilterLifecycle()) {
            delegate.init(getFilterConfig());
        }
        return delegate;
    }
    // ...
}
```

### 3. WebSecurity 和 HttpSecurity

最后，**WebSecurity** 是 **FilterChainProxy** 的 Builder 类，**HttpSecurity** 是 **DefaultSecurityFilterChain** 的 Builder 类。它们都继承了 AbstractConfiguredSecurityBuilder 类，而 AbstractConfiguredSecurityBuilder 又继承 AbstractSecurityBuilder 类，AbstractSecurityBuilder 则实现了 SecurityBuilder 接口。

#### 3.1. SecurityBuilder 接口

SecurityBuilder 接口中只定义了 *build*() 方法。

```java
package org.springframework.security.config.annotation;
// ...
public interface SecurityBuilder<O> {

    O build() throws Exception;
}
```

#### 3.2. AbstractSecurityBuilder 抽象类

AbstractSecurityBuilder 的 *build*() 方法直接调用 *doBuild*() 方法。

```java
package org.springframework.security.config.annotation;
// ...
public abstract class AbstractSecurityBuilder<O> implements SecurityBuilder<O> {
    // ...
    public final O build() throws Exception {
        if (this.building.compareAndSet(false, true)) {
            this.object = doBuild();
            return this.object;
        }
        throw new AlreadyBuiltException("This object has already been built");
    }
    // ...
    protected abstract O doBuild() throws Exception;
    // ...
}
```

#### 3.4. AbstractConfiguredSecurityBuilder 抽象类

AbstractConfiguredSecurityBuilder 重写了 *doBuild*() 方法，依次调用 *beforeInit*(), *init*(), *beforeConfigure*(), *configure*() 方法进行初始化和配置，最后调用 *performBuild*() 方法完成对象创建过程。

```java
package org.springframework.security.config.annotation;
// ...
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
        extends AbstractSecurityBuilder<O> {
    // ...
    @Override
    protected final O doBuild() throws Exception {
        synchronized (configurers) {
            buildState = BuildState.INITIALIZING;

            beforeInit();
            init();

            buildState = BuildState.CONFIGURING;

            beforeConfigure();
            configure();

            buildState = BuildState.BUILDING;

            O result = performBuild();

            buildState = BuildState.BUILT;

            return result;
        }
    }

    protected void beforeInit() throws Exception {
    }

    protected void beforeConfigure() throws Exception {
    }

    protected abstract O performBuild() throws Exception;

    @SuppressWarnings("unchecked")
    private void init() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();

        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.init((B) this);
        }

        for (SecurityConfigurer<O, B> configurer : configurersAddedInInitializing) {
            configurer.init((B) this);
        }
    }

    @SuppressWarnings("unchecked")
    private void configure() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();

        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.configure((B) this);
        }
    }
    // ...
}
```

值得注意的是，*init*() 方法中分别对 **configurers** 和 **configurersAddedInInitializing** 做了遍历，并且 **configures** 对象通过 getConfigurers() 方法获取的 List 对象。因为在第一次遍历过程中（对 configurers 的遍历）可以继续向 Builder 中添加 Configurer，这些 Configurer 会被加入到 **configurersAddedInInitializing** 中，并延迟到第二次遍历过程才进行初始化。

```java
package org.springframework.security.config.annotation;
// ...
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
        extends AbstractSecurityBuilder<O> {
    // ...
    private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();
    private final List<SecurityConfigurer<O, B>> configurersAddedInInitializing = new ArrayList<>();
    // ...
    @SuppressWarnings("unchecked")
    private <C extends SecurityConfigurer<O, B>> void add(C configurer) {
        Assert.notNull(configurer, "configurer cannot be null");

        Class<? extends SecurityConfigurer<O, B>> clazz = (Class<? extends SecurityConfigurer<O, B>>) configurer
                .getClass();
        synchronized (configurers) {
            if (buildState.isConfigured()) {
                throw new IllegalStateException("Cannot apply " + configurer
                        + " to already built object");
            }
            List<SecurityConfigurer<O, B>> configs = allowConfigurersOfSameType ? this.configurers
                    .get(clazz) : null;
            if (configs == null) {
                configs = new ArrayList<>(1);
            }
            configs.add(configurer);
            this.configurers.put(clazz, configs);
            if (buildState.isInitializing()) {
                this.configurersAddedInInitializing.add(configurer);
            }
        }
    }
    // ...
    private Collection<SecurityConfigurer<O, B>> getConfigurers() {
        List<SecurityConfigurer<O, B>> result = new ArrayList<>();
        for (List<SecurityConfigurer<O, B>> configs : this.configurers.values()) {
            result.addAll(configs);
        }
        return result;
    }
    // ...
}
```

#### 3.5. WebSecurity 类

WebSecurity 重写了 *performBuild*() 方法，先遍历 securityFilterChainBuilders 创建 **SecurityFilterChain** 列表，再用该列表对象创建 FilterChainProxy 对象。这里的 securityFilterChainBuilders 就是 HttpSecurity 对象的列表。

```java
package org.springframework.security.config.annotation.web.builders;
// ...
public final class WebSecurity extends
        AbstractConfiguredSecurityBuilder<Filter, WebSecurity> implements
        SecurityBuilder<Filter>, ApplicationContextAware {
    // ...
    private final List<RequestMatcher> ignoredRequests = new ArrayList<>();
    private final List<SecurityBuilder<? extends SecurityFilterChain>> securityFilterChainBuilders = new ArrayList<>();
    // ...
    @Override
    protected Filter performBuild() throws Exception {
        // ...
        int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
        List<SecurityFilterChain> securityFilterChains = new ArrayList<>(chainSize);
        for (RequestMatcher ignoredRequest : ignoredRequests) {
            securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
        }
        for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
            securityFilterChains.add(securityFilterChainBuilder.build());
        }
        FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
        // ...
        Filter result = filterChainProxy;
        // ...
        return result;
    }
    // ...
}
```

#### 3.6. HttpSecurity 类

HttpSecurity 同样重写了 *performBuild*() 方法，先对 Filter 对象列表排序，再创建 DefaultSecurityFilterChain 对象。

```java
package org.springframework.security.config.annotation.web.builders;
// ...
public final class HttpSecurity extends
        AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
        implements SecurityBuilder<DefaultSecurityFilterChain>,
        HttpSecurityBuilder<HttpSecurity> {
    // ...
    private List<Filter> filters = new ArrayList<>();
    private RequestMatcher requestMatcher = AnyRequestMatcher.INSTANCE;
    private FilterComparator comparator = new FilterComparator();
    // ...
    @Override
    protected DefaultSecurityFilterChain performBuild() {
        filters.sort(comparator);
        return new DefaultSecurityFilterChain(requestMatcher, filters);
    }
    // ...
}
```

## 从 Filter 到 Filter

### 1. Spring Boot 自动装配 DelegatingFilterProxy 对象

众所周知，Spring Boot 应用 SPI 机制来实现自动配置。查看 spring-boot-autoconfigure 包下的 META-INF/spring.factories，可以看到 org.springframework.boot.autoconfigure.EnableAutoConfiguration 属性的值中包含 org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration。

```java
package org.springframework.boot.autoconfigure.security.servlet;
// ...
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class, SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {

    private static final String DEFAULT_FILTER_NAME = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;

    @Bean
    @ConditionalOnBean(name = DEFAULT_FILTER_NAME)
    public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
            SecurityProperties securityProperties) {
        DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(
                DEFAULT_FILTER_NAME);
        registration.setOrder(securityProperties.getFilter().getOrder());
        registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
        return registration;
    }

    private EnumSet<DispatcherType> getDispatcherTypes(SecurityProperties securityProperties) {
        if (securityProperties.getFilter().getDispatcherTypes() == null) {
            return null;
        }
        return securityProperties.getFilter().getDispatcherTypes().stream()
                .map((type) -> DispatcherType.valueOf(type.name()))
                .collect(Collectors.toCollection(() -> EnumSet.noneOf(DispatcherType.class)));
    }

}
```

SecurityFilterAutoConfiguration 的 *securityFilterChainRegistration*() 方法创建了一个 DelegatingFilterProxyRegistrationBean 类型的 Bean，该类型实现了 ServletContextInitializer 接口，会在执行 ServletWebServerApplicationContext.selfInitialize() 进行初始化的时候调用 RegistrationBean.*onStartup*() -> DynamicRegistrationBean.*register*() -> AbstractFilterRegistrationBean.*addRegistration*() 方法。*addRegistration*() 方法中先调用 DelegatingFilterProxyRegistrationBean.getFilter() 方法获取 DelegatingFilterProxy 对象，然后调用 ServletContext.*addFilter*() 方法将该 Filter 对象加入 ServletContext。

```java
package org.springframework.boot.web.servlet;
// ...
public class DelegatingFilterProxyRegistrationBean extends AbstractFilterRegistrationBean<DelegatingFilterProxy>
        implements ApplicationContextAware {
    // ...
    private final String targetBeanName;
    // ...
    public DelegatingFilterProxyRegistrationBean(String targetBeanName,
            ServletRegistrationBean<?>... servletRegistrationBeans) {
        super(servletRegistrationBeans);
        Assert.hasLength(targetBeanName, "TargetBeanName must not be null or empty");
        this.targetBeanName = targetBeanName;
        setName(targetBeanName);
    }
    // ...
    @Override
    public DelegatingFilterProxy getFilter() {
        return new DelegatingFilterProxy(this.targetBeanName, getWebApplicationContext()) {

            @Override
            protected void initFilterBean() throws ServletException {
                // Don't initialize filter bean on init()
            }

        };
    }
    // ...
}
```

### 2. springSecurityFilterChain Bean 的创建

WebSecurityConfiguration 是 Spring Security 的配置类，其中的 *springSecurityFilterChain*() 方法就是用来创建 springSecurityFilterChain Bean 的。

```java
package org.springframework.security.config.annotation.web.configuration;
// ...
@Configuration(proxyBeanMethods = false)
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
    private WebSecurity webSecurity;
    private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;
    // ...
    @Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
    public Filter springSecurityFilterChain() throws Exception {
        boolean hasConfigurers = webSecurityConfigurers != null
                && !webSecurityConfigurers.isEmpty();
        if (!hasConfigurers) {
            WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
                    .postProcess(new WebSecurityConfigurerAdapter() {
                    });
            webSecurity.apply(adapter);
        }
        return webSecurity.build();
    }
    // ...
}
```

*springSecurityFilterChain*() 中调用 WebSecuity.*build*() 方法创建了 FilterChainProxy 对象。这里用到的 WebSecurity 对象是在 *setFilterChainProxySecurityConfigurer*() 方法中创建的。

```java
package org.springframework.security.config.annotation.web.configuration;
// ...
@Configuration(proxyBeanMethods = false)
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
    // ...
    @Autowired(required = false)
    public void setFilterChainProxySecurityConfigurer(
            ObjectPostProcessor<Object> objectPostProcessor,
            @Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
            throws Exception {
        webSecurity = objectPostProcessor
                .postProcess(new WebSecurity(objectPostProcessor));
        if (debugEnabled != null) {
            webSecurity.debug(debugEnabled);
        }

        webSecurityConfigurers.sort(AnnotationAwareOrderComparator.INSTANCE);

        Integer previousOrder = null;
        Object previousConfig = null;
        for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
            Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
            if (previousOrder != null && previousOrder.equals(order)) {
                throw new IllegalStateException(
                        "@Order on WebSecurityConfigurers must be unique. Order of "
                                + order + " was already used on " + previousConfig + ", so it cannot be used on "
                                + config + " too.");
            }
            previousOrder = order;
            previousConfig = config;
        }
        for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
            webSecurity.apply(webSecurityConfigurer);
        }
        this.webSecurityConfigurers = webSecurityConfigurers;
    }
    // ...
}
```

不仅如此，*setFilterChainProxySecurityConfigurer*() 方法还向 WebSecurity 对象，并注入了所有 WebSecurityConfigurer 类型（典型如 **WebSecurityConfigurerAdapter**）的 Bean。WebSecurity 的 **configurers** 成员对象的 Configurer 元素来原于此。

```java
package org.springframework.security.config.annotation;
// ...
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
        extends AbstractSecurityBuilder<O> {
    // ...
    public <C extends SecurityConfigurer<O, B>> C apply(C configurer) throws Exception {
        add(configurer);
        return configurer;
    }
    // ...
}
```

值得注意的是，尽管 **webSecurityConfigurers** 参数是 **SecurityConfigurer** 的 List，但 autowiredWebSecurityConfigurersIgnoreParents.*getWebSecurityConfigurers*() 的返回结果中只包含 **WebSecurityConfigurer** 子接口的对象。

```java
package org.springframework.security.config.annotation;
// ...
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {

    void init(B builder) throws Exception;

    void configure(B builder) throws Exception;

}
```

```java
package org.springframework.security.config.annotation.web;
// ...
public interface WebSecurityConfigurer<T extends SecurityBuilder<Filter>> extends
        SecurityConfigurer<Filter, T> {

}
```

```java
package org.springframework.security.config.annotation.web.configuration;
// ...
final class AutowiredWebSecurityConfigurersIgnoreParents {
    // ...
    @SuppressWarnings({ "rawtypes", "unchecked" })
    public List<SecurityConfigurer<Filter, WebSecurity>> getWebSecurityConfigurers() {
        List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers = new ArrayList<>();
        Map<String, WebSecurityConfigurer> beansOfType = beanFactory
                .getBeansOfType(WebSecurityConfigurer.class);
        for (Entry<String, WebSecurityConfigurer> entry : beansOfType.entrySet()) {
            webSecurityConfigurers.add(entry.getValue());
        }
        return webSecurityConfigurers;
    }
}
```

## 通过 WebSecurityConfigurerAdapter 配置 Spring Security

WebSecurityConfigurerAdapter 是 Spring Security 框架中最常用的配置类，定义了多个扩展点用来实现对 Spring Security 框架不同方面的配置，其中最重要的两个扩展点是 ```void configure(WebSecurity web)``` 和 ```void configure(HttpSecurity http)``` 分别用来配置 WebSecurity 对象和 HttpSecurity 对象。

```java
package org.springframework.security.config.annotation.web.configuration;
// ...
@Order(100)
public abstract class WebSecurityConfigurerAdapter implements
        WebSecurityConfigurer<WebSecurity> {
    // ...
    public void configure(WebSecurity web) throws Exception {
    }
    // ...
    protected void configure(HttpSecurity http) throws Exception {
        logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

        http
            .authorizeRequests()
                .anyRequest().authenticated()
                .and()
            .formLogin().and()
            .httpBasic();
    }
    //..
}
```

### 1. 配置 WebSecurity

WebSecurityConfigurerAdapter 类继承 WebSecurityConfigurer 接口，因此会由 Spring 构架调用 WebSecurityConfiguration.*setFilterChainProxySecurityConfigurer*() 方法自动注入到 WebSecurityConfiguration 配置对象中，并通过 WebSecurity.*apply*() 方法注入到 **webSecurity** 对象里。

Spring 架构调用 WebSecurityConfiguration.*springSecurityFilterChain*() 方法实例化 Filter 对象时会调用 **webSecurity** 对象的 *build*() 方法，从而调用 WebSecurityConfigurerAdapter 对象的 *init*() 和 *configure*() 方法。

```java
package org.springframework.security.config.annotation.web.configuration;
// ...
@Order(100)
public abstract class WebSecurityConfigurerAdapter implements
        WebSecurityConfigurer<WebSecurity> {
    // ...
    private HttpSecurity http;
    // ...
    public void init(final WebSecurity web) throws Exception {
        final HttpSecurity http = getHttp();
        web.addSecurityFilterChainBuilder(http).postBuildAction(() -> {
            FilterSecurityInterceptor securityInterceptor = http
                    .getSharedObject(FilterSecurityInterceptor.class);
            web.securityInterceptor(securityInterceptor);
        });
    }

    public void configure(WebSecurity web) throws Exception {
    }
    // ...
}
```

```void configure(WebSecurity web)``` 方法就是留给开发者用来配置 **webSecurity** 对象的扩展点。

### 2. 配置 HttpSecurity

WebSecurityConfigurerAdapter 对象的 *init*() 方法中，首先调用 *getHttp*() 方法取得 **HttpSecurity** 对象，然后调用 **webSecurity** 对象的 *addSecurityFilterChainBuilder*() 方法将 **HttpSecurity** 对象加入到 WebSecurity 的 **securityFilterChainBuilders** 列表中。

```java
public final class WebSecurity extends
        AbstractConfiguredSecurityBuilder<Filter, WebSecurity> implements
        SecurityBuilder<Filter>, ApplicationContextAware {
    // ..
    private final List<SecurityBuilder<? extends SecurityFilterChain>> securityFilterChainBuilders = new ArrayList<>();
    // ...
    public WebSecurity addSecurityFilterChainBuilder(
            SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder) {
        this.securityFilterChainBuilders.add(securityFilterChainBuilder);
        return this;
    }
    // ...
}
```

*getHttp*() 方法先调用 *authenticationManager*() 方法取得 AuthenticationManager 对象，并以该对象作为 AuthenticationManagerBuilder 的 parentAuthenticationManager。然后创建 HttpSecurity 对象，并调用 ```void configure(HttpSecurity http)``` 方法对其进行配置。

```java
public abstract class WebSecurityConfigurerAdapter implements
        WebSecurityConfigurer<WebSecurity> {
    // ...
    private AuthenticationManagerBuilder authenticationBuilder;
    // ...
    @SuppressWarnings({ "rawtypes", "unchecked" })
    protected final HttpSecurity getHttp() throws Exception {
        if (http != null) {
            return http;
        }

        AuthenticationEventPublisher eventPublisher = getAuthenticationEventPublisher();
        localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

        AuthenticationManager authenticationManager = authenticationManager();
        authenticationBuilder.parentAuthenticationManager(authenticationManager);
        Map<Class<?>, Object> sharedObjects = createSharedObjects();

        http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
                sharedObjects);
        if (!disableDefaults) {
            // @formatter:off
            http
                .csrf().and()
                .addFilter(new WebAsyncManagerIntegrationFilter())
                .exceptionHandling().and()
                .headers().and()
                .sessionManagement().and()
                .securityContext().and()
                .requestCache().and()
                .anonymous().and()
                .servletApi().and()
                .apply(new DefaultLoginPageConfigurer<>()).and()
                .logout();
            // @formatter:on
            ClassLoader classLoader = this.context.getClassLoader();
            List<AbstractHttpConfigurer> defaultHttpConfigurers =
                    SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);

            for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
                http.apply(configurer);
            }
        }
        configure(http);
        return http;
    }
    // ...
    protected void configure(HttpSecurity http) throws Exception {
        logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

        http
            .authorizeRequests()
                .anyRequest().authenticated()
                .and()
            .formLogin().and()
            .httpBasic();
    }
    // ...
}
```

### 3. 配置 AuthenticationManager

WebSecurityConfigurerAdapter 对象的 *authenticationManager*() 方法首先调用 ```void configure(AuthenticationManagerBuilder auth)``` 方法对 localConfigureAuthenticationBldr 进行配置。若未重写 *configure*() 方法，则默认实现将 disableLocalConfigureAuthenticationBldr 设置为 ```true```，引发 **authenticationConfiguration**.*getAuthenticationManager*() 的执行。

authenticationBuilder 和 localConfigureAuthenticationBldr 是通过 *setApplicationContext*() 方法自动注入 ApplicationContext 对象时创建的；全局 AuthenticationManager 对象是通过自动注入的 AuthenticationConfiguration 配置对象的 *getAuthenticationManager*() 方法获取的。

```java
public abstract class WebSecurityConfigurerAdapter implements
        WebSecurityConfigurer<WebSecurity> {
    // ...
    private AuthenticationConfiguration authenticationConfiguration;
    private AuthenticationManagerBuilder localConfigureAuthenticationBldr;
    private boolean disableLocalConfigureAuthenticationBldr;
    private boolean authenticationManagerInitialized;
    private AuthenticationManager authenticationManager;
    // ...
    protected AuthenticationManager authenticationManager() throws Exception {
        if (!authenticationManagerInitialized) {
            configure(localConfigureAuthenticationBldr);
            if (disableLocalConfigureAuthenticationBldr) {
                authenticationManager = authenticationConfiguration
                        .getAuthenticationManager();
            }
            else {
                authenticationManager = localConfigureAuthenticationBldr.build();
            }
            authenticationManagerInitialized = true;
        }
        return authenticationManager;
    }
    // ...
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        this.disableLocalConfigureAuthenticationBldr = true;
    }
    // ...
    @Autowired
    public void setApplicationContext(ApplicationContext context) {
        this.context = context;

        ObjectPostProcessor<Object> objectPostProcessor = context.getBean(ObjectPostProcessor.class);
        LazyPasswordEncoder passwordEncoder = new LazyPasswordEncoder(context);

        authenticationBuilder = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, passwordEncoder);
        localConfigureAuthenticationBldr = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, passwordEncoder) {
            @Override
            public AuthenticationManagerBuilder eraseCredentials(boolean eraseCredentials) {
                authenticationBuilder.eraseCredentials(eraseCredentials);
                return super.eraseCredentials(eraseCredentials);
            }

            @Override
            public AuthenticationManagerBuilder authenticationEventPublisher(AuthenticationEventPublisher eventPublisher) {
                authenticationBuilder.authenticationEventPublisher(eventPublisher);
                return super.authenticationEventPublisher(eventPublisher);
            }
        };
    }
    // ...
    @Autowired
    public void setAuthenticationConfiguration(
            AuthenticationConfiguration authenticationConfiguration) {
        this.authenticationConfiguration = authenticationConfiguration;
    }
    // ...
}
```

## 通过 AuthenticationConfiguration 配置全局 AuthenticationManager 对象

AuthenticationConfiguration 是 Spring Security 框架的默认 AuthenticationManager 对象配置类，由 @EnableGlobalAuthentication 注解导入。引入 @EnableWebSecurity 注解的同时就会引入 @EnableGlobalAuthentication 注解。

```java
package org.springframework.security.config.annotation.web.configuration;
// ...
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import({ WebSecurityConfiguration.class,
        SpringWebMvcImportSelector.class,
        OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {

    boolean debug() default false;
}
```

```java
package org.springframework.security.config.annotation.authentication.configuration;
// ...
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import(AuthenticationConfiguration.class)
@Configuration
public @interface EnableGlobalAuthentication {
}
```

### AuthenticationConfiguration.*getAuthenticationManager*()

AuthenticationConfiguration 配置类中最重要的是 *getAuthenticationManager*() 方法，它是全局 AuthenticationManager 对象的单例工厂。

*getAuthenticationManager*() 方法中，首先获取 AuthenticationManagerBuilder 类型的 Bean，然后对该 builder 应用 GlobalAuthenticationConfigurerAdapter，最后构造 AuthenticationManager 对象。

```java
package org.springframework.security.config.annotation.authentication.configuration;
// ...
@Configuration(proxyBeanMethods = false)
@Import(ObjectPostProcessorConfiguration.class)
public class AuthenticationConfiguration {
    // ...
    public AuthenticationManager getAuthenticationManager() throws Exception {
        if (this.authenticationManagerInitialized) {
            return this.authenticationManager;
        }
        AuthenticationManagerBuilder authBuilder = this.applicationContext.getBean(AuthenticationManagerBuilder.class);
        if (this.buildingAuthenticationManager.getAndSet(true)) {
            return new AuthenticationManagerDelegator(authBuilder);
        }

        for (GlobalAuthenticationConfigurerAdapter config : globalAuthConfigurers) {
            authBuilder.apply(config);
        }

        authenticationManager = authBuilder.build();

        if (authenticationManager == null) {
            authenticationManager = getAuthenticationManagerBean();
        }

        this.authenticationManagerInitialized = true;
        return authenticationManager;
    }
    // ...
    @Bean
    public AuthenticationManagerBuilder authenticationManagerBuilder(
        ObjectPostProcessor<Object> objectPostProcessor, ApplicationContext context) {
        LazyPasswordEncoder defaultPasswordEncoder = new LazyPasswordEncoder(context);
        AuthenticationEventPublisher authenticationEventPublisher = getBeanOrNull(context, AuthenticationEventPublisher.class);

        DefaultPasswordEncoderAuthenticationManagerBuilder result = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, defaultPasswordEncoder);
        if (authenticationEventPublisher != null) {
            result.authenticationEventPublisher(authenticationEventPublisher);
        }
        return result;
    }
    // ...
    @Autowired(required = false)
    public void setGlobalAuthenticationConfigurers(
            List<GlobalAuthenticationConfigurerAdapter> configurers) {
        configurers.sort(AnnotationAwareOrderComparator.INSTANCE);
        this.globalAuthConfigurers = configurers;
    }
    // ...
}
```

## SecurityFilterChain 中的 Filter

### 默认 Filter 顺序

- ChannelProcessingFilter
- ConcurrentSessionFilter
- SecurityContextPersistenceFilter
- LogoutFilter
- X509AuthenticationFilter
- AbstractPreAuthenticatedProcessingFilter
- CasAuthenticationFilter
- UsernamePasswordAuthenticationFilter
- ConcurrentSessionFilter
- OpenIDAuthenticationFilter
- org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter
- org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter
- ConcurrentSessionFilter
- DigestAuthenticationFilter
- org.springframework.security.oauth2.server.resource.web.BearerTokenAuthenticationFilter
- BasicAuthenticationFilter
- RequestCacheAwareFilter
- SecurityContextHolderAwareRequestFilter
- JaasApiIntegrationFilter
- RememberMeAuthenticationFilter
- AnonymousAuthenticationFilter
- SessionManagementFilter
- ExceptionTranslationFilter
- FilterSecurityInterceptor
- SwitchUserFilter

### BearerTokenAuthenticationFilter

BearerTokenAuthenticationFilter 由 OAuth2ResourceServerConfigurer 进行配置，用来处理 OAuth2 认证。

BearerTokenAuthenticationFilter 首先调用 **bearerTokenResolver**.*resolve*() 方法从 request 中提取 token （**bearerTokenResolver** 默认为 DefaultBearerTokenResolver 的对象）。若 token 不存在，则调用 filterChain 中的后续 Filter；反之通过 authenticationManagerResolver.*resolve*() 方法取得 AuthenticationManager，并调用 authenticationManager.*authenticate*() 方法解析 Authentication（authenticationManager 默认为 ProviderManager 的对象，并由 JwtAuthenticationProvider 提供对 OAuth2 JWT 的支持）。

### ExceptionTranslationFilter

ExceptionTranslationFilter 由 ExceptionHandlingConfigurer 进行配置，它拦截所有异常：

1. 对于 AuthenticationException 异常：调用 **authenticationEntryPoint**.*commence*() 方法发起认证过程；
2. 对于 AccessDeniedException 异常：若当前认证为匿名（Anonymous）或记住登录（RememberMe），同样发起认证过程；否则调用 **accessDeniedHandler**.*handle*() 方法进行异常处理；
3. 对于 IOException、ServletException 和 RuntimeException 异常：直接抛出；
4. 其它异常：包装成 RuntimeException 异常抛出。
