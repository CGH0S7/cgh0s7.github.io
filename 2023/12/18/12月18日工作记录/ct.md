---
title: GCC15 函数指针类型检查可能导致编译错误
date: 2025-05-23 21:49:02
tags: 技术
---

今天准备继续做一下OS课的实验，实验代码当然是基于著名的xv6项目，意外地发现原本明明正常运行的系统再次构建时却失败了，具体报错如下：

```GCC
user/usertests.c:2598:4: error: initialization of 'void (*)(char *)' from incompatible pointer type 'void (*)(void)' [-Wincompatible-pointer-types]
 2598 |   {rwsbrk, "rwsbrk" },
      |    ^~~~~~
user/usertests.c:2598:4: note: (near initialization for 'quicktests[5].f')
user/usertests.c:247:1: note: 'rwsbrk' declared here
  247 | rwsbrk()
      | ^~~~~~
make: *** [<内置>：user/usertests.o] 错误 1

```
而具体定位到的代码则是：
```C
struct test {
  void (*f)(char *);
  char *s;
} quicktests[] = {
  {copyin, "copyin"},
  {copyout, "copyout"},
  {copyinstr1, "copyinstr1"},
  {copyinstr2, "copyinstr2"},
  {copyinstr3, "copyinstr3"},
  {rwsbrk, "rwsbrk" },
  ...
  { 0, 0},
};
// See if the kernel refuses to read/write user memory that the
// application doesn't have anymore, because it returned it.
void
rwsbrk()
{
  int fd, n;
  
  uint64 a = (uint64) sbrk(8192);

  if(a == 0xffffffffffffffffLL) {
    printf("sbrk(rwsbrk) failed\n");
    exit(1);
  }
  ...
  exit(0);
}
```

这段代码定义了一个函数指针数组，其中每个元素都指向一个函数和它的名称。结构体期望的函数签名是 `void(*)(char*)`，但传入的是一个没有参数的函数 `void(*)(void)`，这会导致函数指针类型不匹配。

但是之前明明是没有问题的怎么现在突然出错呢？仔细一想立刻就怀疑是之前arch更新的时候把gcc升上去导致的，结果查了下大概确实是这样。

- 在 **GCC 14 及之前版本**，这类赋值虽然不符合 C 标准，但 GCC 作为扩展允许这种行为。
- 从 **GCC 15 开始**，这类隐式转换默认触发编译错误或警告（取决于编译器配置），因为 GCC 变得更加严格地遵循标准 C 的函数指针类型规则。

这并不是 xv6 本身的 bug，而是 GCC 编译器对语言规范支持的更新带来的兼容性问题。解决方法倒是简单，直接给`rwsbrk`函数添加参数改为`void rwsbrk(char* arg)`即可解决，强制类型转换也是一种办法。这次事件再次告诉我们什么都升到最新并不是什么好事情，arch用户总是要花时间处理一些莫名其妙的兼容性问题:-)


