# Spring Bean 的生命周期

## 1. 实例化

IoC 容器通过获取 BeanDefinition 中该 Bean 的 classType 后默认使用 CGLIB 来对该 bean 进行实例化，并将实例化出的对象封装在 BeanWrapper 对象中。

## 2. 属性赋值（ 依赖注入 ）

根据 BeanDefinition 中的信息调用 BeanWrapper 接口提供的 setPropertyValue() 方法对 Bean 中的属性逐一进行赋值。

## 3. 初始化

### 3.1 调用 Aware 接口

检测该 Bean 是否实现了特定的 Aware 接口，并将相关的 Aware 实例注入到该 Bean 中。

### 3.2 调用 BeanPostProcessor 接口的 postProcessBeforeInitialzation() 方法（ 前置处理 ）

### 3.3 调用 InitializingBean 接口和 init-method

检测该 Bean 是否实现了 InitializingBean 接口，并执行 afterPropertiesSet() 方法。同时，Spring 为了降低对客户代码的侵入性，给 Bean 的配置项提供了 init-method 属性，该属性指定了在这一阶段需要执行的方法名，Spring 便会在初始化阶段执行设置的方法。

### 3.4 调用 BeanPostProcessor 接口的 postProcessAfterInitialzation() 方法（ 后置处理 ）

## 4. 销毁

与 invokeInitMethods() 方法中类似，销毁阶段会检查是否实现了 DisposableBean 接口和 destroy-method 方法。
