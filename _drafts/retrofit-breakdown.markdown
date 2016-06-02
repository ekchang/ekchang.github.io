---
layout: post
title: Library Breakdown - Retrofit 2.0.2
categories: breakdown
---

This is part of a series of library breakdowns to better understand what these libraries do for us. Diving deeper into the internals lets us appreciate what good software looks like and give us confidence in emulating these practices in our own code. These breakdowns assume you know how to use them, but have always been wondering how they work and what they're doing under the hood.

### Constructing a Retrofit instance

[Retrofit][retrofit] is a typesafe REST client for Java and Android. Basic usage, as noted in the Javadoc, looks like:

```java
Retrofit retrofit = new Retrofit.Builder()
  .baseUrl("https://api.github.com/")
  .addConverterFactory(GsonConverterFactory.create())
  .build();
```

A Retrofit instance is immutable (and thus, thread-safe), and is constructed with a standard Builder pattern.

```java
public static class Builder {

  // ... Other setters

  public Retrofit build() {
    if (baseUrl == null) { throw new IllegalStateException("Base URL required."); }

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) { callFactory = new OkHttpClient(); }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) { callbackExecutor = platform.defaultCallbackExecutor(); }

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

- **`callFactory`**: The factory that constructs `Call` objects. This is what makes the actual network calls--a Call makes a request and receives a response. By default it uses `OkHttpClient`. 
- **`baseUrl`**: The HttpUrl, part of OkHttp. If you set the `baseUrl` as a String, Retrofit converts it with `HttpUrl.parse`.
- **`converterFactories`**: This converts your requests and responses--`ResponseBody` and `RequestBody`--to your POJOs. That's what `GsonConverterFactory` does: it takes a `ResponseBody#charStream()`, creates a `JsonReader` from it, then feeds it to a `TypeAdapter<T>` to give you the object you're looking for. More on `Converter` and `Converter.Factory` later.
- **`adapterFactories`**: By default Retrofit interfaces must return `Call<T>` objects. In a similar fashion as converter factories, which convert the response/request to another type, you can provide a converter that converts the Call object itself. The most common one I've seen is `RxJavaCallAdapterFactory` (`Observable`), but the other built in ones include `Java8CallAdapterFactory` (`CompleteableFuture`) and `GuavaCallAdapterFactory` (`ListenableFuture`). More on this later.
- **`callbackExecutor`**: The Executor to perform Callback methods on. On Android, this returns an Executor that passes all tasks to the main thread (via a Handler constructed with `Looper.getMainLooper()`). On IOS, `NSOperationQueue#addOperation` and `NSOperationQueue#getMainQueue` are obtained reflectively and invoked during `execute`. Everything else, the default executor is null--meaning callbacks are all executed on the same thread as `Call#enqueue`.
- **`validateEagerly`**: By default, whenever you call one of your REST interface methods (GET, POST, etc), Retrofit lazily creates these methods for you internally as `ServiceMethods`. Once created, they're cached in a map. There are some reflective calls needed to parse your defined method parameters and annotations--if you deem the construction of these methods to be a bottleneck in your app, you can turn on eager validation to move up the reflective operations ahead of time. Typically the reflective cost of creating these methods are outweighed by the actual network calls, so it is fine to leave it as false by default.

The heart of Retrofit lies in its three factories: one which defines the client that makes the network calls, one that acts as a bridge between request/responses and your Java objects, and one that acts as a converter for the `Call` objects.

### Service Creation

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}

// TODO set up retrofit

