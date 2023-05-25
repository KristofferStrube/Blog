### Classic example of handling exceptions from JSInterop
With the current way that Blazor propagates errors to us, we are often left with some rather simple ways of finding out what actually happened. Let's see what happens when we try to call some JS that will fail. The example we will work with is invoking a method that does not exist but which we expect might throw a JS `AbortError`.
```razor
@inject IJSRuntime JSRuntime

@code {
    protected override async Task OnInitializedAsync()
    {
        await JSRuntime.InvokeVoidAsync("nonExistingMethod");
    }
}
```
We will see the following error in the console of our browser.
```console
crit: Microsoft.AspNetCore.Components.WebAssembly.Rendering.WebAssemblyRenderer[100]
      Unhandled exception rendering component: Could not find 'nonExistingMethod' ('nonExistingMethod' was undefined).
      Error: Could not find 'nonExistingMethod' ('nonExistingMethod' was undefined).
          at http://localhost:5278/_framework/blazor.webassembly.js:1:328
          at Array.forEach (<anonymous>)
          at a.findFunction (http://localhost:5278/_framework/blazor.webassembly.js:1:296)
          ... This continues with a very long stack trace that is not relevant to the exception. Not until this part which is from the .NET call stack:
   at Microsoft.JSInterop.JSRuntime.<InvokeAsync>d__16`1[[Microsoft.JSInterop.Infrastructure.IJSVoidResult, Microsoft.JSInterop, Version=7.0.0.0, Culture=neutral, PublicKeyToken=adb9793829ddae60]].MoveNext()
   at Microsoft.JSInterop.JSRuntimeExtensions.InvokeVoidAsync(IJSRuntime jsRuntime, String identifier, Object[] args)
   at test_project.Pages.Index.OnInitializedAsync() in F:\repos\test-project\Pages\Index.razor:line 8
   at Microsoft.AspNetCore.Components.ComponentBase.RunInitAndSetParametersAsync()
   at Microsoft.AspNetCore.Components.RenderTree.Renderer.GetErrorHandledTask(Task taskToHandle, ComponentState owningComponentState)
```
And if we try to capture the exception and read its `Message` we still get the same exception except we don't get the .NET call stack.
```razor
@inject IJSRuntime JSRuntime

<pre><code>
@exceptionMessage
</code></pre>

@code {
    string? exceptionMessage;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            await JSRuntime.InvokeVoidAsync("nonExistingMethod");
        }
        catch (JSException jsex)
        {
            exceptionMessage = jsex.Message;
        }
    }
}
```
And this message isn't really a nice user-facing error to get. So what we have done earlier is either to forget entirely what the exception said and just write something like **"Something bad happened!"** which doesn't help anyone. Or the better case which is to check for matching parts of the exception message. That could look something like this if we expect that JS could either throw an `AbortError` or some other unexpected exception.
```
@inject IJSRuntime JSRuntime
@inject ILogger Logger

<span style="color: red;">@exceptionMessage</span>

