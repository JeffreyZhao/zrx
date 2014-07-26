---
title: "快速问答（4）：需要对缓冲状态做哪些检验？"
layout: "note"
link: http://msft.moe/
note_date: "2014-07-25"
---

大好周末，怎能不继续做题？我们来看看之前关于“缓冲依赖”的参考解答，然后再来补充一道新的题目吧——还是关于缓冲的。

## 上期解答

照例先来回答上期问题。上期累计有将近20位同学参与答题，最终有7位同学拿到了20到45不等的分数（满分50），参与度比我想象中高，让我颇感欣慰，于是对于搞好“赵人希”更有信心了，在这里先多谢大家支持。

回到上期的问题。首先，这段代码：

```cs
var firstBuffer = new FirstBuffer();
var secondBuffer = new SecondBuffer();

// Thread A
new Thread(() =>
{
  while (true)
  {
    var item = new Item();

    firstBuffer.Add(item);
    secondBuffer.Add(item);

    Thread.Sleep(1);
  }
}).Start();

// Thread B
new Thread(() =>
{
  while (true)
  {
    firstBuffer.Flush();
    secondBuffer.Flush();

    Thread.Sleep(100);
  }
}).Start();
```

显然是不能保证两个缓冲的依赖关系的，因为对于某个特定的`item`，四个`Add`及`Flush`调用的时序可能会是这样的：

```cs
firstBuffer.Flush(); // Thread B
firstBuffer.Add(item); // Thread A
secondBuffer.Add(item); // Thread A
secondBuffer.Flush(); // Thread B
```

于是，对于这个特定的`item`，它的`Second`方法显然会先于`First`方法执行。

### 零分答案

于是有的同学就很自然地想到加锁，也就是为所有的`Add`和`Flush`方法都加上同一把锁：

```cs
private readonly object _gate = new object();

// Thread A
lock (_gate) {
    firstBuffer.Add(item);
    secondBuffer.Add(item);
}

// Thread B
lock (_gate) {
    firstBuffer.Flush();
    secondBuffer.Flush();
}
```

这当然可能保证先后顺序，但是这个答案我给零分。因为这导致了两个缓冲在执行`Flush`的时候，任何一个缓冲都无法`Add`元素。既然是缓冲，那它的`Flush`操作预期就会花较长时间，现在还得把两个`Flush`操作的耗时加起来一起阻塞掉，那假如是三个或者更多呢？这种做法无法令人接受。所以没法在`Flush`时执行`Add`操作的答案我都给了零分，原题中的示例`AddBuffer`还特意强调了`Flush`中的锁只做了引用复制，因此开销很低。

总之要拿分，光从“效果”上执行正确没用，我给分是有门槛的。说起来今天的主要工作成果之一，就是把一行代码从`Double.Parse(value.ToString("N4"))`修改成了`Math.Round(value, 4)`，性能顿时提升一大截。真为项目里出现这种愚蠢的代码感到痛心……

还有同学的做法是修改`Item`对象等等，首先这种做法的原理还是基于阻塞，效果肯定不好。更重要的是，这种做法的通用性很差，在实际情况中，可能两个缓冲中并非放入同一类型的元素，甚至其中一个缓冲还会交给其他代码块使用。有个同学的做法是从`firstBuffer`返回实际`Flush`的元素数量，然后传入`secondBuffer`的`Flush`方法中去，按照类似原因也只能拿到零分。

至于还有一些特别复杂或是稀奇古怪的答案，我就不一一描述了。

### 得分答案

拿到20分的答案，几乎都是让`firstBuffer`引用了`secondBuffer`，先只在`firstBuffer`里添加元素，然后在`firstBuffer`的`Flush`里再往`secondBuffer`里面添加。这个答案让我感到很为难，因为它的适用度远小于后面的参考答案（往后看了就知道），但似乎也不是不能通过一些改造来迎合各种变化，换句话说它还是稍微有那么一点点点点点……的通用性的，于是就给了20分。

你看我多有诚意，为了给分都不惜脑补各种答案，说不定写这类代码的同学还脑补不出来呢。“赵人希”简直实至名归。

拿到40分的答案，跟后面的参考答案理念较为接近，因此可以拿到九成分数。我们再来想想原代码为什么会“失败”：其实直到`firstBuffer`进行提交之前，两个缓冲的元素还是有正确的依赖关系，但`firstBuffer`在提交过程中，`secondBuffer`的状态发生改变了，因此它包含了`firstBuffer`在提交那一刹那还不存在的元素。

于是有的同学就想到在提交`firstBuffer`前锁住`secondBuffer`，这便得到了零分答案。但我们完全可以换个思路——不要“锁定”，而是让`secondBuffer`先生成一份当前状态的“快照”，这不就可以了么！于是有的同学就是这么回答的：首先为缓冲模型（即`IBuffer`接口）增加一个`Prepare`方法，作用是保留当前状态，然后在`Flush`方法内部提交的是已保留的状态，而不是正在进行收集的状态。代码大致是这样的：

