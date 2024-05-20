- string.create
- go back and fix up the tabs and spaces
- refs for structs
- thread locals

# Dotnet Optimization Cheatsheet


## 🟢 Least Risky 😇
Nothing too scary here. The documentation should provide you with all the knowledge you need.


### Removing unnecessary boxing/unboxing

#### References
[.NET Performance Tips - Boxing and Unboxing](https://learn.microsoft.com/en-us/dotnet/framework/performance/performance-tips#boxing-and-unboxing)


## 🟡 A Little Risky 🤔
All the usual .NET safeties are included, however you may need to have a deeper understanding of the topic to not run into trouble in order to successfully use them.



### AOT

#### References
[Native AOT Deployment Official Guide](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

[.NET AOT Resources: 2022 Edition](https://www.nikouusitalo.com/blog/net-aot-resources-2022-edition/) (Self plug)

[.NET 7 Self-Contained, NativeAOT, and ReadyToRun Cheatsheet](https://www.nikouusitalo.com/blog/net-7-self-contained-nativeaot-and-readytorun-cheatsheet/) (Self plug)


### Loop Alignment

#### References
[DevBlog Post](https://devblogs.microsoft.com/dotnet/loop-alignment-in-net-6/)




## 🔴 Very Risky ☠

### Unsafe

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/unsafe)

### CollectionsMarshal

#### References
[Official Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.collectionsmarshal?view=net-7.0)
[Double the Performance of your Dictionary in C# - Nick Chapsas](https://www.youtube.com/watch?v=rygIP5sh00M)


### Loop Unrolling

#### References
["Don't Use Loops, They Are Slow! Do This Instead" | Code Cop #010 - Nick Chapsas](https://www.youtube.com/watch?v=tllygkj0czw)
