- go back and fix up the tabs and spaces
- refs for structs
- thread locals
- different lock types like interlocked and semaphoreslims


# Dotnet Optimization Cheatsheet


## 🟢 Least Risky 😇
Nothing too scary here. The documentation should provide you with all the knowledge you need.


## 🟡 A Little Risky 🤔
All the usual .NET safeties are included, however you may need to have a deeper understanding of the topic to not run into trouble in order to successfully use them.


## 🔴 Very Risky ☠


### Loop Unrolling

#### References
["Don't Use Loops, They Are Slow! Do This Instead" | Code Cop #010 - Nick Chapsas](https://www.youtube.com/watch?v=tllygkj0czw)