@code {
    string? exceptionMessage;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            await JSRuntime.InvokeVoidAsync("nonExistingMethod");
        }
        catch (JSException jsex)
        {
            if (jsex.Message.Contains("is aborted"))
            {
                exceptionMessage = "The execution of the method was canceled.";
            }
            else
            {
                exceptionMessage = "An unexpected error happened.";
                Logger.LogError(exceptionMessage, jsex);
            }
        }
    }
}
```
This is a bit hand-wavy. We first need to know that the exception message contained the words `"is aborted"` and we then have to hope that no other exception could happen that would also contain those words which would make for a false positive.


### How Blazor currently handles errors in JSInterop
It can be tough to find out why an error happens when an invocation is not successful as we saw above. The root of the problem is that we don't get all information about why an error happens when Blazor handles an error. Most noticeably we don't get information about the `name` of the specific type of error. To find out why let's look at the TypeScript code that handles Blazors dynamic invocation of functions in the [beginInvokeJSFromDotNet function in Microsoft.JSInterop.ts](https://github.com/dotnet/aspnetcore/blob/main/src/JSInterop/Microsoft.JSInterop.JS/src/src/Microsoft.JSInterop.ts#L401-L430):
```ts
beginInvokeJSFromDotNet: (asyncHandle: number, identifier: string, argsJson: string, resultType: JSCallResultType, targetInstanceId: number): void => {
    // Coerce synchronous functions into async ones, plus treat
    // synchronous exceptions the same as async ones
    const promise = new Promise<any>(resolve => {
        const synchronousResultOrPromise = findJSFunction(identifier, targetInstanceId).apply(null, parseJsonWithRevivers(argsJson));
        resolve(synchronousResultOrPromise);
    });

    // We only listen for a result if the caller wants to be notified about it
    if (asyncHandle) {
        // On completion, dispatch result back to .NET
        // Not using "await" because it codegens a lot of boilerplate
        promise
        .then(result => stringifyArgs([asyncHandle, true, createJSCallResult(result, resultType)]))
        .then(
            result => getRequiredDispatcher().endInvokeJSFromDotNet(asyncHandle, true, result),
            error => getRequiredDispatcher().endInvokeJSFromDotNet(asyncHandle, false, JSON.stringify([
                asyncHandle,
                false,
                formatError(error)
            ]))
        );
    }
},
```
They start off by making any potential synchronous method asynchronous by wrapping it in a promise. In the promise, they follow the identifier path to the function that should be invoked and then apply the arguments to that function using the `apply` method. This is all good and basically just means that they have fewer places in the code that varies. Next, they resolve the newly created promise and handle the two outcomes of the function. Either it was successful or it errored. In either case, we call back to .NET with some payload and a flag indicating if the invocation was successful. If it was successful we simply serialize the result and return that. But when it errors we instead call and return the result of the method [formatError function in Microsoft.JSInterop.ts](https://github.com/dotnet/aspnetcore/blob/main/src/JSInterop/Microsoft.JSInterop.JS/src/src/Microsoft.JSInterop.ts#L540-L546):
```ts
function formatError(error: Error | string): string {
    if (error instanceof Error) {
        return `${error.message}\n${error.stack}`;
    }

    return error ? error.toString() : "null";
}
```
This is what formats the error in the well-known format we see in the browser console or the `JSException`'s `Message` when making JSInteorp calls. Do you see what we are missing? Yes exactly, the `name` of the specific `Error`. We have lost the type of the original error. So that is exactly what we want to get back so that we can handle the original grouping of errors instead of resorting to matching on parts of the message.

### Mapping JS errors to C# exceptions
We want to do something similar to what the standard JSInterop has already done but include some other information. Sadly we can't change or use the existing code for the JSInterop functionality as most of it is internal. So instead we need to wrap the existing implementations and work around some of the mechanisms. But before we go there let's look at what different kinds of errors there are in JS.
#### Overview of JS error types
There are 6 types of errors that we should be concerned with when working within the browser.
- EvalError
- RangeError
- ReferenceError
- SyntaxError
- TypeError
- URIError

Each of these is used in a different scenario and can also be referred to as _Native Error Types_.
Apart from these we also have some standard exceptions called the `DOMException`s. They are generally used for errors that happens as an effect of the user, the browser, or the system doing something unexpected.
`DOMException`s are also errors, so we will simply refer to both JS errors and JS exceptions as errors from now on. There are many different kinds of `DOMExceptions` as well that are defined in the [WebIDL standard specifications](https://webidl.spec.whatwg.org/#idl-DOMException-error-names). They differ only by having different names and then of cause by being used in different scenarios. There are `29` predefined error names that are currently valid, so let's not list them all. The ones you might already know could be the `NotFoundError`, the `NotSupportedError`, and of cause the `AbortError`.

#### Capturing and rethrowing in JavaScript
The first step after having made clear that we are interested in the name of the exceptions is to somehow get this information into the exception that gets thrown at us. To do this we need to catch all exceptions on the JS side and rethrow them back for the Blazor implementation to handle. We could do this in all places where we need to invoke a method, but that would mean that we would need custom JS wrapper methods for **ALL** JS invocations we wanted to make. Instead, let's make a simple method that can execute any general method similar to how the Blazor implementation does it, but then throw another exception. We make a new JS module (a JS file with exported functions) and start with the following function which we will import and call later.
```js
export async function callAsyncGlobalMethod(identifier, args) {
    return await callAsyncInstanceMethod(window, identifier, args);
}
```
The method has two parameters. The first is the identifiers for the method that we will invoke and the second is the arguments that we will apply to this method. It then simply calls another method that we will define now and parses the `window` object to this method. The `window` object is the topmost context for the browser and thereby exposes most methods that you can otherwise invoke. So you could call `alert("Hello!")` or you could call `window.alert("Hello!")` and get the same result.
```js
export async function callAsyncInstanceMethod(instance, identifier, args) {
    try {
        var [functionObject, functionInstance] = resolveFunction(instance, identifier);
        return await functionInstance.apply(functionObject, args);
    }
    catch (error) {
        throw new DOMException(formatError(error), "AbortError");
    }
}
```
This function starts off by creating a `try-catch` statement. In the `try` block we call a function that resolves the reference to our function. Then it actually applies the arguments to the function, awaits it, and returns that. If it throws an error while resolving or awaiting the function we hit the `catch` block. We capture the `error` in the `catch` block and throw a new error. The new error is a `DOMException` constructed with the message given from calling `formatError` with the `error` and the `name` set to `"AbortError"`. It actually doesn't matter what type of error we throw as we already know that the Blazor code will completely ignore the `name`, but we chose an `AbortError` as that fits what we have done. Now let's first look at the `resolveFunction` function that we used to get the function instance and the function object.
```js
function resolveFunction(instance, identifier)
{
    let identifierParts = identifier.split(".");
    var functionObject = instance;
    var functionInstance = instance[identifierParts[0]];
    for (let i = 1; i < identifierParts.length; i++) {
        if (functionInstance == undefined) {
            throw new ReferenceError(`Cannot read properties of undefined (reading '${identifierParts[i - 1]}').`);
        }
        functionObject = functionInstance;
        functionInstance = functionInstance[identifierParts[i]];
    }
    if (!(functionInstance instanceof Function)) {
        throw new TypeError(`'${identifierParts.slice(-1)}' is not a function.`);
    }
    return [functionObject, functionInstance];
}
```
`resolveFunction` first splits the identifier into its parts. This is to support when calling methods on attributes like `window.caches.open` with a single invocation. We then set that the object that the function should be called on (`functionObject`) is the `instance` parsed as a start and that the function itself is the property with the name matching the first part of the identifier. Then we go through all consecutive identifier parts _(if there are any left)_ and update the `functionObject` to be the `functionInstance` of the previous iteration, as it was actually an attribute, and once again set the `functionInstance` to the next part of the identifier. Finally, we return the object that the function should be called on and the function itself. We do this as functions in JS always have to be called in some context.

If an error were to happen when invoking the returned function or while resolving it then we format that error and re-throw it. Below we see the function we use to format the error. It is a rather simple function that simply returns a stringified object containing the error `name`, `message`, and `stack`. This is similar to what Blazor does, but we use JSON as our structure instead of concatenating the details and we also include the `name`.
```js
function formatError(error) {
    var name = error.name;
    if (error instanceof DOMException && name == "SyntaxError") {
        name = "DOMExceptionSyntaxError";
    }
    return JSON.stringify({
        name: error.name,
        message: error.message,
        stack: error.stack
    })
}
```
The one special thing we do is check if it is a `DOMException` and if the `name` is `"SyntaxError"`. In that case we prepend the name with `"DOMException"` as to disambiguate it from the `SyntaxError` error type that also has this name. This is important as they have different purposes despite sharing a name.
And now the JS module is done and ready to be imported from C#.
#### Capturing and rethrowing in C#
The C# part should be a reusable class that follows the same pattern as the existing `IJSRuntime` so that it will be easy to use instead. And how lucky can we be? `IJSRuntime` is just an interface that we can implement. We create a new class called `ErrorHandlingJSRuntime` that implements `IJSRuntime`.
```csharp
public class ErrorHandlingJSRuntime : IJSRuntime
{
    public ValueTask<TValue> InvokeAsync<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors | DynamicallyAccessedMemberTypes.PublicFields | DynamicallyAccessedMemberTypes.PublicProperties)] TValue>(string identifier, object?[]? args)
    {
        throw new NotImplementedException();
    }

    public ValueTask<TValue> InvokeAsync<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors | DynamicallyAccessedMemberTypes.PublicFields | DynamicallyAccessedMemberTypes.PublicProperties)] TValue>(string identifier, CancellationToken cancellationToken, object?[]? args)
    {
        throw new NotImplementedException();
    }
}
```
Before we implement the methods let's create a constructor that sets up the members we will need to make invocations.
```csharp
    private Lazy<Task<IJSObjectReference>> helperTask;

    public ErrorHandlingJSRuntime(IJSRuntime jSRuntime)
    {
        helperTask = new(async () =>
            await jSRuntime.InvokeAsync<IJSObjectReference>("import", "./_content/KristofferStrube.Blazor.WebIDL/KristofferStrube.Blazor.WebIDL.js")
        );
    }
