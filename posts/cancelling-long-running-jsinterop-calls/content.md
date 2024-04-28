### Parsing a `CancellationToken` to invocations
As developers who are used to using async-await, we will try to see if an async method takes a `CancellationToken` when we want the ability to cancel tasks. We see that the async invocation tasks for `JSInterop` actually take a `CancellationToken` parameter. But this won't work. Let's check the source code and see what the `CancellationToken` is used for. First, we check the parameter description for `cancellationToken`.

```xml
<param name="cancellationToken">
A cancellation token to signal the cancellation of the operation. Specifying this parameter will override any default cancellations such as due to timeouts
(<see cref="DefaultAsyncTimeout"/>) from being applied.
</param>
```

So, from the description, it seems like this makes it possible to cancel the invocation. But to go even deeper down the rabbit hole, let's see what it is used for. First, if the `CancellationToken` can be canceled, and this happens, it tries to set the state of the resulting task to canceled and cleans up the task and cancellation callback registration.

```csharp
if (cancellationToken.CanBeCanceled)
{
    _cancellationRegistrations[taskId] = cancellationToken.Register(() =>
    {
        tcs.TrySetCanceled(cancellationToken);
        CleanupTasksAndRegistrations(taskId);
    });
}
```

Then it does the same if the `CancellationToken` had already been cancelled so that we don't need to start the invocation if it was immediately canceled.

```csharp
if (cancellationToken.IsCancellationRequested)
{
    tcs.TrySetCanceled(cancellationToken);
    CleanupTasksAndRegistrations(taskId);

    return new ValueTask<TValue>(tcs.Task);
}
```

