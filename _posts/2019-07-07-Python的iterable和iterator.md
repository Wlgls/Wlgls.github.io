---
layout: post
title: Python中可迭代对象和迭代器的区别以及生成器
categories: [Python]
---

刚刚在复习`map()`和`filter()`函数时，遇到了`iterable`和`iterator`两个相近的单词，所以在此处探究一下两者的区别

## 简单的说
简单的从应用上说：  
* iterable: 可以进行for循环的是iterable，对于一个iterable而言，其序列长度是已知的，而且我们可以多次调用。
* iterator: 可以使用next()函数的是iterator。他的长度是未知的，我们通过使用next(Iterator)来获得其中的元素，但是next()方法是不会回退的，当没有值返回时，我们就会得到一个stopIteration异常。也就是说iterator只能遍历一次。
```python
>>>import collections as cl
>>>a = range(4)
>>>isinstance(a, cl.Iterable)
True
>>>b = iter(a)
>>>isinstance(b, cl.Iterator)
True
>>> next(b)
0
>>> next(b)
1
>>> next(b)
2
>>> next(b)
3
>>> next(b)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

但实际上两者拥有不同的概念，iterable是一个可迭代的对象，而iterator是一个迭代器。在python中，对于iterable我们可以使用iter(Iterable)来获得一个迭代器。python中的list，tuple，dict等都是iterable，他们都遵循了可迭代协议。

### 所有的序列都是可迭代的

解释器在需要迭代对象x时，会自动调用iter(x)，而内置的iter函数有以下作用
1. 检查对象是否实现了__iter__方法，如果实现了就调用他，获取一个迭代器
2. 如果没有实现__iter__方法，但是实现了__getitem__方法，Python会自动创建一个迭代器，尝试按顺序（从0开始）获取元素
3. 如果尝试失败，则抛出TypeError异常。

而所有的序列都实现了__getitem__方法，所以他们都可迭代。但实际上，标准的序列都实现了__iter__方法，所以我们在自建序列时也应该这么做。


## 如何手动实现iterable和iterator呢。

* 作为一个iterable，我们只需要实现__iter__()方法。且这个方法要返回一个对应的iterator。
* 而作为一个iterator，我们需要实现__next__()方法和__iter__()方法。

我们需要注意的是，在iterator中实现__iter__()方法返回的是其本身，之所以这么做，是因为在很多地方接受的参数是iterable，如果iterator也是iterable，那么接受参数时，就不需要这么繁琐的转换了。但是虽然说iterator是iterable，在对iterator使用for循环的时候，还是只能遍历一次的。

```python
>>> a = iter(range(4))
>>> isinstance(a, cl.Iterator)
True
>>> for i in b:
...     print("first", i)
... 
first 0
first 1
first 2
first 3
>>> for i in b:
...     print("second", i)
... 
```

现在就可以尝试实现自己的iterable和iterator了
```python
import collections.abc as cl
class myrange(object):
    def __init__(self, n):
        self.l = list(range(n))
    def __iter__(self):
        return myrange_interator(self.l)
    
class myrange_interator(myrange):
    # 继承了myrange

    def __init__(self, l):
        self.list1 = l
        self.index = 0
    def __next__(self):
        try:
            num = self.l[self.index]
        except IndexError:
            raise StopIteration
        self.index += 1
        return num

if __name__ == '__main__':
    myrange = myrange(4)
    print(isinstance(myrange, cl.Iterable))
    myrange_interator = iter(myrange)
    print(isinstance(myrange_interator, cl.Iterator))
    print(next(myrange_interator))
    print(next(myrange_interator))
    print(next(myrange_interator))
    print(next(myrange_interator))
    print(next(myrange_interator))
```
```python
结果：
True
True
0
1
2
3
Traceback (most recent call last):
    ...
    raise StopIteration
StopIteration
```

### For循环

在Python中，for循环就是一个使用迭代器的有力实例。

当我们通过for循环对一个可迭代对象进行迭代时， for循环内部就会自动调用iter()方法执行可迭代对象内部定义的__iter__()方法获得一个迭代器。然后一次次的通过调用next()方法获取数据，当没有下一个数据时， for循环自动捕获并处理StopIteration异常。

## 生成器

Python中存在`yield`关键字。这个关键字就是为了构建生成器。生成器函数调用时产生的对象就是生成器， 所有的生成器都是迭代器，因为生成器实现了迭代器接口， 即__next__()和__iter__()方法。不过，迭代器是用于从集合中取出数据，生成器用于‘凭空’产生数据。但是在Python社区中，大多数都把迭代器和生成器视作同一概念。

### 生成器函数

只要Pyhton函数体中有有yield关键字，该函数就是生成器函数。调用生成器函数的时候会返回一个生成器对象。

生成器函数会创建一个生成器对象，包装生成器函数的定义体。把生成器传给next()函数时，生成器函数会向前，执行函数体中的下一条yield语句，返回产生的值，并在函数定义体的当前位置暂停。最终，函数的定义体返回时，外层的生成器对象会抛出StopInteration异常。

```python
>>> def gen_123():
...     yield 1
...     yield 2
...     yield 3
... 
>>> gen_123
<function gen_123 at 0x7f0639ce7bf8>
>>> gen_123()
<generator object gen_123 at 0x7f0639ce57c8>
>>> for i in gen_123():
...     print(i)
... 
1
2
3
>>> g = gen_123()
>>> next(g)
1
>>> next(g)
2
>>> next(g)
3
>>> 
>>> next(g)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

## 惰性实现

在我们自动实现迭代器的时候，我们是通过将数据方法列表中，然后在逐一遍历，但是这样列表使用的内存量就和文本一样多，所以我们可以惰性求知，也就是使用生成器来实现迭代器。

在我们调用生成器的时候， 只会产生一个值， 他不会一次性将所有的值返回给你， 你需要一个那么就调用一个next()值， 然后就运行到下一个yield关键字处。这样一次返回一个值， 并不会占用太大内存。

但是这也意味着我们无法得知，下一个元素是什么， 也不会知道后面还有多少元素。


## 使用yield实现迭代器

```python
import collections.abc as cl
class myrange(object):
    def __init__(self, n):
        self.l = list(range(n))
    def __iter__(self):
        for i in self.l:
            yield i
        return 
    


if __name__ == '__main__':
    myrange = myrange(4)
    print(isinstance(myrange, cl.Iterable))
    myrange_interator = iter(myrange)
    print(isinstance(myrange_interator, cl.Iterator))
    print(next(myrange_interator))
    print(next(myrange_interator))
    print(next(myrange_interator))
    print(next(myrange_interator))
    print(next(myrange_interator))
```

这一次我们不实现单独的迭代器，而是直接将`__iter__`变成一个生成器函数，迭代器其实就是生成器对象，每次调用__iter__方法都会自动创建。然后我们在调用next方法时生成一个元素并返回，而不是重新存放在列表中。