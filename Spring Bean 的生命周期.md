# Spring Bean 的生命周期

## 1. 实例化

IoC 容器通过获取 BeanDefinition 中该 bean 的 classType 后默认使用 CGLIB 来对该 bean 进行实例化，并将实例化出的对象封装在 BeanWrapper 对象中。BeanWrapper 提供了设置对象属性的接口，从而避免了使用反射机制设置属性。

## 2. 属性赋值（ 依赖注入 ）

调用 populateBean() 方法根据 BeanDefinition 中的信息进行依赖注入。

## 3. 初始化

### 3.1 调用 Aware 接口

检测该 bean 是否实现了特定的 Aware 接口，并将相关的 Aware 实例注入到该 bean 中。

### 3.2 调用 BeanPostProcessor 接口的 postProcessBeforeInitialzation() 方法（ 前置处理 ）

### 3.3 调用 InitializingBean 接口和 init-method

检测该 bean 是否实现了 InitializingBean 接口，执行 afterPropertiesSet() 方法。同时，Spring 为了降低对客户代码的侵入性，给 bean 的配置项提供了 init-method 属性，该属性指定了在这一阶段需要执行的方法名。Spring 便会在初始化阶段执行设置的方法。

### 3.4 调用 BeanPostProcessor 接口的 postProcessAfterInitialzation() 方法（ 后置处理 ）

## 4. 销毁

与 invokeInitMethods() 方法中类似，销毁阶段会检查是否实现了 DisposableBean 接口和 destroy-method 方法。
