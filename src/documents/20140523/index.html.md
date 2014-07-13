---
title: "编程技巧：一个开销很小的集合实现"
layout: "note"
link: http://referencesource.microsoft.com/
note_date: "2014-05-23"
---

集合（Set）是一种很常用的数据结构。在.NET基础类库中定义了标准的`ISet`接口，并提供了`HashSet`和`SortedSet`两种实现，我们先来简单回顾下这两个类：

1. `HashSet`使用和`Dictionary`类似的实现方式，基于散列值和相等性比较来实现，各操作的时间复杂度为O(1)。
2. `SortedSet`使用和`SortDictionary`类似的实现方式（红黑树），基于两两大小比较来实现，各操作的时间复杂度为O(logN)，不过能够直接按照排序后的顺序输出元素。

作为一个合格的.NET程序员，这些都是您必须烂熟于心的知识。此外，假如您还没有看过这两个类的实现，则建议还是尽早去看一下。目前.NET已经将源代码公开了，并基于Roslyn生成了索引，可以进行快速查找。您可以点击文末“<span style="color: blue;">阅读原文</span>”跳转至Reference Source网站浏览.NET源代码。本文的封面便是`SortedSet`的头部注释。挺有意思的，还推荐了一本书呢。

下面这个场景可能大伙都不陌生：我们准备了大量的对象，在某些时候我们需要对其中的一部分进行标记，并且在稍后进行统一处理。过段时间再标记，再处理，如此往复不断。这时候我们便可以使用集合来辅助实现，例如创建一个`HashSet`，保存需要标记的元素，然后只需遍历集合内的元素便是。

不过，无论是`HashSet`还是`SortedSet`，都需要进行多次比较或是计算，性能相对较差。事实上，它们对于这个场景来说都显得有些过重，我们完全可以用一个简单得多的实现，几十行代码即可：

```cs
public interface ILinkedSetElement<T>    where T : class, ILinkedSetElement<T> {        T Next { get; set; }}public class LinkedSet<T>    where T : class, ILinkedSetElement<T> {    private T _top;    private T _bottom;    public void Add(T ele) {        Debug.Assert(ele.Next == null, "It's already in a set!");        ele.Next = _top;        _top = ele;        if (_bottom == null) {            _bottom = _top;        }    }    public T RemoveOne() {        var top = _top;        _top = top.Next;        if (_top == null) {            _bottom = null;        }        top.Next = null;        return top;    }    public bool IsEmpty {        get { return _top == null; }    }    public bool Contains(T ele) {        return ele.Next != null || ele == _bottom;    }}
```

`LinkedSet`的原理是，通过要求每个元素强制实现一个接口，让其自带一个与自身类型相同的`Next`指针，这样在`LinkedSet`内部，我们可以将元素依次串联起来，而无需分配任何额外的对象。至于判断一个元素是否在集合中，只要检查它的`Next`是否为空，以及它是否为集合的第一个元素（`_bottom`）即可。它的接口很有限，几乎就是一个“栈”，但足以满足我们的场景了。

当然，这个所谓的“集合”十分不安全。例如，`Next`指针可以被外部修改。再例如，一个元素不能被存放在两个集合中。还例如，它的元素只能是自定义的class。因此，这个解决方案只能用于程序内部，不能暴露给外部代码。

最后，大家再来思考一个问题。假如我们真的需要将同一个元素放在多个的集合中呢？