```cs
private readonly object _gate = new object();
private List<Item> _items;
private List<Item> _prepared;

public void Prepare() {
    lock (_gate) {
        _parepared = _items;
        _items = null;
    }
}

public void Flush() {
    List<Item> prepared;
    lock (_gate) {
        prepared = _prepared;
        _prepared = null;
    }
    
    if (prepared == null)
        return;
    
    // do something
}
```

这样，我们按照“逆序”来`Prepare`，再顺序进行`Flush`，便可以保证`secondBuffer`提交的状态肯定早于`firstBuffer`的状态——至于`firstBuffer`可能一并提交了更多地元素，那无所谓，反正会在`secondBuffer`中随着下一批次的元素一起提交呗。这种做法可谓通用性上佳，甚至可以组合任意多个缓冲。

想到这个办法，便可以拿到40分了。至于有人35分，是因为虽然原理想到了，但实现太过粗糙，还有明显的bug。另一位同学45分，是因为他还做了参数检验：假如`Prepare`调用时发现还没有`Flush`过，那就抛出一个异常，值得鼓励。

### 参考答案

当然，没有一个人的答案能够拿到满分。就基于40分的答案来说吧，虽然思路到位了，但是还是有显著缺点。例如，程序员假如先`Flush`再`Prepare`会怎么样？假如反复调用`Prepare`或`Flush`又如何？当然，我们可以抛出异常，但这就只能在运行时才能发现问题。一套好的API，应该尽可能的减少误用的可能，其中一个原则之一，便是入口尽可能少。现在的`IBuffer`出现了两个接口，这就带来误用的可能。

那么更好的做法是怎么样的呢？我认为是这样的：

```cs
public interface IBufferState {
    void Commit();
}

public interface IBuffer {
    IBufferState CaptureState();
}
```

在缓冲对象接口上，我们依旧只有一个入口，而API的使用者只能在通过第一个入口之后，才能发现第二个入口，这就避免了误用。即使为了方便，我们需要一个方法可以直接提交缓冲，也只要再补充一个`Flush`扩展方法就行了（用Java的同学眼馋不）：

```cs
public static void Flush(this IBuffer buffer) {
    buffer.CaptureState().Commit();
}
```

我们还可以对任意多个有依赖的缓冲进行统一提交：

```cs
public static void FlushAll(this IList<IBuffer> buffers) {
    var states = new IBufferState[buffers.Count];
    
    for (var i = buffers.Count - 1; i >= 0; i--) {
        states[i] = buffers[i].CaptureState();
    }
    
    foreach (var s in states) {
        s.Commit();
    }
}
```

甚至，我们可以多次获取同一个缓冲的状态，再统一进行提交。这种做法赋予我们更大的灵活性，这是40分的做法所不具备的。

是不是很简单？具体实现可谓是小菜一碟了。

## 本期问题

不过菜虽然小，我就是不端上来给你尝，因为这会是本期的问题，嘿嘿嘿……

且看下方代码片段。`BufferBase`是一个抽象的基类，我们可以继承它并实现具体的`Flush`逻辑：

```cs
public abstract class BufferBase<T> : IBuffer {
    private readonly object _gate = new object();
    private List<T> _items;
    
    public void Add(T item) {
        lock (_gate) {
            if (_items == null) {
                _items = new List<T>();
            }

            _items.Add(item);
        }
    }

    public IBufferState CaptureState() {
        List<T> items;
        lock (_gate) {
            items = _items;
            _items = null;
        }

        // ...
    }
    
    protected abstract void Flush(List<T> items);
}
```

这一期的题目是这样的：请将`BufferBase`补充完整，除了基本逻辑之外，请做出必要的检验，并在合适的时候抛出异常，以防止误用。为了简化问题，我们还是规定这么一个简单的并发环境：

1. `Add`方法可以被任意多个线程随意并行调用，而且，调用`Add`时，可能会有另一个线程正在调用`CaptureState`和`Commit`方法。
2. `CaptureState`和`Commit`方法只会被同一个线程调用，但于此同时，可能会有其他线程正在调用`Add`方法。

不难吧？我觉得比上一期要简单多了，大家可以继续尝试。不过请尽量不要问我问题，我认为问题描述已经足够清晰和准确了，必要的时候我会做出提示的。总有同学要我给出更多补充条件，似乎没有规定具体方向就写不来代码了，要知道实际开发过程中，我们往往都只有一个目标，要发散思路寻找合适的道路，不要脑补无谓的“限制”来束缚自己。

最后，大家可以点击下方“<span style="color:blue;">阅读原文</span>”链接来访问一个神奇的网站。一定要戴上耳机或是打开音响——网站内容我不管，但我已经被它的背景音乐洗脑了，单曲循环八百遍，这篇文章从头至尾都是听着它写下来的……

至于本期封面，还是那奇妙的TPL Dataflow示意图之一，与本文无关，仅供欣赏……