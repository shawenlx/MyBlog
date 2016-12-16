---
title: 为什么我选择使用Blocks
date: 2016-06-08 21:00:00
tags: iOS
categories: iOS
---
- 扯淡：到了新公司接手新框架之后，发现大量的使用Blocks，之前很多时候都是使用代理，突然面对这个陌生的语法，特地科普总结了一番。

------

### 什么是Blocks

- 一句话概括就是，带有局部变量的匿名函数（即不带名称的函数）。也称为<strong>闭包</strong>。

### Blocks语法

- <strong> ^ </strong>返回值类型 参数列表 <strong>表达式</strong>

```objc
//例如
^int (int count) { return count + 1; }
```

- 上述表达式中，返回值类型 以及 参数列表可以省略(如下), 语法加粗表明不可缺少的部分。

```objc
^ {
      printf("Hello, blocks");
}
```

------
<!--more-->
### 将Blocks声明为属性

- 多数情况下，我们会把Blocks声明为属性，便于调用。可以利用关键字typedef 对Blocks进行别名。

  ```objc
  typedef 返回值类型 (^Blocks别名)(参数列表);
  typedef int (^blk_t)(int);
  ```

  ```objc
  //将Blocks声明为属性
  @property (nonatomic, copy) blk_t blk;
  ```

  ```objc
  //对Blocks进行声明定义
  blk = ^(int count) { return count + 1; };
  ```

  ```objc
  //调用Blocks 和 调用C语言声明函数方式一样
  NSLog(@"%d\n", blk(10));
  ```

------

### 将Blocks作为参数传入方法中

- 通常都会事先为Blocks取一个别名，一来方便理解，好吧不装逼，就是偷懒。

```objc
//定义：
typedef void(^Block)(int a);
- (void)nslogParameterWithBlock:(Block)block { 
    if (block) { 
        block(10); 
    }
}
```

```objc
//赋值并调用
[object nslogParameterWithBlock:^(int a) { 
    NSLog(@"%d", a);
}];
```

------

### Blocks截获自动变量值

- 什么是截获自动变量值，所谓自动变量，就是局部变量。那为什么会截获呢？先来看个例子。

```objc
int val = 10;
void  (^block)(void) = ^ { printf("val = %d", val); };
val = 20;
block();  //此处调用block，打印为val = 10；
```

- 之所以没有打印val = 20，是因为该Blocks在编译期就进行了初始化，对声明的val进行截获，也就是说，把val拷贝一份存入Blocks中。所以当外部val进行了修改时，不影响Blocks截获的val值。

------

### __block 关键字

- 从上个例子中，我们知道Blocks会对变量进行截获。那如果我非要在Blocks中修改外部参数怎么办呢？我们可以通过__block关键字做到！

```objc
__block int val = 10;
void  (^block)(void) = ^ { printf("val = %d", val); };
val = 20;
block();  //此处调用block，打印为val = 20;
```

- <strong>通过__block说明符对外部参数进行声明后，实际上，相当于在声明定义Blocks的时候，不是通过拷贝内存的方式拷贝外部参数，而是通过指针指向该外部变量，从而达到可以在Blocks中修改的效果。</strong>

------

### 总结：

- Blocks使用方便，但是block的内存管理确实是一项非常重要也非常容易出错的地方，稍不注意便会造成内存泄露，所以，在使用Blocks的时候，也要小心。
- 因为是匿名函数，所以在项目中多人合作的时候，最后写上每个Blocks的注释说明。
- 后续将继续总结Blocks的内存管理内容。