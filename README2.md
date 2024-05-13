# Dotnet Optimization Cheatsheet

Donald Knuth said:

>Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

If you're the 3%, this is an educational look at .NET performance tricks I've gathered over time. By reading this I assume you understand that architecture, data flow, general performance decisions, hardware, etc have a far larger impact to your codebase than optimising for single digit milliseconds or nanoseconds. 

You'll need to consider:
1. Will I be pre-maturely optimising?
1. Understand the risks involved
1. Is the time spent doing this worth it?


I've rated each optimisation here in terms of difficulty. You may gauge the difficulties differently. 

| Difficulty       | Reasoning                                                                                                                                                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🟢 Easy         | These are either well known, or simple to drop into existing code with only some knowledge.                                                                                                                                               |
| 🟡 Intermediate | Mostly accessible. May require a bit more coding or understanding of specific caveats of use. <br> I won't explain all the caveats, but you'll have knowledge that there may need to be some reasearch on something you need to watch out for. |
| 🔴 Advanced     | Implementation and caveat understanding is critical to prevent app crashes or accidentally reducing performance.                                                                                                                          |

.NET is always evolving. Some of these may be invalid in the future due to breaking changes or they've been baked into the framework under the hood. 

The optimizations below can range from efficient object overloads to language features. They'll include relevant documentation and attribution to where I originally found them.

## Beginning Optimisation

In the shortest way possible:

