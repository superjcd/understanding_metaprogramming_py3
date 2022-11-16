# 理解Python3中的元编程

## type和class 
一种最为常见的定义python类的方式， 为:
```python
In [1]: class Foo:
   ...:     pass
   ...:

In [2]: foo = Foo()

In [3]: foo.__class__
Out[3]: __main__.Foo

In [4]: type(foo)
Out[4]: __main__.Foo
```
上面， 用传统的方式定义了一个类`Foo`， 然后分别使用`__class__`属性和`type`方法获得了实例`foo`的类, 然后发现这两者是等价的。

然后， 我们换一种方式定义一个类
```
In [5]: Foo2 = type('Foo', (), {})

In [6]: foo2 = Foo2()

In [7]: type(foo2)
Out[7]: __main__.Foo
```
可以看到通过`type`定义的一个`Foo`的类和之前传统范式定义的效果是一样的。
那么， type又是啥？

```python
In [8]: type(type)
Out[8]: type
```
> 所有的类的类都是type, type的类是自身

然后我们使用type定义类的时候,它需要接收的参数分别是：
- 类名, 比如上面的A
- 父类, 一个元祖， Foo2没有父类, 所以是空的元祖
- 一个属性字典， Foo2没有其他属性， 所以也是空的字典

## 一个例子
```python
In [10]: Bar = type('Bar', (Foo,), dict(value=1))

In [11]: b = Bar()

In [12]: b.value
Out[12]: 1

In [13]: issubclass(Bar, Foo)
Out[13]: True
```
上面用了type定义了一个全新的类`Bar`， 它继承自类Foo; 然后也被赋予了属性`value`(通常我们是通过__init__方法初始化的)， 当然我们除了一般的属性之外， 也会想要给属性再加上函数， 那么怎么在type中实现呢？

```python
In [14]: Foo3 = type('Foo', (),{ 'value': 1, 'get_value': lambda x: x.value})

In [15]: foo3 = Foo3()

In [16]: foo3.get_value()
Out[16]: 1
```
很酷对吧？
## 自定义元类
先回到最开始的地方：
```python
class Foo:
    pass
```

> 首先需要说明的是， 在我们定义类的时候， Foo的父类（这里是type）的__call__方法会被触发， 然后`__call__`会相继调用`__new__`(其实是生成一个类对象的过程， 注意python中一切皆对象， 类也是对象)和`__init__`方法

如果我想要`Foo`的实例带有属性`value`以及一个获取实例value的方法`get_value`, 一般的方法是在`__init__`里面定义`self.value`属性， 然后为这个类定义一个返回`self.value`属性， 然后我想把把
`get_value`这项能力抽离出来， 给其他的类使用， 那么是不是也需要在新类里面定义`value`属性和`get_value`方法呢(当然我们也可以通过继承， 这里先装作不知道)？  
既然type是一切类的元类， 那么我们只要在type的__new__()方法中定义相关的方法不就好了？
但是问题在于， type的__new__方法是不可以更改的， 所以我们需要定义一个可以更改__new__方法的类作为我们的元类

```python
In [27]: class Meta(type):
    ...:     def __new__(cls, name, bases, dct):
    ...:         x = super().__new__(cls, name, bases, dct)  # 其实就是调用了type(name, bases, dct)
    ...:         x.value=1
    ...:         x.get_value=lambda x : x.value
    ...:         return x
    ...: 
    ...: 

In [28]: class Foo(metaclass=Meta):
    ...:     pass
    ...: 

In [29]: f = Foo()

In [30]: f.value
Out[30]: 1

In [31]: f.get_value()
Out[31]: 1
```
如果别的类也想拥有`value`属性和`get_value`方法， 只要为其他类加上元类, 或者直接继承`Foo`就万事大吉了！

## 其他办法
事实上， 我完全可以通过定义一个一般的父类达到到同样效果。但是元类和一般的父类有一个非常明显的不同， 在于元类不会显式地出现在类的继承链条上
```python
In [32]: class Bar(Foo):
    ...:     pass
    ...: 

In [33]: b = Bar()

In [34]: b.value
Out[34]: 1

In [35]: b.get_value()
Out[35]: 1

In [36]: Bar.__mro__
Out[36]: (__main__.Bar, __main__.Foo, object)

In [37]: issubclass(Bar, Foo)
Out[37]: True

In [38]: issubclass(Bar, Meta)
Out[38]: False
```
这里我们让`Bar`继承自`Foo`，我们在`Foo`中定义了元类`Meta`， 但是元类并没有出现在`Bar`的继承链中(也不在Foo的继承链)， 这样我们只想为对象赋予方法和属性， 又不想弄脏对象的继承链， 那么通过元类是一个不错的方法；当然元类也不是唯一方法， 类装饰器可以得到类似的效果。

## 参考
[Python Metaclasses](https://realpython.com/python-metaclasses/)