---
layout: post
title: c++smart pointer 分析
categories:
- c++11
tags: [c++11]
status: publish
type: post
published: true
meta:
  _edit_last: '1' 
  views: '2' 
  author:
  login: cothee
  email: cotheehi@gmail.com
  display_name: cothee
  first_name: ''
  last_name: ''
---

# c++smart pointer 分析: unique_ptr, shared_ptr & weak_ptr
## unqique_ptr
  提供对对象的单独唯一引用,可用构造函数创建(c++14支持std::make_unique, 在c++11中没有仅仅是因为忘了。。[4])
## shared_ptr
  提供对对象的共享引用，可用构造函数或std::make_shared(c++11)创建;
## weak_ptr
  首先，weak_ptr有几个特点：
  (1) 增加weak_ptr不会增加shared_ptr的引用计数，其use_count方法返回的是shared_ptr的引用计数；
  (2) 不能通过weak_ptr来使用实际的对象(除非借助lock方法来生成shared_ptr对象)，但可以借助它来确认指向的对象是否已经被delete掉了(expired方法)；
  (3) 从weak_ptr的constructor可以看出，weak_ptr不能脱离shared_ptr而单独存在，也就是说，weak_ptr必须从shared_ptr或其他weak_ptr生成。
  weak_ptr主要用于以下场景：
(1)  解决循环引用问题: 这个应该是最广泛使用的场景了吧，在循环引用条件下，必须使用weak_ptr。而且一般两个对象一个是owner,一个是owned,owner里使用shared_ptr,owned里使用weak_ptr[2];
(2)与shared_ptr结合使用，使用shared_ptr来管理数据，而使用weak_ptr来给数据的实际使用者，利用expired和lock方法，可以方便使用数据，并确定shared_ptr指向的实际数据是否已经被delete掉，当需要使用shared_ptr的时候，用lock方法返回一个shared_ptr[1,2]。“std::weak_ptr models temporary ownership: when an object needs to be accessed only if it exists, and it may be deleted at any time by someone else, std::weak_ptr is used to track the object, and it is converted to std::shared_ptr to assume temporary ownership. If the original std::shared_ptr is destroyed at this time, the object's lifetime is extended until the temporary std::shared_ptr is destroyed as well”[1]

## smart pointer 和 raw pointer的性能对比
参考博客[5]，先说下博客结论： There are only few reasons in modern C++ justifying the memory management with new and delete.
自己写了测试代码实际测了下，代码如下[6] (https://github.com/cothee/interesting_test)：
test.h ：
```cpp 
//test.h
#include <chrono>
#include <iostream>
 
#define TEST_NUM 1000000
 
#define TEST_FUN1(op1, fun_name) {int count = 0;                                              \
                                  auto start = std::chrono::high_resolution_clock::now();      \
                                  do {op1;} while (++count < TEST_NUM);                         \
                                  auto end =std::chrono::high_resolution_clock::now();           \
                                  std::chrono::duration<double,std::nano> elapsed = end - start;  \
                                  std::cout << fun_name << " use: " << elapsed.count() / TEST_NUM << "  nanoseconds" << std::endl;  \
                                 }
 
#define TEST_FUN2(op1, op2, fun_name) {int count = 0;                                              \
                                       auto start = std::chrono::high_resolution_clock::now();      \
                                       do {op1; op2;} while (++count < TEST_NUM);                    \
                                       auto end =std::chrono::high_resolution_clock::now();           \
                                       std::chrono::duration<double,std::nano> elapsed = end - start;  \
                                       std::cout << fun_name << " use: " << elapsed.count() / TEST_NUM << "  nanoseconds" << std::endl;  \

```
test.cc ：
```cpp
#include <iostream>
#include <memory>
#include "test.h"
 
struct Node {
  int a;
  int b;
  int c;
  Node(int m, int n, int p) {
    a = m;
    b = n;
    c = p;
  }
};
 
void test_shared() {
  TEST_FUN1(std::shared_ptr<Node> ptr(new Node(1,2,3)), "new shared_ptr");
}
 
void test_unique() {
  TEST_FUN1(std::unique_ptr<Node> ptr(new Node(1,2,3)), "new unique_ptr");
}
 
void test_raw() {
    TEST_FUN2(Node* n = new Node(1,2,3), delete n, "new raw_pointer");
}
 
void test_malloc() {
    Node* p;
    TEST_FUN2(p = (Node*)malloc(sizeof(Node)), free(p), "malloc & free");
}
 
void copy_shared() {
  std::shared_ptr<Node> ptr_ori;
  TEST_FUN1(auto ptr = ptr_ori, "copy shared_ptr");
}
 
void copy_weaked() {
  std::shared_ptr<Node> ptr_ori;
  TEST_FUN1(std::weak_ptr<Node> ptr = ptr_ori, "copy weak_ptr from shared_ptr");
}
 

 
int main() {
  test_shared();
  test_unique();
  test_raw();
  test_malloc();
  copy_shared();
  copy_weaked();
}

```
测试结果:

| operation | optimization level | time(nanoseconds) |
| ------------- |:-------------:| -----:|
| new shared_ptr| O0|102.5|
| new unique_ptr| O0|76.9|
| new raw_ptr| O0|28.6|
| malloc| O0|23.6|
| copy shared_ptr| O0|17.3|
| new weak_ptr from shared_ptr| O0|17.4|
| new shared_ptr| O1|53.1|
| new unique_ptr| O1|26.1|
| new raw_ptr| O1|25.6|
| malloc| O1|0.31|
| copy shared_ptr| O1|0.31|
| new weak_ptr from shared_ptr| O1|0.31|
| new shared_ptr| O2|52.9|
| new unique_ptr| O2|25.5|
| new raw_ptr| O2|25.6|
| malloc| O2|4.2e-06|
| copy shared_ptr| O2|3.9e-06|
| new weak_ptr from shared_ptr| O2|2.9e-06|

<p>
可见，基本上，新创建一个shared_ptr还是比较耗时间的，基本为unique_ptr和 raw new的1.5~2.5倍，但是shared_ptr和weak_ptr的copy基本不花多少时间。需要注意的是，在加优化的情况下，malloc还是比new更加快的，而且不只一个数量级。所以，个人理解，如果一定要用raw pointer，new还不如malloc来的更快，当然这就需要以舍弃c++的一些特性作为代价了。但是另一方面也要想到，创建10,000,000个shared_ptr也只需要花费大概0.5～1s，而且实际中也不可能一上来就创建这么多shared_ptr，肯定是在代码运行过程中逐渐创建的。所以，实际中真的需要以舍弃c++的方便特性来换取这种提升吗？感觉意义不是很大。
</p>
### references：
[1]https://en.cppreference.com/w/cpp/memory/weak_ptr
[2]https://stackoverflow.com/questions/12030650/when-is-stdweak-ptr-useful
[3]https://stackoverflow.com/questions/5671241/how-does-weak-ptr-work/5671308#5671308
[4]https://herbsutter.com/gotw/_102/
[5]http://www.modernescpp.com/index.php/memory-and-performance-overhead-of-smart-pointer
[6]https://github.com/cothee/interesting_test
