# Dotnet Optimization Cheatsheet

# Still under construction 🏗️

Practical neato performance/speed/"hacks"/good practices for .NET I've collected over time. Like bypassing safeties in your car and ripping out the seats to make it go faster, some of these come with a ⚠**RISK**⚠. This means you should understand your own scenario and what the relevant risks are. 

So let's just call these *educational*. Each item is fully referenced so you can educate yourself too. Oh and they're mostly about language features over things like what should be existing good coding practices and architecture.

## ‼ Read This First ‼
1. Try not to pre-maturely optimise
1. Understand the risks involved
1. Is the time spent doing this worth it?

This readme is giving you the tools. It's up to you to understand how and where to use them. *Or don't, I'm not gonna tell you how to live your life.*

## Starting Optimizing

In the shortest way possible:

1. Write unit tests so you have a baseline for your functionality
1. Use something like [BenchmarkDotNet](https://benchmarkdotnet.org/) to get a before performance baseline
1. Write your change
1. Run tests
1. Benchmark to see if it did improve what you expected

To clarify some key concepts that keep coming back below in as short as possible (as each one can be it's own book):

| Concept     | Synopsis                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Performance | Speed                                                                                                                                                                                                   |
| Scalability | Throughput                                                                                                                                                                                              |
| Allocations | Boils down to "work the garbage collector need to do". The GC stops our code entirely when cleaning up so if we can relieve pressure. We can also stop inefficient copying of objects around in memory. |

By the way, the risk ratings are arbitrary 🤷‍♀️

## 🟢 Least Risky 😇
Nothing too scary here. The documentation should provide you with all the knowledge you need.

### Span<T>, Memory<T>

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.span-1?view=net-7.0)

[Memory<T> and Span<T> usage guidelines](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines)

[Turbocharged: Writing High-Performance C# and .NET Code - Steve Gordon](https://www.youtube.com/watch?v=CwISe8blq38)

### Faster loops

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

### Struct

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct)

### AOT

#### References
[Native AOT Deployment Official Guide](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

[.NET AOT Resources: 2022 Edition](https://www.nikouusitalo.com/blog/net-aot-resources-2022-edition/) (Self plug)

[.NET 7 Self-Contained, NativeAOT, and ReadyToRun Cheatsheet](https://www.nikouusitalo.com/blog/net-7-self-contained-nativeaot-and-readytorun-cheatsheet/) (Self plug)

### Vectorizing

#### References
[Use SIMD-accelerated numeric types](https://learn.microsoft.com/en-us/dotnet/standard/simd)

## 🔴 Very Risky ☠
🔥 Prod might go up in a spectacular ball of flame 🔥

![](images/mad-max-fireball.gif)

### Even faster loops

#### References
[The weirdest way to loop in C# is also the fastest - Nick Chapsas](https://www.youtube.com/watch?v=cwBrWn4m9y8)

[Unusual optimizations; ref foreach and ref returns - Marc Gravell](https://blog.marcgravell.com/2022/05/unusual-optimizations-ref-foreach-and.html)

### Unsafe

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/unsafe)
