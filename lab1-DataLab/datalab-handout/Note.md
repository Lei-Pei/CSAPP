# Lab1 DataLab笔记

该实验主要涉及整数类型的二进制表示和补码表示形式、IEEE规定的浮点数二进制表示形式，通过与、或、非以及移位等运算实现规定的函数功能。同时在实现函数的过程中限制了操作符的种类以及数量，使得实验难度较大。

## 写在前面

1. 本实验中`Makefile`有这么一条语句：

~~~
CFLAGS = -O -Wall -m32
~~~

表明该实验是在32位环境下进行的，但经过多次尝试后（谷歌，百度。。。）无果，本人将其改成了64位环境（由于int和float在32位和64位环境下位数和表示方式相同，所以对实验结果并没有影响）

2. 函数中的注释部分简单地指明了本人在实现该函数中的思路过程，开始时总是会本能地使用一些规定外的操作符实现函数功能，但是在执行`./dlc`指令后发现有些操作符并不能使用。。。。

3. 通过阅读README文件可以得知，对于实验结果的正确性测试指令主要有以下几条：

~~~
unix> ./dlc bits.c // 测试实验代码是否符合规定要求
unix> ./dlc -e bits.c // 查看每个函数实现运用到的操作符的个数
unix> make btest // 编译并运行btest文件

# 注意执行下面两条指令前需要先更新btest文件
unix> ./btest // 查看实验结果得分
unix> ./btest -f 函数名 // 测试单个函数的正确性
~~~

## 实验部分

### 1. bitXor

该函数的功能就是实现参数x与y的异或运算，但规定在函数实现过程中只能用按位非(~)和按位与(&)运算。

通过离散数学知识可知推到公式如下：

**a ^ b = (a & ~b) | (~a & b)** 

​		 **= ((a & ~b) | ~a) & ((a & ~b) | b)** 

​		 **= (~b | ~a) & (a | b)**

​		 **= (~(a & b)) & (\~(~a & ~b))**

从而得出了我们需要的结果如下：

~~~c
int bitXor(int x, int y) {
    /* x^y = (x & ~y) | (~x & y) */
    return (~(x & y)) & (~(~x & ~y));
}
~~~

### 2. tmin

该函数较为简单，就是要得到整数在二进制补码表示下的最小值(1000...0000，共31个0)，代码实现如下：

~~~c
int tmin(void) {
    return 1 << 31;
}
~~~

### 3. isTmax

该函数主要功能是判断参数x是否是二进制补码表示下的最大整数(0111...1111，共31个1)

> 首先我的想法是通过`(x << 1) >> 1 == ~0`来判断x是否为Tmax，但是后面发现移位运算不能使用（后面我们会介绍`==`可以通过异或运算实现）

转化思路发现`~0111...1111 = 1000...000`，同时`0111...1111 + 1 = 1000...0000`，从而想到利用`~x == x + 1`来判断x是否为Tmax，但是经测试后发现`1111...1111`也满足该式子，因此在实现过程中需要排除`1111...1111`的情况。

> **通过异或运算实现==：**
>
> > `x == y <==> x ^ y == 0`，从而当`!(x ^ y)`为1时，x与y相等，`!(x ^ y)`为0时，x与y不等。

最终得到代码如下：

~~~c
int isTmax(int x) {
    // return (x << 1) >> 1 == ~0;
    return !((x + 1) ^ (~x)) & !!(x + 1);
}
~~~

### 4. allOddBits

该函数功能是判断x的二进制表示下所有奇数位的值是否都为1。

不难想到，只需要将x与`0101...0101`进行按位与操作，如果得到的结果为`0101...0101`则该函数返回1，否则返回0，在十六进制表示下有如下表示形式：

~~~
x & 0xAAAAAAAA == 0xAAAAAAAA
~~~

但是经测试发现只允许使用0到255(0xFF)的常量值，因此我们需要用下面的方式得到`0xAAAAAAAA`：

~~~
(0xAA << 24) + (0xAA << 16) + (0xAA << 8) + 0xAA
~~~

最终得到代码如下：

~~~c
int allOddBits(int x) {
    // return x & 0xAAAAAAAA == 0xAAAAAAAA;
    // return !((x & 0xAAAAAAAA) ^ 0xAAAAAAAA);
    int allA = (0xAA << 24) + (0xAA << 16) + (0xAA << 8) + 0xAA;
    return !((x & allA) ^ allA);
}
~~~