```
We take an `IJSRuntime` in the constructor and create a new lazily loaded `IJSObjectReference` helper task using it. The helper is created once needed by importing the module we made in the previous section. Now we are ready to implement the methods. The two methods are very similar but differ by having an extra `CancellationToken` argument that can be used to cancel an invocation. An example of a time you would cancel an invocation is if the user navigates away from the page before the invocation is done. This makes the first method easy to implement as we can simply call the other method with the default value for the `CancellationToken`.
```csharp
public ValueTask<TValue> InvokeAsync<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors | DynamicallyAccessedMemberTypes.PublicFields | DynamicallyAccessedMemberTypes.PublicProperties)] TValue>(string identifier, params object?[]? args)
{
    return InvokeAsync<TValue>(identifier, CancellationToken.None, args);
}
```
You might have questioned the rather long signature for the methods by now. Especially the very long `DynamicallyAccessedMembers` attribute added to the generic type parameter. These are used to tell that there will be used reflection on the types passed to the method. This is relevant in cases where you use AOT compilation as that will normally not allow reflection, but this makes it clear to the compiler that there will be used reflection on these types. It needs reflection since `System.Text.Json` goes through all properties and constructors when deserializing. 

But now to the method that makes the invocation.
```csharp
public async ValueTask<TValue> InvokeAsync<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors | DynamicallyAccessedMemberTypes.PublicFields | DynamicallyAccessedMemberTypes.PublicProperties)] TValue>(string identifier, CancellationToken cancellationToken, params object?[]? args)
{
    var helper = await helperTask.Value;
    try
    {
        return await helper.InvokeAsync<TValue>("callAsyncGlobalMethod", cancellationToken, identifier, args);
    }
    catch (JSException exception)
    {
        if (UnpackMessageOfExeption(exception) is not Error { } error)
        {
            throw;
        }
        throw MapToWebIDLException(error, exception);
    }
}
```
We first resolve our lazy-loaded helper module and then we start a `try-catch` statement. In the `try` block we invoke the `callAsyncGlobalMethod` method from our helper module and parse in the `CancellationToken` that might be the default `None` value and the original `identifier` and `args` arguments as the `args` parameter for the `InvokeAsync` method on our helper module. If it is successful then that will be it and we just return the value. But if it fails then we hit the `catch` block. We only catch `JSException` as we don't want to handle any other errors like JSON serialization errors or the like. In the block, we call the method `UnpackMessageOfExeption` that has the task of deserializing the `Message` we packed in the JS function `formatError`. If it could not unpack it then we throw the original exception as it clearly was not an exception that we had formatted. If we could unpack it then we map the `Error` and the `JSException` to some other `Exception` type and throw that instead. Let's first implement the `UnpackMessageOfExeption` method.
```csharp
internal Error? UnpackMessageOfExeption(JSException exception)
{
    return JsonSerializer.Deserialize<Error?>(exception.Message[..^9].Trim());
}
```
We first trim the last `9` characters which are `"undefined"` as this is the `stack` that Blazor postfixes the `message` with (because we threw the exception) and then we call `Trim` to remove the line break. Then we use `System.Text.Json` to deserialize the payload as an `Error?` and return that. Pretty straightforward. Now to the `MapToWebIDLException` method that maps this `Error` to a new `Exception`.
```csharp
internal WebIDLException MapToWebIDLException(Error error, JSException exception)
{
    if (ErrorMapper.TryGetValue(error.name, out Func<string, string, string?, Exception, WebIDLException>? creator))
    {
        return creator(error.name, error.message, error.stack, exception);
    }
    else
    {
        return new WebIDLException($"{error.name}: \"{error.message}\"", null, exception);
    }
}
```
The method checks inside a dictionary for a key equal to the error name. The type of the dictionary is `Dictionary<string, Func<string, string, string?, Exception, WebIDLException>>` but we will get back to that later. The value returned from the dictionary is a `Func` that takes 3 strings and an `Exception` and parses back a `WebIDLException`. If we successfully find a key that matches then we invoke the `Func` that is in the value by parsing the `name`, the `message`, the `stack`, and the original `JSException` to this. If we could not find a matching key then we simply return a `WebIDLException`. This is an `Exception` type that we have implemented to give all `Exceptions` from this `ErrorHandlingJSRuntime` a common type they can be derived from.
```csharp
/// <summary>
/// A common exception class for all exceptions from the <c>Blazor.WebIDL</c> library.
/// </summary>
public class WebIDLException : Exception
{
    private readonly string? jSStackTrace;

