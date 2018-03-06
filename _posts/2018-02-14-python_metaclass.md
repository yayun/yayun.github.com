---
layout: post
title: "Python metaclass"
---

*翻译自: [what are metaclasses in Python](https://stackoverflow.com/questions/100003/what-are-metaclasses-in-python/6581949#6581949)*

## 1. Classes as objects  

对大多数语言来说，class 是一些用来描述怎样生成对象的代码模板。Python 中也差不多：

```python
In [1]: class ObjectCreator(object):
   ...:     pass
   ...:

In [2]: my_object = ObjectCreator()

In [3]: print my_object
<__main__.ObjectCreator object at 0x10de1d1d0>
```
但是在 Python 中，class 不只是产生对象的代码，Class 也是对象！

当我们使用 *class* 关键字时，Python 执行它并且创建一个 *对象*. eg:
声明：
```pythoon
In [1]: class ObjectCreator(object):
   ...:     pass
   ...:
```
会在内存中创建名为 *ObjectCreator* 的对象。

*这个对象（类）自己也有创建对象（实例）的能力，这就是为什么它是一个类*

但是他自己也是一个对象，因为：

* 将类赋值给一个变量 
* 复制
* 给类添加一个属性
* 可以作为一个函数的参数传递

```python
In [4]: print(ObjectCreator) # you can print a class because it's an object
<class '__main__.ObjectCreator'>

In [5]: def echo(o):
   ...:     print(o)
   ...:

In [6]: echo(ObjectCreator) # you can pass a class as a parameter
<class '__main__.ObjectCreator'>

In [7]: print(hasattr(ObjectCreator, 'new_attribute'))
False

In [8]: ObjectCreator.new_attribute = 'foo' # you can add attributes to a class

In [9]: print(hasattr(ObjectCreator, 'new_attribute'))
True

In [10]: print(ObjectCreator.new_attribute)
foo

In [11]: ObjectCreatorMirror = ObjectCreator # you can assign a class to a variable

In [12]: print(ObjectCreatorMirror.new_attribute)
foo

In [13]: print(ObjectCreatorMirror())
<__main__.ObjectCreator object at 0x10de1d650>
```
## 2. 动态创建类

既然类是对象，我们就可以像创建对象一样随手创建一个类.

首先，我们可以在一个函数中使用 *class* 关键字创建类：
```python
In [14]: def choose_class(name):
   ....:     if name == 'foo':
   ....:         class Foo(object):
   ....:             pass
   ....:         return Foo # return the class, not an instance
   ....:     else:
   ....:         class Bar(object):
   ....:             pass
   ....:         return Bar
   ....:

In [15]: MyClass = choose_class('foo')

In [16]: print(MyClass) # the function returns a class, not an instance
<class '__main__.Foo'>
In [18]: print(MyClass()) # you can create an object from the class
<__main__.Foo object at 0x10de1d890>
```

这样看起来并不是很动态，因为你仍然需要自己写整个类

既然 classes 是对象，那么类就可以动态的生成.

当你使用 *class* 关键字的时候，Python 动态创建对象. 我们也可以手动创建对象！

还记的 *type* 函数吧，一个可以让我们知道一个对象是什么类型的函数：
```python
In [19]: print(type(1))
<type 'int'>

In [20]: print(type("1"))
<type 'str'>

In [21]: print(type(ObjectCreator))
<type 'type'>

In [22]: print(type(ObjectCreator()))
<class '__main__.ObjectCreator'>
```

*type* 也有另外一个完全不一样的功能： 它可以随手创建一个类. *type* 可以用类的描述作为参数，并且返回一个新的类。

 (一个相同的函数因为你传递的参数不同而有两种不同的使用方法看起来很愚蠢。对于 Python 这种需要一直向后兼容的语言来来说确实是一个问题)

*type* 可以用这种方式工作：

```
type(name of the class,
     tuple of the parent class (for inheritance, can be empty),
     dictionary containing attributes names and values)
```
eg:

```python
In [24]: class MyShinyClass(object):
   ....:     pass
```

可以用这种方式手动创建：

```python
In [25]: MyShinyClass = type('MyShinyClass', (), {}) # returns a class object
In [26]: print(MyShinyClass)
<class '__main__.MyShinyClass'>
In [28]: print(MyShinyClass()) # create an instance with the class
<__main__.MyShinyClass object at 0x10de1d450>
```
我们使用 "MyShinyClass" 作为类名和保存类引用的变量.

*type* 接收 dict 定义类的变量：

```python
In [29]: class Foo(object):
   ....:     bar = True
```

上面的代码也可以这样写：
```python
In [30]: Foo = type('Foo', (), {'bar': True})
```

上面创建的类可以像一个正常的类一样使用：
```python
>>> print(Foo)
<class '__main__.Foo'>
>>> print(Foo.bar)
True
>>> f = Foo()
>>> print(f)
<__main__.Foo object at 0x8a9b84c>
>>> print(f.bar)
True
```

我们也可以继承上面创建的类：
```python
>>>   class FooChild(Foo):
...         pass
>>> FooChild = type('FooChild', (Foo,), {})
>>> print(FooChild)
<class '__main__.FooChild'>
>>> print(FooChild.bar) # bar is inherited from Foo
True
```

最后我们可能想给我们的类添加新的方法.我们只需要使用适当的名称定义函数并且将这个函数作为变量赋给类就可以了

```python
>>> def echo_bar(self):
...       print(self.bar)
...
>>> FooChild = type('FooChild', (Foo,), {'echo_bar': echo_bar})
>>> hasattr(Foo, 'echo_bar')
False
>>> hasattr(FooChild, 'echo_bar')
True
>>> my_foo = FooChild()
>>> my_foo.echo_bar()
True
```

我们设计可以像给正常创建的的类对象添加方法一样，给动态创建类添加更多的方法：

```
>>> def echo_bar_more(self):
...       print('yet another method')
...
>>> FooChild.echo_bar_more = echo_bar_more
>>> hasattr(FooChild, 'echo_bar_more')
True
```

在 Python 中 类就是对象，我们可以随手动态创建一个类.
这就是 Python 在你使用 *class* 关键字的时候做的事情，但是 Python 是用 metaclass 来做这件事情的

## 3. 什么是 MetaClass （元类）

MetaClasses 是创建类的 `原料`.

我们为了创建对象而定义一个类，但是我们知道 Python 的 Class 也是对象.

MetaClasses 就是用来创建这样对象的！ 他们是类的类. 我们可以这样描述:

```
MyClass = MetaClass()
MyObject = MyClass()
```

使用type:

```
MyClass = type('MyClass', (), {})
```

因为*type* 事实上就是一个 metaclass, *type* 是Python 用来创建所有类的 metaclass！
Now you wonder why the heck is it written in lowercase, and not Type?

为什么我们把 *type* 写成小写的type 而不是大写的呢？
我们可以类比这些：
str 类创建字符串对象
int 类创建整型对象
type 类创建类对象

可以使用 __class__ 属性来检查

Python 中一切皆是对象. 包括: ints string function 和 class. 所有的都是对象。所有的对象都是从类中创建的：

```python
>>> age = 35
>>> age.__class__
<type 'int'>
>>> name = 'bob'
>>> name.__class__
<type 'str'>
>>> def foo(): pass
>>> foo.__class__
<type 'function'>
>>> class Bar(object): pass
>>> b = Bar()
>>> b.__class__
<class '__main__.Bar'>
```

__class__ 的 __class__ 是什么呢？

```python
>>> age.__class__.__class__
<type 'type'>
>>> name.__class__.__class__
<type 'type'>
>>> foo.__class__.__class__
<type 'type'>
>>> b.__class__.__class__
<type 'type'>
```

metaclass 是创建类对象的原料. 你也可以把他叫成是类工厂.

*type* 是 Python 内置的 metaclass, 我们也可以使用自己的 metaclass 来创建类.


## 4. __metaclass__ 属性

在写一个类的时候我们可以给一个类创建一个 __metaclass__ 属性:

```python
class Foo(object):
    __metaclass__ = something ...
    ...
```

如果我们这样写，Python 就会使用 metaclass 来创建 Foo 类.

如果这样写，小心些。我们开始写了 class Foo(object), 但是 Foo 对象还没有在内存中创建。

Python 会在类定义中查看 __metaclass__, 如果发现了它，就会使用这个 metaclass 来创建 Foo class 对象. 如果在类定义中没有发现 metaclass，就会使用 type 来创建 class.

当我们使用

```python
class Foo(Bar):
    pass
```
Python 做了这些事情：

查看在 Foo 类中有没有 __metaclass__ 属性?

如果有，就会在在内存中创建一个类对象，使用 Foo 名字并且使用 metaclass.

如果 Python 没有发现 __metaclass__, 就会继续在当前模块中找有没有 __metaclass__, 如果找到了就会做相同的事情。（这种情况只会发生在 old-style class 中，就是那种不继承任何类的类）

如果没有发现任何 __metaclass__, 就会使用 Bar 的 __metaclass__ （有可能是type）来创建类对象.

Be careful here that the __metaclass__ attribute will not be inherited, the metaclass of the parent (Bar.__class__) will be. If Bar used a __metaclass__ attribute that created Bar with type() (and not type.__new__()), the subclasses will not inherit that behavior.
注意：__metaclass__ 属性不会被继承,



