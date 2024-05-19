# Dotnet Optimization Cheatsheet

[Donald Knuth](https://en.wikipedia.org/wiki/Donald_Knuth):

>Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

If you're the 3%, this is an educational look at .NET performance tricks I've gathered over time. By reading this I assume you understand that architecture, patterns, data flow, general performance decisions, hardware, etc have a far larger impact to your codebase than optimising for potentially single digit milliseconds or nanoseconds. 

You'll need to consider:
1. Will I be pre-maturely optimising?
1. Understand the risks involved
1. Is the time spent doing this worth it?

I've rated each optimisation here in terms of difficulty. You may gauge the difficulties differently. Remember, we're looking at the last 3%, so while some are easy to implement, they may only be measurably effective on an extremely hot path.

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

| Resource                                                                                                                                                                      | Description                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [PerformanceTricksAzureSDK](https://github.com/danielmarbach/PerformanceTricksAzureSDK)                                                                                       | Tricks written by [Daniel Marbach](https://github.com/danielmarbach)                                                   |
| [.NET Memory Performance Analysis](https://github.com/Maoni0/mem-doc/blob/master/doc/.NETMemoryPerformanceAnalysis.md)                                                        | A thorough document on analyzing .NET performance by [Maoni Stephens](https://github.com/Maoni0)                       |
| [Classes vs. Structs. How not to teach about performance!](https://sergeyteplyakov.github.io/Blog/benchmarking/2023/11/02/Performance_Comparison_For_Classes_vs_Structs.html) | A critique of bad benchmarking by [Sergiy Teplyakov](https://sergeyteplyakov.github.io/Blog/)                          |
| [Diagrams of .NET internals](https://goodies.dotnetos.org/)                                                                                                                   | In depth diagrams from the [Dotnetos courses](https://dotnetos.org/products/)                                          |
| [Turbocharged: Writing High-Performance C# and .NET Code](https://www.youtube.com/watch?v=CwISe8blq38)                                                                        | A great video describing multiple ways to improve performance by [Steve Gordon](https://www.stevejgordon.co.uk/)       |
| [Performance tricks I learned from contributing to open source .NET packages](https://www.youtube.com/watch?v=pGgsFW7kDKI)                                                    | A great video describing multiple ways to sqeeze out performance by [Daniel Marbach](https://github.com/danielmarbach) |

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

### 🟡 `ValueTask`

[Official DevBlog by Stephen Toub](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)

[Official Documentation for ManualResetValueTaskSourceCore](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.sources.manualresetvaluetasksourcecore-1)

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

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct)

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

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)

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

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/high-performance/stringpool)

[Official Reference](https://learn.microsoft.com/en-us/dotnet/api/microsoft.toolkit.highperformance.buffers.stringpool)

This can be thought of as configurable/managed [string interning](https://learn.microsoft.com/en-us/dotnet/api/system.string.intern) with useful methods. Brought to us by the [.NET Community Toolkit](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/introduction).

```csharp
public string GetString(ReadOnlySpan<char> readOnlySpan)
{
	return StringPool.Shared.GetOrAdd(readOnlySpan);
}
```

### 🟡 `ObjectPool`

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.objectpool.objectpool-1?view=dotnet-plat-ext-7.0)

[Official Guide](https://learn.microsoft.com/en-us/aspnet/core/performance/objectpool?view=aspnetcore-8.0)

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

### 🔴 `SkipLocalsInit`

[C# 9 - Improving performance using the SkipLocalsInit attribute by Gérald Barré](https://www.meziantou.net/csharp-9-improve-performance-using-skiplocalsinit.htm)

[Feature spec](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/skip-localsinit)

[Feature design for SkipLocalsInit](https://github.com/dotnet/roslyn/blob/main/docs/features/skiplocalsinit.md)

By default the CLR forces the JIT to set all local variabls to their default value meaning your variable won't be some leftover value from memory. In high performance situations, this may become noticeable and we can skip this initialization as long as we understand the risk being taken on. Also see [`Unsafe.SkipInit<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.skipinit).

```csharp
[SkipLocalsInit]
byte SkipInitLocals()
{
    Span<byte> s = stackalloc byte[10];
    return s[0];
}
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

### 🔴 Fastest Loops

[I Lied! The Fastest C# Loop Is Even Weirder - Nick Chapsas](https://www.youtube.com/watch?v=KLtMtxUihBk)

With this we've mostly removed the safety .NET provides us in exchange for speed. It is also difficult to understand at a glance without being familiar with `MemoryMarshal` methods.

Don't mutate the collection during looping.

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

[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool)

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
[SIMD-accelerated numeric types](https://learn.microsoft.com/en-us/dotnet/standard/simd)

[Vectorization Guidelines](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/vectorization-guidelines.md)

[Great self-learning post by Alexandre Mutel](https://xoofx.github.io/blog/2023/07/09/10x-performance-with-simd-in-csharp-dotnet/)

Single Instruction, Multiple Data (SIMD) allows us to act on multiple values per iteration rather than just a single value via vectorization. As .NET has moved forward, it's been made easier to take advantage of vectors.

Vectorizing adds complexity to your codebase, but thankfully under the hood, common .NET methods have been written using vectorization and we get the benefit such as for [string.IndexOf() for OrdinalIgnoreCase](https://github.com/dotnet/runtime/pull/67758).


### 🔴 Inline Arrays

[C# 12 release notes](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#inline-arrays)

[Inline array language proposal document](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/inline-arrays)

[Fun example via David Fowler](https://x.com/davidfowl/status/1678190691841716224)

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
