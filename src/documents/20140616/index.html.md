---
title: "快速问答（2）：将自定义的值类型用作字典的键，需要做什么特别的事情吗？"
layout: "note"
link: http://blog.zhaojie.me/2014/05/zrx-quiz-1-answer.html
note_date: "2014-06-16"
---

第一期的“快速问答”发布至今已经过了三个星期了，不过一个正确答案也没有。有几位同学尝试做了解答，但都有点答非所问的样子。这次先对这个问题进行简单讲解，然后再提新一期的问题。

上一期的问题是，如何可以让同一个元素存放入多个`LinkedSet`中。尝试回答的同学都把它当做数据结构的问题来做了，这个思维实在有些禁锢。事实上，这是一道标准的.NET实践题，就看你能否灵活运用已有知识了。相关内容之前刚在第一期“有奖征答”活动中提起过，点击下方“<span style="color:blue;">阅读原文</span>”链接再回顾一下吧！

（输入<span style="color:red;">3</span>可以查看“有奖征答”相关内容，输入<span style="color:red;">4</span>可以查看“快速问答”相关内容，或访问<span style="color:blue;">zrx.zhaojie.me</span>查看所有消息归档。）

文章里提到，使用不同具体类型来特化泛型类型，可以得到完全不同的类型，这点和Java的“伪泛型”完全不同。为什么一个元素对象无法放入多个`LinkedSet`？正是因为`ILinkedSetElement`接口只有一个，而一个类型无法多次实现同一个接口。那么，我们使用泛型的特化来实现多个接口不就可以了吗？且看下方代码：

```cs
public interface ILinkedSetElement<T, TKind>    where T : class, ILinkedSetElement<T, TKind> {        T Next { get; set; }}public class LinkedSet<T, TKind>    where T : class, ILinkedSetElement<T, TKind> {}
```

我们为`ILinkedSetElement`和`LinkedSet`各自添加一个泛型参数，这样在使用时我们便可以通过指定不同的具体参数，来作为两个完全不同的类型使用：

```
public class Set1 { }

public class Set2 { }

public class MyElement :
    ILInkedSetElement<MyElement, Set1>,
    ILInkedSetElement<MyElement, Set2> {
    
    MyElement ILinkedSetElement<MyElement, Set1>.Next { get; set; }
    
    MyElement ILinkedSetElement<MyElement, Set2>.Next { get; set; }
}
```

配合C#的“显式实现接口”，同时定义两个`Next`属性毫无压力。其实就是这么简单。

这里我又要说了，Java在很多细节方面真的很不为开发人员考虑，例如，两个接口中存在同名的方法，居然就不能将它们区分开来，简直岂有此理。

好吧，接下来发布“快速问答”第二期的题目。

“字典”是我们常用的数据容器，它的作用是从“键”映射到“值”。我们这里只讨论泛型的字典，也就是`Dictionary<TKey, TValue>`，它的键和值都是强类型的。那么现在的问题是，假如我们要把一个自定义的值类型（即`struct`）作为键来使用，需要做什么特别的事情吗？还是只要如此简单即可？

```cs
public struct MyKey {    // Some fields}Dictionary<MyKey, int> _dict = new Dictionary<MyKey, int>();
```

请大家将答案想清楚并描述完整后，贴在<span style="color:blue;">gist.github.com</span>上再发给我，自然越详细越好。有必要的话，也可以附带一些示例代码。

本期的奖品如封面所示，是闪迪32G的USB 3.0优盘，刚看下来京东价是89元，暂时准备了五份。这是道标准的.NET基础题，对于日常开发十分重要，按理说也挺简单的。大家加油。