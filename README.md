# Dotnet Optimization Cheatsheet

Real neato performance/speed/"hacks" for .NET. Like bypassing safties in your car and ripping out the seats in your car to make it go faster, some of these come with a ⚠**RISK**⚠. This means you should understand your own scenario and what the relevant risks are. 

So let's just call these *educational*.

In respect to taking on something from below:
1. Write unit tests so you have a baseline for your functionality
1. Use something like [BenchmarkDotNet](https://benchmarkdotnet.org/) to get a before performance baseline
1. Write your change
1. Run tests
1. Benchmark to see if it did improve what you expected

To clarify some key terms:

| Term        | Meaning    |
| ----------- | ---------- |
| Performance | Speed      |
| Scalability | Throughput |



## `Span`

## Faster loops

## `ValueTask`

### `ManualResetValueTaskSourceCore`

## `Async`

## Pooling
Wrote a little about this on my own site: [Pooling in C#](https://www.nikouusitalo.com/blog/pooling-in-c/).

### `ThreadPool`

### `ArrayPool`

### `RecyclableMemoryStream`

### `MemoryPool`

### `ObjectPool`