Then, it starts the invocation, which means we will have no more opportunities for the task to be stopped if a cancellation is requested before the JS function is done and calls back. If the source of the `CancellationToken` was canceled while the invocation happened, then the entire JS function will finish first. Then, when the method that receives the response is invoked, it checks whether the task callback has already been removed and exits early in that case.
```csharp
if (!_pendingTasks.TryRemove(taskId, out var tcs))
{
    // We should simply return if we can't find an id for the invocation.
    // This likely means that the method that initiated the call defined a timeout and stopped waiting.
    return false;
}
```
This is great as we spared the work of deserializing the response, which could have been expensive, as we have seen in our previous post on [Blazor performance](https://kristoffer-strube.dk/post/a-holistic-comparison-of-blazor-wasm-performance-from-aspnet-core-5-to-8/). But this still doesn't enable us to cancel JS functions while they are running.

### Cancelling using the `AbortController` and `AbortSignal`
To support canceling a JS function, we need to parse something into it that it can use to check whether it should be stopped early. JS has a standard equivalent to the `CancellationToken` and `CancellationTokenSource` types: the `AbortSignal` and `AbortController` types. If we create an `AbortController`, we can get its `AbortSignal`. Using the `AbortController`, we can then abort the operation, and using the `AbortSignal`, we can check whether the operation has been canceled. The introduction of this has a similar history to that of the `CancellationToken` as it was not introduced when JS first got support for the async-await pattern. It was instead introduced later when we discovered that some operations would be nice to be able to abort. This also means that it is sadly not used in all APIs, as some APIs were standardized before its introduction. However, a lot of new APIs utilize it to cancel asynchronous operations.

One API that utilizes the `AbortSignal` is the [Fetch API](https://fetch.spec.whatwg.org/). The fetch API standardizes ways to fetch resources in JavaScript. In Blazor WASM, the `HttpClient` is implemented so that it uses this API to make its requests, but we can also use the API directly without this abstraction. In JS, we can write the following to make a simple GET request:
```js
async function getCharacterInfo(characterNumber) {
    let response = await fetch("https://api.sampleapis.com/futurama/characters/" + characterNumber);
    let json = response.json();
    return json;
}
```
The above sample URL returns some information about a character from Futurama and is actually pretty fast, but we could imagine that it was a slow endpoint and that we would like to abort the fetch request mid-way in some cases. To do this we simple need to specify some options for the optional `init` parameter for the fetch method.
```js
async function getCharacterInfo(characterNumber) {
    let abortController = new AbortController();
    let abortSignal = abortController.signal;

    let requestInit = {
        "signal": abortSignal
    };

    let response = await fetch("https://api.sampleapis.com/futurama/characters/" + characterNumber, requestInit);
    let json = response.json();
    return json;
}
```
In principle, the above is all we need. But currently, we cannot tell the `AbortController` that the action should be canceled as it is not exposed anywhere outside the function. There are multiple ways to achieve this. One way is to save the abort controllers in a map where the key is defined by some extra parameter like this:
```js
window.abortControllers = {};

async function getCharacterInfo(characterNumber, abortControllerKey) {
    let abortController = new AbortController();
    window.abortControllers[abortControllerKey] = abortController;
    let abortSignal = abortController.signal;

    let requestInit = {
        "signal": abortSignal
    };

    let response = await fetch("https://api.sampleapis.com/futurama/characters/" + characterNumber, requestInit);
    let json = response.json();
    return json;
}

function abortFetch(abortControllerKey) {
    window.abortControllers[abortControllerKey].abort();
}
```
The above makes it possible to abort a fetch using some `abortControllerKey` to identify the operation that should be canceled. But this requires us to remember this arbitrary key and ensure that the key is unique. This also pollutes the global window with the `abortControllers` attribute that could potentially collide with attributes used by other libraries. One last problem with this solution is that the `AbortController`s would never be garbage collected as they continue to be referenced in the `abortControllers` map. To solve that problem, we would need to make a separate third call to remove the reference to the `AbortController` once the operation is finished.

Let's instead use another approach that mitigates these problems. Instead of keeping the `AbortController` in a map, we could construct the `AbortController` outside the JS function and then parse it in as an `IJSObjectReference`. With this approach, our JS functions would look like this:
```js
async function getCharacterInfo(characterNumber, abortController) {
    let abortSignal = abortController.signal;

    let requestInit = {
        "signal": abortSignal
    };

    let response = await fetch("https://api.sampleapis.com/futurama/characters/" + characterNumber, requestInit);
    let json = response.json();
    return json;
}

function createAbortController() {
    return new AbortController();
}
```
This could then be used from Blazor like this:
```csharp
public partial class Index : IDisposable
{
    private string age = "";
    private int characterNumber = 1;
    private CancellationTokenSource? fetchCancellationTokenSource;

    [Inject]
    public required IJSRuntime JSRuntime { get; set; }

    public async Task UpdateCharacterAge()
    {
        age = "";

        // Get CancellationToken.
        fetchCancellationTokenSource = new CancellationTokenSource();
        CancellationToken token = fetchCancellationTokenSource.Token;

        // Create AbortController
        await using IJSObjectReference abortController = await JSRuntime.InvokeAsync<IJSObjectReference>("createAbortController");

        // Register that it should abort when the CancellationToken is canceled.
        token.Register(async () =>
        {
            await abortController.InvokeVoidAsync("abort");
        });

        // Make JS invocation.
        Character character = await JSRuntime.InvokeAsync<Character>("getCharacterInfo", token, characterNumber, abortController);

        // Reset the fetchCancellationTokenSource and set result.
        fetchCancellationTokenSource = null;
        age = character.Age;
    }

    public record Character(string Occupation, string Age);

    public void Dispose()
    {
        fetchCancellationTokenSource?.Cancel();
    }
}
```
This might seem like a lot of C# code, but I promise that it is less code than what would have been needed for the JS map-based solution that we saw the JS code for earlier. We use the `AbortController` and the `CancellationTokenSource` we discussed previously. They work nicely together. The `AbortController` ensures that we can exit the JS function early. The `CancellationTokenSource` ensures we don't use time to deserialize the `AbortError` that gets thrown when the JS function is aborted.

### Using Blazor.DOM
In the above solution, we needed to write a function to construct an `AbortController`, and we needed to know that it had a function called `abort` used for aborting the `AbortController`. Another thing to point out is that we parsed the `AbortController` itself to the `getCharacterInfo` function when it really only needed the `AbortSignal`.

I've implemented wrappers for the relevant types used in the above sample in my library [Blazor.DOM](https://github.com/KristofferStrube/Blazor.DOM). So let's try to rewrite the above sample to be a bit more strongly typed and minimal. Let's first make some changes to the JS part.

```js
async function getCharacterInfo(characterNumber, abortSignal) {
    let requestInit = {
        "signal": abortSignal
    };

    let response = await fetch("https://api.sampleapis.com/futurama/characters/" + characterNumber, requestInit);
    let json = response.json();
    return json;
}
```
As you can see, we now need a lot less JS. We like this. The main difference is that `getCharacterInfo` takes the `abortSignal` directly instead of the `abortController`. This is nice as we really shouldn't be able to access the controller in the function as we could then potentially abort the controller from within the function, giving unwanted side effects or errors in our C# code. Apart from this, we have also removed the method that constructed an `AbortController` as we will no longer need to call this. Now, let's see the C# part.
```csharp
public partial class Index : IDisposable
{
    private string age = "";
    private int characterNumber = 1;
    private CancellationTokenSource? fetchCancellationTokenSource;

    [Inject]
    public required IJSRuntime JSRuntime { get; set; }

    public async Task UpdateCharacterAge()
    {
        age = "";

        // Get CancellationToken.
        fetchCancellationTokenSource = new CancellationTokenSource();
        CancellationToken token = fetchCancellationTokenSource.Token;

        // Create AbortController and AbortSignal
        await using AbortController abortController = await AbortController.CreateAsync(JSRuntime);
        await using AbortSignal abortSignal = await abortController.GetSignalAsync();

        // Register that it should abort when the CancellationToken is canceled.
        token.Register(async () =>
        {
            await abortController.AbortAsync();
        });

        // Make JS invocation.
        Character character = await JSRuntime.InvokeAsync<Character>("getCharacterInfo", token, characterNumber, abortSignal);

        // Reset the fetchCancellationTokenSource and set result.
        fetchCancellationTokenSource = null;
        age = character.Age;
    }

    public record Character(string Occupation, string Age);

    public void Dispose()
    {
        fetchCancellationTokenSource?.Cancel();
    }
}
```
It has not changed much, apart from the fact that we now use strongly-typed types from the [KristofferStrube.Blazor.DOM](https://www.nuget.org/packages/KristofferStrube.Blazor.DOM) NuGet package.

### Other JS functions that can be aborted
We have now seen one sample of a JS function that can be aborted and understand how we can parse an `AbortSignal` to it to have the option to stop the JS function before it finishes. We also understand how to create this `AbortSignal` in C# and signal it to abort. So, to better understand how we can use this, let's see some other samples of JS functions to cancel.
#### Infinite loop
```js
async function infiniteLoop(abortSignal) {
    while(!abortSignal.aborted) {
        console.log("Do some actual work forever in 1 second intervals.");
        await new Promise(r => setTimeout(r, 1000));
    }
}
```
If we have some infinite loop that ensures that some function is called forever with some interval, we also need to be able to stop the loop. For this, we can use that the `AbortSignal` has an attribute that tells us whether the signal has been aborted.
#### Piping ReadableStream
```js
function compressReadableStream(readableStream, abortSignal) {
    let compressionStream = new CompressionStream("gzip");
    let compressed = readableStream.pipeThrough(compressionStream, { "signal": abortSignal });
    return compressed;
}
```
Another API that incorporates the `AbortSignal` is the [Streams API](https://streams.spec.whatwg.org/). When we pipe some readable stream to a writable stream or through a transformer, we can parse options to the function, enabling us to abort it. This is a good use case as a `ReadableStream` could potentially be large or even infinite.

### Conclusion
In this article, we explored the source code of JSInterop to see what it does when we parse a `CancellationToken` to it. We introduced the `AbortController` and `AbortSignal` types and used them together with the Fetch API to cancel a potentially long-lived operation. Then we saw how we can use the [Blazor.DOM](https://github.com/KristofferStrube/Blazor.DOM) library to make some of this easier and simpler. In the end, we quickly took a look at some other scenarios where we could use `AbortSignal`s. If you have any questions related to the article or use cases I did not cover, feel free to reach out to me.
