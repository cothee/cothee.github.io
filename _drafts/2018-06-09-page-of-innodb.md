
在innodb中，当使用独立表空间时，innodb_file_per_table = 1(在my.cnf中设置)，每个表的结构信息(即create 时用到的信息)存在于tablename.frm文件中，表中的实际行数据存在于tablename.ibd文件中，这两个文件统一位于tablename目录下。因此，当数据库出现问题时，想要恢复数据，就得从这两个文件中恢复，ibd文件是重中之重。
page是innodb的最小文件单元，一个page的默认大小是16KB(机械硬盘的一个扇区是512B, 文件系统的一个块是4KB)，ibd文件就由一个一个page组成。老版本的innodb的行格式是Redundant, 自MySQL5.6以后，默认都是Compact格式。相比Redundant格式来说，Compact格式节约了大量的存储空间。本文也仅讨论Compact格式。

上面说到，一个page的大小是16KB。实际上，每一个page都由固定的几部分构成，(1)file header;(2)page header;(3)行记录;(4)page Directory;(5)File Tailer。其中，file header 占38个字节，后面紧跟着的page header占用56个字节。这里先简略介绍一下Page header, 
PAGE_HEAP_TOP  2 Bytes  指向堆中的第一条记录;
PAGE_N_HEAP    2 Bytes  堆中的记录总数，初始值为2（后面会解释原因）
PAGE_FREE      2 Bytes  指向第一条空闲记录
PAGE_GARBAGE   2 Bytes  所有已删除记录占用的总的字节数;
PAGE_N_RECS    2 Bytes  页中的用户记录数目

在这38+56之后的字节中，紧接着的就是行记录。最先开始的是两条特殊的行记录，"Infimum" 和 "Supremum", 所有的记录被串成一个链表，链表的头节点就是Infimum记录，末尾节点就是Supremum,中间的就是真正的用户记录。对于Compact格式来说，一个记录除了记录内容之外，还有变长字段列表+固定头部。因此，一个完整的记录在Page中的构成就是 变长字段列表+固定头部(5 Bytes)+ 用户记录，三者在页中的顺序也是这样子。以下按顺序介绍这三部分：

(1)变长字段长度头(variable-length header, https://dev.mysql.com/doc/refman/5.6/en/innodb-physical-record.html#innodb-compact-row-format-characteristics)

变长字段长度头包含一个比特向量，用于指示一个允许为null的字段是不是为null。如果一个表中的允许为null的字段数目是N个，那么这个比特向量就占用(N+8-1)/8个字节。例如，N=9,则比特向量会占用两个字节;N=0,则没有比特向量。比特向量后面紧跟着的是变长字段长度。这里需要注意一下，一定是变长字段的长度，而固定长度的字段长度是不会显示在这里的，例如int,tinyint,char等等。每一个变长字段的长度会在这里占用一个或两个字节，具体取决于该字段的最大长度。对于每一个非null的变长字段来说，当字段值存在于溢出页或者实际长度超过127bytes并且最大长度超过255字节时，该字段的长度就会占用两个字节。更准确的说，这里并不是字节数，而是字符个数。因为在不同编码方式下，每个字符占用的字节数是不一样的，如果每个字符占三个字节，那这里的最大长度就是765字节。

(2)固定头部:
在Compact格式下, 固定头部占用5 Bytes = 40 bits.

保留位  1   保留位
保留位  1   保留位
deleted_flag 1  标志删除
min_rec_flag 1  为1表示该记录是预先被定义为最小的记录
n_owned 4       该记录拥有的记录数
heap_no  13  索引堆中的排序记录
record_type 3  记录类型，000:普通记录，001:B+树节点指针，010:infimum, 011:supermum
next_record 16 页中下一条记录的相对位置(注意是相对位置)








