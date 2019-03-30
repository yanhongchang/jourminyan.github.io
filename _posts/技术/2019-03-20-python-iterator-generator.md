---
layout: post
title: python 中的迭代器和生成器
category: 技术
---

更新历史：

- 2019.03.20 完成
- 2019.03.30 修改完善

------

### 迭代器
想要弄清楚迭代器，先理什么是可迭代对象？
可迭代对象可以理解为:一个对象如果拥有`__iter__`方法，就可以说这个对象是可迭代对象。

在python中如果一个对象满足如下协议（约定）时，就可以成为迭代器。
1. 必须是可迭代对象，也就是说对象必有有`__iter__`方法
2. 可以通过next()方法获取下一个元素。
3. 可以在最后没有元素的情况下，返回StopIteration


这么看来我们可以自己实现一个迭代器，类中重写`__iter__`方法，返回一个对象。这个对象有next()方法。并且在没有元素的时候，可以返回stopIteration。例如：

```
class MyIter（ilist）：
    def __init__(self, ilist):
        self.list = ilist
        self.c = 0

    def __iter__(self):
        return self

    def next(self):
        if self.c == len(self.list):
            self.c = 0
            raise StopIteration

        self.c += 1
        return self.list[self.c-1]
```

如上就是可以实现一个迭代器。但是一般我们不自己实现，可以通过生成器来实现迭代器。生成器稍后我们介绍。这里先看一下，为什么要有迭代器？或者迭代器的好处是什么？

迭代器最大的好处就是每次调用next()只会返回一个元素，即便迭代器可以获取非常庞大数量级的元素，但是每次只返回一个元素。这样对内存的使用非常友好，不会造成内存大量开销。

如何获取一个迭代器：
#### 1、iter（）
可以通过iter（）将一个可迭代对象转化成一个迭代器。可迭代对象例如：list、dict、string、tuple等等

```
>>> a
[1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> b = iter(a)
>>> b
<listiterator object at 0x7f38dce1c850>
```

将一个可迭代对象list，转化成列表迭代器listiterator。

#### 2、通过生成器获取一个迭代器。
具体见下面生成器介绍。

### 生成器

生成器其实是一种特殊的迭代器，生成器自动实现了或者说满足了迭代器所要求的条件（`__iter__`和next()）。

那么怎么获得生成器呢？常用有如下两种方法。

#### 1、通过修改列表生成式获得
将列表生成式 [x for x in range(5)] 中的[]修改为(),既可以获得一个生成器
(x for x in range(5))

```
>>> type((x for x in range(5)))
<type 'generator'>
```

我们证明一下生成器也是迭代器。

```
>>> a = (x for x in range(5))
>>> dir(a)
['__class__', '__delattr__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__iter__', '__name__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'close', 'gi_code', 'gi_frame', 'gi_running', 'next', 'send', 'throw']
```

可以看到生成器包含了__iter__ 和next方法，满足迭代器的要求。

#### 2、通过yield 实现生成器
普通函数中如果包含了yield关键字，就可以称这个函数整体为生成器。
yield关键字的含义是：当程序执行到yield时，返回yield后面的值给调用方，然后中断。当通过next()再次调用是时，继续执行yield后续代码。并且遇到yield时，再次返回并中断程序。

```
>>> def count(i):
...     while True:
...         yield i
...         i += 1
... 
>>> 
>>> co = count(3)
>>> type(co)
<type 'generator'>
>>> co.next()
3
>>> co.next()
4
>>> co.next()
5
>>> dir(co)
['__class__', '__delattr__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__iter__', '__name__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'close', 'gi_code', 'gi_frame', 'gi_running', 'next', 'send', 'throw']
```

可以看到这里的co就是一个生成器。通过dir(co),发现存在__iter__ 和next，所以再次证明生成器是一种特殊的迭代器。