    /// <summary>
    /// Returns the stack trace as a string. The stack is prepended with the JS stack trace if there is any. If no stack trace is available, null is returned.
    /// </summary>
    public override string? StackTrace => jSStackTrace is not null ? jSStackTrace + '\n' + base.StackTrace : base.StackTrace;

    /// <summary>
    /// Constructs a wrapper Exception for the given error.
    /// </summary>
    /// <param name="message">User agent-defined value that provides human readable details of the error.</param>
    /// <param name="jSStackTrace">The stack trace from JavaScript if there is any.</param>
    /// <param name="innerException">Inner exception which is the cause of this exception.</param>
    public WebIDLException(string message, string? jSStackTrace, Exception innerException) : base(message, innerException)
    {
        this.jSStackTrace = jSStackTrace;
    }
}
```
The `WebIDLException` has a single constructor that parses the `message` and `innerException` down to the base `Exception` and sets the field `jSStackTrace` with the potential stack trace. If the `jSStackTrace` is not null then it is prepended to the base `StackTrace`. Our goal is to map each standard error name to some `Exception` type that inherits from `WebIDLException`. For this purpose, we define the dictionary `ErrorMapper` as a public property on the `ErrorHandlingJSRuntime`.
```csharp
public Dictionary<string, Func<string, string, string?, Exception, WebIDLException>> ErrorMapper { get; set; } = ErrorMappers.Default;
```
We define a static readonly default dictionary that maps the ones defined in the specifications. This opens up for anyone to add to the error mapper if they need to support some custom JS error type. There are a lot of predefined errors in JS as we have seen previously so I will not show all the different `Exception` types I have created. But I have grouped them so that all the _Native Error Types_ like `ReferenceErrorException` and `TypeErrorException` are derived from a common `NativeErrorException` and all the different `DOMException`s are derived from a common `DOMException` class.
```csharp
/// <summary>
/// An exception that encapsulates a name for an exception.
/// </summary>
/// <remarks><see href="https://webidl.spec.whatwg.org/#idl-DOMException">See the WebIDL definition here</see></remarks>
public class DOMException : WebIDLException
{
    public const string NotFoundError = "NotFoundError";
    public const string SyntaxError = "DOMExceptionSyntaxError";
    public const string AbortError = "AbortError";
    // Constants for all the other predefined `DOMException` names are also included here but have been omitted for brevity.