### 5. negate

该函数功能为返回x的加法逆元-x。

通过`x + (~x) = (1111...1111) = -1` 以及 `x + (-x) = 0`可以得到`-x = (~x) + 1`，从而得到代码如下：

~~~c
int negate(int x) {
    return ~x + 1;
}
~~~

### 6. isAsciiDigit

该函数功能为判断x是否为Ascii码下的数字(0x30到0x39之间)

首先，如果x为Ascii码下的数字，需要满足：

+ **x的后四位取值范围为[0, 9]**。先通过`x & 0xF`得到x的后四位的值，然后通过`((x & 0xF) + 0x6) >> 4 == 0`判断是否在[0, 9]中。
+ **x的倒数第5位到倒数第8位表示的值为`0011`，即3**。可以通过`x >> 4 == 0x3`来判断这一结果。

最终得到代码如下：

~~~c
int isAsciiDigit(int x) {
    // (x >> 4 == 0x3) && (((x & 0xF) + 0x6) >> 4 == 0)
    int fifthToEighthToLastBits = !((x >> 4) ^ 0x3);
    int firstToFourthToLastBits = !(((x & 0xF) + 0x6) >> 4);
    return fifthToEighthToLastBits & firstToFourthToLastBits;
}
~~~

### 7. conditional

该函数功能为利用按位与、按位或、非等操作符实现三元运算符(x ? y : z)

首先通过三元运算符的定义`x != 0 return y else return z`不难联想到式子`x & y + ~x & z`，但是该式子需要满足在`x != 0`的情况下`x = 1111...1111`，因此我们需要想办法在x不为0的情况下，将x转化为`1111...1111`，同时`x = 0`时保持x的值不变。

+ 首先利用非(!)运算统一x不为0的情况。此时有`!!x = 1`，然后通过`~x + 1`得到`1111...1111`
+ 当x = 0时，此时`!!x = 0`，通过`~x + 1`后仍然为0

因此得到最终代码如下：

~~~c
int conditional(int x, int y, int z) {
    // change x to 0xFFFFFFFF if x != 0 
    int zeroOrOne = !!x; // 0 if x == 0, 1 otherwise
    int zeroOrMinusOne = ~zeroOrOne + 1; // 0 if x == 0, -1(0xFFFFFFFF) otherwise
    x = zeroOrMinusOne;
    return (x & y) + (~x & z);
}
~~~

### 8. isLessOrEqual

该函数功能为判断`x <= y`是否成立。

在将`x <= y`转化成`x - y <= 0`之前需要考虑溢出(overflow)的情况：

+ `x < 0`，`y > 0`，`x - y > 0`，出现负溢出(negative overflow)。该情况下`x - y > 0`但是满足`x <= y`。
+ `x > 0`，`y < 0`， `x - y < 0`，出现正溢出(positive overflow)。该情况下`x - y < 0`但是不满足`x <= y`

排除上面两种情况，可以通过`x - y = x + (-y) = x + (~y + 1) <= 0`来判断`x <= y`是否成立，最终代码如下：

~~~c
int isLessOrEqual(int x, int y) {
    // x < 0 && y >= 0 || ((!(x >= 0 && y < 0)) && (x - y = x + (~y + 1) < 0 || x == y))
    int xGE0 = !(x >> 31); // x >= 0
    int xLT0 = !xGE0; // x < 0
    int yGE0 = !(y >> 31); // y >= 0
    int yLT0 = !yGE0; // y < 0
    int xMinusyLT0 = !!((x + (~y + 1)) >> 31); // x - y < 0
    int isEqual = !(x ^ y); // x == y
    return (xLT0 & yGE0) | ((!(xGE0 & yLT0)) & (xMinusyLT0 | isEqual));
}
~~~

### 9. logicalNeg

该函数功能为用其他的运算符实现非(!)的功能。

同问题7，这里我们需要想办法将x不为0的情况统一起来处理，但是在问题7中我们用到了!，本问题中我们需要想其他的办法。

观察0和非0的二进制表示后可以发现：**当x不为0时，x与-x至少有一个为负，其符号位为1；当x为0时，x与-x均为0，符号位均为0**。从而有：

