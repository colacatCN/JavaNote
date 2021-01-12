# BeanFactory 和 FactoryBean 的区别

## BeanFactory

BeanFactory，以 Factory 结尾，表示它是一个工厂类或接口，负责生产和管理 Bean 的一个工厂。在 Spring 中，BeanFactory 是 IoC 容器的核心接口，它含了各种 Bean 的定义、加载、实例化、依赖注入和生命周期管理的方法。BeanFactory 只是个接口，并不是 IoC 容器的具体实现，同时 Spring 容器也给出了很多种实现，如 DefaultListableBeanFactory、XmlBeanFactory 和 ApplicationContext 等，其中 ApplicationContext 接口作为 BeanFactory 的子类，除了提供 BeanFactory 所具有的功能外，还提供了更完整的框架功能：

1. 继承 MessageSource，因此支持国际化。

2. 资源文件访问，如 URL 和文件（ ResourceLoader ）。

3. 载入多个（有继承关系）上下文（即同时加载多个配置文件），使得每一个上下文都专注于一个特定的层次，比如应用的 Web 层。

4. 提供在监听器中注册 Bean 的事件。

## FactoryBean

在某些情况下实例化 Bean 的过程比较复杂，如果按照传统的方式则需要在 &#60;bean&#62; 标签中提供大量的配置信息，配置方式的灵活性和扩展受到了极大的限制。这时采用编码的方式可能会得到一个简单的方案，为此 Spring 提供了一个 org.springframework.bean.factory.FactoryBean 的工厂类接口，用户可以通过实现该接口定制实例化 Bean 的逻辑，可以通过 FactoryBean 接口的 getObject() 方法获取目标对象。

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
  SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
  factory.setDataSource(dataSource);
  factory.setVfs(SpringBootVFS.class);
  if (StringUtils.hasText(this.properties.getConfigLocation())) {
    factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
  }
  applyConfiguration(factory);
  if (this.properties.getConfigurationProperties() != null) {
    factory.setConfigurationProperties(this.properties.getConfigurationProperties());
  }
  if (!ObjectUtils.isEmpty(this.interceptors)) {
    factory.setPlugins(this.interceptors);
  }
  if (this.databaseIdProvider != null) {
    factory.setDatabaseIdProvider(this.databaseIdProvider);
  }
  if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
    factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
  }
  if (this.properties.getTypeAliasesSuperType() != null) {
    factory.setTypeAliasesSuperType(this.properties.getTypeAliasesSuperType());
  }
  if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
    factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
  }
  if (!ObjectUtils.isEmpty(this.typeHandlers)) {
    factory.setTypeHandlers(this.typeHandlers);
  }
  if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
    factory.setMapperLocations(this.properties.resolveMapperLocations());
  }
  Set<String> factoryPropertyNames = Stream
      .of(new BeanWrapperImpl(SqlSessionFactoryBean.class).getPropertyDescriptors()).map(PropertyDescriptor::getName)
      .collect(Collectors.toSet());
  Class<? extends LanguageDriver> defaultLanguageDriver = this.properties.getDefaultScriptingLanguageDriver();
  if (factoryPropertyNames.contains("scriptingLanguageDrivers") && !ObjectUtils.isEmpty(this.languageDrivers)) {
    // Need to mybatis-spring 2.0.2+
    factory.setScriptingLanguageDrivers(this.languageDrivers);
    if (defaultLanguageDriver == null && this.languageDrivers.length == 1) {
      defaultLanguageDriver = this.languageDrivers[0].getClass();
    }
  }
  if (factoryPropertyNames.contains("defaultScriptingLanguageDriver")) {
    // Need to mybatis-spring 2.0.2+
    factory.setDefaultScriptingLanguageDriver(defaultLanguageDriver);
  }

  return factory.getObject();
}
```