    /// <summary>
    /// Error name which should be one of the ones listed in this <see href="https://webidl.spec.whatwg.org/#dfn-error-names-table">error names table</see>.
    /// </summary>
    public string Name { get; set; }

    /// <summary>
    /// Constructs a wrapper Exception for the given error.
    /// </summary>
    /// <param name="message">User agent-defined value that provides human readable details of the error.</param>
    /// <param name="name">Error name which should be one of the ones listed in this <see href="https://webidl.spec.whatwg.org/#dfn-error-names-table">error names table</see>.</param>
    /// <param name="jSStackTrace">The stack trace from JavaScript if there is any.</param>
    /// <param name="innerException">Inner exception which is the cause of this exception.</param>
    protected DOMException(string message, string name, string? jSStackTrace, Exception innerException) : base(message, jSStackTrace, innerException)
    {
        Name = name;
    }
}
```
The `DOMException` defines a lot of constants for the different names of errors, but also adds a property `Name` that defines the error name. The constructor is protected since one should always define a derived `Exception` type for each additional custom `DOMException`. A concrete example of a `DOMException` could be the `NotFoundErrorException`.
```csharp
/// <summary>
/// The object can not be found here.
/// </summary>
/// <remarks><see href="https://webidl.spec.whatwg.org/#notfounderror">See the WebIDL definition here</see></remarks>
public class NotFoundErrorException : DOMException
{
    /// <summary>
    /// Constructs a wrapper Exception for the given error.
    /// </summary>
    /// <param name="message">User agent-defined value that provides human readable details of the error.</param>
    /// <param name="jSStackTrace">The stack trace from JavaScript if there is any.</param>
    /// <param name="innerException">Inner exception which is the cause of this exception.</param>
    public NotFoundErrorException(string message, string? jSStackTrace, Exception innerException) : base(message, NotFoundError, jSStackTrace, innerException) { }
}
```
Now we are ready to define our `Default` `ErrorMapper`.
```csharp
public static class ErrorMappers
{
    public static Dictionary<string, Func<string, string, string?, Exception, WebIDLException>> Default { get; } = new()
    {
        { DOMException.NotFoundError, (name, message, jSStackTrace, innerException) => new NotFoundErrorException(message, jSStackTrace, innerException) },
        { DOMException.SyntaxError, (name, message, jSStackTrace, innerException) => new SyntaxErrorDOMException(message, jSStackTrace, innerException) },
        { DOMException.AbortError, (name, message, jSStackTrace, innerException) => new AbortErrorException(message, jSStackTrace, innerException) },
        // Also includes all other DOMException types, but they have been omitted from the article for brevity again.
        { "EvalError", (name, message, jSStackTrace, innerException) => new EvalErrorException(message, jSStackTrace, innerException) },
        { "RangeError", (name, message, jSStackTrace, innerException) => new RangeErrorException(message, jSStackTrace, innerException) },
        { "ReferenceError", (name, message, jSStackTrace, innerException) => new ReferenceErrorException(message, jSStackTrace, innerException) },
        { "SyntaxError", (name, message, jSStackTrace, innerException) => new TypeErrorException(message, jSStackTrace, innerException) },
        { "TypeError", (name, message, jSStackTrace, innerException) => new TypeErrorException(message, jSStackTrace, innerException) },
        { "URIError", (name, message, jSStackTrace, innerException) => new URIErrorException(message, jSStackTrace, innerException) },
    };
}
```
The `Default` mapper is pretty straight forward as we would expect so now we are done with implementing the `ErrorHandlingJSRuntime`. Let's make a small sample of using this to call a function that will fail and then catch the exception.
```razor
@inject IJSRuntime JSRuntime

