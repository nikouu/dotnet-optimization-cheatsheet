# Dotnet Optimization Cheatsheet

# Still under construction 🏗️

Practical neato performance/speed/"hacks"/good practices for .NET I've collected over time. Like bypassing safeties in your car and ripping out the seats to make it go faster, some of these come with a ⚠**RISK**⚠. You are here to have a reference of what is possible, this means you should understand your own scenario and what the relevant risks are. 

Donald Knuth said:

>Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

So let's just call these *educational* for that 3%. Each item is fully referenced so you can educate yourself too. Oh and they're mostly about language features over things like what should be existing good coding practices and architecture.

You might also need to consider: if a microbenchmark for function A is 2ms faster than function B, but function B does no allocations, which to pick? 

This is different from my [Optimization-Tools-And-Notes](https://github.com/nikouu/Optimization-Tools-And-Notes) repo as that is mostly tools for performance, and this is mostly code to help understand and make performance better.

## ‼ Read This First ‼

> [_We always want our code to "run faster". But rarely do we ask – what is it running from?_](https://twitter.com/moyix/status/1449397749343047681)

1. Try not to pre-maturely optimise
1. Understand the risks involved
1. Is the time spent doing this worth it?

This readme is giving you the tools. It's up to you to understand how and where to use them. 

*Or don't, I'm not gonna tell you how to live your life.*

## Starting Optimizing

In the shortest way possible:

1. Write unit tests so you have a baseline for your functionality
1. Use something like [BenchmarkDotNet](https://benchmarkdotnet.org/) or the [Performance Profiler](https://learn.microsoft.com/en-us/visualstudio/profiling/?view=vs-2022) in Visual Studio to get a before performance baseline
1. Write your change
1. Run tests
1. Benchmark to see if it did improve what you expected

We're looking at the following:
1. Making items/second go up and/or
2. Allocations go down

By the way, the risk ratings are arbitrary 🤷‍♀️

Oh and later .NET versions might render some of these invalid. For instance string interpolation used to be a tad slow, but had [a big performance glow up in .NET 6](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/).

## 🟢 Least Risky 😇
Nothing too scary here. The documentation should provide you with all the knowledge you need.

### Use less value nullable types
[Nullable value types are boxed](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types#boxing-and-unboxing) meaning there is unboxing work and null checking work happening. 

```csharp
int? Slow(int? x, int? y) {
	int? sum = 0;
	for (int i = 0; i < 1000; i++){
		sum += x * y;
	}
	return sum;
}
```

vs

```csharp
int? Fast(int x, int y) {
	int? sum = 0;
	for (int i = 0; i < 1000; i++){
		sum += x * y;
	}
	return sum;
}
```

#### References
[Tweet via Bartosz Adamczewski with infographics and explainers](https://twitter.com/badamczewski01/status/1542799681717149697)

### Use String.Compare()
`String.Compare()` is a memory efficient way to compare strings. This is in respect to the memory inefficient method of comparing by doing `stringA.ToLower() == stringB.ToLower()` - which due to strings being immutable, has to create a new string for each `.ToLower()` call.

```csharp
var result = String.Compare("StringA", "StringB", StringComparison.OrdinalIgnoreCase);
```

Note: It's recommended to use the overload of [`Compare(String, String, StringComparison)`](https://learn.microsoft.com/en-us/dotnet/api/system.string.compare?view=netcore-3.1#system-string-compare(system-string-system-string-system-stringcomparison)) rather than `Compare(String, String)` as via the documentation: 

> When comparing strings, you should call the Compare(String, String, StringComparison) method, which requires that you explicitly specify the type of string comparison that the method uses. For more information, see Best Practices for Using Strings.

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.string.compare?view=netcore-3.1)

[Twitter post via Daniel Lawson](https://twitter.com/danylaws/status/1504381170347294727)

### String to GUID

Tries to make the conversion from string to GUID without allocation. However, according to the Twitter replies (see references below) there are more ways to improve this by:
- Moving off MD5
- Using UTF16
- Using `MemoryMarshal.AsBytes` instead of the UTF8 work

```csharp
Guid ComputeGuid(string stringToConvert)
{
    const int maxBytesOnStack = 256;

    var encoding = Encoding.UTF8;
    var maxByteCount = encoding.GetMaxByteCount(stringToConvert.Length);

    if (maxByteCount <= maxBytesOnStack)
    {
        var buffer = (Span<byte>)stackalloc byte[maxBytesOnStack];
        var written = encoding.GetBytes(stringToConvert, buffer);
        var utf8Bytes = buffer[..written];
        return HashData(utf8Bytes);
    }
    else
    {
        var utf8Bytes = encoding.GetBytes(stringToConvert);
        return HashData(utf8Bytes);
    }
}

Guid HashData(ReadOnlySpan<byte> bytes)
{
    var hashBytes = (Span<byte>)stackalloc byte[16];
    var written = MD5.HashData(bytes, hashBytes);

    return new Guid(hashBytes);
}
```

#### References
[Via Immo Landwerth (terrajobst)](https://twitter.com/terrajobst/status/1507808952146223106)

[GitHub repo it's used in](https://github.com/terrajobst/apisof.net/blob/31398940e1729982a7f5e56e0656beb55045c249/src/Terrajobst.UsageCrawling/ApiKey.cs#L11)

[My project with benchmarks against variations](https://github.com/nikouu/String-to-GUID-Dotnet-Benchmarking)

### Don't use async with large SqlClient data

#### References
[dotnet GitHub issue](https://github.com/dotnet/SqlClient/issues/593)


### Span<T>, Memory<T>

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.span-1?view=net-7.0)

[Memory<T> and Span<T> usage guidelines](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)

[Turbocharged: Writing High-Performance C# and .NET Code - Steve Gordon](https://www.youtube.com/watch?v=CwISe8blq38)

### Faster loops with Array and Span<T>

```csharp
var items = new int[] { 1, 2, 3, 4, 5 };
var span = items.AsSpan();

for (int i = 0; i < span.Length; i++)
{
    Console.WriteLine(span[i]);
}
```

Don't mutate the collection during looping.

### ThreadPool

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool?view=net-7.0)

### ArrayPool

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1?view=net-7.0)

### MemoryPool

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.memorypool-1?view=net-7.0)

### ObjectPool

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.objectpool.objectpool-1?view=dotnet-plat-ext-7.0)

### StringPool

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/microsoft.toolkit.highperformance.buffers.stringpool?view=win-comm-toolkit-dotnet-7.1)

### RecyclableMemoryStream

#### References
[Official GitHub Repo](https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream)

### Removing unnecessary boxing/unboxing

#### References
[.NET Performance Tips - Boxing and Unboxing](https://learn.microsoft.com/en-us/dotnet/framework/performance/performance-tips#boxing-and-unboxing)

### Use StringBuilder for larger strings

#### References
[.NET Performance Tips - Strings](https://learn.microsoft.com/en-us/dotnet/framework/performance/performance-tips#strings)

## 🟡 A Little Risky 🤔
All the usual .NET safeties are included, however you may need to have a deeper understanding of the topic to not run into trouble in order to successfully use them.

### Faster loops with List and Span<T>

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var span = CollectionsMarshal.AsSpan(numbers);

for (int i = 0; i < span.Length; i++)
{
    Console.WriteLine(span[i]);
}
```

If the collection is mutated during looping, no error will be thrown. Be careful.

If a `Span` isn't viable, you can create your own enumerator with `ref` and `readonly` keywords. Information can be found at [Unusual optimizations; ref foreach and ref returns](https://blog.marcgravell.com/2022/05/unusual-optimizations-ref-foreach-and.html) by Marc Gravell.

#### References

[The weirdest way to loop in C# is also the fastest - Nick Chapsas](https://www.youtube.com/watch?v=cwBrWn4m9y8)

[CollectionsMarshal.AsSpan<T> Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.collectionsmarshal.asspan?view=net-8.0)

### Parallel ForEach

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreachasync?view=net-7.0)

[Official Guide](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop)

### Async

#### References
[Official Guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)

### ValueTask

#### ManualResetValueTaskSourceCore

#### References
[Official DevBlog](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)

[Official Documentation for ManualResetValueTaskSourceCore](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.sources.manualresetvaluetasksourcecore-1?view=net-7.0)

### Garbage Collection

#### References
[Garbage collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/)

[Workstation and server garbage collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc)

### HttpClient

#### References

[Updated benchmarking comparisons using `HttpClient`](https://github.com/nikouu/HttpClientBenchmarking)

[Efficiently Streaming Large HTTP Responses With HttpClient - Tugberk Ugurlu](https://www.tugberkugurlu.com/archive/efficiently-streaming-large-http-responses-with-httpclient)

[Streaming with New .NET HttpClient and HttpCompletionOption.ResponseHeadersRead - Tugberk Ugurlu](https://www.tugberkugurlu.com/archive/streaming-with-newnet-httpclient-and-httpcompletionoption-responseheadersread)

[Efficient api calls with HttpClient and JSON.NET - John Thiriet](https://johnthiriet.com/efficient-api-calls/)

### Struct

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct)

### Overriding GetHashCode() and Equals for Structs

#### References
[Official DevBlog](https://devblogs.microsoft.com/premier-developer/performance-implications-of-default-struct-equality-in-c/)

[Jon Skeet SO answer](https://stackoverflow.com/a/263416)

[Struct equality performance in .NET - Gérald Barré](https://www.meziantou.net/struct-equality-performance-in-dotnet.htm)

### AOT

#### References
[Native AOT Deployment Official Guide](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

[.NET AOT Resources: 2022 Edition](https://www.nikouusitalo.com/blog/net-aot-resources-2022-edition/) (Self plug)

[.NET 7 Self-Contained, NativeAOT, and ReadyToRun Cheatsheet](https://www.nikouusitalo.com/blog/net-7-self-contained-nativeaot-and-readytorun-cheatsheet/) (Self plug)

### Vectorizing

#### References
[Use SIMD-accelerated numeric types](https://learn.microsoft.com/en-us/dotnet/standard/simd)
[dotnet/runtime Vectorization Guidelines](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/vectorization-guidelines.md)

### Loop Alignment

#### References
[DevBlog Post](https://devblogs.microsoft.com/dotnet/loop-alignment-in-net-6/)

### SkipLocalsInit

#### References

[Meziantou's Blog](https://www.meziantou.net/csharp-9-improve-performance-using-skiplocalsinit.htm)

### Inline Arrays

#### References

[David Fowler](https://twitter.com/davidfowl/status/1678190691841716224)

### High performance byte/char manipulation 1
[Via David Fowler](https://twitter.com/davidfowl/status/1520966312817664000)

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
Answer from [Miha Zupan](https://twitter.com/_MihaZupan/status/1521135981356896258) for why to use a flat 256 instead of the `bytesNeeded` for the stack allocation:
> If you use SkipLocalsInit (perf sensitive code like this should and does), a constant size stackalloc is more efficient. It also means you don't have to explicitly guard against passing a negative value to the stackalloc.

Answer to why 256 is used. You can also see [other places](https://grep.app/search?q=stackalloc&filter[lang][0]=C%23&filter[repo][0]=dotnet/runtime) it's used in .NET:
> It's conservative to avoid stack overflows.

### High performance byte/char manipulation 2
[A more in-depth version via David Fowler](https://twitter.com/davidfowl/status/1521008356864843777/photo/1)

```csharp
using var memory = bytesNeeded <= 256
    ? new StackOrPooledMemory(stackalloc byte[256])
    : new StackOrPooledMemory(bytesNeeded);

Span<byte> span = memory.Buffer;
var written = Populate(span);
Process(span[written..]);

ref struct StackOrPooledMemory
{
    private readonly Span<byte> _span;
    private byte[]? _arrayPooled;
    
    public StackOrPooledMemory(Span<byte> buffer)
    {
        _span = buffer;
        _arrayPooled = null;
    }

    public StackOrPooledMemory(int size)
    {
        var buffer = ArrayPool<byte>.Shared.Rent(size);
        _arrayPooled = buffer;
        _span = buffer;
    }

    public Span<byte> Buffer => _span;

    public void Dispose()
    {
        if (_arrayPooled is not null)
        {
            var pooled = _arrayPooled;
            _arrayPooled = null;
            ArrayPool<byte>.Shared.Return(pooled);
        }
    }
}
```

Answer from [David Fowler](https://twitter.com/davidfowl/status/1521011080373293056) as to why no try/finally:
> We often skip that in high performance code, if the buffer doesn't go back to the pool because of exceptions, it'll get GCed.
> 
And [another note](https://twitter.com/davidfowl/status/1521127114124058626):
> Try/finally prevents inlining

### Ref fields

For more complex data structures, having your deep down property as a ref could improve speed.

```csharp
public struct D
{
    public int Field;

    [UnscopedRef]
    public ref int ByRefField => ref Field;
}

```

#### Refereces
[Via Konrad Kokosa](https://twitter.com/konradkokosa/status/1729908112960667991)

[Gist via Konrad Kokosa](https://gist.github.com/kkokosa/b13b4ada81dc77d2f79eaf05b9f87d37)


## 🔴 Very Risky ☠
🔥 Prod might go up in a spectacular ball of flame 🔥

![](images/mad-max-fireball.gif)

☠ These also might be detrimental to the health of your codebase, colleagues, and mental fortitude. They could be antipatterns, malpractice, or just overall disgusting. If you think these are good ideas for your codebase, stop and think: *are they really?* Because they're probably not. ☠

### Even faster loops

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

#### References
[The weirdest way to loop in C# is also the fastest - Nick Chapsas](https://www.youtube.com/watch?v=cwBrWn4m9y8)

[MemoryMarshal.GetReference Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.memorymarshal.getreference?view=net-8.0)

[Unsafe.Add Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.add?view=net-8.0)

### Even even faster loops

```csharp
ref var start = ref MemoryMarshal.GetArrayDataReference(items);
ref var end = ref Unsafe.Add(ref start, items.Length);

while(Unsafe.IsAddressLessThan(ref start, ref end)){
    Console.WriteLine(start);
    start = ref Unsafe.Add(ref start, 1);
}
```

Don't mutate the collection during looping.

#### References
[I Lied! The Fastest C# Loop Is Even Weirder - Nick Chapsas](https://www.youtube.com/watch?v=KLtMtxUihBk) (Nick, how do you keep finding these?)

### Unsafe

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/unsafe)

### CollectionsMarshal

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.collectionsmarshal?view=net-7.0)
[Double the Performance of your Dictionary in C# - Nick Chapsas](https://www.youtube.com/watch?v=rygIP5sh00M)

### Disallow Garbage Collection (for a time)

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.gc.trystartnogcregion)

### Loop Unrolling

#### References
["Don't Use Loops, They Are Slow! Do This Instead" | Code Cop #010 - Nick Chapsas](https://www.youtube.com/watch?v=tllygkj0czw)
