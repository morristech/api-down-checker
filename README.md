# Api Down Checker
> Is my API down or what?

This library let's you easily get notified in your client code when your API is not working, but some other trusted endpoint is. We use this to determine when the API is down.

<br/>
<p align="center">
<b><a href="#features">Features</a></b>
|
<b><a href="#usage">Usage</a></b>
|
<b><a href="#download">Download</a></b>
|
<b><a href="#under-the-hood">Under the hood</a></b>
|
<b><a href="#customization">Customization</a></b>
|
<b><a href="#contributions">Contributions</a></b>
</p>
<br/>


### Features
- Set your trusted and untrusted endpoints.
- Checks connectivity to *Google.com* by default.
- Customize the "is ok" criteria of the endpoints.
- Add an [OkHttp](http://square.github.io/okhttp/) Interceptor to automatically perform the checking when you receive failing responses.
- Caches the "is api down" result for 10 seconds to avoid doing too many requests.


### Usage
Create a instance of ApiDownChecker
```java
ApiDownChecker checker = new ApiDownChecker.Builder()
  .check("http://my.api/status")
  .trustGoogle() // default if none specified
  .build();
```

Now you can use it on demand
```java
boolean isMyApiDown = checker.isApiDown();
```


Or you can let the library automatically check if your when it detects network or http errors by adding an interceptor to your OkHttpClient. When using the interceptor, you will receive an ApiDownException when performing a request

```java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
  .addInterceptor(ApiDownInterceptor.create()
    .checkWith(checker)
    .build())
  .build();
  
// Then catch the exception when using the client...
Request request = new Request.Builder()
  .get().url("http://my.api/method")
  .build();
try {
  okHttpClient.newCall(request).execute();
} catch (ApiDownException e) {
  // Your api is down, warn the user or something
}
```

And that's all! If you're using Retrofit you can handle the exception in your API call or your callback.


### Download
**Sorry!** As of right now the library isn't published to any dependency repository. [Grab the JAR](https://github.com/scm-spain/ApiDownChecker/releases) and import in into your project.

This will be changed soon, don't worry.


### Under the hood

What do you do when a web page doesn't load? You check [google.com](www.google.com) to see if it's that web's problem or your connection.
The idea behind this library is the same. You might have some backend system to nofity you when the API is down. But in our experience it doesn't always work well. So this is our *workaround* for that.

This library is based on two [ApiValidator](https://github.com/scm-spain/ApiDownChecker/blob/master/apidownchecker/src/main/java/net/infojobs/apidownchecker/ApiValidator.java) instances, a **trusted** and an **untrusted** validator, which are consulted to determine if your API is down. The **trusted** validator is someone you trust will *always* work, like google's home page. The **untrusted** validator represents your api, and tells you whether your API is responding properly or not.

A validator is a simple interface that tells if it's working fine. 
```java
public interface ApiValidator {
    boolean isOk();
}
```


### Customization

You can customize some aspect of the behavior:

##### Validators
The library includes a simple implementation of the [ApiValidator](https://github.com/scm-spain/ApiDownChecker/blob/master/apidownchecker/src/main/java/net/infojobs/apidownchecker/ApiValidator.java), the [HttpValidator](https://github.com/scm-spain/ApiDownChecker/blob/master/apidownchecker/src/main/java/net/infojobs/apidownchecker/HttpValidator.java), which receives an url and answers **isOk** if that url is responding a successful HTTP status code (2xx). If the api responds with an errored code or throws an exception (like Unknown host or Timeout) the validator gives a negative response.

```java
public class HttpValidator implements ApiValidator {

    // stuff...

    protected boolean validateResponse(Response response) {
        return response.isSuccessful();
    }
}
```

Maybe you need to do a different checking. Maybe your status endpoint always responds with a 200 status code and you need to read some value in the return body. Or maybe you must use some kind of special authentication. In that case, just implement your own ApiValidator or extend HttpValidator.

```java
public class MyApiValidator extends HttpValidator {

    public MyApiValidator(OkHttpClient httpClient) {
        super(httpClient, "http://my.api/status");
    }

    @Override
    protected boolean validateResponse(Response response) {
        // parse a json, read a header or something
    }
}
```

Pass the validators to the Builder
```java
ApiDownChecker checker = new ApiDownChecker.Builder()
  .check(myApiValidator)
  .trust(myTrustedValidator)
  .build();
```

Note: when you pass a String as a parameter to `check()` or `trust()` a new HttpValidator is created for you.

##### OkHttpClient
By default a new OkHttpClient is used when building ApiDownChecker. You can use a custom implementation by using `withClient()` in the builder.

```java
OkHttpClient client = getSomeCustomOkHttpClient();
ApiDownChecker checker = new ApiDownChecker.Builder()
  .check("http://my.api")
  .withClient(client)
  .build();
```

Warning: note that this OkHttp client cannot be the same that the one used to consume your API if you want to use the automagical Interceptor. That would be a cyclic dependency, and the ApiDownException thrown by the interceptor would be capture by itself.

##### Logging
You can add a simple logger to follow the library operation. By default an empty logger is used, but you can add your own.

```java
ApiDownChecker checker = new ApiDownChecker.Builder()
  .check("http://my.api/status")
  .logWith(new Logger() {
      @Override
      public void log(String message) {
          Log.w(TAG, message);
      }
  })
  .build();
```


### Contributions
For bugs, requests, questions and discussions please use the [Github Issues](https://github.com/scm-spain/ApiDownChecker/issues).