<span style="color: red;">@exceptionMessage</span>

@code {
    string? exceptionMessage;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            var errorHandlingJSRuntime = new ErrorHandlingJSRuntime(JSRuntime);
            var url = await errorHandlingJSRuntime.InvokeAsync<string>("URL.createObjectURL", "not a blob");
        }
        catch (TypeErrorException ex)
        {
            exceptionMessage = $"Wrong type for argument. {ex.Message}";
        }
        catch (WebIDLException ex)
        {
            exceptionMessage = $"An {ex.GetType().Name} happened: \"{ex.Message}\"";
        }
        catch (Exception)
        {
            exceptionMessage = "An unexpected error happened.";
        }
    }
}
```
In the above example, we try to call the `createObjectURL` method from the browser File API. This method takes a `Blob`, a `File`, or a `MediaSource` as its argument, but we passed a string to it. So it throws a `TypeErrorException` since there was no overload for the method taking a string. This results in the following message in our app:
```console
Wrong type for argument. Failed to execute 'createObjectURL' on 'URL': Overload resolution failed.
```
### Blazor.WebIDL
As the JS errors are declared in the WebIDL standard specifications I have included the work that we have done above in my [Blazor.WebIDL](https://github.com/KristofferStrube/Blazor.WebIDL) package. You can add it to your project by running the following in the terminal while located in the folder of your Blazor project.
```bash
dotnet add package KristofferStrube.Blazor.WebIDL
```
And instead of having to construct the `ErrorHandlingJSRuntime` yourself, I've added an extension method that adds the needed services to the service collection so that you can just inject the `ErrorHandlingJSRuntime` as an `IErrorHandlingJSRuntime` in the pages that need it.
```csharp
builder.Services.AddErrorHandlingJSRuntime();
```
I've made some more simplifications that make it easier to use. But for this to work we also need to call and await the extension method `SetupErrorHandlingJSInterop` on our `ServiceProvider` after we have `Build` the application, but before we run it.
```csharp
var app = builder.Build();

