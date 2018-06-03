---
title: Retrofit基础
date: 2017-04-17 10:01:14
---

## 资料

* **[Github] [Github]**
* **[官网] [官网]**
* **[深入浅出 Retrofit，这么牛逼的框架你们还不来看看？] [深入浅出 Retrofit，这么牛逼的框架你们还不来看看？]**
* **[你真的会用Retrofit2吗?Retrofit2完全教程] [你真的会用Retrofit2吗?Retrofit2完全教程]**
* **[Retrofit2 使用方法] [Retrofit2 使用方法]**


## 摘要

* [0 简介](#0-简介)
* [1 配置](#1-配置)
* [2 基础用法](#2-基础用法)
* [3 注解详解](#3-注解详解)
* [4 Retrofit设置](#4-Retrofit设置)
* [5 RxJava支持](#5-RxJava支持)

### 0 简介
Retrofit结构简单，源码只有37个文件，其中22个文件是注解以及HTTP相关的

### 1 配置
### 1.1导入
模组build.gradle文件
```groovy
compile 'com.squareup.retrofit2:retrofit:2.2.0'
```
### 1.2 混淆
```proguard
# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions
```
### 2 基础用法
### 2.1 创建Retrofit实例
> 注：Retrofit2的`baseUrl`必须以`/(斜线)`结束，不然后抛出`IllegalArgumentException`异常

```java
// Retrofit 实例
Retrofit retrofit = new Retrofit.Builder()
.baseUrl("https://api.github.com/")  // 主机
.build();
```
### 2.2 定义接口和Bean
> 这里`interfacer`无法直接调用该方法，需要用`Retrofit`实例创建一个GitHubService代理对象

```java
// Bean
public class Repo {
public long id;
public String name;
}

// 请求接口
public interface GitHubService {
@GET("users/{user}/repos")
Call<List<Repo>> listRepos(@Path("user") String user);
}
```
### 2.3 发起请求

> 下面包含了`异步请求`、`同步请求`的示例，同时也包含了`取消请求`、`克隆请求`的示例

```java
// 网络请求
Call<List<Repo>> call = retrofit.create(GitHubService.class)  // 获取服务
.listRepos("octocat"); // 获取请求

// 响应处理，异步
call.enqueue(new Callback<List<Repo>>() {  // 获取响应
@Override
public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
if (response.isSuccessful()) {  // 响应码 200~300
Log.d(TAG, "onResponse: " + response.body()); // 显示 List<Repo>
}
}

@Override
public void onFailure(Call<List<Repo>> call, Throwable t) {

}
});

// 响应处理，同步
try {
Response<List<Repo>> response = call.execute();
} catch (IOException e) {
e.printStackTrace();
}

// 响应处理，取消 Call
call.cancel();

// 响应处理，克隆 clone
call.clone();
```

### 3 注解详解
Retrofit中的注解共分为三类，分别是`HTTP请求方法`、`标记类`、`参数类`
### 3.1 HTTP请求方法
#### 3.1.1 对应HTTP标准请求方法的注解

> 注解接收一个字符串表示接口路径，与BaseUrl组成完整Url

请求方法类型`@GET/@POST/@PUT/@DELETE/@PATCH/@HEAD/@OPTIONS`
```java
@GET("users/list")
@GET("users/list?sort=desc")
```

#### 3.1.2 @HTTP注解

代替以上任意一个注解，有3个属性：`method`、`payh`、`hasBody`

```java
public interface BlogService {
/**
* method 表示请求的方法，区分大小写
* path表示路径
* hasBody表示是否有请求体
*/
@HTTP(method = "GET", path = "blog/{id}", hasBody = false)
Call<ResponseBody> getBlog(@Path("id") int id);
}

```
### 3.2 标记类
#### 3.2.1 表单请求
1.@FormUrlEncoded 
> 表示请求体是一个Form表单，例如在网站上登录页面就是此种方式
Content-Type:application/x-www-form-urlencoded

2.@Multipart
> 表示请求体是一个支持文件上传的Form表单，例如在网站上传文件就是使用此种方式
Content-Type:multipart/form-data

#### 3.2.2 其他标记

1.@Streaming
> 表示响应的数据用流的形式返回，如果使用该注解，默认会把数据全部载入内存之后通过流读取内存中的数据。返回数据较大时使用此注解

### 3.3 参数类

### 3.3.1 请求地址
`@Query`参数为null,会自动忽略

`@Path/@Query/@QueryMap/@Url`
```java
// 通过 @Path 替换相对地址中的 {}
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId);

// 通过 @Query 设置请求参数
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);

// 通过 @QueryMap 设置请求参数 Map
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);
```
### 3.3.2 请求头
1.静态请求头

`@Headers`

请求头不会覆盖,同名头都会包含进请求头中
```java
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();

@Headers({
"Accept: application/vnd.github.v3.full+json",
"User-Agent: Retrofit-Sample-App"
})
@GET("users/{username}")
Call<User> getUser(@Path("username") String username);
```
2.动态请求头

`@Header`

值为null，请求头被删除，否则值调用toString();
```java
@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)
```

3.批量请求头

所有请求都需要请求头，使用`OkHttp interceptor`添加

### 3.3.3 请求体
1.非表单请求体
非表单请求体 `@Body`

使用指定Retrofit的转换器转化请求体，如果没有就使用RequestBody
`例如Gson转换器将请求体转换为Json数据`
```java
@POST("users/new")
Call<User> createUser(@Body User user);
```
2.表单请求体

表单请求体 `@Field/@FieldMap/@Part/@PartMap`

> 1.@Field和FieldMao与@FormUrlEncoded注解配合使用
2.@Part和PartMap与Multipart注解配合使用，可以进行文件上传
3.FieldMap的接受类型是Map<String,String>,非String类型会调用其toString方法
4.PartMap的默认接受类型是Map<String,RequestBody>,非RequestBody类型会通过Converter转换
5.{占位符}和`PATH`尽量只用在URL的path部分，URL中的参数使用`Query`和`QueryMap`代替,保证接口定义的简洁
6.`Query`、`Field`和`Part`这三者都支持**数组**和实现了`Iterable`接口的类型，如`List`,`Set`等，方便向后台传递数据

```java
@FormUrlEncoded  // Form表单
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);

@Multipart  // 上传文件表单
@PUT("user/photo")
Call<User> updateUser(@Part("photo") RequestBody photo, @Part("description") RequestBody description);
```

### 4 Retrofit设置
### 4.1 转换器(Converter)
ResponseBody为默认响应体类型，即`Call<ResponseBody>`,可设置转换器转换
* Gson: `com.squareup.retrofit2:converter-gson`
* Jackson: `com.squareup.retrofit2:converter-jackson`
* Moshi: `com.squareup.retrofit2:converter-moshi`
* Protobuf: `com.squareup.retrofit2:converter-protobuf`
* Wire: `com.squareup.retrofit2:converter-wire`
* Simple XML: `com.squareup.retrofit2:converter-simplexml`
* Scalars (primitives, boxed, and String): `com.squareup.retrofit2:converter-scalars`

设置转换器
```java
Retrofit retrofit = new Retrofit.Builder()
.baseUrl("https://api.github.com")
.addConverterFactory(GsonConverterFactory.create(gson))  // 设置转换器，可传入Gson实例
.build();

```

### 4.2 调用适配器(CallAdapter)

`Converter`是对于`Call<T>`中`T`的转换，而`CallAdapter`则可以对`Call`进行转换
* Rxjava `com.squareup.retrofit2:adapter-rxjava`
* Guava `com.squareup.retrofit2:adapter-guava`
* Java8 `com.squareup.retrofit2:adapter-java8`

设置调用适配器
```java
Retrofit retrofit = new Retrofit.Builder()
.baseUrl("https://api.github.com")
.addCallAdapterFactory(RxJavaCallAdapterFactory.create())// 设置调用适配器
.build();
```
### 5 RxJava支持
> 需要在build.gradle配置RxJava调用适配器

使用`RxJava`获取`Header`和`响应码`的方法
1.用`Observable<Response<T>>`代替`Observable<T>`,这里的`Response`指`retrofit.Response`
2.用`Observable<Result<T>>`代替`Observable<T>`，这里的`Result`是指`retrofit2.adapter.rxjava.Result`,这个`Result`中包含了`Response`的实例


```java
// Bean
public class Repo {
public long id;
public String name;
}

// 请求接口
public interface GitHubService {
@GET("users/{user}/repos")
Observable<List<Repo>> listReposBy(@Path("user") String user);
}

// Retrofit 实例
Retrofit retrofit = new Retrofit.Builder()
.baseUrl("https://api.github.com/")
.addConverterFactory(GsonConverterFactory.create())
.addCallAdapterFactory(RxJava2CallAdapterFactory.create())  // 转换器
.build();

// 网络观察者
Observable<List<Repo>> observable = retrofit.create(GitHubService.class)
.listReposBy("octocat");

// 数据处理
observable.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(new Consumer<List<Repo>>() {
@Override
public void accept(@NonNull List<Repo> repos) throws Exception {
Log.d(TAG, "Rxjava2 - onResponse: " + repos);
}
});
```

[Github]:
https://github.com/square/retrofit
[官网]:
http://square.github.io/retrofit/
[你真的会用Retrofit2吗?Retrofit2完全教程]:
http://www.jianshu.com/p/308f3c54abdd
[Retrofit2 使用方法]:
http://www.jianshu.com/p/93bd0b322dc3
[深入浅出 Retrofit，这么牛逼的框架你们还不来看看？]:
http://www.println.net/post/deep-in-retrofit
