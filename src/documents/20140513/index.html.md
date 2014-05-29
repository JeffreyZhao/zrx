---
title: "关于.NET程序性能的基本要领"
layout: "note"
link: http://download-codeplex.sec.s-msft.com/Download?ProjectName=roslyn&DownloadId=838017
note_date: "2014-05-13"
---

话说 Roslyn 大家肯定都已经有所耳闻了，这是下一代 C# 和 VB.NET 的编译器实现。Roslyn 使用纯托管代码开发，但性能超过之前使用 C++ 编写的原生实现。

Bill Chiles 是 Roslyn 的 PM（程序经理，Program Manager），他最近写了一篇文章叫做《Essential Performance Facts and .NET Framework Tips》，其中总结了几条经验，点击底部“阅读原文”可以看到这篇文章，目前是个 CodePlex 上的 PDF 文件，以后可能会发布在 MSDN 上。

他在文章里谈到以下几点：

1. 不要进行过早优化。有的时候重复计算都比使用哈希表进行缓存来的快。
2. 没有评测，便是猜测。
3. 好工具很重要。这里他推荐了 PerfView（封面便是），这是个微软发布的免费工具。
4. 性能的关键，在于内存分配。
5. 其他一些细节。

关于第4点需要多说几句。对于托管环境来说，GC 对于性能的影响可谓事关重大。假如一段程序写的不够 GC 友好，让 GC 发生的多，尤其是那种 Stop-the-World GC，这对性能的影响远胜某些“多花了几条拷贝指令”之类的“探索”。很多时候，用户眼中的“性能”在于程序的“响应程度（responsiveness）”，而 GC 一旦暂停所有线程，对响应程度的影响可想而知。

相较于 Java 平台来说，.NET 已经是个相对 GC 友好的运行环境了。其中重要的方面之一便是自定义值类型，即 struct。struct 让程序员进行一定程度上可控的内存分配，避免在堆上产生对象。而在Java中，只有几种原生类型是值类型，值类型还不能包含成员。对一个 .NET 程序员来说可能很难想象，在 Java 里无法使用一个未装箱的 `int` 值作为一个字典的键，但事实便是如此。

当然，Java 似乎已经有打算作这方面的改进（可以搜索“Value Types for Java”），但离真正可用还遥遥无期。目前 Java 只能通过一些如逃逸分析的手段，发现某个对象不会被共享到堆上，于是便将其分配在栈上。

当然 .NET 提供再多对 GC 友好的功能，也抵不过开发人员的误用。Bill 的文章里举了一些常见案例，当然其实这些都是每个 .NET 开发人员必须了解的基础。那个例子颇为有趣，他谈到对于性能敏感的地方，有时候都要避免 LINQ 或 Lambda。因为使用 Lambda 构造匿名函数时，编译器会产生闭包，因为所谓闭包，便是一个用来保存上下文的，分配在堆上的对象。此外，如 `List<T>` 的迭代器被有意实现为 struct，但使用通用的 LINQ 接口，则会被转化为 `IEnumerable<T>` 和 `IEnumerator<T>`，产生装箱。

无独有偶，今天早上 @连城404 在新浪微博上说到：“按照 Michael 的建议把 HiveTableScan 关键路径上的 FP 风格的代码换成 while 循环加可复用的 mutable 对象，扫表性能提升40%。”，这其实也正和这次的话题密切相关。在 SICP 的脚注中 Alan Perlis 同样写到：“Lisp programmers know the value of everything but the cost of nothing.”，颇为有趣。

常用的 FP 手段的确会带来性能开销，这是事实，不过假如你现在立即得出“不要用 FP”这样的结论，那我也只能用怜悯的眼光看着你了。

最后，您有减少内存分配，优化 GC 这方面的实践吗？告诉我吧，有机会我也会谈一下我在这方面的一些技巧和案例的。