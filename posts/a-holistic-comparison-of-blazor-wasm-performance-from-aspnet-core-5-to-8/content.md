### The sample project
I have tried to make a sample project that represents some common work. For this, I started with my relatively new [DocumentSearching](https://github.com/KristofferStrube/DocumentSearching) repository that implements a Suffix Trie for document searching. This is one of the simplest approaches for indexing text so that it is faster to search for all indexes of any substring in an input string. Below is a demo of what using this index can look like. You can try out this demo yourself here: [https://kristofferstrube.github.io/DocumentSearching/](https://kristofferstrube.github.io/DocumentSearching/)

<video width="800px" src="{site}/videos/search-periodic-table.mp4" autoplay muted controls loop></video>

The implementation is pretty naive (let's say that's intentional), so constructing the index reads through the input text many times, making it a rather intensive operation. Since the construction is expensive, it will make sense to save the constructed index somewhere in some cases so that it simply needs to be read to be ready for use. Inspired by this use case, I made a series of 7 tasks representing some common tasks.

#### Read CSV
We need some text to search in before constructing the search index itself. For this, I have found a CSV file containing descriptions for all the elements of the Periodic Table as a sample of something to search through. We will load this file using a `HttpClient` and then parse it using [CsvHelper](https://joshclose.github.io/CsvHelper/). To limit the size of this test, we will only load the descriptions of the first 10 elements. The summaries of the first 10 elements consist of 3026 characters. We place a scope around this part so that the `StreamReader` and `CsvReader` get disposed of before measuring the time.

```csharp
{
    Stream periodicTableCsv = await HttpClient.GetStreamAsync("data/periodic-table-detailed.csv");
    using StreamReader streamReader = new StreamReader(periodicTableCsv);
    using CsvReader csv = new(streamReader, CultureInfo.InvariantCulture);
    elements = csv.GetRecords<Element>().Take(10).ToArray();
}
```

#### Construct Index
Using the entries from the CSV file, we will then construct the search index using the `DocumentIndex` class from the repository. I have made a branch of the repository where the class library targets .NET 5 so that it can be used in all the versions we run tests for. The branch is here: [DocumentSearching/experiment/aspnetcore5](https://github.com/KristofferStrube/DocumentSearching/tree/experiment/aspnetcore5). With the `DocumentIndex<Element>.Create` method, we create an index over all the summary for each element. We lower the summary text so that we don't need to think about the casing as long as we also lower our search query.

```csharp
index = DocumentIndex<Element>.Create(elements, e => e.Summary.ToLower());
```

#### JSON Serialize
We will then serialize the search index. The Suffix Trie, the index's main part, consists of a large hierarchy of `Node` classes that point to each other. This is not unlike the very large models that some send between their clients and servers when the users work with complex models.

We define the following serialization options object outside the main test loop for this and the deserialization.
```csharp
private readonly JsonSerializerOptions serializerOptions = new()
{
    ReferenceHandler = ReferenceHandler.Preserve
};
```
And then use it like this when serializing:
```csharp
string serialized = JsonSerializer.Serialize(index, serializerOptions);
```

#### Write to LocalStorage
Next, we need to save it somewhere. We will use JSInterop to write the serialized index to LocalStorage for this. This is also a common part of many Blazor WASM applications that rely on APIs from the browser through JSInterop to deliver a rich experience.

```csharp
JSRuntime.InvokeVoid("window.localStorage.setItem", "index", serialized);
```

#### Read from LocalStorage
In a real use case, we could imagine that we had left the site and revisited it. So, we need to read the serialized index back from LocalStorage. For this, we likewise use JSInterop to read the string back. I don't expect this to be very different from the writing to LocalStorage, but it completes the story.

```csharp
string readIndex = JSRuntime.Invoke<string>("window.localStorage.getItem", "index");
```

#### JSON Deserialize
Now, we reconstruct the search index from the previously serialized string. It is not uncommon to talk with some JSON-based API in a Blazor WASM application, which also involves a lot of deserialization. Depending on how data-driven your site is, this might be a hot-path in your application.

```csharp
DocumentIndex<Element> deserializedIndex = JsonSerializer.Deserialize<DocumentIndex<Element>>(readIndex, serializerOptions);
```

#### Search
With the index back, we can perform some searches in the elements. I have chosen to search for 10 words that I know appear in the summaries of the first 10 elements. The search is so fast that the `Stopwatch` class I'm using for my timings wouldn't be able to measure 10 searches alone, so I'm putting these 10 searches in a loop that iterates 10 times so that we have something comparable. When searching, the index does a lot of equality checks and lookups in arrays to find the matching indexes, which could mimic the work done in applications that evaluate a lot of business logic on the client side.

```csharp
for (int j = 0; j < 10; j++)
{
    searchResults = deserializedIndex.ExactSearch("hydrogen");
    searchResults = deserializedIndex.ExactSearch("oxygen");
    searchResults = deserializedIndex.ExactSearch("light");
    searchResults = deserializedIndex.ExactSearch("heavy");
    searchResults = deserializedIndex.ExactSearch("chemical");
    searchResults = deserializedIndex.ExactSearch("element");
    searchResults = deserializedIndex.ExactSearch("symbol");
    searchResults = deserializedIndex.ExactSearch("atomic");
    searchResults = deserializedIndex.ExactSearch("number");
    searchResults = deserializedIndex.ExactSearch("weight");
}
```

### Testing
I ran the steps above in their natural order inside a loop that repeated 110 times. I timed how fast each part took using the `Stopwatch` class by starting one and then measuring how many milliseconds passed. After collecting 110 measurements for each part, I removed the first and last 5 measurements and calculated the minimum, average, and maximum values. This is not a recommendation for how to make benchmarks for Blazor WASM, as many factors can influence the results when other work is done on the computer simultaneously. But it is better than simply measuring the time once.

The following is the main loop that we will execute in our tests: 
```csharp
private Element[] elements = default!;
private DocumentIndex<Element> index = default!;
private SearchResult<Element>[] searchResults = [];

public async Task Start()
{
    for (int i = 0; i < 110; i++)
    {
        // Read CSV

        // Construct Index

        // JSON Serialize Index

        // Write to LocalStorage

        // Read from LocalStorage

        // JSON Deserialize Index

        // Search for words in summaries.

        // Cleanup
        elements = default!;
        index = default!;
        JSRuntime.InvokeVoid("window.localStorage.clear");
        searchResults = [];
        await Task.Delay(200);
    }
}
```
At the end of every loop cycle, we also do some cleanup. Most of it is simple: returning the intermediate variables to their initial values. We also clear LocalStorage so it is empty at the start of every iteration. We also make a 200-millisecond delay. This is so that the runtime gets time to run any garbage collection it needs to perform and to ensure that the UI does not become unresponsive when using the single thread available in Blazor WASM for long periods.

After running the loop, we get the aforementioned statistics from all the collections of measurements. We do this by parsing the collections to the following method that formats the statistics we are interested in before we output them for each part of the loop.
```csharp
private string GetStatistics(List<double> timings)
{
    double[] middleTimings = timings.Skip(5).SkipLast(5).ToArray();

    double min = Math.Round(middleTimings.Min(), 2);
    double average = Math.Round(middleTimings.Average(), 2);
    double max = Math.Round(middleTimings.Max(), 2);

    return $"Min: {min} ms; Average: {average} ms; Max: {max} ms;";
}
```

I will use the fastest of the 100 measurements in the plots I make later, so it wasn't necessary to throw away the first and last 5 as they could only be slower than the rest depending on how the runtime optimizes the loop. But it still makes sense to skip these to get a more meaningful insight into a long-running process's average and maximum values. Here is an example of the minimal, average, and maximal measurements made for the test project targeting ASP.NET Core 5 with .NET 5 published with the standard release configuration:
```bash
Read CSV: Min: 5.8 ms; Average: 9.77 ms; Max: 29.3 ms;
Construct Index: Min: 5.7 ms; Average: 6 ms; Max: 7.9 ms;
JSON Serialize: Min: 331 ms; Average: 336.53 ms; Max: 349.2 ms;
Write LocalStorage: Min: 208.4 ms; Average: 211.46 ms; Max: 220.8 ms;
Read LocalStorage: Min: 239.4 ms; Average: 242.46 ms; Max: 255.8 ms;
JSON Deserialize: Min: 1041.7 ms; Average: 1052.56 ms; Max: 1093.3 ms;
Search: Min: 12.5 ms; Average: 12.87 ms; Max: 17.3 ms;
```
I had expected that the measurements would vary a lot more. If we look at the proximity of the min values and average values, we see that they are generally pretty close. Without doing any statistical analysis, this tells me that most measurements are close to the minimum value.

### Test scenarios
I started by making a sample Blazor WASM project that targeted .NET 5. In this, I created a page with the above methods defined. It is really easy to upgrade a Blazor WASM project from .NET 5 and ASP.NET Core 5 up to 8. It is just to update the versions of the Blazor-specific NuGet packages to match the selected .NET version. Apart from this, we also want to try out AOT for all the versions of Blazor WASM that support it. Actually, of the versions we want to test, only Blazor in ASP.NET Core 5 didn't have it yet. But that is fine, as it simply re-iterates how great a jump AOT was when introduced in .NET 6. To configure the application to use AOT, we only need to add the following to the csproj:

```xml
<RunAOTCompilation>true</RunAOTCompilation>
```

Now, we are ready to run our tests. We are going to run tests on the following configurations:

- .NET 5 - ASP.NET Core 5.0.17 Release
- .NET 6 - ASP.NET Core 6.0.26 Release
- .NET 6 - ASP.NET Core 6.0.26 Release AOT
- .NET 7 - ASP.NET Core 7.0.15 Release
- .NET 7 - ASP.NET Core 7.0.15 Release AOT
- .NET 8 - ASP.NET Core 8.0.0 Release
- .NET 8 - ASP.NET Core 8.0.0 Release AOT

You can find the ASP.NET Core 5 base project that we will upgrade to each of the above configurations here: [KristofferStrube.DocumentSearching.BlazorWasm5](https://github.com/KristofferStrube/DocumentSearching/tree/experiment/aspnetcore5/samples/KristofferStrube.DocumentSearching.BlazorWasm5)

### Results
And now for the results! I have made some plots presenting the results for each part,

#### Read CSV
Reading the CSV had some improvement when moving to ASP.NET Core 6, but it did not seem like AOT has a large impact in this version. Surprisingly, ASP.NET Core 7 performed worse but did have another great jump in performance for AOT. In  ASP.NET Core 8, we got the standard release down to the same level that we had in 6 and matched the AOT level of 7, so it seems like the greatest of both versions. Generally, there are not any huge improvements, which makes sense as there are other network-related constraints to reading a file fast with a `HttpClient`.

![Plot showing the performance of reading CSVs in different versions of Blazor WASM]({site}/images/read-csv.png)

#### Construct Index
Constructing the search index also improved when moving to ASP.NET Core 6. Though, again not a huge difference with and without AOT. Then, at ASP.NET Core 7, we see a huge improvement for AOT and again see an actual regression in the standard release. As for ASP.NET Core 8, we again remain around the same level for AOT but have a nice catchup for the non-AOT that is now twice as fast as the ASP.NET Core 6 AOT version.

![Plot showing the performance of constructing a search index in different versions of Blazor WASM]({site}/images/construct-index.png)

#### JSON Serialize
Serializing the search index seems to have followed an almost identical pattern of improvements. I had expected this to have a much greater percentage improvement as we have heard a lot about the improvements made to System.Text.Json throughout every release. But if we instead look at the absolute improvement of the task, this is a pretty impressive improvement, cutting a serialization task that previously took more than 300 milliseconds, which is a great enough delay for a person to perceive it, down to less than 25 milliseconds in ASP.NET Core 8 with AOT.

![Plot showing the performance of JSON serializing in different versions of Blazor WASM]({site}/images/json-serialize.png)

#### Write LocalStorage
Again, it is a similar picture to what we had in the two other parts when we look at the performance of writing to LocalStorage. The one noticable thing is that writing to LocalStorage in ASP.NET Core 6 AOT is much better than its non-AOT counterpart. So, it seems that JSInterop is especially receptive to AOT's optimizations. 

![Plot showing the performance of writing to LocalStorage with JSInterop in different versions of Blazor WASM]({site}/images/write-localstorage.png)

#### Read LocalStorage
I had expected that returning values from JSInterop would be consistently slower than writing as we need to allocate a new string. But it seems like they are close to being equally matched. The one surprising thing to notice is that reading in ASP.NET Core 6 AOT is more than twice as fast as writing. I will definitely have this in the back of my mind when I make systems that are JSInterop intensive in the future.

![Plot showing the performance of reading from LocalStorage with JSInterop in different versions of Blazor WASM]({site}/images/read-localstorage.png)

#### JSON Deserialize
The JSON deserialization is the slowest of all the parts. I assume that this is because deserialization relies heavily on reflection combined with some good old string parsing. That we start off with having the deserialization take more than a second in ASP.NET Core 5 only makes it even more impressive that we got down below 50ms in the ASP.NET Core 8 AOT release.

![Plot showing the performance of JSON deserializing in different versions of Blazor WASM]({site}/images/json-deserialize.png)

#### Search
And now to the final part that all the previous steps have led to. The search itself. In ASP.NET Core 6, AOT was slower than the non-AOT counterpart, which is a contradiction to what almost seems like a rule from the previous plots. But overall, they still follow the pattern we have seen in the other plots, i.e., that non-AOT has gotten better through all versions except ASP.NET Core 7 and that the AOT version gets better over time, where the one to 7 is the greatest jump.

![Plot showing the performance of searching with constructed search index in different versions of Blazor WASM]({site}/images/search.png)

### Conclusion
Now, we have seen how fast some common tasks are performed in Blazor WASM with different versions of ASP.NET Core. A couple of tests did not have the outcome I had expected before I started the experiments. The greatest surprise is how ASP.NET Core 7 non-AOT was worse than 6 in almost all tasks. I want to explore the source of this in the future as it seems like something others should have reported on before me. Apart from this, I also found that JSON is not a valid serialization mechanism for the sample use case we created, as it was the limiting part across all the different ASP.NET Core versions we tested. But it was okay as a sample. I can't wait to see what improvements we will see in ASP.NET Core 9 when we get the first previews pretty soon. I do a lot of JSInterop, so I especially hope for more improvements to that both for AOT and non-AOT, as only Blazor WASM can employ AOT.