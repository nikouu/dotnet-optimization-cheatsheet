# Dotnet Optimization Cheatsheet

[Donald Knuth](https://en.wikipedia.org/wiki/Donald_Knuth):

>Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

If you're the 3%, this is an educational reference on .NET performance tricks I've gathered. By reading this I assume you understand that architecture, patterns, data flow, general performance decisions, hardware, etc have a far larger impact to your codebase than optimising for potentially single digit milliseconds, nanoseconds, or memory bytes with niche and potentially dangerous strategies. For instance, you'll get far more out of caching database values than you'll ever get out of skipping local variable initialization for an object - by several orders of magnitude. I also expect that you already understand best practices e.g. properly dealing with `IDisposable` objects.

You'll need to consider:
1. Will I be pre-maturely optimising?
1. Do I understand the risks involved?
1. Is the time spent doing this worth it?

I've rated each optimisation here in terms of difficulty and you may gauge the difficulties differently. Remember, we're looking at the last 3%, so while some are easy to implement, they may only be measurably effective on an extremely hot path.

| Difficulty       | Reasoning                                                                                                                                                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🟢 Easy         | These are either well known, or simple to drop into existing code with only some knowledge.                                                                                                                                               |
| 🟡 Intermediate | Mostly accessible. May require a bit more coding or understanding of specific caveats of use. Own research is advised. |
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

| Resource                                                                                                                                                                      | Description                                                                                                                                                                                              |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Pro .NET Memory Management](https://prodotnetmemory.com/)                                                                                                                    | A heavy and incredibly informative read on everything memory and memory-performance in .NET. A lot of knowledge below originally was seeded by this book and should be treated as an implicit reference. |
| [PerformanceTricksAzureSDK](https://github.com/danielmarbach/PerformanceTricksAzureSDK)                                                                                       | Tricks written by [Daniel Marbach](https://github.com/danielmarbach)                                                                                                                                     |
| [.NET Memory Performance Analysis](https://github.com/Maoni0/mem-doc/blob/master/doc/.NETMemoryPerformanceAnalysis.md)                                                        | A thorough document on analyzing .NET performance by [Maoni Stephens](https://github.com/Maoni0)                                                                                                         |
| [Classes vs. Structs. How not to teach about performance!](https://sergeyteplyakov.github.io/Blog/benchmarking/2023/11/02/Performance_Comparison_For_Classes_vs_Structs.html) | A critique of bad benchmarking by [Sergiy Teplyakov](https://sergeyteplyakov.github.io/Blog/)                                                                                                            |
| [Diagrams of .NET internals](https://goodies.dotnetos.org/)                                                                                                                   | In depth diagrams from the [Dotnetos courses](https://dotnetos.org/products/)                                                                                                                            |
| [Turbocharged: Writing High-Performance C# and .NET Code](https://www.youtube.com/watch?v=CwISe8blq38)                                                                        | A great video describing multiple ways to improve performance by [Steve Gordon](https://www.stevejgordon.co.uk/)                                                                                         |
| [Performance tricks I learned from contributing to open source .NET packages](https://www.youtube.com/watch?v=pGgsFW7kDKI)                                                    | A great video describing multiple ways to sqeeze out performance by [Daniel Marbach](https://github.com/danielmarbach)                                                                                   |

## Optimisations

[_We always want our code to "run faster". But rarely do we ask – what is it running from?_](https://twitter.com/moyix/status/1449397749343047681)

### 🟢 Remove unncessary boxing/unboxing

[.NET Performance Tips - Boxing and Unboxing](https://learn.microsoft.com/en-us/dotnet/framework/performance/performance-tips#boxing-and-unboxing)

[Boxing and Unboxing (C# Programming Guide)](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)

Boxing is converting an instance of a value type to an instance of a reference type. And unboxing the reverse. In most cases ([but not all](https://twitter.com/konradkokosa/status/1432622993390424064)), we'll be referring to putting a value type on the heap.

This slows down performance because the runtime needs to do a two step operation:
1. allocate the new boxed object on the heap which now has an object header and method table reference
1. the copy to the heap

This is two fold as it also eventually increases garbage collection pressure.

We can recognise this as the emitted `box`/`unbox` operations in the IL code, or by using tools such as the [Clr Heap Allocation Analyzer](https://marketplace.visualstudio.com/items?itemName=MukulSabharwal.ClrHeapAllocationAnalyzer).

```
No example
```

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

### 🟢 Use `String.Compare()` or `String.Equals()`

[Twitter post via Daniel Lawson](https://twitter.com/danylaws/status/1504381170347294727)

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.string.compare)

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

### 🟢 `string.Create()`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.string.create)

[Post via Dave Callan](https://x.com/Dave_DotNet/status/1682699744747958274)

If you know the length of your final string, perhaps you're concatenating various strings to one, you can use `string.Create()` to speed up the final string creation with minimal allocations.

It looks a complex, but this method allows us to manipulate the string as if it were mutable inside the lambda via a `Span<T>` and guarantees us an immutable string afterwards. 

Example via [Stephen Toub](https://youtu.be/5KdICNWOfEQ?t=2669)
```csharp
string s = string.Create(34, Guid.NewGuid(), (span, guid) => 
{
	"ID".AsSpan().CopyTo(span);
	guid.TryFormat(span.Slice(2), out _, "N");
});

Console.WriteLine(s); // ID3d1822eb1310418cbadeafbe3e7b7b9f
```

### 🟢 Don't use async with large SqlClient data

There is a long time open [dotnet GitHub issue](https://github.com/dotnet/SqlClient/issues/593) where larger returned query data takes a lot longer with async compared to sync.

```
No example
```

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

### 🟢 EF Core compiled queries

[Official Article](https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/ef/language-reference/compiled-queries-linq-to-entities)

There is also a `CompileAsyncQuery` available too.

```csharp
// EF Core compiled query
private static readonly Func<MyDbContext, IEnumerable<User>> _getUsersQuery =
	EF.CompileQuery((MyDbContext context) => context.Users.ToList());

public IEnumerable<User> GetUsers()
{
	return _getUsersQuery(_context);
}
```

### 🟢 Keep "exceptions, exceptional"

[Via Kevlin Henney](https://x.com/KevlinHenney/status/1566350355188912129)

[Some numbers via Peter Morris](https://x.com/MrPeterLMorris/status/1796136008326500756)

In distributed systems especially, networking issues become business as usual, not exceptional circumstances. As an example, instead of [`HttpResponseMessage.EnsureSuccessStatusCode()`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpresponsemessage.ensuresuccessstatuscode) you may want to consider [`HttpResponseMessage.IsSuccessStatusCode`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpresponsemessage.issuccessstatuscode) then deal with it accordingly sans exception.

```
No example
```

### 🟡 Async

[Official Article](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)

Asynchronous programming allows us to continue with other tasks while waiting for I/O bound calls or expensive computations. In .NET we use the `async` and `await` keywords. There's too much to cover for this documentation, but once you get used to it, it's easy enough to use daily.

The user needs to ensure they put the `await` keyword in for an async method call.

```csharp
public async Task<string> GetFileContentsAsync(string path)
{
	return await File.ReadAllTextAsync(path);
}
```

### 🟡 `ValueTask`

[Official DevBlog by Stephen Toub](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)

[Official Reference for `ManualResetValueTaskSourceCore`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.sources.manualresetvaluetasksourcecore-1)

`Task`s are very flexible to suit the varied needs asked of them. However, a lot of use cases of `Task` are simple single awaits and it may also be the case that during high throughput code there could be a performance hit of the allocations from `Task`. To solve these problems we have `ValueTask` which allocates less than `Task`. However there are caveats with the less flexibility compared to `Task` such as you shouldn't await a `ValueTask` more than once. 

```csharp
var result = await GetValue();


public async ValueTask<int> GetValue()
{
	// implementation omitted
	return value;
}
```

### 🟡 Structs

[Official Article](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct)

[Class vs Struct in C#: Making Informed Choices via NDepend blog](https://blog.ndepend.com/class-vs-struct-in-c-making-informed-choices/)

[Value vs Reference type allocation locations via Konrad Kokosa](https://twitter.com/konradkokosa/status/1432622993390424064)

There are times when structs may help us with performance compared to a class. Structs are lighter than classes and _often_ end up on the stack meaning no allocations for the GC to deal with. It's better to use them in place of simple objects.

However, there are a few caveats here including the size and complexity of the struct, nullability, and no inheritance.

```csharp
var coordinates = new Coords(0, 0);

public struct Coords
{
    public Coords(double x, double y)
    {
        X = x;
        Y = y;
    }

    public double X { get; }
    public double Y { get; }

    public override string ToString() => $"({X}, {Y})";
}
```

### 🟡 `Parallel.ForEach()`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreach)

[Official Article](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop)

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

[`Span<T>` Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.span-1)

[`ReadOnlySpan<T>` Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.readonlyspan-1)

[`Memory<T>` Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.memory-1)

[`Memory<T>` and `Span<T>` usage guidelines](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)

[A Complete .NET Developer's Guide to Span with Stephen Toub](https://www.youtube.com/watch?v=5KdICNWOfEQ)

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

### 🟡 `ArrayPool<T>`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)

[How to use ArrayPool and MemoryPool in C# by Joydip Kanjilal](https://www.infoworld.com/article/3596289/how-to-use-arraypool-and-memorypool-in-c.html)

Allows us to rent an array of type `T` that has equal to or greater size than what we ask for. This helps us reduce allocations on hot paths as instead of creating and destroying array instances, we reuse them. 

Note: you can also use the more flexible [`MemoryPool`](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.memorypool-1), however there are [further caveats](https://endjin.com/blog/2020/09/arraypool-vs-memorypool-minimizing-allocations-ais-dotnet) to that as it allocates to understand the owner of the item - which `ArrayPool` avoids.

```csharp
var buffer = ArrayPool<int>.Shared.Rent(5);

for (var i = 0; i < 5; i++)
{
    buffer[i] = i * 2;
}

for (var i = 0; i < 5; i++)
{
    Console.WriteLine(buffer[i]);
}

ArrayPool<int>.Shared.Return(buffer);
```

### 🟡 `StringPool`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/microsoft.toolkit.highperformance.buffers.stringpool)

[Official Article](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/high-performance/stringpool)

This can be thought of as configurable/managed [string interning](https://learn.microsoft.com/en-us/dotnet/api/system.string.intern) with useful methods. Brought to us by the [.NET Community Toolkit](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/introduction).

```csharp
public string GetString(ReadOnlySpan<char> readOnlySpan)
{
	return StringPool.Shared.GetOrAdd(readOnlySpan);
}
```

### 🟡 `ObjectPool`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.objectpool.objectpool-1?view=dotnet-plat-ext-7.0)

[Official Article](https://learn.microsoft.com/en-us/aspnet/core/performance/objectpool?view=aspnetcore-8.0)

A general use pool. Has varying ways to setup.

```csharp
var objectPool = ObjectPool.Create<HeavyObject>();

var pooledObject = objectPool.Get();

// then some work

objectPool.Return(pooledObject);
```

### 🟡 `RecyclableMemoryStream`

[Official GitHub Repo](https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream)

This is designed to be a near-drop in replacement for `MemoryStream` objects, with configuration and metrics.

To use an example from the link above:

```csharp
private static readonly RecyclableMemoryStreamManager manager = new RecyclableMemoryStreamManager();

static void Main(string[] args)
{
	var sourceBuffer = new byte[] { 0, 1, 2, 3, 4, 5, 6, 7 };
	
	using (var stream = manager.GetStream())
	{
		stream.Write(sourceBuffer, 0, sourceBuffer.Length);
	}
}
```

### 🟡 Server Garbage Collection

[Garbage Collection Article](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/)

[Workstation GC Official Article](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc)

[Runtime configuration options for garbage collection](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/garbage-collector)

[.NET Memory Performance Analysis section with server GC](https://github.com/Maoni0/mem-doc/blob/master/doc/.NETMemoryPerformanceAnalysis.md#server-gc)  

[Twitter post with graphs from Sergiy Teplyakov](https://x.com/STeplyakov/status/1729204671133413815)

[DevBlog post from Sergiy Teplyakov](https://devblogs.microsoft.com/premier-developer/understanding-different-gc-modes-with-concurrency-visualizer/)

[Blog post about .NET 8 GC feature: DATAS by Maoni Stephens](https://maoni0.medium.com/dynamically-adapting-to-application-sizes-2d72fcb6f1ea)

Having concurrent threads help with garbage collection can minimise the GC pause time in an application. However there are caveats to do with the amount of logical processors and how many applications are running on the machine at once.

```
No example
```

### 🟡 Ahead of Time Compilation (AOT)

[Native AOT Deployment Official Guide](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

[.NET AOT Resources: 2022 Edition (Self plug)](https://www.nikouusitalo.com/blog/net-aot-resources-2022-edition/) 

AOT allows us to compile our code natively for the target running environment - removing the JITer. This allows us to have a smaller deployment footprint and a faster startup time.

Caveats here is that reflection (used in a lot of .NET) is extremely limited, we have to compile for each target environment, no COM, and more. We can get around some of these limitations with [source generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview).

```
No example
```

### 🔴 `stackalloc`

[Official Article](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc)

[High performance byte/char manipulation via David Fowler](https://x.com/davidfowl/status/1520966312817664000)

[My favourite examples](https://github.com/nikouu/Neat-CSharp-Snippets?tab=readme-ov-file#stackalloc-patterns)

[Dos and Don'ts of stackalloc via Kevin Jones](https://vcsjones.dev/stackalloc/)

`stackalloc` gives us a block of memory on the stack. We can easily use it with `Span<T>` and `ReadOnlySpan<T>` to work as small buffers. Highly performance stack-only work, no heap, no garbage collector.

A caveat here is it must be small to prevent stack overflows. I've seen developers at Microsoft use a [max size of 256 bytes](https://grep.app/search?q=stackalloc&filter[lang][0]=C%23&filter[repo][0]=dotnet/runtime).

Example from David Fowler, linked above.
```csharp
// Choose a small stack threshold to avoid stack overflow
const int stackAllocThreshold = 256;

byte[]? pooled = null;

// Either use stack memory or pooled memory
Span<byte> span = bytesNeeded <= stackAllocThreshold
    ? stackalloc byte[stackAllocThreshold]
    : (pooled = ArrayPool<byte>.Shared.Rent(bytesNeeded);

// Write to the span and process it
var written = Populate(span);
Process(span[..written]);

// Return the pooled memory if we used it
if (pooled is not null)
{
    ArrayPool<byte>.Shared.Return(pooled);
}
```

### 🔴 `MemoryMarshal`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.memorymarshal)

More easily allows us to interoperate with `Memory<T>`, `ReadOnlyMemory<T>`, `Span<T>`, and `ReadOnlySpan<T>`. A lot of performant `Unsafe` (class) code is abstracted away from us when using `MemoryMarshal`. 

There are little safeties granted to us in `MemoryMarshal` when compared to raw `Unsafe` calls, so the user must keep in mind that while they don't see `Unsafe` they are still at risk of the same type safety risks.

Note: This advice roughly applies to all within the [`System.Runtime.InteropServices`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices) namespace, including `MemoryMarshal`.

The following adapted from [Immo Landwerth](https://x.com/terrajobst/status/1507808952146223106):
```csharp
ReadOnlySpan<byte> bytes = MemoryMarshal.AsBytes(input.String.AsSpan());
return HashData(bytes);

Guid HashData(ReadOnlySpan<byte> bytes)
{
	var hashBytes = (Span<byte>)stackalloc byte[20];
	var written = SHA1.HashData(bytes, hashBytes);

	return new Guid(hashBytes[..16]);
}
```
### 🔴 `CollectionsMarshal`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.collectionsmarshal)

`CollectionsMarshal` gives us handles to the underlying data representations of collections. It makes use of `MemoryMarshal` and `Unsafe` under the hood and as such any lack of safety that is found there extends to this, such as type safety risks.

Assuming the developer keeps within safety boundaries, this allows us to make highly performant access to the collection.

The following is an absolutely key caveat:
>Items should not be added or removed while the span/ref is in use.

While it means more than this, an easy point to remember is: don't add/remove from the collection while using the returned object.

```csharp
var items = new List<int> { 1, 2, 3, 4, 5 };
var span = CollectionsMarshal.AsSpan(items);

for (int i = 0; i < span.Length; i++)
{
    Console.WriteLine(span[i]);
}
```

### 🔴 `SkipLocalsInit`

[C# 9 - Improving performance using the SkipLocalsInit attribute by Gérald Barré](https://www.meziantou.net/csharp-9-improve-performance-using-skiplocalsinit.htm)

[Feature spec](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/skip-localsinit)

[Feature design for SkipLocalsInit](https://github.com/dotnet/roslyn/blob/main/docs/features/skiplocalsinit.md)

By default the CLR forces the JIT to set all local variables to their default value meaning your variable won't be some leftover value from memory. In high performance situations, this may become noticeable and we can skip this initialization as long as we understand the risk being taken on. Also see [`Unsafe.SkipInit<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.skipinit).

```csharp
[SkipLocalsInit]
byte SkipInitLocals()
{
    Span<byte> s = stackalloc byte[10];
    return s[0];
}
```

### 🔴 `unsafe` (keyword)

[Official Article](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/unsafe)

[Unsafe code, pointer types, and function pointers](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code)

The `unsafe` keyword allows us to work with pointers in C# to write unverifyable code. We can allocate memory without safety from the garbage collector, use pointers, and call methods with function pointers.

However, we are on our own. No GC collection, no security guarantee, no type safety, and other caveats. 

Example via documentation:
```csharp
class UnsafeTest
{
    // Unsafe method: takes pointer to int.
    unsafe static void SquarePtrParam(int* p)
    {
        *p *= *p;
    }

    unsafe static void Main()
    {
        int i = 5;
        // Unsafe method: uses address-of operator (&).
        SquarePtrParam(&i);
        Console.WriteLine(i);
    }
}
```

### 🔴 `Unsafe` (class)

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe)

Safer than the `unsafe` keyword, the `Unsafe` class allows us to do lower level manipulation for performant code by supressing type safety while still being tracked by the garbage collector. There are caveats, especially around type safety.

```csharp
var items = new Span<int>([ 0, 1, 2, 3, 4 ]);
ref var spanRef = ref MemoryMarshal.GetReference(items);

var item = Unsafe.Add(ref spanRef, 2);

Console.WriteLine(item); // prints 2
```

### 🔴 Ref fields (with `UnscopedRef`)

[Twitter post via Konrad Kokosa](https://twitter.com/konradkokosa/status/1729908112960667991)

[Gist via Konrad Kokosa](https://gist.github.com/kkokosa/b13b4ada81dc77d2f79eaf05b9f87d37)

[Low Level Struct Improvements proposal](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/low-level-struct-improvements#parameter-scope-variance)

[UnscopedRefAttribute documentation](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.unscopedrefattribute)

For more complex data structures, having your deep down property as a ref could improve speed.

```csharp
public struct D
{
    public int Field;

    [UnscopedRef]
    public ref int ByRefField => ref Field;
}
```

### 🔴 Overriding `GetHashCode()` and `Equals` for Structs

[Official DevBlog](https://devblogs.microsoft.com/premier-developer/performance-implications-of-default-struct-equality-in-c/)

[Jon Skeet SO answer](https://stackoverflow.com/a/263416)

[Struct equality performance in .NET - Gérald Barré](https://www.meziantou.net/struct-equality-performance-in-dotnet.htm)

The default value type implementation of `GetHashCode()` trades speed for a good hash distribution across any given value type - useful for dictionary and hashset types. This happens by reflection, which is incredibly slow when looking through our micro-optimization lens.

An example from the Jon Skeet answer above of overriding:
```csharp
public override int GetHashCode()
{
    unchecked // Overflow is fine, just wrap
    {
        int hash = (int) 2166136261;
        // Suitable nullity checks etc, of course :)
        hash = (hash * 16777619) ^ field1.GetHashCode();
        hash = (hash * 16777619) ^ field2.GetHashCode();
        hash = (hash * 16777619) ^ field3.GetHashCode();
        return hash;
    }
}
```

### 🔴 Even Faster Loops

[The weirdest way to loop in C# is also the fastest - Nick Chapsas](https://www.youtube.com/watch?v=cwBrWn4m9y8)

[MemoryMarshal.GetReference Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.memorymarshal.getreference?view=net-8.0)

[Unsafe.Add Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.add?view=net-8.0)

Using both `MemoryMarshal`, `CollectionsMarshal`, and `Unsafe` we're able to loop directly on the underlying array inside a `List<T>` and index quickly to the next element.

Do not add or remove from the collection during looping.

```csharp
var items = new List<int> { 1, 2, 3, 4, 5 };
var span = CollectionsMarshal.AsSpan(items);
ref var searchSpace = ref MemoryMarshal.GetReference(span);

for (int i = 0; i < span.Length; i++)
{
    var item = Unsafe.Add(ref searchSpace, i);
    Console.WriteLine(item);
}
```

### 🔴 Fastest Loops

[I Lied! The Fastest C# Loop Is Even Weirder - Nick Chapsas](https://www.youtube.com/watch?v=KLtMtxUihBk)

With this we've mostly removed the safety .NET provides us in exchange for speed. It is also difficult to understand at a glance without being familiar with `MemoryMarshal` methods.

Do not add or remove from the collection during looping.

```csharp
int[] items = [1, 2, 3, 4, 5];

ref var start = ref MemoryMarshal.GetArrayDataReference(items);
ref var end = ref Unsafe.Add(ref start, items.Length);

while (Unsafe.IsAddressLessThan(ref start, ref end))
{
    Console.WriteLine(start);
    start = ref Unsafe.Add(ref start, 1);
}
```

### 🔴 Removing Closures

[How we achieved 5X faster pipeline execution by removing closure allocations by Daniel Marbach](https://particular.net/blog/pipeline-and-closure-allocations)

[Dissecting the local functions in C# 7 by Sergey Tepliakov](https://devblogs.microsoft.com/premier-developer/dissecting-the-local-functions-in-c-7/)

[StackOverflow question on closures](https://stackoverflow.com/questions/595482/what-are-closures-in-c)

In-line delegates and anonymous methods that capture parental variables give us closure allocations. A new object is allocated to hold the value from the parent for the anonymous method.

We have a few options including, but not limited to:
1. If using LINQ, moving that code into a regular loop instead
1. Static lambdas ([lambdas are anonymous functions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions)) that accept state
	1. Further helpful if the static lambda is generic to removing boxing

Example via the above link by Daniel Marbach:
```csharp
static void MyFunction<T>(Action<T> action, T state) => action(state);

int myNumber = 42;
MyFunction(static number => Console.WriteLine(number), myNumber);
```

### 🔴 `ThreadPool`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool)

[Offical Guide](https://learn.microsoft.com/en-us/dotnet/standard/threading/the-managed-thread-pool)

If you're going to use low level threads, you'll probably be using `ThreadPool` as it manages the threads for us. As .NET has moved forwards, a lot of raw thread or thread pool usage has been superseded (in my experience) by PLINQ, Parallel foreach calls, [`Task.Factory.StartNew()`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskfactory.startnew), [`Task.Run()`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.run) etc - I.E. further abstraction over threads.

The following is via the documentation above:
```csharp
ThreadPool.QueueUserWorkItem(ThreadProc);
Console.WriteLine("Main thread does some work, then sleeps.");
Thread.Sleep(1000);

Console.WriteLine("Main thread exits.");

static void ThreadProc(Object stateInfo) 
{
    // No state object was passed to QueueUserWorkItem, so stateInfo is null.
    Console.WriteLine("Hello from the thread pool.");
}
``` 

### 🔴 Vectorizing

[Use SIMD-accelerated numeric types documentation](https://learn.microsoft.com/en-us/dotnet/standard/simd)

[SIMD-accelerated numeric types](https://learn.microsoft.com/en-us/dotnet/standard/simd)

[Vectorization Guidelines](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/vectorization-guidelines.md)

[Great self-learning post by Alexandre Mutel](https://xoofx.github.io/blog/2023/07/09/10x-performance-with-simd-in-csharp-dotnet/)

Single Instruction, Multiple Data (SIMD) allows us to act on multiple values per iteration rather than just a single value via vectorization. As .NET has moved forward, it's been made easier to take advantage of this feature.

Vectorizing adds complexity to your codebase, but thankfully under the hood, common .NET methods have been written using vectorization and as such we get the benefit for free. E.g. [string.IndexOf() for OrdinalIgnoreCase](https://github.com/dotnet/runtime/pull/67758).

The following is a simple example:
```csharp
// Initialize two arrays for the operation
int[] array1 = [1, 2, 3, 4, 5, 6, 7, 8];
int[] array2 = [8, 7, 6, 5, 4, 3, 2, 1];
int[] result = new int[array1.Length];

// Create vectors from the arrays
var vector1 = new Vector<int>(array1);
var vector2 = new Vector<int>(array2);

// Perform the addition
var resultVector = Vector.Add(vector1, vector2);

// Copy the results back into the result array
resultVector.CopyTo(result);

// Print the results
Console.WriteLine(string.Join(", ", result));  // Outputs: 9, 9, 9, 9, 9, 9, 9, 9
```

### 🔴 Inline Arrays

[C# 12 release notes](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#inline-arrays)

[Inline array language proposal document](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/inline-arrays)

[Fun example via David Fowler](https://x.com/davidfowl/status/1678190691841716224)

The documentation states: 
>You likely won't declare your own inline arrays, but you use them transparently when they're exposed as `System.Span<T>` or `System.ReadOnlySpan<T>` objects from runtime APIs

Inline arrays are more a runtime feature for the .NET development team at Microsoft as it allows them to give us featurs such as `Span<T>` and interaction with unmanaged types. They're essentially fixed-sized stack allocated buffers.

Example from documentation:

```csharp
[InlineArray(10)]
public struct Buffer
{
    private int _element0;
}

var buffer = new Buffer();
for (int i = 0; i < 10; i++)
{
    buffer[i] = i;
}

foreach (var i in buffer)
{
    Console.WriteLine(i);
}
```

### 🔴 `SuppressGCTransition`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.suppressgctransitionattribute)

[Great writeup by Kevin Gosse](https://minidump.net/suppressgctransition-b9a8a774edbd/)

This attribute prevents the thread transition from cooperative GC move to preemptive GC mode when applied to a `DllImport`. It shaves only nanoseconds and has many caveats for usage. 

Example via Kevin Gosse:
```csharp
public int PInvoke_With_SuppressGCTransition()
{
	[DllImport("NativeLib.dll")]
	[SuppressGCTransition]
	static extern int Increment(int value);
}
```

### 🔴 `GC.TryStartNoGCRegion()`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.gc.trystartnogcregion)

[Preventing .NET Garbage Collections with the TryStartNoGCRegion API by Matt Warren](https://mattwarren.org/2016/08/16/Preventing-dotNET-Garbage-Collections-with-the-TryStartNoGCRegion-API/)

Used for absolutely critical hotpaths, `TryStartNoGCRegion()` will attempt to disallow the garbage collection until the corresponding `EndNoGCRegion()` call. There are many caveats here that may throw exceptions, and lead to accidental misuse.

```csharp
if (GC.TryStartNoGCRegion(10000))
{
	Console.WriteLine("No GC Region started successfully.");

	int[] values = new int[10000];

	// do work

	GC.EndNoGCRegion();
}
else
{
	Console.WriteLine("No GC Region failed to start.");
}
```

### 🔴 Alignment

[DevBlog Post](https://devblogs.microsoft.com/dotnet/loop-alignment-in-net-6/)

[Example via Egor Bogatov](https://x.com/EgorBo/status/1646922981744992261)

Alignment is around the CPU caches and we can spend extra effort in making sure our data is fitted to the CPU cache size. However, the CLR will put in a lot of effort to align this for us by adding padding if needed.

However there may be times where we may have some influence such as with setting the `StructLayout` attribute on a `struct` or how we control our nested loops accesses.

```
No example
```

### 🔴 `[MethodImpl(MethodImplOptions.AggressiveInlining)]`

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.methodimploptions)

While there are many in the `MethodImplOptions` enum, I'd wager the most common is `MethodImplOptions.AggressiveInlining` which takes a piece of code and inserts it where the caller wants it as opposed to making a call out to a separate location, if possible.

It's more appropriate to use this for small functions that are very hot. Caveats here is that it can increase the size of your application and could make it slower overall.

The following example is from [System.Math](https://github.com/dotnet/runtime/blob/5535e31a712343a63f5d7d796cd874e563e5ac14/src/libraries/System.Private.CoreLib/src/System/Math.cs#L457C1-L475C10)
```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static byte Clamp(byte value, byte min, byte max)
{
	if (min > max)
	{
		ThrowMinMaxException(min, max);
	}

	if (value < min)
	{
		return min;
	}
	else if (value > max)
	{
		return max;
	}

	return value;
}
```

### 🔴 Improving `ReadOnlySequence<T>` performance

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.readonlysequence-1?view=net-8.0)

[Twitter post via @neuecc](https://x.com/neuecc/status/1629725919735840770)

[Cysharp/MemoryPack benchmark test](https://github.com/Cysharp/MemoryPack/blob/753a5557e9415201c01dae34b1a551b95ef1a150/sandbox/Benchmark/Micro/SpanSliceTest.cs)

The `Slice()` operation on a `ReadOnlySequence<T>` is slow and can be worked around by wrapping the `ReadOnlySequence<T>` with another struct and copying it into a `Span<T>` and using the `Slice()` operation on span.

The following example is a simplified version from the MemoryPack benchmark tests.

```csharp
ref struct SpanWriter
{
    Span<byte> raw;

    public SpanWriter(byte[] buffer)
    {
        raw = buffer;
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Advance(int count)
    {
        raw = raw.Slice(count);
    }
}
```

## To be written

- Locks: `lock`, `SemaphmoreSlim`, `Interlocked`, and the new `System.Threading.Lock` lock.
    - [The Danger of Atomic Operations by Dmitry Vyukov et al](https://abseil.io/docs/cpp/atomic_danger) 
    - [`Lock` object](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/lock-object)
- Thread locals
- More `ref`
- Loop unrolling
- [CLR Configs](https://x.com/EgorBo/status/1794394757218615603?t=oUFEAbbzTxl7CD78vgr-GQ&s=19)
- `StringValue`
    - [A brief look at StringValues by Andrew Lock](https://andrewlock.net/a-brief-look-at-stringvalues/)
- `System.Text.Json.Utf8*`
    - [MS Blog](https://devblogs.microsoft.com/dotnet/the-convenience-of-system-text-json/)
- Having a class as `sealed`
    - [Performance Improvements in .NET 6 by Stephen Toub](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-6/)
- LINQ and how there may be times on very hot paths where it might be more useful to write your own for loop; with the caveat that LINQ has fantastic optimizations with SIMD under the hood which may be in play and a developer may be de-optimizing with a for loop.
