# Spring 循环依赖

## 基本流程

getBean() ---> doGetBean() ---> getSingleton() ---> createBean() ---> addSingleton()

所以 Spring “多此一举”的将实例先封装到 ObjectFactory 中（三级缓存），主要关键点在 getObject() 方法并非直接返回实例，而是对实例又使用 SmartInstantiationAwareBeanPostProcessor 的 getEarlyBeanReference 方法对 bean 进行处理，也就是说，当 Spring 中存在该后置处理器，所有的单例 bean 在实例化后都会被进行提前曝光到三级缓存中，但是并不是所有的 bean 都存在循环依赖，也就是三级缓存到二级缓存的步骤不一定都会被执行，有可能曝光后直接创建完成，没被提前引用过，就直接被加入到一级缓存中。因此可以确保只有提前曝光且被引用的 bean 才会进行该后置处理。

## Spring 是如何解决循环依赖？

Spring 为了解决单例的循环依赖问题，使用了三级缓存。其中一级缓存为单例池 `singletonObjects`，二级缓存为提前曝光对象 `earlySingletonObjects`，三级缓存为提前曝光对象工厂 `singletonFactories`。

假设 A、B 循环引用，实例化 A 的时候就将其放入三级缓存中，接着填充属性的时候发现依赖了 B，同样的流程再执行一遍，实例化 B 后将其放入三级缓存中。接着对 B 填充属性时又发现依赖了 A，这时候尝试从二级缓存中查找早期暴露的 A，没有 AOP 代理的话，直接将 A 的原始对象注入 B，完成 B 的初始化后，就去完成剩下的 A 的步骤。如果有 AOP 代理，就进行 AOP 处理获取代理后的对象 A，注入 B，走完剩下的流程。

B 在三级缓存中查询到了 A 放入的 ObjectFactory 对象。如果 A 实现了 SmartInstantiationAwareBeanPostProcessor 接口则执行 getEarlyBeanReference() 方法来构建动态代理对象，随后 B 从三级缓存中将 A 的 ObjectFactory 对象删除后，将 A 的动态代理对象放入二级缓存中。

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```
