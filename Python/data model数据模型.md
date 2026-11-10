## 以后怎么读一个新对象

python

```
print([n for n in dir(obj) if n.startswith('__') and n.endswith('__')])
```

然后查这张表：

|看到|意味着|
|---|---|
|`__init__`|能被调用造对象|
|`__iter__` `__next__`|**能被 `for`**|
|`__len__`|能被 `len()`|
|`__getitem__`|**能下标 `[ ]`**|
|`__contains__`|能 `in`|
|`__enter__` `__exit__`|**能 `with`**|
|`__str__` `__repr__`|能 `print()`|
|`__eq__` `__lt__` `__gt__`|能比较、能排序|
|`__add__` `__mul__`|能 `+` `*`|

**终极参考**（官方原文，收藏它）：[Python 语言参考 · 数据模型](https://docs.python.org/3/reference/datamodel.html)

python

```python
class B:        # ← B 是"类"（Class）：设计图纸
    def f(self, x):
        return x * 2

b = B()         # ← b 是"实例"（object）：拿图纸造出来的那一件东西
```

那行等式现在读起来就顺了：

python

```
b.f(5)   ≡   B.f(b, 5)
  ↑            ↑  ↑
实例调 f     类B的f 作用在实例b上
```

我完全可以写成：

python

```python
class Cat:
    ...
my_cat = Cat()
my_cat.f(5)  ≡  Cat.f(my_cat, 5)     # 一模一样的道理
```