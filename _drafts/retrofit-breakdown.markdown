---
layout: post
title: Library Breakdown - Retrofit 2.0.2
categories: breakdown
---

This is part of a series of library breakdowns to better understand what these libraries do for us. Diving deeper into the internals lets us appreciate what good software looks like and give us confidence in emulating these practices in our own code.

### Constructing a Retrofit instance

[Retrofit][retrofit] is a typesafe REST client for Java and Android. Basic usage, as noted in the Javadoc, looks like:

```java
Retrofit retrofit = new Retrofit.Builder()
  .baseUrl("https://api.example.com/")
  .addConverterFactory(GsonConverterFactory.create())
  .build();
```

A Retrofit instance is immutable (and thus, thread-safe), and is constructed with a standard Builder pattern--the Builder class represents the mutable instance that allows you to configure Retrofit's internals before calling build, finalizing the Retrofit object.

```java
public static class Builder {

  // ... Other setters

  public Retrofit build() {
    if (baseUrl == null) {
      throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
      callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
      callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
    adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

    return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
        callbackExecutor, validateEagerly);
  }	
}
```
Here you can see what the core components of Retrofit are:

- **`callFactory`**: The factory that constructs `Call` objects. A Call makes a request and receives a response. By default it uses `OkHttpClient`. 
- **`baseUrl`**: The HttpUrl, part of OkHttp. If you set the `baseUrl` as a String, Retrofit converts it with `HttpUrl.parse`.
- **`converterFactories`**: These factories define how to convert `ResponseBody` and `RequestBody` to your POJOs. That's what `GsonConverterFactory` does: it takes a `ResponseBody#charStream()`, creates a `JsonReader` from it, then feeds it to a `TypeAdapter<T>` to give you the object you're looking for. More on `Converter` and `Converter.Factory` later.
- **`adapterFactories`**: By default Retrofit interfaces must return `Call<T>` objects. In a similar fashion as converter factories, which convert the response/request to another type, you can provide a converter that converts the Call object itself. The most common one I've seen is `RxJavaCallAdapterFactory` (`Observable`), but the other built in ones include `Java8CallAdapterFactory` (`CompleteableFuture`) and `GuavaCallAdapterFactory` (`ListenableFuture`). More on this later.
- **`callbackExecutor`**: The Executor to perform Callback methods on. On Android, this returns an Executor that passes all tasks to the main thread (via a Handler constructed with `Looper.getMainLooper()`). On IOS, `NSOperationQueue#addOperation` and `NSOperationQueue#getMainQueue` are obtained reflectively and invoked during `execute`. Everything else, the default executor is null--meaning callbacks are all executed on the same thread as `Call#enqueue`.
- **`validateEagerly`**: 


 [retrofit]: http://square.github.io/retrofit/