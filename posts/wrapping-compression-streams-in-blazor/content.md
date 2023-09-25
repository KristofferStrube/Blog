*Disclaimer: "Most modern browsers" means Chrome, Edge, Safari, and Opera. (Note that the list doesn't include Firefox)*

### Setup project
The newest version of [Blazor.Streams](https://github.com/KristofferStrube/Blazor.Streams) uses packages that depend on .NET 7 so we need to download the newest version of .NET 7 from [dotnet.microsoft.com/en-us/download/dotnet/7.0](https://dotnet.microsoft.com/en-us/download/dotnet/7.0).

Using the dotnet CLI we can create a new Razor class library in a new folder.
```bash
dotnet new razorclasslib
```
Then we add a reference to [Blazor.Streams](https://github.com/KristofferStrube/Blazor.Streams) using the CLI as well.
```
dotnet add package KristofferStrube.Blazor.Streams
```
The `razorclasslib` template adds a couple of sample razor components that we don't need, but it also creates a sample JSInterop class which we can change every so little to be our base JS Wrapper class. The JS Wrapper base class should make a couple of fields and properties available: an `IJSObjectReference` representing the wrapped object, an `IJSRuntime` for invoking JS methods, and a lazily evaluated reference to a helper JS script file.

##### BaseJSWrapper.cs
```csharp
public abstract class BaseJSWrapper : IAsyncDisposable
{
    public IJSObjectReference JSReference { get; }
    public IJSRuntime JSRuntime { get; }

    protected readonly Lazy<Task<IJSObjectReference>> helperTask;

    internal BaseJSWrapper(IJSRuntime jSRuntime, IJSObjectReference jSReference)
    {
        helperTask = new(jSRuntime.GetHelperAsync);
        JSReference = jSReference;
        JSRuntime = jSRuntime;
    }

    public async ValueTask DisposeAsync()
    {
        if (helperTask.IsValueCreated)
        {
            IJSObjectReference module = await helperTask.Value;
            await module.DisposeAsync();
        }
        GC.SuppressFinalize(this);
    }
}
```

We have an internal constructor as we only want wrapper classes from this project to be able to extend the class.
We initialize the lazy helper task using an `IJSRuntime` extension method called `GetHelperAsync`.

`GetHelperAsync` is defined like so:
##### IJSRuntimeExtensions.cs
```csharp
internal static class IJSRuntimeExtensions
{
    internal static async Task<IJSObjectReference> GetHelperAsync(this IJSRuntime jSRuntime)
    {
        return await jSRuntime.InvokeAsync<IJSObjectReference>(
            "import",
            "./_content/KristofferStrube.Blazor.CompressionStreams/KristofferStrube.Blazor.CompressionStreams.js"
        );
    }
}
```
The method references our helper JS script file. The first part of the path `./content/` is where Blazor stores all resources from packages that it uses. The next part `/KristofferStrube.Blazor.CompressionStreams/` is the name of the namespace of our project and after this part of the path we find all the files from the `/wwwroot/` folder of our project which is where our helper script is placed. We call the helper script `KristofferStrube.Blazor.CompressionStreams.js` so that it is easy to find the specific script from the browser developer tool if necessary which wouldn't be as easy if everyone called their helper scripts `helper.js`. We will add some methods to the script file when we need any methods that Blazor doesn't have support for natively.

### Looking at the WebIDL specification
A good start for wrapping any browser API is to look at the WebIDL specification and we are very lucky that the specification for the Compressions Streams API is very short and concise. We can [find the specification here](https://wicg.github.io/compression/#idl-index) but I have also written it out below as it is very short.
```webidl
[Exposed=*]
interface CompressionStream {
  constructor(DOMString format);
};
CompressionStream includes GenericTransformStream;

[Exposed=*]
interface DecompressionStream {
  constructor(DOMString format);
};
DecompressionStream includes GenericTransformStream;
```
We see that it defines two interfaces that are very similar. They are both exposed in `*` meaning that they can be constructed in all contexts. We see that they both have a constructor that takes a format. If we scroll up a bit in the specification we find that the API supports 3 different compression algorithms which is what we must parse as the format. It supports the following formats:
- `deflate` (ZLIB Compressed Data Format)
- `deflate-raw` (The DEFLATE algorithm)
- `gzip` (GZIP file format)

The next thing we note is that the interfaces both include `GenericTransformStream`. This means that it inherits the members defined in the `GenericTransformStream` mixin interface from the browser Streams API.
```webidl
interface mixin GenericTransformStream {
  readonly attribute ReadableStream readable;
  readonly attribute WritableStream writable;
};
```
We don't have extension properties nor multiple inheritances in C#. But we have defined an interface called `IGenericTransformStream` in [Blazor.Streams](https://github.com/KristofferStrube/Blazor.Streams) that can help us ensure that we implement wrappers for these attributes.

### Wrapping CompressionStream

Let's start by writing the wrapper class for the `CompressionStream`.

##### CompressionStream.cs
```csharp
public class CompressionStream : BaseJSWrapper, IGenericTransformStream
{
    public static Task<CompressionStream> CreateAsync(IJSRuntime jSRuntime, IJSObjectReference jSReference)
    {
        return Task.FromResult(new CompressionStream(jSRuntime, jSReference));
    }

    protected CompressionStream(IJSRuntime jSRuntime, IJSObjectReference jSReference)
        : base(jSRuntime, jSReference) { }

    // Rest of the methods that need to be implemented.
}
```
We make the constructor protected so that consumers of the wrapper class need to use the creator method `CreateAsync` to instantiate an instance. This method did not need to return a Task, but for uniformity, we do this as other creator methods might need to do some asynchronous work like registering event listeners or similar through JSInterop. The `IGenericTransformStream` interface defines that we need to implement two methods `GetReadableAsync` and `GetWritableAsync`. So let's do so.
```csharp
// Other methods in CompressionStream.cs above here

public async Task<ReadableStream> GetReadableAsync()
{
    IJSObjectReference helper = await helperTask.Value;
    IJSObjectReference jSInstance = await helper.InvokeAsync<IJSObjectReference>("getAttribute", JSReference, "readable");
    return await ReadableStream.CreateAsync(JSRuntime, jSInstance);
}

public async Task<WritableStream> GetWritableAsync()
{
    IJSObjectReference helper = await helperTask.Value;
    IJSObjectReference jSInstance = await helper.InvokeAsync<IJSObjectReference>("getAttribute", JSReference, "writable");
    return await WritableStream.CreateAsync(JSRuntime, jSInstance);
}
```
The two methods make it possible to access rich wrapper instances of the `readable` and `writable` counterparts in the transform stream. In each method, we first await the lazily defined helper. This uses the existing helper `IJSObjectReference` for the helper if it has been used before and else creates a new one using the `GetHelperAsync` as we defined it in the `BaseJSWrapper` constructor. Next, we use the helper to get either the `readable` or `writable` attribute from our wrapped `JSReference` using a helper method called `getAttribute`. Then we have an `IJSObjectReference` to that attribute which we can create a rich Stream wrapper from. To use the `getAttribute` method we need to add it to the helper JS script file.
##### KristofferStrube.Blazor.CompressionStreams.js
```js
export function getAttribute(object, attribute) {
    return object[attribute];
}
```
It is a really simple method, but currently, Blazor doesn't have a method for accessing the attributes of a JS object so we need it. It has been planned for .NET 8 to add methods for this so that we don't need this JS method. You can check out [the related issue](https://github.com/dotnet/aspnetcore/issues/31151) which I have linked many times before.

The last thing we need to wrap for the `CompressionStream` is its constructor. This will be a second static `CreateAsync` method that takes an `IJSRuntime` and the `format` as it was defined in the WebIDL specification. In the specification, it was defined as being a string, but actually, only 3 different values were possible. So let's create an enum for those options.
##### CompressionAlgorithm.cs
```csharp
public enum CompressionAlgorithm
{
    Deflate,
    DeflateRaw,
    Gzip
}

public static class CompressionAlgorithmsExtensions
{
    public static string AsString(this CompressionAlgorithm compressionAlgorithm) => compressionAlgorithm switch
    {
        CompressionAlgorithm.Deflate => "deflate",
        CompressionAlgorithm.DeflateRaw => "deflate-raw",
        CompressionAlgorithm.Gzip => "gzip",
        _ => throw new NotSupportedException($"Value {compressionAlgorithm} not supported as a Compression Algorithm format."),
    };
}
```
We also define an extension function for the enum which maps it to a string. We could have created this mapping using a custom JSON serializer, but I find this more readable and versatile. Now we are ready to make the constructor wrapping method.
```csharp
// Other methods in CompressionStream.cs above here

public static async Task<CompressionStream> CreateAsync(IJSRuntime jSRuntime, CompressionAlgorithm format)
{
    IJSObjectReference helper = await jSRuntime.GetHelperAsync();
    IJSObjectReference jSInstance = await helper.InvokeAsync<IJSObjectReference>("createCompressionStream", format.AsString());
    return new CompressionStream(jSRuntime, jSInstance);
}
```
In this method we can't use the lazily evaluated helper as the method is static so we simply just await it directly. Using the helper we invoke a method called `createCompressionStream` which calls the constructor with our `format`. We then create and return a new `CompressionStream` using our protected constructor. So the last part is to add the `createCompressionStream` method to our helper JS script file before being done.
##### KristofferStrube.Blazor.CompressionStreams.js
```js
// Other helper methods above here

export function createCompressionStream(format) {
    return new CompressionStream(format);
}
```
This method is also very simple but is necessary as we can't invoke constructors directly from Blazor. This feature is also planned to be added in .NET 8 as a part of the previously linked issue.

Now we have wrapped `CompressionStream` and as we remember `DecompressionStream` is very similar so we won't go through that in this article.

### CompressionStream InProcess
Blazor has an InProcess variant of `IJSObjectReference` called `IJSInProcessObjectReference` which can also make JSInterop calls synchronously. This can be very useful for accessing attributes as we can then use C# properties. [Blazor.Streams](https://github.com/KristofferStrube/Blazor.Streams) also defines an InProcess interface for `IGenericTransformStreamInProcess`. It has the same shape as `IGenericTransformStream` but returns InProcess variants of `ReadableStream` and `WritableStream` called `ReadableStreamInProcess` and `WritableStreamInProcess`. So the primary benefit of making an InProcess variant of `CompressionStream` and `DecompressionStream` is its interoperability with the Blazor.Streams package. As we have already seen the process I will just share the result.
##### CompressionStream.InProcess.cs
```csharp
public class CompressionStreamInProcess : CompressionStream, IGenericTransformStreamInProcess
{
    public new IJSInProcessObjectReference JSReference { get; set; }

    public static Task<CompressionStreamInProcess> CreateAsync(IJSRuntime jSRuntime, IJSInProcessObjectReference jSReference)
        => Task.FromResult(new CompressionStreamInProcess(jSRuntime, jSReference));

    public new static async Task<CompressionStream> CreateAsync(IJSRuntime jSRuntime, CompressionAlgorithm format)
    {
        IJSObjectReference helper = await jSRuntime.GetHelperAsync();
        IJSInProcessObjectReference jSInstance = await helper.InvokeAsync<IJSInProcessObjectReference>("createCompressionStream", format.AsString());
        return await Task.FromResult(new CompressionStreamInProcess(jSRuntime, jSInstance));
    }

    protected CompressionStreamInProcess(IJSRuntime jSRuntime, IJSInProcessObjectReference jSReference) : base(jSRuntime, jSReference)
    {
        JSReference = jSReference;
    }

    public new async Task<ReadableStreamInProcess> GetReadableAsync()
    {
        IJSObjectReference helper = await helperTask.Value;
        IJSInProcessObjectReference jSInstance = await helper.InvokeAsync<IJSInProcessObjectReference>("getAttribute", JSReference, "readable");
        return await ReadableStreamInProcess.CreateAsync(JSRuntime, jSInstance);
    }

    public new async Task<WritableStreamInProcess> GetWritableAsync()
    {
        IJSObjectReference helper = await helperTask.Value;
        IJSInProcessObjectReference jSInstance = await helper.InvokeAsync<IJSInProcessObjectReference>("getAttribute", JSReference, "writable");
        return await WritableStreamInProcess.CreateAsync(JSRuntime, jSInstance);
    }
}
```

The primary difference is that we hide a lot of the members of the `CompressionStream` and define InProcess variants instead.

You can checkout the full implementation at [github.com/KristofferStrube/Blazor.CompressionStreams/](https://github.com/KristofferStrube/Blazor.CompressionStreams/)

### Validation Sample
Now we just need to check that our implementation works. To validate it we will construct a stream, compress it, decompress it, and check that the content is still valid.

We create a new Blazor WASM project in an empty folder using the CLI once again.
```bash
dotnet new blazorwasm
```
From this project, we add a reference to the class library. If you followed the previous steps and made the class library yourself then you can use the [dotnet add reference](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-add-reference) command to reference that project. Else you can add my published NuGet package using this command:
```bash
dotnet add package KristofferStrube.Blazor.CompressionStreams
```
Once we have added the library reference to our Blazor WASM project we can begin to make our sample page. We will simply modify the pre-generated page `Index.razor`. We define the following scaffold for our page:
##### Index.razor
```razor
@page "/"
@using KristofferStrube.Blazor.CompressionStreams

@inject IJSRuntime JSRuntime
@inject HttpClient HttpClient

@if (compressedStreamSize is not 0)
{
    <div><label style="width:200px;">Compressed size was:</label> @compressedStreamSize</div>
}
@if (decompressedStreamSize is not 0)
{
    <div><label style="width:200px;">Decompressed size was:</label> @decompressedStreamSize</div>
}

<p>
    @content
</p>

@code {
    string content = "";
    long compressedStreamSize;
    long decompressedStreamSize;

    protected override async Task OnInitializedAsync()
    {
        // Our compression and decompression code here
    }
}
```
In this scaffold, we already do a lot. We add an `using` for the `KristofferStrube.Blazor.CompressionStreams` library and inject the `IJSRuntime` and `HttpClient` services. We will use the `HttpClient` to download some data as our compression guinea pig for compression in a bit. After this, we make some markup that will present the sizes of our stream compressed and decompressed followed by the final content itself. After this, we define these fields in our code section.

Now we just need to fill out our `OnInitializedAsync` method with the actual code for compressing and decompressing a stream. For a start we need some stream of data. For this, we use the aforementioned `HttpClient`. We can stream the result of an HTTP request using the `GetStreamAsync` method. So we need something to stream and for this purpose, we will use the world-renowned Lorem-Ipsum text which we will copy into a file called `lorem.txt` in the `/wwwroot/data/` folder of our Blazor WASM project. Then we can fetch it and construct a `ReadableStream` from the resulting stream.
```csharp
var data = await HttpClient.GetStreamAsync("data/lorem.txt");
var streamRef = new DotNetStreamReference(stream: data, leaveOpen: false);
var jSStreamReference = await JSRuntime.InvokeAsync<IJSObjectReference>("jSStreamReference", streamRef);
var readableStream = await ReadableStream.CreateAsync(JSRuntime, jSStreamReference);
```
With ASP.NET Core 6, Blazor got a `DotNetStreamReference` type which can be used to construct a JS stream from any .NET `Stream` as we do in the above code. From the `IJSObjectReference` to this JS stream we construct a `ReadableStream` from the `Blazor.Streams` library.

Then we compress the stream using a new instance of a `CompressionStream`.
```csharp
var compressionStream = await CompressionStream.CreateAsync(JSRuntime, CompressionAlgorithm.DeflateRaw);
var compressedStream = await readableStream.PipeThroughAsync(compressionStream);
```
The `ReadableStream` wrapper has a method called `PipeThroughAsync` which can be used to transform a stream in any way. It takes a class that implements `IGenericTransformStream` as its only parameter which we luckily had `CompressionStream` implement.

Now we want to measure the compressed size but reading the compressed stream will consume it. So we first _tee_ the stream. By teeing the stream we create 2 new identical streams that can be consumed however we want. This also locks the original stream making it impossible to read.
```csharp
var (tee1, tee2) = await compressedStream.TeeAsync();
```
Then we read the first tee and counts its size.
```csharp
var reader = await tee1.GetDefaultReaderAsync();
await foreach (var byteArrayChunk in reader.IterateByteArraysAsync())
{
    compressedStreamSize += byteArrayChunk.Length;
}
```
And finally we decompress the second tee and read and measure its size.
```csharp
var decompressionStream = await DecompressionStream.CreateAsync(JSRuntime, CompressionAlgorithm.DeflateRaw);
var decompressedStream = await tee2.PipeThroughAsync(decompressionStream);

var writeStream = new System.IO.MemoryStream();
await decompressedStream.CopyToAsync(writeStream);
decompressedStreamSize = writeStream.Length;
content = System.Text.Encoding.UTF8.GetString(writeStream.ToArray());
```
Then we are done. Let's run it and see the result.
```bash
dotnet run
```
And then we go to the index page. Normally I would have made a video for this, but this sample isn't really that interesting, so I've just copied the result here as what is actually interesting is that the text was compressed and that the resulting content was intact.
```bash
Compressed size was: 3696
Decompressed size was: 11481
Lorem ipsum dolor sit amet, consectetur adipiscing elit. Morbi et ex a dolor pulvinar euismod... (continuing)
```
You can check out the demo here yourself: [https://kristofferstrube.github.io/Blazor.CompressionStreams/](https://kristofferstrube.github.io/Blazor.CompressionStreams/)

And you can check out the GitHub project here: [https://github.com/KristofferStrube/Blazor.CompressionStreams](https://github.com/KristofferStrube/Blazor.CompressionStreams) (Throw me a star if you enjoyed the post.)

### Conclusion
Now, we have seen an approach for wrapping browser APIs in Blazor WASM. We have wrapped the `CompressionStream` interface and the other related interfaces defined in the Compressions Streams API. And in the end, we have shown a small example that validates that the wrapper has the intended behavior. This post is a precursor to a series of posts that I will make on the topic of streaming files to and from Blazor WASM clients using my wrapper classes which I'm looking forward to getting started with. If you have any questions related to the article or comments then feel free to reach out.