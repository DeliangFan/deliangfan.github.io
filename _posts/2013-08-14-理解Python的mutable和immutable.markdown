---
layout: post
title:  "理解 Python object 的 mutable 和 immutable"
categories: Python
---

![object](http://7xp2eu.com1.z0.glb.clouddn.com/pythonobjectmutable.png)

在 Python 的世界里，一切皆对象，每个对象包含各包含一个 idendity、type 和 value。

- identity: identity 可理解为 object 的内存地址空间，其值可用 id() 函数获取。一旦 object 被创建，其 identity 不可变。
- type: type 可理解为 object 的类名，其值可用 type() 函数获取。一旦 object 被创建，其 type 也不可变。
- value: value 可理解为 object 的值，和 identity 和 type 不同，有些 object 的 value 可变，有些 object 的 value 永不可变。我们把 value 可变动的 object 成为 mutable object，把 value 不可变动的 object 称之为 immutable。

常见的 immutable objects:

- Numeric types: int, float, complex
- string
- tuple
- frozen set

常见的 mutable objects:

- list
- dict
- set
- byte array

# Case One

~~~ bash
>>> a = [1, 2, 3]
>>> id(a)
4317018016
>>> a[1] = 10
>>> id(a)
4317018016
>>> a
[1, 10, 3]
~~~

不难理解，列表的值在 a[1] = 10 执行后被改变，但是其地址空间没有改变。

# Case Two

~~~ bash
>>> a = 1
>>> id(a)
140207626202088
>>> a = 2
>>> id(a)
140207626202064
~~~

和 C 语言不同，Python 的运算符 = 是引用，而非赋值运算，所以 a = 2 的含义是创建一个值为 2 的 int 对象，再将 a 指向该对象的地址，所以上例的 1 和 2 这两个 int 对象的值并没有改变。

# Case Three

~~~ bash
>>> a = (1, 2, [3])
>>> id(a)
4317137728
>>> a[0] = 1
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
>>>
>>> a[2][0] = 100
>>> id(a)
4317137728
>>> a
(1, 2, [100])
~~~

从上例可看出，immutable 不能理解为绝对的不可变，对于某些 container 类型的 object，如果它包含了一个 mutable 的 object，那么 mutable object 的值是可变的。因为 tuple 所含 list 改变的只是 value 而非 identity，所以 tuple 依旧被称为 immutable object。

~~~ bash
>>> a = (1, 2, [3])
>>> id(a[2])
4317018016
>>> a[2][0] = 100
>>> id(a[2])
4317018016
~~~
