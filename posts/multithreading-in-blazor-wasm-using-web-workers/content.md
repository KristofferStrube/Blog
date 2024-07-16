### Web Workers
Web Workers are defined as a part of [the HTML specification](https://html.spec.whatwg.org/multipage/workers.html#workers). They enable us to run some scripts in the background to avoid blocking the primary thread when we do heavy work. This can be a common problem in Blazor WASM, where the UI can become unresponsive when a heavy workload is executed.

In JavaScript, we can create a worker and listen for messages posted in the worker like so:
```js
let worker = new Worker("my-worker-script.js", { type: "classic" });

worker.addEventListener("message", log)

function log(e) {
    console.log("Some message from the worker: " + e.data);
}
```

The content of the `my-worker-script.js` file would then define the worker's work. An example could be posting a message to itself, which is what we listen for in the previous code block.

#### my-worker-script.js
```js
self.postMessage("This was posted from the worker!");

console.log("We can also log to the console directly from here.")
```

### Tewr.BlazorWorker
[Tewr.BlazorWorker](https://github.com/Tewr/BlazorWorker) is one of the first Web Workers abstractions made for Blazor WASM. It is made by [Tewr](https://github.com/Tewr), whom you might also know from the [BlazorFileReader](https://github.com/Tewr/BlazorFileReader) library. It uses the [Serialize.Linq](https://github.com/esskar/Serialize.Linq) library to serialize expressions that represent the work that is to be done on the background thread. First, the worker is initialized, which also starts the Blazor runtime from the worker thread. Then, the serialized expression is posted to the worker. The Blazor application, which the worker initializes, listens for messages sent to the worker, and when it sees a new message, it deserializes the expressions and invokes it.

The following is a minimal sample of what you need to do some simple work on another thread.
First, you need to install this NuGet package
```console
Tewr.BlazorWorker.BackgroundService
```
And then add its service to your service collection in `Program.cs`.
```csharp
builder.Services.AddWorkerFactory();
```
Then, on a page, make a setup like this.

```razor
@using BlazorWorker.BackgroundServiceFactory
@using BlazorWorker.Core
@inject IWorkerFactory workerFactory

<button @onclick=ExecuteOnTewrBlazorWorker>Execute</button>
<br/>
<code>@result</code>

@code {
    private string result = "";

    private async Task ExecuteOnTewrBlazorWorker()
    {
        IWorker worker = await workerFactory.CreateAsync();
        var service = await worker.CreateBackgroundServiceAsync<MathService>();

        double input = Random.Shared.NextDouble();

        double output = await service.RunAsync(math => math.MutliplyByTwo(input));

        result = $"{input} * 2 = {output}";
    }

    public class MathService
    {
        public double MutliplyByTwo(double input)
        {
            return input * 2;
        }
    }
}
```
The above code sample initializes a service called `MathService` in the worker and then invokes the method `MutliplyByTwo` with a random number created on the main thread. The sample is very basic, and obviously, multiplying a number by two is not very expensive. But the idea is still recognizable: Taking some input from the main thread, running some method on the worker thread, and, in the end, returning the result to the main thread.

After going through this minimal sample, my first impression is that it indeed feels pretty minimal. The only part I'm unsure about is the use of `Serialize.Linq`, which it uses to deserialize and compile the expression every time it is called. Depending on how well the .NET WASM runtime can optimize the generated IL code, this could be expensive.

### SpawnDev.BlazorJS.WebWorkers
[SpawnDev.BlazorJS.WebWorkers](https://github.com/LostBeard/SpawnDev.BlazorJS?tab=readme-ov-file#spawndevblazorjswebworkers) is a newer implementation which is a part of [LostBeard](https://github.com/LostBeard)s project [SpawnDev.BlazorJS](https://github.com/LostBeard/SpawnDev.BlazorJS). It likewise uses [Serialize.Linq](https://github.com/esskar/Serialize.Linq) to serialize the expression that should be evaluated on the worker. But it requires a bit more setup.

Let's see an equivalent to the previous sample but with `SpawnDev.BlazorJS.WebWorkers`. We first need to add their NuGet package:
```console
SpawnDev.BlazorJS.WebWorkers
```
Then we need to do a bit of setup in `Program.cs`.
```csharp
builder.Services.AddBlazorJSRuntime();
builder.Services.AddWebWorkerService();
builder.Services.AddSingleton<IMathService, MathService>();

await builder.Build().BlazorJSRunAsync();
```
We add the services needed for `BlazorJS` to work, a service needed for accessing and creating web workers, and inject `MathService` as a service. Then, we change a significant part of our `Program.cs` file. We normally call `RunAsync()` after building the `WebAssemblyHost` but here we call `BlazorJSRunAsync()` instead. This is the primary part that changes the control flow of our Blazor application. The `BlazorJSRunAsync` method checks whether we are running in the window context, in which case it starts Blazor like usual. Otherwise, it simply idles and listens for messages.

We have updated our `MathService` a bit by making it implement an interface. This is necessary so the library can create a proxy for the service.
```csharp
public interface IMathService
{
    public double MutliplyByTwo(double input);
}

public class MathService : IMathService
{
    public double MutliplyByTwo(double input)
    {
        return input * 2;
    }
}
```
Then, on some page, we can make our minimal sample again:
```razor
@using SpawnDev.BlazorJS.WebWorkers
@inject WebWorkerService WebWorkerService

<button @onclick=ExecuteOnSpawnDevWebWorker>Execute</button>
<br/>
<code>@result</code>

@code {
    private string result = "";

    private async Task ExecuteOnSpawnDevWebWorker()
    {
        WebWorker? worker = await WebWorkerService.GetWebWorker();

        double input = Random.Shared.NextDouble();

        double output = await worker!.Run<IMathService, double>(job => job.MutliplyByTwo(input));

        result = $"{input} * 2 = {output}";
    }
}
```

This was a lot more setup than the `Tewr` sample, even though it seems to use the same core principles. However, that can also be helpful, as it makes certain parts of the internals more transparent to us. This can help library users troubleshoot problems with greater ease.

### KristofferStrube.Blazor.WebWorkers
Before looking at how the other solutions worked, I tried to implement my own wrapper for calling .NET through Web Workers. It differs by two key points.

1. It does not start a separate Blazor instance in the worker but instead starts a .NET application using the `wasm-experimental` workload.
2. It does not use `Serialize.Linq` and instead enforces a standard format for a job that needs to be implemented in a separate project.

This makes the implementation a bit more limited. The work being executed by the worker must have a single input and output. Apart from this, you can't use normal Blazor JSInterop in a `wasm-experimental` project. But if we assume you don't need to use JSInterop for your background job, then this should be fine. So, let's see what it looks like.

You first need to install the `wasm-experimental` workload
```bash
dotnet workload install wasm-experimental
```

Then, you need to create a separate project in your solution. You can create this project using the standard .NET 8 console template
```bash
dotnet new console
```
To make this a `wasm-experimental` project, you must adjust the `.csproj` file to look like this.
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <OutputType>Exe</OutputType>
    <RuntimeIdentifier>browser-wasm</RuntimeIdentifier>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="KristofferStrube.Blazor.WebWorkers" Version="0.1.0-alpha.6" />
  </ItemGroup>

</Project>
```
Next, we need to add a class that implements the abstract `JsonJob` class to the `wasm-experimental` project. The `JsonJob` class has a single abstract method called `Work`, where you should implement the work that the job needs to do. The two type parameters of the `JsonJob` class define the input and output of the job.
```csharp
using KristofferStrube.Blazor.WebWorkers;

public class MultiplyByTwoJob : JsonJob<double, double>
{
    public override double Work(double input)
    {
        return input * 2;
    }
}
```
To finish the worker part, we just need to update our `Program.cs` to start our job. Before this, we also checked that it was running in the browser, as this depends on using the interop services for JavaScript, which are only available when running in the browser.
```csharp
if (!OperatingSystem.IsBrowser())
    throw new PlatformNotSupportedException("Can only be run in the browser!");

await new MultiplyByTwoJob().StartAsync();
```
We then need to take a dependency on the `wasm-experimental` project from our main Blazor project, and then we are ready to make our minimal sample
```razor
@using KristofferStrube.Blazor.WebWorkers
@inject IJSRuntime JSRuntime

<button @onclick=ExecuteOnBlazorWebWorkers>Execute</button>
<br/>
<code>@result</code>

@code {
    private string result = "";

    private async Task ExecuteOnBlazorWebWorkers()
    {
        var worker = await JobWorker<double, double, MultiplyByTwoJob>.CreateAsync(JSRuntime);

        double input = Random.Shared.NextDouble();

        double output = await worker.ExecuteAsync(input);

        result = $"{input} * 2 = {output}";
    }
}
```
My solution required even more setup to get the minimal sample up and running, but it also differs a lot, so there might be other gains from this.
### Performance comparisons
Now that we have seen three ways to do the same work in the background, let's make some simple benchmarks of how the different solutions perform in various settings. In all these experiments, we will repeat the experiment 1000 times and then take the average of the best 100 results to avoid including outliers. We will also compare the three wrapper implementations with the same work implemented in JavaScript.

All of the following comparisons are also available here if you want to check the code directly or try to reproduce the results on your own machine:

Repository: [https://github.com/KristofferStrube/Blazor.WorkersBenchmarks](https://github.com/KristofferStrube/Blazor.WorkersBenchmarks)

Live Demo: [https://kristofferstrube.github.io/Blazor.WorkersBenchmarks/](https://kristofferstrube.github.io/Blazor.WorkersBenchmarks/)

#### Sending large inputs
It can be expensive to send the parameters needed to run the job. This can be one of the reasons why it might not be worth it to use a worker, as the additional workload of serializing the input would still need to be done on the main thread.

To compare this, we have constructed a simple job that receives a string and always returns the number 42. We intentionally do not use the input to calculate something, as we would then measure the processing time of the input as well.
```csharp
public class LargeInputJob : JsonJob<string, int>
{
    public override int Work(string input)
    {
        return 42;
    }
}
```

Let's see the result of this small benchmark.

![Results from measuring the average running time for different worker wrappers](https://kristoffer-strube.dk/images/large-input.png)
Surprisingly, `Tewr.Blazor.Worker` seems to be much slower than `SpawnDev.BlazorJS.WebWorkers`. It appears to have a larger overhead from starting some work, but it also grows in running time much faster than the other implementations. If we create a linear fit for the four different sets of measurements, we get the following base overheads and growth rates.

<br />
<table>
<tr><th>Worker Implementation</th><th>Base in milliseconds</th><th>Growth in milliseconds per 10000 chars</th></tr>
<tr><td>Tewr.BlazorWorker</td><td>5.776</td><td>1.033</td></tr>
<tr><td>SpawnDev.BlazorJS.WebWorkers</td><td>1.576</td><td>0.196</td></tr>
<tr><td>KristofferStrube.Blazor.WebWorkers</td><td>0.813</td><td>0.150</td></tr>
<tr><td>JS Worker</td><td>0.317</td><td>0.087</td></tr>
<table>
<br />

I like to present these numbers as they make clear what can be challenging to read from the plot. The primary interesting part that we can read from these numbers is that the different sets of measurements will get the same order if we sort them by their base overhead or if we sort them by their growth rate. This means that using JS for this type of work will always be most favorable, no matter how big our input is. But if we were to limit this comparison to only include .NET implementations, then our solution would be the best. That JS won is unsurprising as we need to send our string through JSInterop two times when working with the .NET implementations. First, send it from .NET to JS, then send it back to our Blazor or `wasm-experimental` application from JS in the worker.

#### CPU-intensive work
Our next benchmark is going to measure the speed of the work being done on the worker itself. This is an especially interesting experiment as it will compare JavaScript against Compiled Expressions and .NET pre-compiled. For this reason, we will also measure both AOT and non-AOT in this benchmark to see how big this influence has on the different .NET implementations. The work we are going to measure in this sample is going to be very simple but, nonetheless, very CPU intensive when run for a big input. We will run a for loop up to the input number and sum up all the even numbers. This is the kind of work that could freeze the UI in Blazor if we were to run it on the main thread. This time, we will return the result, not because we wish to measure how long larger numbers take to return but for reasons that will become clear soon. This job is implemented like this in .NET:
```csharp
public class SumEvenNumbersJob : JsonJob<int, int>
{
    public override int Work(int input)
    {
        int sum = 0;
        for (int i = 0; i < input; i++)
        {
            if (i % 2 == 0)
            {
                sum += i;
            }
        }
        return sum;
    }
}
```

Let's see the results.

![Results from measuring the average running time for summing even numbers up to some input with different worker wrappers](https://kristoffer-strube.dk/images/sum-even-numbers.png)

We first see that we have the same order as before when sorting by speed. But this time, we don't see a clear difference in which .NET implementation grows the fastest. We still see that the JS worker is the fastest and it seems to grow much slower than the other implementations.

But all is not lost. Let's repeat this benchmark with the project AOT compiled. This was why we ensured that the sum was returned in this sample, as AOT would else remove the for-loop entirely if the sum had not been used. This should make a considerable improvement for our .NET implementations, as we have seen in our last article on this topic: [A holistic comparison of Blazor WASM performance from ASP.NET Core 5 to 8](https://kristoffer-strube.dk/post/a-holistic-comparison-of-blazor-wasm-performance-from-aspnet-core-5-to-8/). We will start this experiment with some bigger inputs as the .NET implementations are now so efficient that they would seem not to change at the previous scale.

![Results from measuring the average running time for summing even numbers up to some input with different worker wrappers that are AOT compiled
](https://kristoffer-strube.dk/images/sum-even-numbers-aot.png)

Notice that we now increment our input with 1 million instead of 10 thousand for each of the points at which we measure time. This is why the JS workers' change in speed seems steeper now, even though the speed has not changed significantly. But let's focus on what we are interested in. .NET is faster than JS for the same job on a worker thread! This is the first time I have seen a direct comparison of .NET against JavaScript, so I was excited to know that .NET can outperform JS for CPU-intensive work in the browser.

#### Reading, processing, and outputting
The next case is a more complete demonstration of a real workload. We will make a mini-version of the [The One Billion Row Challenge](https://github.com/gunnarmorling/1brc). The challenge at its core is to read some files with temperature measurements and then output each city's minimum, average, and maximum temperatures. To make this a bit simpler, I have chosen to create an endpoint on my personal API that can return measurements for some number of cities. Apart from this, the task is still the same. This also makes it closer to some standard browser work: getting some string, parsing it, reading through it, and reporting aggregated results. I have tried to find the simplest possible solution, as I could not make the same performance improvements I know of in .NET when making the JS implementation.

This is the simple implementation in .NET:

```csharp
using KristofferStrube.Blazor.WebWorkers;

public class AverageCityTemperaturesJob(HttpClient httpClient) : TaskJsonJob<int, CityStatistics[]>
{
    private const string MeasurementsEndpoint = "https://kristoffer-strube.dk/API/not-the-measurment-endpoint/";
    private string? responseAsText;

    public override async Task<CityStatistics[]> Work(int input)
    {
        if (responseAsText is null)
        {
            responseAsText = await httpClient.GetStringAsync(MeasurementsEndpoint + input);
        }

        Dictionary<string, CityAggregate> aggregates = new();

        foreach (string line in responseAsText.Split("\n"))
        {
            string[] parts = line.Split(";");
            string city = parts[0];
            float temperature = float.Parse(parts[1]);

            if (!aggregates.TryGetValue(city, out CityAggregate? aggregate))
            {
                aggregate = new();
                aggregates[city] = aggregate;
            }
            aggregate.Min = temperature < aggregate.Min ? temperature : aggregate.Min;
            aggregate.Max = temperature > aggregate.Max ? temperature : aggregate.Max;
            aggregate.Sum += temperature;
            aggregate.Count++;
        }

        var result = new List<CityStatistics>();
        foreach ((string city, CityAggregate aggregate) in aggregates)
        {
            result.Add(new()
            {
                City = city,
                MinTemperature = aggregate.Min / 10.0,
                AverageTemperature = Math.Round(aggregate.Sum / 10.0 / aggregate.Count, 1),
                MaxTemperature = aggregate.Max / 10.0
            });
        }

        CityStatistics[] resultAsArray = result.ToArray();

        return resultAsArray;
    }

    private class CityAggregate
    {
        public float Min { get; set; } = float.MaxValue;
        public float Max { get; set; } = float.MinValue;
        public float Sum { get; set; }
        public int Count { get; set; }
    }
}
```

and here an implementation that uses the same structure, but in JavaScript:

```js
const measurementsEndpoint = "https://kristoffer-strube.dk/API/not-the-measurment-endpoint/"

let responseAsText = undefined;

async function work(input) {
    if (responseAsText == undefined) {
        let response = await fetch(measurementsEndpoint + input);
        responseAsText = await response.text();
    }
    
    let aggregates = new Map();

    for (line of responseAsText.split('\n')) {
        let parts = line.split(';');
        let city = parts[0];
        let temperature = parseFloat(parts[1]);

        let aggregate = aggregates.get(city);
        if (aggregate == undefined) {
            aggregate = { min: Number.MAX_VALUE, max: -Number.MAX_VALUE, sum: 0, count: 0 };
            aggregates.set(city, aggregate);
        }
        aggregate.min = temperature < aggregate.min ? temperature : aggregate.min;
        aggregate.max = temperature > aggregate.max ? temperature : aggregate.max;
        aggregate.sum += temperature;
        aggregate.count++;
    }

    let result = [];

    aggregates.forEach((aggregate, city) => {
        result.push({
            city: city,
            minTemperature: aggregate.min,
            averageTemperature: Math.round((aggregate.sum / aggregate.count + Number.EPSILON) * 10) / 10,
            maxTemperature: aggregate.max,
        })
    })

    return result;
}
```

Common for the two implementations is that they cache the result from the API as a string. We do this for two reasons. We are not interested in comparing how fast JavaScript or .NET can download something in the browser. Secondly, download speed can vary a lot, which would add unnecessary noise to the results or the benchmark. We ran the .NET implementation with AOT compilation as I don't expect it to be comparable with the speed of JS without.

Let's see the results.

![Results from measuring the average running time for finding the average temperature of cities with different worker wrappers](https://kristoffer-strube.dk/images/average-city-temperature-aot.png)

Not the result we were hoping for, but let's look at it anyway. None of the .NET workers were faster than the JS worker. I tried to dig deeper into the code and make some ad-hoc micro-benchmarks of the core parts, and it seems like JS is faster at most of them, even when we AOT compile. Some of the critical parts of performance are the lookups in the dictionary/map and the conversion from string to float. Both of these were faster in JS. My assumption is that the browsers have some efficient implementation that is not written in JS, which they use for these operations. It makes sense that the browser has efficient map implementations, as most browsers use the same underlying implementation for maps and objects. So, every time some attribute of an object in JS is accessed, it essentially equates to a map lookup. Furthermore, I expect that this is optimized for string keys as most attributes on objects are identified by a string key. For conversion from string to float, it also makes sense that the browsers have some efficient implementation they use for making this conversion for any input that is number typed or when parsing JSON with numbers in them.

### Conclusion
Now, we have seen how we can make some work in the background using JS and Web Workers. Following that, we have seen 2 of the most popular Blazor wrappers for Web Workers that make it possible to evaluate some .NET code in a background thread. Next, we presented our own novel approach for making a .NET Web Worker, which uses the `wasm-experimental` workload. In the end, we compared three different equivalent pieces of work in JS and each of the three wrapper implementations. The first comparison looked at the performance of workloads that depend on large inputs, and for this, we saw that JS was the fastest but that our new approach for Web Workers outperformed the existing implementations. The general picture we saw from this was that there was a 2X overhead from using our Web Worker implementation compared to starting a JS worker from Blazor. Depending on your work, this might still make it worth using our implementation. You might be better equipped to maintain the .NET implementation or might not have an implementation available for JS. The next comparison was on CPU-intensive work. We first saw that JS was still faster, but after turning on AOT compilation for the .NET workers we saw that they all outperformed JS for sufficiently large workloads. Our .NET worker implementation still outperformed the others, but this time, it had the same scaling factor and only won due to a smaller startup overhead. The last benchmark was intended to represent some varied work involving reading and processing data. For this, we again saw that the JS worker was faster than the .NET workers, even with AOT on. We reasoned that JS probably uses some optimized subroutines for some of the most performance-heavy parts, which might be why it outperformed the .NET worker implementations. Even though JS outperformed .NET, it could still make sense to use .NET workers to keep your codebase coherent or if you don't have a JS implementation for the work you need to do. To conclude, using Web Workers in Blazor looks very promising, and I look forward to using this in some projects very soon. If you have any questions related to this article or our Worker implementation, feel free to reach out to me.