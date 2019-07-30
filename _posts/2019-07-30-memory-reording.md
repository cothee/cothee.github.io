---
layout: post
title: 内存乱序、内存屏障以及Acquire/Release语义等若干问题
categories:
- programming
tags: [programming]
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
---
### 内存乱序、内存屏障以及Acquire/Release语义等若干问题
```
If you understand quantum physics, you don't understand quantum physics.
											           -Richard Feynman
```

#### 1. 为什么会有内存乱序(memory reording)
为了加快代码的执行，编译器或CPU通常会对内存操作顺序进行一些修改，这就是memory reording 。因为内存乱序是由编译器或CPU造成的，所以其发生的时间在编译期（compiler reording），或运行期（CPU reording）。即使在多线程的程序中，内存乱序也很少被注意到，这是因为内存乱序有一个基本的原则：不能修改单线程程序的行为。而且，在通常的多线程程序中，通常会通过加锁等方式来保持同步，这些方式通常都会阻止内存乱序的产生。但是，如果一个在多个线程间共享的内存资源没有锁保护的话，内存乱序的效果就会显现。下面这段话具体解释编译器和CPU为什么能通过乱序来加快执行速度：
*‘’Memory access instructions, such as loads and stores, typically take longer to execute than other instructions. Therefore, compilers use registers to hold frequently used values and processors use high speed caches to hold the most frequently used memory locations. Another common optimization is for compilers and processors to rearrange the order that instructions are executed so that the processor does not have to wait for memory accesses to complete. This can result in memory being accessed in a different order than specified in the source code. While this typically will not cause a problem in a single thread of execution, it can cause a problem if the location can also be accessed from another processor or device.‘’*
总结一下，就是编译器乱序和CPU乱序的原因大致有两个：(1)局部性原理; (2)CPU乱序则更多的是由于CPU和内存读写的速度差距造成的，当然，归根到底都是CPU和memory的速度差距而引起的优化。
#### 2.内存乱序会有什么问题
下面这段代码，如果在编译的时期发生了乱序，那么value = v 和 flag = 1这两个操作有可能交换顺序执行。在单线程环境下，这个不会有任何问题，因为这两个操作并不相互依赖，这个乱序保证了程序执行的正确性。但是，在多线程环境下，就有可能会出问题。如果thread1调用了setValue(), 而thread2调用checkAndDo()。那对于thread2来说对于flag的判断就很重要了，thread2如果以为flag为true的时候，thread1肯定已经把value值给修改了，这时我是不是可以拿这个value去做些什么。如果没有reording，thread2的这个判断就是正确的（单处理器下），没问题，程序正常执行。但是如果setValue里的顺序被reording了，这时thread2就不应该依靠flag去对value做任何假设判断，否则就会导致错误。通常，在两个函数里面加上锁，那这个问题就被解决掉了。
```cpp
int value = 0;
int flag = false;
void setValue(int v)
{
    value = v;
    flag= true;
}
void checkAndDo() {
	if (flag) {
		//do something with value....
	} 
}
```
#### 3.编译期乱序(compiler reording)
加锁可以解决这个问题，那有没有更简单的方法来阻止乱序？在GCC编译器下，只需要阻止乱序的代码中间插入一条汇编语句就可以了。
```cpp
//asm volatile("" ::: "memory");
void setValue(int v)
{
    value = v;
    asm volatile("" ::: "memory");  //used in GCC
    //atomic_signal_fence(memory_order_acq_rel); in c11/c++11
    flag= true;
}
```
这段汇编的作用是，告诉GCC编译器，这段代码不要优化，不要乱序。所以其充当了编译器屏障的作用。事实上，一个普通的函数，即使函数内部不包含任何屏障代码，也可以充当编译器屏障的作用（不包含[pure functions](https://lwn.net/Articles/285332/)和内联函数）。
```cpp
void doSomeStuff(Foo* foo)
{
    foo->bar = 5;
    someFun();  // prevents reordering of neighboring assignments, whether the fun() have any barrier or not, it will work.
    foo->bar2 = foo->bar;
}
```
编译器屏障在单处理器系统上已经足够了，因为不会有CPU乱序出现，即使多线程也不会有问题。但是现代的计算机基本都是多处理器的。因此，要解决内存乱序问题，多数情况下只有编译器屏障是不够的。
#### 4.CPU 乱序(CPU reording)和内存屏障(memory barrier)
#####  几种乱序表现
#LoadLoad  ==> #LoadLoad memory barrier
#StoreStore ==> #StoreStore memory barrier
#LoadStore ==> #LoadStore memory barrier
#StoreLoad ==> #StoreLoad memory barrier, 在大多数处理器上，实现这一种barrier的开销要比其他高。
每一种reording类型，都相应的有一种memory barrier（通过cpu fence instruction来实现，类似compiler barrier）去阻止reording。这些fence instruction同时也阻止了compiler reording。不同的mb性能开销也不一样，#StoreLoad memory barrier的开销会比较高一点，而且这是一个full memory barrier， 它同时也会阻止其他三种类型的memory reording。有的处理器允许所有种类的reording，有的处理器只允许其中一种reording， 这和处理器具体的内存模型有关。
#### 5. Acquire语义和Release语义
 Acquire语义 ： **Acquire semantics** is a property that can only apply to operations that **read** from shared memory, whether they are read-modify-write operations or plain loads. The operation is then considered a **read-acquire**. Acquire semantics prevent memory reordering of the read-acquire with any read or write operation that **follows** it in program order.
 Release 语义：**Release semantics** is a property that can only apply to operations that **write** to shared memory, whether they are read-modify-write operations or plain stores. The operation is then considered a **write-release**. Release semantics prevent memory reordering of the write-release with any read or write operation that **precedes** it in program order.
也就是说，Read_Acquire语义不允许它后面的操作重排到它前面来；而Write_Release则不允许它前面的操作重排序到它的后面去。这两个语义既阻止了编译器乱序，也阻止了CPU乱序。我们可以用memory barrier来实现这两个语义。在具体的平台上，可以用特定的cpu instruction来代替这里的atomic_thread_fence()。根据Acquire和Release的定义，即使是我们实现了这两个语义，#StoreLoad乱序还是有可能存在。以下是C++11实现的两种语义。
```cpp
int value = 0;
//make sure store&load on flag is atomic, a plain bool type may not be atomic
std::atomic<bool> flag = false; 

//write release1
value = 20;
std::atomic_thread_fence(std::memory_order_release);
/**(原子并不代表不乱序，这里只是利用其原子操作，
如果没有atomic_thread_fence，还是有可能乱序) **/
flag.store(true, std::memory_order_relaxed); 

//write release2
value = 20;
flag.store(true, std::memory_order_release);

//read acquire1
r1 = flag.load(std::memory_order_relaxed);
std::atomic_thread_fence(std::memory_order_acquire);
r2 = value;

//read acquire2
r1 = flag.load(std::memory_order_acquire);
r2 = value;  //r代表寄存器
```
 
#### 6.弱内存模型VS强内存模型VS顺序一致性内存模型
不同处理器，不同语言都有自己的内存模型。对于之前提到的几种乱序类型，在弱内存模型(weak memory model)下都有可能发生。但是在强内存模型(strong memory model)下，每一条指令都天然带有acquire或release语义，因此，只有可能存在#StoreLoad这种类型的乱序。X86/64架构就是属于强内存模型，其Load操作带有acquire语义，Store操作则带有release语义；ARM则属于弱内存模型。另外就是，顺序一致性模型，在这种模型下，不会有任何reording问题出现，限制更加严格。
#### 7. volatile 关键字能提供什么
volatile关键字告诉编译器，去 **内存**  里面取“最新”值。但是，即使取的是内存里的所谓“最新”值，事实上并不能保证最新。回到setValue和checkAndDo那个例子，假设flag和value两个变量都设置为volatile类型。这时，变量虽然是从内存而不是寄存器里取的值，但是，这个关键字并不能像编译器屏障那样告诉编译器不要在这里乱序，因此，这依旧没有办法阻止多线程运行中会出现的问题。这里volatile关键字能干什么以及不能干什么就很清楚了，用volatile可以时时刻刻去内存里取最新值，但是不要期待它能解决内存乱序问题。在一些需要考虑顺序的场景下，还是要小心使用volatile变量。
#### 8. atomic变量能提供什么
写过C++11的都知道，atomic能提供原子语义。跟volatile相比，它还能解决乱序的问题。更直接一点说，它提供了 Acquire语义和Release语义，而这两个语义正是阻止内存乱序的关键所在。
```cpp
//这里的flag是否带有Acquire语义？？
std::atomic<bool> flag;
if (flag == true) {
//if (flag.load(std::memory_order_acquire) == true) 
	//do something
}
```
(1)这里两个if条件里的判断肯定不是原子的啊，因为首先要读出来，然后才是比较操作。[[here]](https://stackoverflow.com/questions/49225302/using-compare-and-read-write-operations-for-stdatomicbool-in-c)
(2)至于第一个if，atomic<T>本身并没有提供==操作，它首先得调用operator T()去load去读取，然后再去调用T的operator ==()操作，load()默认情况下用的是std::memory_order_seq_cst，所以除此之外，这两个if的操作是等价的，都保证了acquire语义，但是并非原子操作。
#### 9.锁能提供什么
即使不了解reording，可能多数人的代码也并没有因此而出现过问题，因为我们知道在需要并发访问的地方加锁。除了保护临界区资源不被并发访问外，加锁操作自动提供了Acquire语义，解锁操作自动提供了Release语义。这样的结果就是，我们的临界区代码既不会被reording到lock操作的前面去，也不会被reording到unlock操作的后面去，而是被牢牢的限制在了lock和unlock之间。所以，要实现一个锁，至少要实现加锁时候的acquire语义和和解锁时候的release语义。有一个疑问，lock和unlock之间有多行代码，那这几行代码之间有没有可能乱序？会的。
#### 10.leveldb中的AtomicPointer实现
不看std::atomic的实现方式，只看x86平台相关实现。从本意上来说，这个AtomicPointer是想要实现两点：（1）原子的load/store;(2)Acquire和Release语义。AtomicPointer的原子性操作和Acquire, Release语义是没有关系的，如果没有特别要求的话，完全可以不实现Acquire和Release语义，就是只用NoBarrier_*就可以了。下面是leveldb中用的指令，但是前面说过，该指令只能防止编译期reording，那么在SMP系统上，如果出现了CPU乱序，那该实现岂不是有问题的，所有关于Acquire_Load和Release_Store的操作都应该是有问题的。但是，由于x86是强内存模型，store天然有release语义，load天然有acquire语义，所以这种情况下，它根本不需要做特殊处理，只要防止编译期乱序就好了[here](https://stackoverflow.com/questions/19965076/gcc-memory-barrier-sync-synchronize-vs-asm-volatile-memory)。
```cpp
asm volatile("" : : : "memory"); 
```
另外，在x86上，以下操作是原子的[here](https://xem.github.io/minix86/manual/intel-x86-and-64-manual-vol3/o_fe12b1e2a880e0ce-258.html)：
- Reading or writing a byte
- Reading or writing a word aligned on a 16-bit boundary
- Reading or writing a doubleword aligned on a 32-bit boundary
- Reading or writing a quadword aligned on a 64-bit boundary（64位）
AtomicPointer中存的指针变量肯定是双字对齐（32位）或四字对齐的（64位），所以该变量的读写肯定是原子的。也就是说，对于一般的变量读写都是原子的，但并不保证不乱序,要保证顺序就需要上面的两个语义做保证。

#### 11. [一个特殊的例子： Out-Of-Thin-Air Storesl](https://gcc.gnu.org/ml/gcc/2007-10/msg00597.html)
Out-Of-Thin-Air，就翻译为凭空而来吧，凭空而来的store。
```cpp
//这个例子先记这里
int v;
void f(int set_v) {
  if (set_v) {
    v = 1;
  }
//===================
/**
   pushl   %ebp
   movl    %esp, %ebp
   cmpl    $0, 8(%ebp)
   movl    $1, %eax
   cmove   v, %eax        ; load (maybe)
   movl    %eax, v        ; store (always)
   popl    %ebp
   ret
*/
/*
是的，v被读出来又被放进去了，如果不加锁的话，
set_v != 0的情况下，这段原本*只读*的代码
在多线程并发访问下会出问题吗？？(假设还有一个线程是写)
*/
```
[https://en.wikipedia.org/wiki/Memory_ordering#Compile-time_memory_barrier_implementation](https://en.wikipedia.org/wiki/Memory_ordering#Compile-time_memory_barrier_implementation)
[https://preshing.com/20120625/memory-ordering-at-compile-time/](https://preshing.com/20120625/memory-ordering-at-compile-time/)
[https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
[https://preshing.com/20120930/weak-vs-strong-memory-models/](https://preshing.com/20120930/weak-vs-strong-memory-models/)
[https://preshing.com/20120515/memory-reordering-caught-in-the-act/](https://preshing.com/20120515/memory-reordering-caught-in-the-act/)
[https://preshing.com/20120913/acquire-and-release-semantics/](https://preshing.com/20120913/acquire-and-release-semantics/)
[https://preshing.com/20120930/weak-vs-strong-memory-models/](https://preshing.com/20120930/weak-vs-strong-memory-models/)
[http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
[http://hedengcheng.com/?p=803](http://hedengcheng.com/?p=803)
[https://yarchive.net/comp/linux/compiler_barriers.html](https://yarchive.net/comp/linux/compiler_barriers.html)
[https://www.kernel.org/doc/Documentation/memory-barriers.txt](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
[https://stackoverflow.com/questions/49225302/using-compare-and-read-write-operations-for-stdatomicbool-in-c](https://stackoverflow.com/questions/49225302/using-compare-and-read-write-operations-for-stdatomicbool-in-c)
[https://en.cppreference.com/w/cpp/atomic/atomic](https://en.cppreference.com/w/cpp/atomic/atomic)
[https://stackoverflow.com/questions/19965076/gcc-memory-barrier-sync-synchronize-vs-asm-volatile-memory](https://stackoverflow.com/questions/19965076/gcc-memory-barrier-sync-synchronize-vs-asm-volatile-memory)
[http://bruceblinn.com/linuxinfo/MemoryBarriers.html](http://bruceblinn.com/linuxinfo/MemoryBarriers.html)
[https://xem.github.io/minix86/manual/intel-x86-and-64-manual-vol3/o_fe12b1e2a880e0ce-258.html](https://xem.github.io/minix86/manual/intel-x86-and-64-manual-vol3/o_fe12b1e2a880e0ce-258.html)