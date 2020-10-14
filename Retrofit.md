# Retrofit2

## Okhttp 发起同步和异步请求的方式：

```java
// 同步请求
Response response = okhttpClient.newCall(request).execute();
// 异步请求
Response response =okhttpClient.newCall(request).enqueue(callback);
```

从上面两行代码，可以发现 Okhttp 发起网络请求有三个步骤：
1. 将所需参数构建出 Request 对象；
2. 根据 Request 对象构建出对应的 Call 对象；
3. 执行 execute() 或 enqueue() 方法来完成同、异步请求。

## 通过 OkhttpCall 来包装调用 Okhttp 的 call 来完成请求发送的过程

```java
@Override public Response<T> execute() throws IOException {
okhttp3.Call call;

synchronized (this) {
    // 省略部分代码

    call = rawCall;
    if (call == null) {
    try {
        call = rawCall = createRawCall(); // @1
    } catch (IOException | RuntimeException | Error e) {
        throwIfFatal(e);
        creationFailure = e;
        throw e;
    }
    }
}

if (canceled) {
    call.cancel();
}

return parseResponse(call.execute()); // @2、@3
}
```

删除一些无关紧要的代码后，发现 OkhttpCall 的 execute() 方法主要做了如下三件事：
1. 调用 createRawCall() 方法创建 Okhttp3 的 Call 对象；
2. 调用 call.execute() 方法返回一个 Okhttp3 的 Response 对象；
3. 调用 parseResponse() 方法对步骤 2 返回的 Response 进行处理，而后返回 Retrofit 泛化的 Response<T> 对象。

## Retrofit 执行过程分析

### Retrofit.create()

Retrofit.create() 方法的入参是一个接口的 Class 对象，但返回给上层业务的却是一个由 Proxy.newProxyInstance() 创建出的动态代理对象。动态代理对象的强大之处在于只需要用户指定目标接口就能在程序运行过程中自动为其生成一个对应的动态代理对象，之后该接口下所有方法的调用都会走动态代理对象的 invoke() 方法。动态代理对象不需要关心具体执行方法的名字，所有方法一律是以 Method 对象的形式存在。InvocationHandler 接口只有一个方法，即 invoke() 方法！它是对动态代理对象所有方法的唯一实现。也就是说无论调用动态代理对象上的哪个方法，其实底层都是在调用 InvocationHandler 的 invoke() 方法。

```java
public <T> T create(final Class<T> service) {
Utils.validateServiceInterface(service);
if (validateEagerly) {
    eagerlyValidateMethods(service);
}
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
        }
        if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        ServiceMethod<Object, Object> serviceMethod =
            (ServiceMethod<Object, Object>) loadServiceMethod(method);
        OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.adapt(okHttpCall);
        }
    });
}

ServiceMethod<?, ?> loadServiceMethod(Method method) {
ServiceMethod<?, ?> result = serviceMethodCache.get(method);
if (result != null) return result;

synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
    result = new ServiceMethod.Builder<>(this, method).build();
    serviceMethodCache.put(method, result);
    }
}
return result;
}
```

Retrofit.create() 方法主要做了如下三件事：
1. 根据目标 method 对象去调用 loadServiceMethod() 方法获取对应的 ServiceMethod 对象；
2. 通过 serviceMethod 和方法参数创建出对应的 OkhttpCall 对象；
3. 执行 serviceMethod 的 adapt() 方法并返回一个 Call 对象（ 根据具体业务需求而定 ）。

### ServiceMethod.Builder(Retrofit retrofit, Method method)

```java
Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
  this.methodAnnotations = method.getAnnotations(); // 获取方法上的所有注解
  this.parameterTypes = method.getGenericParameterTypes(); // 获取所有方法参数的参数类型
  this.parameterAnnotationsArray = method.getParameterAnnotations(); // 获取所有方法参数的注解
}
```