1. Write unit tests so you have a baseline for your functionality
1. Use something like [BenchmarkDotNet](https://benchmarkdotnet.org/) or the [Performance Profiler](https://learn.microsoft.com/en-us/visualstudio/profiling/?view=vs-2022) in Visual Studio to get a before performance baseline
1. Write your change
1. Run tests
1. Benchmark to see if it did improve what you expected

If you're looking for tools to help, I have: [Optimization-Tools-And-Notes](https://github.com/nikouu/Optimization-Tools-And-Notes) 

## Further Resources

| Resource                                                                                                               | Description                                                                                      |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| [PerformanceTricksAzureSDK](https://github.com/danielmarbach/PerformanceTricksAzureSDK)                                | Tricks written by [Daniel Marbach](https://github.com/danielmarbach)                             |
| [.NET Memory Performance Analysis](https://github.com/Maoni0/mem-doc/blob/master/doc/.NETMemoryPerformanceAnalysis.md) | A thorough document on analyzing .NET performance by [Maoni Stephens](https://github.com/Maoni0) |
|                                                                                                                        |                                                                                                  |

## Optimisations

[_We always want our code to "run faster". But rarely do we ask – what is it running from?_](https://twitter.com/moyix/status/1449397749343047681)

### 🟢 Use fewer nullable value types

[Via Bartosz Adamczewski](https://twitter.com/badamczewski01/status/1542799681717149697)

[Nullable value types are boxed](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types#boxing-and-unboxing) meaning there is unboxing work and null checking work happening. 

```csharp
// slower with nullable value types
int? Slow(int? x, int? y) {
	int? sum = 0;
	for (int i = 0; i < 1000; i++){
		sum += x * y;
	}
	return sum;
}

// faster with non-nullable value types
int? Fast(int x, int y) {
	int? sum = 0;
	for (int i = 0; i < 1000; i++){
		sum += x * y;
	}
	return sum;
}
```

### 🟢 Use `String.Compare()`

[Twitter post via Daniel Lawson](https://twitter.com/danylaws/status/1504381170347294727)

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.string.compare?view=netcore-3.1)

`String.Compare()` is a memory efficient way to compare strings. This is in respect to the memory inefficient method of comparing by doing `stringA.ToLower() == stringB.ToLower()` - which due to strings being immutable, has to create a new string for each `.ToLower()` call.

```csharp
var result = String.Compare("StringA", "StringB", StringComparison.OrdinalIgnoreCase);
```

Note: It's recommended to use the overload of [`Compare(String, String, StringComparison)`](https://learn.microsoft.com/en-us/dotnet/api/system.string.compare?view=netcore-3.1#system-string-compare(system-string-system-string-system-stringcomparison)) rather than `Compare(String, String)` as via the documentation: 

> When comparing strings, you should call the Compare(String, String, StringComparison) method, which requires that you explicitly specify the type of string comparison that the method uses. For more information, see Best Practices for Using Strings.

### 🟢 Use `StringBuilder` for larger strings

[.NET Performance Tips - Strings](https://learn.microsoft.com/en-us/dotnet/framework/performance/performance-tips#strings)

Strings are immutable. When you concatenate many strings together each concatenation creates a new string that takes a relatively long amount of time. 

```csharp
string[] words = []; // imagine a large array of strings

// inefficient
var slowWords = "";

foreach(var word in words)
{
	slowWords += word;
}

Console.WriteLine(slowWords);

// more efficient
var stringBuilder = new StringBuilder();

foreach(var word in words)
{
	stringBuilder.AppendLine(word);
}

Console.WriteLine(stringBuilder.ToString());

```

### 🟢 Don't use async with large SqlClient data

There is a long time open [dotnet GitHub issue](https://github.com/dotnet/SqlClient/issues/593) where larger returned query data takes a lot longer with async compared to sync.

### 🟢 Use streams with `HttpClient`

[Performance Improvements in .NET 8 by Stephen Toub](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/)

[HttpClient benchmarking via Niko Uusitalo](https://github.com/nikouu/HttpClientBenchmarking)

If we work with streams out of `HttpClient` we can eliminate a lot of memory copies. 

The following example assumes the response is a JSON string:

```csharp
public async Task GetStreamAsync()
{
    using var stream = await _httpClient.GetStreamAsync(_url);

    var data = await JsonSerializer.DeserializeAsync<List<WeatherForecast>>(stream);
}
```

In fact, this is such a common operation that we have a stream convenience overload for JSON:

```csharp
public async Task GetFromJsonAsync()
{
    var data = await _httpClient.GetFromJsonAsync<List<WeatherForecast>>(_url);
}
```

These two examples above eliminate copying the response from the stream to a JSON string then the JSON string into our objects. 

### 🟡 Async

[Official Guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)

Asynchronous programming allows us to continue with other tasks while waiting for I/O bound calls or expensive computations. In .NET we use the `async` and `await` keywords. There's too much to cover for this documentation, but once you get used to it, it's easy enough to use daily.

The user needs to ensure they put the `await` keyword in for an async method call.

```csharp
public async Task<string> GetFileContentsAsync(string path)
{
	return await File.ReadAllTextAsync(path);
}
```

### 🟡 `Parallel.ForEach()`

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreach)

[Official Guide](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop)

`Parallel.ForEach()` works like executing a `foreach` loop in parallel. Best used if you have lots of independent CPU bound work.

Be wary of threading and concurrency issues.

There is also [`Parallel.ForEachAsync()`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreachasync).

```csharp
List<Image> images = GetImages();

Parallel.ForEach(images, (image) => {
	var resizedImage = ResizeImage(image);
	SaveImage(image);
});
```

### 🟡 `Span<T>`, `ReadOnlySpan<T>`, `Memory<T>`

[`Span<T>` Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.span-1)

[`ReadOnlySpan<T>` Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.readonlyspan-1)

[`Memory<T>` Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.memory-1)

[`Memory<T>` and `Span<T>` usage guidelines](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)

[Turbocharged: Writing High-Performance C# and .NET Code - Steve Gordon](https://www.youtube.com/watch?v=CwISe8blq38)

All three of these "represent an arbitrary contiguous region of memory". Another way it's commonly explained is these can be thought of as a window into the memory. 

For our short purposes here, the performance gains are via low level reading and manipulation of data without copying or allocating. We use them with arrays, strings, and more niche data situations.

There are caveats such as `Span` types only being put on the stack, meaning they can't be used inside async calls (which is what `Memory<T>` allows for). 

```csharp
int[] items = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

// Technically we can use the slicing overload of AsSpan() as well
var span = items.AsSpan();
var slice = span.Slice(1, 5); // 1, 2, 3, 4, 5
```

### 🟡 Faster loops with `Span<T>` and `ReadOnlySpan<T>`

[The weirdest way to loop in C# is also the fastest - Nick Chapsas](https://www.youtube.com/watch?v=cwBrWn4m9y8)

[Improve C# code performance with Span<T> via NDepend blog](https://blog.ndepend.com/improve-c-code-performance-with-spant/)

Don't mutate the collection during looping.

For arrays and strings, we can use `AsSpan()`.

```csharp
int[] items = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
var span = items.AsSpan();

for (int i = 0; i < span.Length; i++)
{
    Console.WriteLine(span[i]);
}
```

However for lists we need to use [`CollectionsMarshal.AsSpan()`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.collectionsmarshal.asspan)

```csharp
List<int> items = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
var span = CollectionsMarshal.AsSpan(items);

for (int i = 0; i < span.Length; i++)
{
    Console.WriteLine(span[i]);
}
```	

If a `Span<T>` isn't viable, you can create your own enumerator with `ref` and `readonly` keywords. Information can be found at [Unusual optimizations; ref foreach and ref returns by Marc Gravell](https://blog.marcgravell.com/2022/05/unusual-optimizations-ref-foreach-and.html).

### 🔴 `SkipLocalsInit`

[C# 9 - Improving performance using the SkipLocalsInit attribute by Gérald Barré](https://www.meziantou.net/csharp-9-improve-performance-using-skiplocalsinit.htm)

[Feature spec](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/skip-localsinit)

[Feature design for SkipLocalsInit](https://github.com/dotnet/roslyn/blob/main/docs/features/skiplocalsinit.md)

By default the CLR forces the JIT to set all local variabls to their default value meaning your variable won't be some leftover value from memory. In high performance situations, this may become noticeable and we can skip this initialization as long as we understand the risk being taken on.

```csharp
[SkipLocalsInit]
byte SkipInitLocals()
{
    Span<byte> s = stackalloc byte[10];
    return s[0];
}
```