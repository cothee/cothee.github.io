---
layout: post
title: WiscKey—— KV 分离的设计与实现
categories:
- database
tags: [database, storage]
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

## 1.背景——LSM-Tree 有什么问题
首先，WiscKey 是在 LSM-Tree 基础上进行的针对 SSD的优化设计。那么，LSM-Tree 存在什么问题？
#### 1.2 读写放大
写放大和读放大是 LSM-Tree 最主要的问题。写放大主要来自 Compaction。作为 LSM-Tree 的实现之一，LevelDB的相邻 level 之间的大小差是10倍，当上层的某个文件要 compact到下一层时，这个 compact 可能引发下层所有文件的读取和重新写入。这一次 compaction 引起的写入就是上层文件的10倍大小(按照论文里的说法是10倍，但我认为应该更大，因为上层的一个文件和下层所有文件的大小差距肯定是大于10倍的，上层不只一个文件)。按照 levelDB的实现，从 Level1到 Level6合计起来就是50倍的写放大。然后再说到读放大，一般查找一个key-value， 最多需要读取L0层的8个文件（交叉无序需遍历），和其他层各一个文件（有序，每个文件的key range在内存中，最多只需读取一个文件），共14个sst文件。由于每个sst文件包含bloom filter + index block + data block，如果读取1KB的KV，需要读取4KB(bloom filter) + 16KB(index block) + 4KB(data block) = 24KB， 24KB × 14 = 336KB的数据，因此读放大是336倍。KV 对的大小越小，读放大会越大。
#### 1.1 写放大引起的 SSD 寿命问题
写放大越大意味着需要擦除写入的数据越多，而SSD的擦除次数是有限的，写放大会降低SSD的寿命。
#### 1.3 在 SSD 上，LSM-Tree 的优势没有那么大
LSM-Tree 适用于写>读的场景中，LSM-Tree 针对写入进行了特定的优化——把随机写转换为顺序写。考虑到 HDD 顺序写性能要远大于其随机写性能，这个优化大大提升了写入性能。但是 SSD 的顺序写和随机写的性能没有 HDD 那样差别那么大，因此，这个优化对SSD 上的写入性能提升并不显著。
#### 1.4 SSD 的随机读性能很好
SSD 的随机读性能其实很好，其内部有很好的并行性，如果通过多线程来并行读，可以有效利用这个特点，从而大大提高吞吐。这个主要是考虑到KV分离后，针对vLog的读都是随机的，range query是否会变得很慢？如果能有效利用SSD的这个优势，range query也并不会慢，后面会细说。
## 2.假设——基于什么样的假设
### 2.1 key-value 对里key和value的相对大小
实际应用中，key通常比较小，大约几十字节以内，而value大小则分布广泛，甚至超过4KB。key的大小通常是小于value的。
## 3.问题——KV 分离会带来哪些新的问题
### 3.1 如何组织value文件（vLog file）
首先要明确的是，顺序写还是顺序写，一定不能引入随机写。这样的话vLog文件还是以append only的方式写入。
### 3.2 range query会不会有性能下降
当然会，因为虽然能在原有的LSM-Tree里顺序扫出一堆key，但是如果vLog文件是顺序写入的话，那意味着后续查找value要靠点读的方式进行了，这意味着随机读，虽然SSD 的随机读性能要高于HDD，但是还是会牺牲性能。所以，这一点在具体实现的时候需要有优化。根据论文中的优化，实际性能并不低。
### 3.3vLog文件如何进行value的删除
KV分离后，LSM-Tree里只存储了Key,当进行删除时，也只是append一条删除记录，但是对于相应vLog文件里的value进行删除会比较困难。因为value也是append进去的，而且不会进行compaction。因此需要专门针对value的删除进行设计。
## 4. 方法——WiscKey是如何设计的
LSM-Tree的写放大主要来源于compaction，而compaction其实只需要对key进行compaction即可，对value只需要该删除的删除该更新的更新。如果将key和value分离的话，compaction就只需要处理key，考虑到2.1的假设——key的大小通常小于value的大小，这将大大减少写放大。
### 4.1设计目标
(1) 降低写放大

(2) 降低读放大

(3) 充分利用SSD的硬件优势（随机读+顺序写）

### 4.2设计细节
#### 4.2.1总体设计
(1) key + value的location 存在LSM-Tree里，value的location意味着filename+offset，这通常不会占太多空间。LSM-Tree里不再存储实际的value。value变小了，这造成的实际效果就是减少了写放大。

(2) value存储在单独的vLog文件里。value的写入是顺序写入。当然，这就首先会引入问题3.3。文件不可能无限增大，被删除的value和过时的value应该能被删除清理。
#### 4.2.2 细节设计
(1) 针对问题3.2，首先要提到1.4中提到的SSD的随机读优势。利用其内部有很好的并行性这个特点，在range query时可以先从 LSM-Tree 中读出一批key，以及 key 对应的 vLog 地址（读出来以后·是否对地址进行一下排序会有更好的查询性能？？），然后再用多线程的方式去vLog里读value（个人认为这里很适合用协程）。这样会充分利用SSD 的优势，增大range query 的吞吐。