```JAVA
public ServiceMethod build() {
  // 创建 CallAdapter 对象 @1
  callAdapter = createCallAdapter();
  // 获取方法的返回类型
  responseType = callAdapter.responseType();
  if (responseType == Response.class || responseType == okhttp3.Response.class) {
    throw methodError("'"
        + Utils.getRawType(responseType).getName()
        + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  // 根据 responseType 创建对应的数据转换器 @2
  responseConverter = createResponseConverter();

  // 解析方法上的所有注解
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }

  // 省略部分代码

  return new ServiceMethod<>(this);
}

private CallAdapter<T, R> createCallAdapter() {
    // 获取方法返回值类型的 Type 对象
    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
    }
    // 获取方法上所有的注解
    Annotation[] annotations = method.getAnnotations();
    try {
        // 遍历 callAdapterFactories 集合，根据方法的 returnType 和 annotations 检索出目标 CallAdapter 对象。
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) {
        throw methodError(e, "Unable to create call adapter for %s", returnType);
    }
}

private Converter<ResponseBody, T> createResponseConverter() {
    Annotation[] annotations = method.getAnnotations();
    try {
        // 原理和 createCallAdapter() 方法类似，同样是通过遍历 converterFactories 集合，然后根据 responseType 检索出目标 ResponseConverter 对象。
        return retrofit.responseBodyConverter(responseType, annotations);
    } catch (RuntimeException e) {
        throw methodError(e, "Unable to create converter for %s", responseType);
    }
}
```

ServiceMethod.build() 方法的第一步骤就是调用 createCallAdapter() 方法创建出一个 callAdapter 对象。还记得上面提到的动态代理对象的 invoke() 方法的最后一行吗？即 return serviceMethod.adapt(OkhttpCall)，其实方法内部就是调用了 callAdapter 对象的 adapt() 方法。

现在我们梳理一下当用户在调用接口中的某个方法时，Retrofit 在后台都为我们操了哪些心？

1. 调用 loadServiceMethod() 方法检查本地缓存中是否存在 method 对象对应的 ServiceMethod 对象；
2. 如果不存在，调用 ServiceMethod.Builder 的 builder() 方法，根据 method 对象的返回值类型和注解分别调用两个核心方法createCallAdapter() 和 createResponseConverter() 在各自的集合中检索出对应的 CallAdapter 对象和 ResponseConverter 对象；
3. 根据 ServiceMethod 对象和方法参数创建对应的 OkhttpCall 对象；
4. 默认情况下，最后一步应该是调用 ServiceMethod 对象的 adapt() 方法无脑返回 Call 对象给上层业务。但是！！！我们是一群有追求的程序员，不希望业务人员在拿到 Call 对象之后再手动地去调用 execute() 或 enqueue() 方法去发起请求。因此我们选择继承 CallAdapter.Factory 这个抽象类自定义一个 CallAdapterFactory，不仅要覆写了它的 get() 方法，目的是在遍历集合 callAdapterFactories 时判断当前接口是否可以使用该 CallAdapterFactory 进行处理，而且还需要修改其内部返回的 CallAdapter 对象的 adapt() 方法的执行逻辑，目的是不让它再无脑返回 Call 对象了，而是直接调用 Call 对象的 execute() 向目标服务端发起请求，并将已经反序列化好的结果返回给业务层。
5. 这一步其实对是第四步末尾处的细化，调用 Call 对象的 execute() 方法实际上走的还是 OkhttpCall 对象的 execute() 方法。
   1. execute() 方法内部首先调用 createRawCall() 方法，底层通过调用 ServiceMethod 对象的 toCall() 方法根据指定的 CallFactory 创建 OkhttpClient 以及对应的 Request；
   2. 接着调用 Call 对象的 execute() 方法正式向目标服务端发起请求；
   3. 最后再调用 parseResponse() 方法，其实底层调用的还是 ServiceMethod 对象的 toResponse() 方法，在该方法内部通过指定的 responseConverter 对 Body 进行反序列化。
