---
layout: post
title:  "理解浮点数"
categories: Algorithm
---

# Example

首先从两个例子开始。

- 例 1，以下输出是？

~~~
#include<stdio.h>

void main(){
    float p = 1000000.9;

    printf("%f", p);
}
~~~

结果：

~~~
1000000.875000
~~~


- 例 2，以下输出是？

~~~
#include<stdio.h>

void main(){
    int i = 1;
    float* p = &i;

    printf("%f", *p);
}
~~~

结果：

~~~
0.000000
~~~

# 理解浮点数

从例 1 可知，[浮点数](https://en.wikipedia.org/wiki/IEEE_floating_point)表示的精度有限；由例 2 可知，同样为 4 Byte，float 和 int 的组织结构不一样。那么浮点数是如何组织的呢？首先回忆[科学计数法](https://zh.wikipedia.org/wiki/%E7%A7%91%E5%AD%A6%E8%AE%B0%E6%95%B0%E6%B3%95)。在科学记数法中，一个有限的实数可用如下表示：

~~~
a * 10 ^ n

其中：
1 <= |a| < 10
n 为整数
~~~

和科学计数法类似，以 4 Byte 的 float 为例，[IEEE 754](https://en.wikipedia.org/wiki/IEEE_754-1985) 的定义如下：

~~~

|s|    q(8 bit)   |                  c(23 bit)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

~~~
(−1)^s × c × 2^q
~~~