# Dotnet Optimization Cheatsheet

Donald Knuth said:

>Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

If you're the 3%, this is an educational look at .NET performance tricks I've gathered over time. By reading this I assume you understand that architecture, data flow, general performance decisions, etc have a far larger impact to your codebase than optimising for single digit milliseconds or nanoseconds. 

You'll need to consider:
1. Will I be pre-maturely optimising?
1. Understand the risks involved
1. Is the time spent doing this worth it?


I've rated each optimisation here in terms of difficulty. You may gauge the difficulties differently. 

| Difficulty       | Reasoning                                                                                                        |
| ---------------- | ---------------------------------------------------------------------------------------------------------------- |
| 🟢 Easy         | These are either well known, or simple to drop into existing code with only some knowledge.                      |
| 🟡 Intermediate | Mostly accessible. May require a bit more coding or understanding of specific caveats of use.                    |
| 🔴 Advanced     | Implementation and caveat understanding is critical to prevent app crashes or accidentally reducing performance. |

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

### 🟡 `async`

[Official Guide](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)


