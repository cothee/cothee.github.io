---
layout: post
title: 时间轮
categories:
- algorithms
tags: [timing wheel, c++11]
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

# 定时器：时间轮  
  一个定时模块需要包含四个接口：
  (1)StartTimer: 插入一个指定过期操作(Interval),过期操作(ExpiryAction)和编号(RequestId)的定时器;
  (2)StopTimer: 找到相应的定时器，并停止(将其移除);
  (3)PerTickBookeeping: 整个定时模块的计时单位为T,则需要在每单位时间消逝后都要检查是否有过期的定时器，如果有的话，针对该过期的timer调用StopTimer;
  (4)ExpiryProcessing:针对过期timer,调用在StartTimer时为该Timer指定的过期操作（ExpireAction);
前两个操作是由外部的用户调用，而后两种则是随着时间的tick，由模块本身进行调用。共有一下几种时间轮:
## 1.简单直接——无序链表/数组  
(1)StartTimer: O(1),直接插在尾部
(2)StopTimer: O(1)
(3)PerTickBookeeping:O(n), 需要遍历整个链表，查找过期Timer

## 2.有序双向链表  
(1)StartTimer: 每次插入都要保证链表是有序的,最坏是O(n),平均情况取决于所有timers的internals的概率分布和timers的到达过程？？？不具体指定分布的化，一般也认为平均情况为O(n)
(2)StopTimer: O(1)
(3)PerTickBookeeping:O(1), 每次Tick,只需要和表头的Timer进行比较
2相对于1,相当于用增加StartTimer的代价来降低PerTickBookeeping的时间。由于用的是双链表，代价为O(n).
事实上,在2的基础上，如果考降低有序插入的时间代价的话，StartTimer的时间也是可以降低的，比如用二叉树，堆等，从而可以使StartTimer降低到O(logn)。其实以上两种方法都比较直接。

## 3.简单时间轮  
之所以是简单时间轮，是该算法有一个简化实际情况的限制条件：要求所有Timer存在一个MaxInterval。这样，保证每一个Timer都会在MaxInterval时间内过期。设置一个数组来存放时间轮链表，数组大小为MaxInterval， 数组每一个元素指向一个时间轮链表，需要保证同一个链表里面的Timer会在相同时刻过期。假设当前时刻指向的元素为i，需要插入的Timer的interval为j，则该Timer会被插入第(i+j)%MaxInterval个数组元素指向的链表里，并且是插入表头，因此，StartTimer为O(1)。而且，也很好理解，StopTimer(采用双链表)和PerTickBookeeping也都为O(1).考虑到实际的情况，这种简化的情况基本已经满足需求，服务端在检查网络上的读写事件的超时时间一般不会很大（百、千秒内）。当需要的MaxInterval比较大时，可以调节每次tick的时间，即将时间粒度变大一点。

## 4.hash时间轮——hash表+有/无序链表  
(1)hash+无序链表
(2)hash+有序链表
这两个时间轮的区别是很容易理解的，无序链表和有序链表的使用影响到了StartTimer，stopTimer的效率。简单来说，它就是一个固定长度的哈希表。而与简单时间轮不同的是，它不再有最大时间限制的假设。尽管如此，hash表的最大长度MaxInterval却是客观存在的。在链表里面会多存一个参数round，round = timeout / MaxInterval,每次tick都需要检查当前链表中是否有round为0的节点，如果有的话，删除之，对于不为0的则需要减1。有序和无序的区别在于一个hash槽里的链表是否需要按照round来排序，如果排序的话，StartTimer时间会比较长(O(n))，如果不排序的话，Tick的时间会比较长。基本的区别就是Tick和Start谁来承担排序的工作量，换汤不换药。

## 5.hierarchical分层时间轮  
这个是最复杂的时间轮,，类似一个钟表，可以表示很细粒度的时间以及很长的时间，因此一般用于比较精细的时间控制。一般存在多种时间粒度的时间轮，每个时间轮代表不同的时间长度(天，时，分，秒)，在走动的时候，要向钟表一样，秒时间轮走一圈之后，分时间轮应该向前走一格，同时查看该格上有没有过期事件，以此类推，具体每种时间轮的实现和前面的也类似，不再赘述。复杂度方面：
(1)StartTimer:O(m), m表示时间轮的个数（如果是时，分，秒，则m=3）
(2)StopTimer: O(1)
(3)PerTickBookeeping:平均来讲为O(1)(有时候可能会去分，时轮上去tick)

## [一个简单时间轮的c++实现](https://github.com/cothee/timewheel)  
  用c++11实现，其中双链表用weak_ptr 和shared_ptr的实现，充分体现两种指针的用法。具体实现不完全线程安全(not thread-safe exactly)：tick()和add()操作之间是线程安全的，tick()和remove()之间也是线程安全的，Add()和remove()之间也是线程安全的(加锁实现)，但是tick本身只允许在一个线程当中调用(考虑到实际情况，我们也只需要在唯一的一个线程当中调用tick)。expired在每次tick之后做，和tick同一线程。

## References:  
[1]http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf 
[2]https://www.cse.wustl.edu/~cdgill/courses/cs6874/TimingWheels.ppt 
[3]https://github.com/cothee/timewheel