(2) 3.1和3.3可以合并成一个问题，都需要有效组织vLog文件，最大化提升性能。从KV分离后的情况来看，如果要删除KV ,则要首先删除key，这和原来的方式没有区别。但这时如果要删除value的话，有两种方式，一种是直接不用管，key已经被删了，意味着value永远也不会被读取，这样vLog文件会无限增大，这当然是不可取的。另一种就是遍历LSM-Tree，获取每一个有效key的value地址，这些value需要在vLog文件中继续保存 ，其他的value则需要被物理删除。不过，这里我还是不知道具体究竟怎么删除。论文里提出了一个优化的轻量级garbage collection（gc）方法。首先， vLog 的一条记录是<key_size, key, value_size, value>，然后，WiscKey将 vLog 文件看成一个链表，有 head 和 tail。其中 head 指向vLog 文件的尾部，这里可以进行 value 的append 操作；tail 则是gc 开始的地方。head 和 tail 中间的内容是有效的 value。在做 gc 的时候，首先从 tail 指向的地方开始scan 出一些 KV来，再通过查找 LSM-Tree确认哪些 value是有效的，然后再把有效的 KV append到 vLog 的 head 所指向的地方，最后释放掉被 scan 出来的 KV 占用的空间，修改tail的指向位置（tail 位置信息存在 LSM-Tree 中，防止宕机丢失数据）。对 vLog 的重新 append 是需要 sync 进行持久化的。同时，由于 value 的位置信息已经改变了，还需要重新写入一次 LSM-Tree 来更新 KV信息。文中提到，gc 可以在后台定时或设置阈值策略来做，在存储空间比较大或者 delete 操作比较少（如果有的write 是 update 类型的，应该和 delete 比较类似，都会造成原有 kv 信息在 vLog 文件中无效）的场景下，可以少做 GCC。
```
// vLog 文件格式
|tail|...|<key_size, key, value_size, value>|...|head|
```
#### 4.3细节优化
(1) 去掉 LSM-Tree 原来的 log 写入。由于在上面介绍的的 vLog细节中针对 gc 的优化方式， 写入记录里已经有了 kv 的信息，并且也是顺序写入，相当于也可以起到 LSM-Tree 的 log 的作用。数据库启动的时候，从 vLog 的 headwe 位置起扫入的 kv 可以写入到 LSM-Tree 中，完成对数据库的恢复。

(2) 对于较小 kv 的写入，采用buffer缓存的方式，达到一定大小后 再刷入磁盘。这样减少了 IO（write）操作，提高磁盘的吞吐。（对，这样做有可能丢数据的）

## 5.实现——WiscKey是如何实现的
(1) vLog 文件的 gc 实现：在 gc过程中，确认0~tail 这一段文件内容失效后，可以用 **fallocate()**来将这段文件空间释放，还给系统。

(2) 范围查找：WiscKey内部维护了一个线程池（32个线程），这个线程池包含一个线程安全的队列。在查找的时候，上层从LSM-Tree 中查找出一系列key 的 value address， 然后将这些 address 放到线程池的队列中，唤醒多个线程，达到并行查找的目的。

## 6.实验——结果验证
  测试主要是在对比 leveldb 和WiscKey。
### 6.1 Load 性能
  就是写入性能，不知道为什么用 Load 这个单词。
(1) 顺序写入场景：对 Leveldb 来说，写入时间主要在三个方面：写 log；写 memtable；memetable dump到磁盘（？？）；在小 value 场景下，Leveldb 的写入时间主要花在写 log 上。在大 value 场景下，时间主要消耗在写 log 和memtable 刷盘上。而WiscKey则在大小 value 场景下都有较好表现，主要是WiscKey不需要写LSM-Tree 的log，而写vLog 是有 buffer 的。

(2) 随机写入：无论是小 value 还是大 value，WiscKey的随机写入要比 Leveldb高几十到100多倍。在 value=4KB 的时候WiscKey是达到了磁盘带宽极限。而 Leveldb则由于compaction 的原因吞吐较低。
### 6.2 Query性能
(1) Random Query
  无论是 大 value 场景还是 小 value 场景，WiscKey 都有更高的吞吐。这里的主要原因是（1）leveldb 的读放大更明显；（2）WiscKey中的 compaction 只占用较少的磁盘带宽。

(2) Range Query
  简单来说，随着 value size的增大，WiscKey 的吞吐主要在变大。而 value size 超过4KB以后，leveldb 的吞吐就开始变小。因此在小 value 场景下，leveldb 更有优势，而在 大 value 场景下，WiscKey则更有优势。
### 6.3 空间放大
  大约60~80G 的用户实际数据, 如果没有 gc，WiscKey会占用100-140GB 的空间。而有 gc 的情况下，WiscKey的占用空间大约也为60-80GB，和用户数据本身差不多， Leveldb 占用的空间则在80-100GB 之间，比用户数据稍大。


## References:  
  
[1][https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

