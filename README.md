# Dotnet Optimization Cheatsheet

Real neato performance/speed/"hacks" for .NET. Like bypassing safties in your car and ripping out the seats in your car to make it go faster, some of these come with a ⚠**RISK**⚠. This means you should understand your own scenario and what the relevant risks are. 

So let's just call these *educational*.

In respect to taking on something from below:
1. Write unit tests so you have a baseline for your functionality
1. Use something like [BenchmarkDotNet](https://benchmarkdotnet.org/) to get a before performance baseline
1. Write your change
1. Run tests
1. Benchmark to see if it did improve what you expected

To clarify some key concepts that keep coming back below in as short as possible (each one can be it's own book):

| Concept     | Synopsis                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Performance | Speed                                                                                                                                                                                                   |
| Scalability | Throughput                                                                                                                                                                                              |
| Allocations | Boils down to "work the garbage collector need to do". The GC stops our code entirely when cleaning up so if we can relieve pressure. We can also stop inefficient copying of objects around in memory. |

These risk ratings are arbitrary 🤷‍♀️

## 🟢 Least Risky 
Nothing too scary here. The documentation should provide you with all the knowledge you need.

### `Span<T>`

### Pooling
Wrote a little about this on my own site: [Pooling in C#](https://www.nikouusitalo.com/blog/pooling-in-c/).

#### `ThreadPool<T>`

#### `ArrayPool<T>`

#### `RecyclableMemoryStream`

#### References
[Microsoft.IO.RecyclableMemoryStream](https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream)

#### `MemoryPool<T>`

#### `ObjectPool<T>`

#### References
[Turbocharged: Writing High-Performance C# and .NET Code - Steve Gordon](https://www.youtube.com/watch?v=CwISe8blq38)

## 🟡 A little Risky
All the usual .NET safeties are included, however you might need to have an understanding of the topic to not run into trouble in order to successfully use them.

### `Async`

### `ValueTask<T>`

#### `ManualResetValueTaskSourceCore`

## 🔴☠ Very Risky
Prod might go up in a spectacular ball of flame.
![](images\mad-max-fireball.gif)

### Faster loops

#### References
[The weirdest way to loop in C# is also the fastest - Nick Chapsas](https://www.youtube.com/watch?v=cwBrWn4m9y8)

### `Unsafe`