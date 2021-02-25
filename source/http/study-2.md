---
title: 实战：装简洁易用低耦合的网络层框架HiRESTful封装
---

## 目录

- 需求分析
- 疑难分析
- 效果展示
- 设计模式

### 需求分析

**统一收拢接口的入参，请求方式, 请求头，返回值，请求资源URL。方便接口维护与复用**。

**隔离三方网络请求框架，利于迭代跟替换**

- 支持GET,POST请求方式，POST支持表单和application/json
- 支持动态更换接口域名BaseUrl
- 支持动态修改接口相对路由
- 支持添加个性化的Header. 
- 支持传递8种基本类型数据
- 支持拦截器 ，支持拓展网络引擎实现方式，自定义序列化方式。

```java
  interface HiApiService{
    @GET("/cities/{province}")
    @POST(value = "/cities/{province}", formPost = false)
    @BaseUrl("https://api.devio.org/as/")
    @Headers({"auth-token:token"})
    HiCall<City> listCities(
            @Path("province") int provinceId,
            @Field("pageCount") int pageCount,
            @Filed("page") int page);
  }
```



### 疑难分析

- 如何实现动态拿到接口的实现类对象。如何动态实现接口中的方法。

```java
 interface HiApiService{
     @GET("/cities")
     HiCall<City> listCities(@Field("pageCount")int pageCount)
 }


class HiApiServiceImpl implements HiApiService{
      public HiCall<City> listCities(int pageCount){
         HiCall call =  new HiCall();
         call.url="/citites"
         return call;
     }
 }
```



- 注解处理器自动生成Api接口的实现类

```java
class HiRestfulProcesor extends AbstractProcessor{
   boolean process(.....,RoundEnvironment roundEnv){
      
     //generate file HiApiServiceImpl implements HiApiService
   }
}
```

- 动态代理

```java
Object proxy = Proxy.newProxyInstance(classloader, HiApiService.class, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args)  {
                return null;
            }
        })
```



### 效果实现

```java
//初始化 
String baseUrl = "https://api.devio.org/as/";
 HiRestful hiRestful = new HiRestful(baseUrl, new RetrofitCallFactory(baseUrl));
 hiRestful.addInterceptor(BizInterceptor());

//发起异步请求
 hiRestful.create(HiApiService.class)
          .listCities("mrme2014", 10, 1)
          .enqueue(new  HiCallback<JsonObject> {
            public void onSuccess(HiResponse<JsonObject> response ) {
                JsonObject data = response.data
            }

            public void onFailed(Throwable throwable) {

            }
    });
```

- Annotaions
<center><img src="/imgs/http/annotations.png" style="zoom:40%;"></center>

<br/>

- HiInterceptor

<img src="/imgs/http/HiInterceptor.png" alt="HiInterceptor" style="zoom: 60%;" />

<br/>

- HiCall

<img src="/imgs/http/HiCall.png" alt="HiCall" style="zoom:35%;" />

<br/>

- HiRestful

<center><img src="/imgs/http/HiRestful.png" style="zoom:35%;"></center>



### 设计模式

- 抽象工厂设计模式，抽象了对象创建的具体细节，创建的时候只需要用特定接口函数隔离创建细节。体现了 “对扩展开发，对修改封闭的设计原则”。
- 适用于每种产品 创建细节不同的场景,在Android源码中比比皆是啊

```java
class OkHttpCallFactory implements HiCall.Factory{
        public OkHttpCallFactory(){
            //初始化 okhttp
        }
        public HiCall newCall(HiRequest request) {
            //给OKhttpCall 设置一堆属性
            return null;
        }
    }
    
class RetrofitCallFactory implements HiCall.Factory{
        public RetrofitCallFactory(){
            //初始化 retrofit 
        }
        public HiCall newCall(HiRequest request) {
            //给retrofit call 设置一堆属性
            return null;
        }
    }
```



- 拦截器模式：使多个**对象**有机会处理**请求**，从而避免了请求的**发送者和接收者**之间的**耦合关系**。将这些对象连成**一条链**，并沿着这条链传递该请求。
- 使用场景：需实现特定功能，比如日志记录、登录判断、权限检查、全局Headers添加。

<img src="/imgs/http/interceptors2.png" alt="interceptors2" style="zoom:25%;" />

### 适配Retrofit

```java
Class RetrofitCallFactory implements HiCall.Factory{
  public RetrofitCallFactory(String baseUrl){
     Retrofit retrofit = Retrofit.Builder()
            .baseUrl(baseUrl)
            .build()
     apiService =  retrofit.create(RetrofitApiService.class)
  }
  
  HiCall newCall(HiRequest request){
    return RetrofitCall(request)
  }
  
  
  
  interface RetrofitApiService {
        @GET
        Call<ResponseBody> get(@HeaderMap Map<String, String> headers, @Url String url, 
                              @QueryMap(encoded = true) Map<String, String> paramMap)

        @POST
        Call<ResponseBody> post(@HeaderMap Map<String, String> headers, @Url String url,
                                @Body RequestBody body): 
    }
}
```
