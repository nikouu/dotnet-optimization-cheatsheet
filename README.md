- add inline arrays
- rewrite with better preamble around optimisation
- add table of contents for the different levels
- string.create
- suggestions on what to optimise first such as architecture, adding more hardware
- go back and fix up the tabs and spaces
- refs for structs

# Dotnet Optimization Cheatsheet


## 🟢 Least Risky 😇
Nothing too scary here. The documentation should provide you with all the knowledge you need.





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


## 🟡 A Little Risky 🤔
All the usual .NET safeties are included, however you may need to have a deeper understanding of the topic to not run into trouble in order to successfully use them.


### Garbage Collection

#### References
[Garbage collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/)

[Workstation and server garbage collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc)


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


### Inline Arrays

#### References

[David Fowler](https://twitter.com/davidfowl/status/1678190691841716224)






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