GitHubService service = retrofit.create(GitHubService.class);
```

The magical create method, whose javadoc is longer than the implementation itself. Here's what it's doing (checks and default/platform method routing omitted):

```java
public <T> T create(final Class<T> service) {

    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
It is clear what is happening under the hood. A [proxy][proxy] instance is created in which every method defined by the interface is used to create an `OkHttpCall`. The call adapter then applies any user-defined transformation to convert the Call object to whatever else they want to wrap `T` (Observable, etc). The default CallAdapter returns the Call itself.

### OkHttpCall - an okhttp3.RealCall wrapper

The default Call returned is always an instance of `OkHttpCall`, which is backed by an `okhttp3.Call`. Given a default `Call.Factory` of `OkHttpClient`, this will actually be an `okhttp3.RealCall` instance. Let's discuss how this is all created.

The two most common uses of a Call is to use `execute()` or `enqueue()` for synchronous and asynchronous requests respectively. These have some guards, error checks, and cancel checks, but the main goal is to invoke the backing `okhttp3.Call` `execute()` and `enqueue()` respectively.

```java
// OkHttpCall.java

public Response<T> execute() throws IOException {
  okhttp3.Call call;

  synchronized (this) {
    // Checks/guards omitted
    ... 
    call = createRawCall();
  }

  return parseResponse(call.execute());
}

private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) { throw new NullPointerException("Call.Factory returned null."); }
    return call;
}
```

Now we see where the `ServiceMethod` comes in. A `Request` is created from the `ServiceMethod` configurations and provided arguments and invoke the CallFactory's `newCall(Request)` to get a new Call. What does `OkHttpCall#newCall(Request)` look like?

```java
// okhttp3.OkHttpClient.java

public Call newCall(Request request) { return new RealCall(this, request); }
```

We won't be diving into `RealCall` -- that's more suitable for an OkHttp breakdown.

The `enqueue()` implementation is identical to `execute()` except `call.enqueue()` is called at the end. An `okhttp3.Callback` which delegates all responses to the callback provided when invoking `Call#enqueue(Callback<T> callback)`.

```java
// OkHttpCall.java

public void enqueue(final Callback<T> callback) {
  // Same implementation as execute except for return statement

  ...

  call.enqueue(new okhttp3.Callback() {
    @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
        throws IOException {
      Response<T> response;
      try {
        response = parseResponse(rawResponse);
      } catch (Throwable e) {
        callFailure(e);
        return;
      }
      callSuccess(response);
    }

    @Override public void onFailure(okhttp3.Call call, IOException e) { callback.onFailure(OkHttpCall.this, e); }

    private void callFailure(Throwable e) { callback.onFailure(OkHttpCall.this, e); }

    private void callSuccess(Response<T> response) { callback.onResponse(OkHttpCall.this, response); }
  });
}
```

Regardless of whether you call `execute()` or `enqueue()`, a successful response is passed through `parseResonse()`. The parse attempts to strip the body from the response and return `Response.success(body, response)` or `Response.error()`. These are static factory methods for getting `Responses`:

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  // Remove the body's source (the only stateful object) so we can pass the response along.
  rawResponse = rawResponse.newBuilder()
      .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
      .build();

  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
    // Buffer the entire body to avoid future I/O.
    ResponseBody bufferedBody = Utils.buffer(rawBody);
    rawBody.close();
    return Response.error(bufferedBody, rawResponse);
  }

  if (code == 204 || code == 205) { return Response.success(null, rawResponse); }

  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  T body = serviceMethod.toResponse(catchingBody);
  return Response.success(body, rawResponse);
}
```

HTTP status codes between 200-300 encompass success and redirects; outside of those ranges are client/server errors. 204 and 205 codes have no body. Note that the body type is determined via the `ServiceMethod`, which runs the `ResponseBody` through its assigned `Converter<ResponseBody, T>`. 

### Converters

Converters are what you use to transform one type to another. In the context of Retrofit, you're interested in converting objects into `RequestBody` and `ResponseBody` into objects--otherwise every service interface you define would be forced to have a return type of `Call<ResponseBody>`. The `BuiltInConverters` factory includes a few default converters, namely ones which don't do any conversion and pass the body through.

Here's the complete `Converter` interface and `Converter.Factory` class:

```java
public interface Converter<F, T> {
  T convert(F value) throws IOException;

  /** Creates {@link Converter} instances based on a type and target usage. */
  abstract class Factory {
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) { return null; }

    public Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) { return null; }

    public Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) { return null; }
  }
}
```

When the `ServiceMethod` is created (upon first service method call, in which it is cached afterward, or at Retrofit creation if we `validateEagerly`), Retrofit iterates through the converter factories to find a factory that can support the required return type for the service method and generates the Converter:

```java
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,
      Type type, Annotation[] annotations) {
  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations, this);
    if (converter != null) {
      return (Converter<ResponseBody, T>) converter;
    }
  }
}
```

When a factory returns a null Converter, it is saying "sorry, I don't support converting this type" and then Retrofit tries again with the next factory. This seems trivial to mention, but this implementation is important because _the order in which you register converters matters_. Recall the Retrofit setup gist:

```java
Retrofit retrofit = new Retrofit.Builder()
  .baseUrl("https://api.github.com/")
  .addConverterFactory(GsonConverterFactory.create())
  .addConverterFactory(ProtoConverterFactory.create()) // Incorrect usage - Never gets used!
  .build();

public final GsonConverterFactory extends Converter.Factory {

  private final Gson gson;

  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonRequestBodyConverter<>(gson, adapter);
  }
}
```

`GsonConverterFactory` will _never return a null converter_ and thus if you register any converter factories after `GsonConverterFactory`, they will be completely ignored. This is documented in `GsonConverterFactory` javadoc.

### Call Adapters

By default, every service method returns a `Call<T>`. Recall that the default will be an `OkHttpCall<T>`, using an `okhttp3.Call` under the hood. 

```java
ServiceMethod serviceMethod = loadServiceMethod(method);
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.callAdapter.adapt(okHttpCall);
```

When a method is invoked, the resulting `OkHttpCall` is passed through the call adapter--which adapts a Call object to another type. 

```java
public interface CallAdapter<T> {

  Type responseType();

  <R> T adapt(Call<R> call);

  abstract class Factory {
    public abstract CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit);

    protected static Type getParameterUpperBound(int index, ParameterizedType type) { 
      return Utils.getParameterUpperBound(index, type); 
    }

    protected static Class<?> getRawType(Type type) { return Utils.getRawType(type); }
  }
}
```

Naturally, the default adapter returns the call itself:

```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {

  public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) { return null; }

    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() { return responseType; }

      @Override public <R> Call<R> adapt(Call<R> call) { return call; }
    };
  }
}
```

It is worth looking at `RxJavaCallAdapterFactory` to see how Call objects are adapted to Observables:

```java
private CallAdapter<Observable<?>> getCallAdapter(Type returnType, Scheduler scheduler) {

  Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);

  Class<?> rawObservableType = getRawType(observableType);

  if (rawObservableType == Response.class) {
    ...
    Type responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    return new ResponseCallAdapter(responseType, scheduler);
  }

  if (rawObservableType == Result.class) {
    ...
    Type responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    return new ResultCallAdapter(responseType, scheduler);
  }

  return new SimpleCallAdapter(observableType, scheduler);
}
```

 [retrofit]: http://square.github.io/retrofit/
 [proxy]: https://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Proxy.html