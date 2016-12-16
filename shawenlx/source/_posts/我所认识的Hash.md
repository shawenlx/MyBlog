---
title: 我所认识的Hash
date: 2016-12-14 10:59:59
tags: c / c++
categories: algorithm
---
## 我所认识的Hash

------

### 前言

​   关于Hash表，是我们经常会碰到的数据结构。大多数时候，它能高效地解决我们的一些实际问题。当然，多数情况下时间和空间正如鱼和熊掌，不可兼得。只有在特定的情况下，我们去选择适合当前需求的数据结构和算法。

​   Hash表本质是可以理解为键值对的集合。key-value一一对应。

------
<!--more-->

### 字符Hash的简单应用

- 先来看这样一个简单的例子，有一个字符串，仅有大写字母’A'~'Z'组成，我们需要对这个字符串中所有的字母进行统计。然后输出对应的表，如 ACAADEB, 则A->3, B->1, C->1, D->1, E->1。
- 那么如何来做这个统计呢，这时候Hash表就可以派上用场了。字符‘A'对应的ASCII码为65，那么根据Hash的思想，我们可以把一个数组中的下标映射为字符，将下标对应的值中存储我们的统计结果，比如Hash['A'] = 3;

```c++
int hash[26];
string str = "ACAADEB";
void count_char_on_string(string &str) {
    for (int i = 0; i < str.length(); ++i) {
        ++hash[str[i] - 'A'];
    }
}
```

通过以上这个代码，我们就能将字符串中的字符一一统计下来。贯彻的思想就是**将字符映射成数组下标**。

------

### 字符串Hash

- 既然能够将字符映射成整数，那么肯定有办法将字符串也映射成整数。
- 关于字符串hash算法，有很多种，可以通过这篇博客了解，不同的字符串hash算法的对比。[各种字符串Hash函数比较](https://www.byvoid.com/blog/string-hash-compare)
- 此处，我仅仅以BKDRHash算法作为例子，核心的思想是通过一个选择器，我们也成为种子(seed)，通常选择种子应该是一个质数，如31， 131，1313等，这样的好处在于减少不同字符串映射为相同整数的冲突。通过一个计算公式，将不同的字符串进行转化为数字。

```c++
// BKDR Hash Function
unsigned int BKDRHash(char *str) {
    unsigned int seed = 131; // 31 131 1313 13131 131313 etc..
    unsigned int hash = 0;
    while (*str) {
        //每转化一位字符，用当前的hash值 * seed，在加上字符的ASCII码。
        //hash * seed, 此处我们可以理解为当前字符串在前一个字符串hash值的基础上，偏移了一个种子的数级距离。
        hash = hash * seed + (*str++);
    }
    return (hash & 0x7FFFFFFF); //最后用hash & Ox7FFFFFFF 确保hash值在unsigned int 范围中。
}
```

利用字符串Hash可以解决很多问题，可以做字符串匹配。效率也蛮高的，相比KMP算法或者后缀数组和AC自动机等数据结构，它也不失为一种巧妙的办法。**我们可以通过将每一位计算得到的hash偏移值存储在一个数组里，然后通过计算偏移得到子串的hash值，在和匹配串的hash值进行对比，如果相同，则说明匹配成功。**

```C++
typedef unsigned long long ull;
const int MOD = 0x7FFFFFFF;
ull hash[1000]; //假设字符串长度小于1000
ull char_hash[1000]; //存储对应字符的hash值
inline ull cal_hash(string &str, const int l, const int r) {
    if (l == 0) return cal_hash[r];
    ull seed = 131;
    hash[0] = 1;
    for (int i = 1; i < str.length(); ++i) {
      hash[i] = hash[i-1] * seed;	//计算偏移
    }
    //用当前r对应的char_hash值-减去当前子串前一部分多计算的偏移hash值。就能得到子串的hash值了。
    return ((char_hash[r] - char_hash[l-1] * hash[r-l+1]) % MOD + MOD) %MOD;
}
```



------

### 解决字符串Hash冲突

- 或多或少，BKDRHash算法在极小的几率下会出现Hash冲突，解决的办法有很多，多数时候，我们采用邻接表来解决这个问题。
- 由于算法涉及&操作，会有导致冲突的可能。在多串匹配的情况下，我们需要将所有匹配字符串计算得到的hash值，存放到邻接表中，所谓的邻接表就是数组套链表。将计算到的匹配串hash值作为数组中的值，并加入到对应的链表中，如果我们计算到的hash值出现冲突的情况，就往对应链表中加入新的节点。在后续匹配的情况下，进行查找链表并做多一次简单的匹配即可。

------

### 关于Hash的一个有趣应用

- 有这么一个4 * 4的棋盘，棋子有两种状态，黑色或白色。我们这个时候也可以通过Hash来记录当前棋盘的信息。

那么对应的棋盘，我们用1表示黑色，用0表示白色。那么就有：

![img](https://odzkskevi.qnssl.com/14b4b3ec0b5261bea3a5ad9f1313252c)

这个图转化成矩阵就是：

​                               1 0 1 0

​                               0 0 0 0

​                               1 1 0 1

​                               1 0 0 1

那么如何转化成整数来记录棋盘呢？

​                       2^0 2^1 2^2 2^3

​                       2^4 2^5 2^6 2^7

​                       2^8 2^9 2^10  2^11

​                       2^12  2^13  2^14  2^15

最后得到的棋盘我们可以通过判断当前棋子颜色，如果是黑色，则在对应位n上加上对应的2^n, n可以通过行列式计算出来。

```C++
int state[4][4]; //预处理记录棋子状态
unsigned int hash = 0;
for (int i = 0; i < 16; ++i) {
    hash += state[i/4][i%4] * (1 << i);
}
```

这样我们就把棋盘信息转化成一个整数了。

------

### 最后

​   关于Hash表仍有很多的应用，还有一种应用是搜索领域，如何将互联网上大量的字符串存储，这里推荐大家了解一下字典树，它也是基于Hash算法的思想的数据结构。在iOS中，NSDictionary就是基于Hash的实现。以后有机会会深入CF框架看看实现源码，到时候再来分享。