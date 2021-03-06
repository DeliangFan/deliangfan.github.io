---
layout: post
title:  "理解浮点数"
categories: Algorithm
---

# Example

首先从两个例子开始。

- 例 1，下例输出是？

~~~ c
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


- 例 2，下例输出是？

~~~ c
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

从例 1 可知，[浮点数](https://en.wikipedia.org/wiki/IEEE_floating_point)表示的精度有限；由例 2 可知，同样为 4 Byte，float 和 int 的数据组织不一样。那么浮点数是如何组织的呢？首先回忆[科学计数法](https://zh.wikipedia.org/wiki/%E7%A7%91%E5%AD%A6%E8%AE%B0%E6%95%B0%E6%B3%95)。在科学记数法中，一个有限的实数可用如下表示：

~~~
a * 10^n

其中：
1 <= |a| < 10
n 为整数
~~~

以上式子亦可表达为如下：

~~~
(-1)^s * a * 10^n

其中：
s = 0 或 1，s 为 0 表示正数，为 1 表示负数
1 <= a < 10
n 为整数
~~~

计算机采用二进制，科学计数法的二进制表达如下：

~~~
(−1)^s × a × 2^n

其中
s = 0 或 1，s 为 0 表示正数，为 1 表示负数
1 <= a < 2
n 为整数
~~~

以 4 Byte 的 float 为例，[IEEE 754](https://en.wikipedia.org/wiki/IEEE_754-1985) 对其的定义如下：

~~~
|s|exponent(8 bit)|            fraction (23 bit)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

- s 表示符号位，共 1 bit，0 表示正数，1 表示负数
- e(exponent) 共 8 bit，它决定了 n 的值，即浮点数能表达的范围
- f(fraction) 共 23 bit，它决定了 a 的值，即浮点数的精度

e 和 n 的关系如下：

~~~
n = e - 127

其中 0<= e <= 255
所以 -127 <= n <= 128。(当 n = 128 时，表示浮点数无穷大)
~~~

f 和 a 的关系如下：

~~~
a = f/(2^23) + 1

其中 0 <= f <= 8388607
所以 1 <= a <= 1.99999988
~~~

所以 float 类型的浮点数能表示的范围为：

~~~
± 1.99999988 * 2^-127 至 ± 1.99999988 * 2^127
即：± 1.18 × 10^-38 至 ± 3.4 × 10^38
~~~

float 类型的浮点数精度为：

~~~
1/(2^23) = 1.19 * 10^-7，十进制下为 7 位。
~~~

同理，double 类型的浮点数能表示的范围和最大精度为：

~~~
范围：± 2.23 × 10^-308 到 ± 1.80 × 10^308
精度：十进制下为 16 位
~~~

例 1 中，因为 1000000.9 长度为 8 位，超出 float 所能表示的最大精度，所以不能被精确表示。

例 2 中，对于 int i = 1，其在内存中的数据为：

~~~
|s|exponent(8 bit)|            fraction (23 bit)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|0|1|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

故有 s = 0, e = 0, f = 1，其值为 (-1)^0 * (1 + 2^-23) * 2^-127，即 5.88 * 10^-39，所以显示的结果为 0。
