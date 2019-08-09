---
layout: post
title: "Retrofit 2.0源码探索"
date: 2019-08-09
description: "Retrofit 2.0源码探索"
tag: android
---

#### Retrofit 2.0源码探索



<h3> Retrofit2.0的具体网络流程<h3

​                           


1. 通过解析网络请求接口的注解 获得 网络请求参数

2. 通过 动态代理 生成网络请求对象

3. 通过 网络请求适配器 将 网络请求对象 进行平台适配
   
   > 平台包括：Android、Rxjava、Guava和java8
   
4. 通过 网络请求执行器 发送网络请求

5. 通过 数据转换器 解析服务器返回的数据

6. 通过 回调执行器 切换线程（子线程 ->>主线程）

7. 用户在主线程处理返回结果

<h3> Retrofit实例创建</h3>	



```java
 Retrofit retrofit = new Retrofit.Builder()
                                 .baseUrl("http://xxxxxx/")
     							  //配置网络请求适配器 默认为ExecutorCallAdapterFactory
     					         .addCallAdapterFactory(
                                     newRxJavaCallAdapterFactory().create())
     							  //配置数据转换器 解析服务器返回的数据
                                 .addConverterFactory(GsonConverterFactory.create())
                                 .build();
```



Retrofit实例是**使用建造者模式通过Builder类**进行创建的

```java
Builder(Platform platform) {
  this.platform = platform;
}
public Builder() {
  this(Platform.get());
}
```



```java
class Platform {
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }
  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        //当前平台未Android 时返回
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
  
  .............

  static class Android extends Platform {
      
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }
    @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor        callbackExecutor) {
      if (callbackExecutor == null) throw new AssertionError();
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
	// 回调执行器 切换线程（子线程 ->>主线程）Android 默认的
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
}
```

<h3>构建Retrofit 时的代码</h3>



```Java
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }
  //配置网络请求执行器
  okhttp3.Call.Factory callFactory = this.callFactory;
    // 如果没指定，则默认使用okhttp
      // 所以Retrofit默认使用okhttp进行网络请求
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }
  
  Executor callbackExecutor = this.callbackExecutor;
    // 如果没指定，则默认使用Platform检测环境时的默认callbackExecutor
      // 即Android默认的callbackExecutor回调执行器
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  //配置网络请求适配器工厂（CallAdapterFactory）
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

  //  配置数据转换器工厂：converterFactory
  List<Converter.Factory> converterFactories =
      new ArrayList<>(1 + this.converterFactories.size());

  // 数据转换器工厂集合存储的是：默认数据转换器工厂（ BuiltInConverters）、自定义1数据转换器工厂
   //  （GsonConverterFactory）、自定义2数据转换器工厂....
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);
  //1. 获取合适的网络请求适配器和数据转换器都是从adapterFactories和converterFactories集合的首位-末 //位开始遍历
// 因此集合中的工厂位置越靠前就拥有越高的使用权限
  return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```

<h3> 创建网络请求接口的实例</h3>



```java
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
      // 判断是否需要提前验证
       // 具体方法作用：
      // 1. 给接口中每个方法的注解进行解析并得到一个ServiceMethod对象
      // 2. 以Method为键将该对象存入LinkedHashMap集合中
     // 特别注意：如果不是提前验证则进行动态解析对应方法（下面会详细说明），得到一个ServiceMethod对象，
      //最后存入到LinkedHashMap集合中，类似延迟加载（默认）
    eagerlyValidateMethods(service);
  }
    // 创建了网络请求接口的动态代理对象，即通过动态代理创建网络请求接口的实例 （并最终返回）
        // 该动态代理是为了拿到网络请求接口实例上所有注解
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();
 // 在 InvocationHandler类的invoke（）实现中，除了执行真正的逻辑（如再次转发给真正的实现类对象），还可
          //以进行一些有用的操作
         // 如统计执行时间、进行初始化和清理、对接口调用进行检查等。
        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
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
// 特别注意
// return (T) roxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces,  //InvocationHandler invocationHandler)
// 可以解读为：getProxyClass(loader, interfaces) //.getConstructor(InvocationHandler.class).newInstance(invocationHandler);
// 即通过动态生成的代理类，调用interfaces接口的方法实际上是通过调用InvocationHandler对象的invoke（）
//来完成指定的功能
private void eagerlyValidateMethods(Class<?> service) {
  Platform platform = Platform.get();
  for (Method method : service.getDeclaredMethods()) {
    if (!platform.isDefaultMethod(method)) {
      loadServiceMethod(method);
    }
  }
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

通过上面的代码最终获得网络请求接口的动态代理类

通过解析网络接口的方法注解 生成相应的`ServiceMethod` 

`serviceMethod.adapt(okHttpCall)`

```java
T adapt(Call<R> call) {
  return callAdapter.adapt(call);
}
```

callAdapter的根据网络请求方法的返回值类型来选择具体要用哪种CallAdapterFactory，然后创建具体的CallAdapter实例

```java
private CallAdapter<T, R> createCallAdapter() {
  Type returnType = method.getGenericReturnType();
  if (Utils.hasUnresolvableType(returnType)) {
    throw methodError(
        "Method return type must not include a type variable or wildcard: %s", returnType);
  }
  if (returnType == void.class) {
    throw methodError("Service methods cannot return void.");
  }
  Annotation[] annotations = method.getAnnotations();
  try {
    //调用 retrofit 的 callAdapter 获得 CallAdapter
    return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}
```



```java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
  return nextCallAdapter(null, returnType, annotations);
}
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {
  checkNotNull(returnType, "returnType == null");
  checkNotNull(annotations, "annotations == null");

  int start = callAdapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      //
    CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }

  ........
}
```



```java
//不同的CallFactory 的get 实现不同 这是 RxJava2CallAdapterFactory
@Override
public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
  Class<?> rawType = getRawType(returnType);

  if (rawType == Completable.class) {
    return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
        false, true);
  }

  boolean isFlowable = rawType == Flowable.class;
  boolean isSingle = rawType == Single.class;
  boolean isMaybe = rawType == Maybe.class;
  if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
    return null;
  }
........
  return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
      isSingle, isMaybe, false);
}
```

Retrofit采用了外观模式统一调用创建网络请求接口实例和网络请求参数配置的方法，具体细节是：

- 动态创建网络请求接口的实例**（代理模式 - 动态代理）** 
- 创建 `serviceMethod` 对象**（建造者模式 & 单例模式（缓存机制））** 
- 对 `serviceMethod` 对象进行网络请求参数配置：通过解析网络请求接口方法的参数、返回值和注解类型，从Retrofit对象中获取对应的网络请求的url地址、网络请求执行器、网络请求适配器 & 数据转换器。**（策略模式）** 
- 对 `serviceMethod` 对象加入线程切换的操作，便于接收数据后通过Handler从子线程切换到主线程从而对返回数据结果进行处理**（装饰模式）** 
- 最终创建并返回一个`OkHttpCall`类型的网络请求对象



`Retrofit` 本质上是一个 `RESTful` 的`HTTP` 网络请求框架的封装，即通过 大量的设计模式 封装了 `OkHttp` ，使得简洁易用。具体过程如下：

1.  `Retrofit` 将 `Http`请求 抽象 成 `Java`接口
2. 在接口里用 注解 描述和配置 网络请求参数
3. 用动态代理 的方式，动态将网络请求接口的注解 解析 成`HTTP`请求
4. 最后执行`HTTP`请求