await app.Services.SetupErrorHandlingJSInterop();

await app.RunAsync();
```
Then we can inject it into some Blazor page and use it directly to make invocations.
```razor
@inject IErrorHandlingJSRuntime ErrorHandlingJSRuntime

<button @onclick="ReadClipboard">Copy from clipboard</button>
<br />
@if (result is not null)
{
    <p>You have the text: '@result' in your clipboard</p>
}
else
{
    <code>@copyError</code>
}

@code {
    private string? result;
    private string copyError = string.Empty;

    private async Task Copy()
    {
        try
        {
            result = await ErrorHandlingJSRuntime.InvokeAsync<string>("navigator.clipboard.readText");
        }
        catch (NotAllowedErrorException)
        {
            copyError = "The user has not given permission to read the clipboard.";
        }
        catch (DOMException exception)
        {
            copyError = $"{exception.Name} (which is a DOMException): \"{exception.Message}\"";
        }
        catch (Exception)
        {
            copyError = "An unexpected error happened.";
        }
    }
}
```
In the above sample, we try to read the content of the clipboard. You might not have given the site permission to do this and in this case, a `NotAllowedErrorException` will be thrown. If it wasn't this specific kind of `DOMException` that happened then it might have been some other `DOMException` type which is the next kind we try to catch and in the end, if it wasn't any of these we give back a general error message.

I have also added error handling wrappers for the `IJSInProcessRuntime`, `IJSObjectReference`, and `IJSInProcessObjectReference` types and made sure that you get back an appropriate error handling version of an `IJSObjectReference` if this is the `TValue` type that you return from an invocation. So in total, we now have the following error-handling JSInterop interfaces that we can inject.
- `IErrorHandlingJSRuntime`
- `IErrorHandlingJSInProcessRuntime`
- `IErrorHandlingJSObjectReference`
- `IErrorHandlingJSInProcessObjectReference`

The reason why we need to call the `SetupErrorHandlingJSInterop` was so that we could return `IErrorHandlingJSObjectReference`s without having to resolve the helper asynchronously when creating it but also so that we can use the `IErrorHandlingJSInProcessRuntime` when using Blazor WASM to make synchronous JS invocations without using some asynchronous factory to create the runtime each time it is needed.

### Future plans
The plan is to also make an error-handling version of the `IJSUnmarshalledRuntime` in the future. And I intend to upgrade all my existing browser API packages [Blazor.FileSystemAccess](https://github.com/KristofferStrube/Blazor.FileSystemAccess), [Blazor.FileSystem](https://github.com/KristofferStrube/Blazor.FileSystem), [Blazor.FileAPI](https://github.com/KristofferStrube/Blazor.FileAPI), [Blazor.Streams](https://github.com/KristofferStrube/Blazor.Streams), [Blazor.CompressionStreams](https://github.com/KristofferStrube/Blazor.CompressionStreams), [Blazor.WebAudio](https://github.com/KristofferStrube/Blazor.WebAudio), [Blazor.MediaCaptureStreams](https://github.com/KristofferStrube/Blazor.MediaCaptureStreams), [Blazor.DOM](https://github.com/KristofferStrube/Blazor.DOM) and all my future ones to use the error-handling versions when they make JSInterop calls.

### Conclusion
At the start of this post, we saw some of the existing approaches for handling different types of exceptions when making JSInterop calls in Blazor. And we now know that this is because we don't get the name of the exception extracted when the error is returned to C#. We then found a way to catch more details of the error on the JS side of the invocation and re-throw these details back to Blazor where we could then unpack the name, message, and stack trace separately and map them to custom C# exception types. In the end, I presented the [Blazor.WebIDL](https://github.com/KristofferStrube/Blazor.WebIDL) package that has made this `ErrorHandlingJSRuntime` easy to add to any project where you want to have typed exceptions when using JSInterop. We also went through a small example of using the package to make handle different error scenarios. I would be very happy if people want to test the package and give feedback on the GitHub repository so that I can make sure that it is both easy and safe to use. The library is still in alpha while I test it in some of my libraries, but there won't go long before I make a first full release. And as always, if you have any feedback or questions for this post then feel free to reach out to me on Mastodon or Twitter.

<script>
    hljs.highlightAll();
</script>