+ x = 0时，`(x | (-x)) >> 31 + 1 = (x | (~x + 1)) >> 31 + 1 = 0 + 1 = 1`
+ x != 0时， `(x | (-x)) >> 31 + 1 = (x | (~x + 1)) >> 31 + 1 = 1111...1111 + 1 = -1 + 1 = 0 `

最终代码如下：

~~~c
int logicalNeg(int x) {
    // the sign bit of (x | ~x + 1) is 0 if and only if x == 0
    int minusOneOrZero = (x | (~x + 1)) >> 31; // 0 if x == 0, -1 otherwise
    return minusOneOrZero + 1;
}
~~~

### 10. howManyBits

此函数实现较为困难，本人不知该如何讲解思路过程，在这里仅附上代码：

~~~c
int howManyBits(int x) {
    /* howManyBits(-5) = 4 = howManyBits(4)
     * howManyBits(-4) = 3 = howManyBits(3)
     * howManyBits(-3) = 3 = howManyBits(2)
     * howManyBits(-2) = 2 = howManyBits(1)
     * howManyBits(-1) = 1 = howManyBits(0)
     * howManyBits(0) = 1
     * howManyBits(1) = 2
     * howManyBits(2) = 3
     * howManyBits(3) = 3
     * howManyBits(4) = 4
     *
     * if x < 0, howManyBits(x) = howManyBits(-x - 1) = howManyBits(~x)
     */
    int sign = x >> 31;
    int pos16, pos8, pos4, pos2, pos1, pos0;
    x = x ^ sign; // flip each bit of x if x < 0
    pos16 = !!(x >> 16) << 4; // pos16 = 16 if (x >> 16) != 0
    x >>= pos16;
    pos8 = !!(x >> 8) << 3; // pos8 = 8 if (x >> 8) != 0
    x >>= pos8;
    pos4 = !!(x >> 4) << 2; // pos4 = 4 if (x >> 4) != 0
    x >>= pos4;
    pos2 = !!(x >> 2) << 1; // pos2 = 2 if (x >> 2) != 0
    x >>= pos2;
    pos1 = !!(x >> 1); // pos1 = 1 if (x >> 1) != 0
    x >>= pos1;
    pos0 = x;
    return pos16 + pos8 + pos4 + pos2 + pos1 + pos0 + 1;
}
~~~

### 11 - 13. floatScale2, floatFloat2Int, floatPower2

这三个函数实现较为简单，只需要熟悉单精度浮点数float的表示方式，注意一些琐碎的细节即可，没有任何的逻辑思维考核，因此这里不进行详细地讲解。代码如下：

~~~c
unsigned floatScale2(unsigned uf) {
    unsigned sign = (uf >> 31) & 0x1;
    unsigned exp = (uf >> 23) & 0xFF;
    unsigned frac = uf & 0x7FFFFF;
    if (exp == 0xFF)  return uf; 
    else if (exp == 0) {
        frac <<= 1;
        // return (sign << 31) + (exp << 23) + frac;
        return (sign << 31) | (exp << 23) | frac;
    } 
    else {
        exp += 1;
        return (sign << 31) | (exp << 23) | frac;
    }
}
~~~

~~~c
int floatFloat2Int(unsigned uf) {
    unsigned sign = (uf >> 31) & 0x1;
    unsigned exp = (uf >> 23) & 0xFF;
    unsigned frac = uf & 0x7FFFFF;
    int E = exp - 127;
    if (E < 0)  return 0;
    if (exp == 0xFF || E >= 31)  return 0x80000000u;
    else {
        frac = frac | (1 << 23);
    	if (E <= 23)  frac >>= (23 - E);
    	else  frac <<= (E - 23);
    }
    
    if (sign) return -frac;
    else  return frac;
}
~~~

~~~c
unsigned floatPower2(int x) {
    if (x > 127)  return 0xFF << 23; // largest normalized < 2 * 2 ^ 127 = 2 ^ 128
    else if (x < -149)  return 0; // smallest denormalized = 2 ^ (1 - 127) + 2 ^ (-23) = 2 ^ (-149)
    else if (x >= -126)  return (x + 127) << 23; // smallest normalized = 2 ^ (-126)
    else return 1 << (x + 126 + 23);
}
~~~

