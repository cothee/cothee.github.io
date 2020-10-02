---
layout: post
title: the use of pipe
categories:
- linux
tags: [cpp]
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

读写锁在 gcc-8.2里有两种实现，编译的时候可以选择关闭宏 _GLIBCXX_USE_PTHREAD_RWLOCK_T和 _GTHREAD_USE_MUTEX_TIMEDLOCK来选择使用其中一种实现。

一种实现封装 了pthread_rwlock, 另外一种实现用了两个条件变量+一个计数器变量+一个 mutex 锁([https://gcc.gnu.org/onlinedocs/gcc-8.2.0/libstdc++/api/a00137_source.html](https://gcc.gnu.org/onlinedocs/gcc-8.2.0/libstdc++/api/a00137_source.html))

【成员变量】
(1) 条件变量 M_gate1: 标志是否有写者和读者数量是否超限；
(2) 条件变量 M_gate2: 标志读者数量是否为0；
(3) 计数器变量 M_state：最高位标志写者，其它位计数读者的数量，每进来一个读者，计数器加1；
(4) mutex: 互斥锁用来保护M_state, 并且用来在两个条件变量wait的时候使用。

【加写锁的过程】

（1）锁住 mutex, 通过条件变量M_gate1，确认M_state没有设置写者标志位；
（2）M_gate1.wait()返回后，设置M_state的写者标志位（在锁里）， 表明自己已经占据写者位置，阻止其它写者进入，但这时还不能进行写操作，还需要确认没有其它读者；
（3）锁住 mutex（M_gate2 wait 返回后，又会获得锁）, 通过条件变量M_gate2, 确认M_state 的读者数量为0（确认没有读者）；
（4）成功，返回。
**分析**：加写锁需要两层条件：（1）当前无写者；（2）进一步判断当前无读者；

【加读锁的过程】

（1）锁住 mutex, 通过条件变量M_gate2， 确认读者数量**小于**允许的最大读者数量且无写者（如果超过了，读者就会持续等待在M_gate2上）；
（2）M_state++， 计数器自增，读者数量加1；
（3）解锁，返回。

**分析：**加读锁需要两个条件：（1）当前无写者；（2）当前读者数没有超过最大读者数量。实际实现的代码中，其实只有一个条件：M_state < S_max_readers，但者一行判断其实包含了前面说的两个条件，因为最高位是标着写者的，这一个条件就能表示无写者且读者数量未超限。完成加写锁的操作，mutex 一共需要进行三次加锁和解锁操作。
```cpp
void  lock_shared() {
	unique_lock<mutex> __lk(_M_mut);
	_M_gate1.wait(__lk, [=]{return  _M_state < _S_max_readers; });
	++_M_state;
}
```

**【**解写锁过程**】**

（1）锁住mutex, 取消M_state写者标志位的设置；
（2）唤醒等待在M_gate1上的写者；
（3）解锁，返回。

**分析：**唤醒其它等待M_gate1的写者。这时候可能并没有写者等待在M_gate1上，唤醒操作不会有其它作用。（这种条件下，可能有读者等待在M_gate2上，为什么不唤醒读者？？）

**【**解读锁过程**】**
（1）锁住mutex；
（2）M_state--， 计数器自减，读者数量减1；
（3）此时如果有写者等待在M_gate2上（写标志位已经设置，等待在M_gate2），判断下当前读者数量是否为0，如果为0，则可以唤醒等待在M_gate2上的写者；
（4）此时如果没有写者等待在M_gate2上，则判断（M_state--）之前的读者数量是否等于最大读者数量，如果等于，则唤醒等待在M_gate2上的读者，（小于的情况其它读者可以获得读锁而无需等待）；

**分析：**写优先。读者等待有两个条件：（1）当前无写者；（2）当前读者数量达到了最大数量；因此，在没有写者的情况下，只有读者数量达到最大时，才需要唤醒可能存在的等待的读者。

**【**其它问题**】**

（1）读者和写者会同时抢 mutex, 写者和写者会抢 mutex, 读者和读者之间也会抢 mutex。所以，即使是在读多写少的情况下，多个读者之间也会抢 mutex，会有竞争。因此多个读者获取读锁也是有代价的。
