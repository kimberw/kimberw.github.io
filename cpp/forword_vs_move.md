# std::forward VS std::move

## 简述

1. 问题: 临时变量copy开销太大
2. 引入: rvalue, lvalue, rvalue reference概念
3. 方法: rvalue reference传临时变量, move语义避免copy
4. 优化: forward同时能处理rvalue/lvalue reference和const reference

## 详细

forward 和 move 需要解决的问题只有一个：避免临时变量的copy

move 解决 T && （rvalue reference）右值引用参数，避免copy

但是此时需要对函数调用进行overload，区分 T&(左值) 和 T&&（右值）。

为了避免重复代码，引入了 forward。

对于 forward 如果外面传来了rvalue临时变量, 它就转发rvalue并且启用move语义. 如果外面传来了lvalue, 它就转发lvalue并且启用复制. 然后它也还能保留const. 这样就能完美转发(perfect forwarding)所有情况了.

从技术角度，forward可以完美替代 move，为什么还要保留move呢？

- forward常用于template函数中, 使用的时候必须要多带一个template参数T: forward<T>, 代码略复杂;
- 明确只需要move的情况而用forward, 代码意图不清晰, 其他人看着理解起来比较费劲.
- 更技术上来说, 他们都可以被static_cast替代. 为什么不用static_cast呢? 也就是为了读着方便易懂.

> https://zhuanlan.zhihu.com/p/55856487