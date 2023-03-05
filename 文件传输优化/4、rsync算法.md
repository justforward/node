

# 核心算法

rsync的算法如下：（假设我们同步源文件名为fileSrc，同步目的文件叫fileDst）

1）**分块Checksum算法。** 首先，我们会把fileDst的文件平均切分成若干个小块，比如每块512个字节（最后一块会小于这个数），然后对每块计算两个checksum，

一个叫rolling checksum，是弱checksum，32位的checksum，其使用的是Mark Adler发明的adler-32算法，  
另一个是强checksum，128位的，以前用md4，现在用md5 hash算法。  
为什么要这样？因为若干年前的硬件上跑md4的算法太慢了，所以，我们需要一个快算法来鉴别文件块的不同，但是弱的adler32算法碰撞概率太高了，所以我们还要引入强的checksum算法以保证两文件块是相同的。也就是说，弱的checksum是用来区别不同，而强的是用来确认相同。（checksum的具体公式可以参看这篇文章）

**2）传输算法。** 同步目标端会把fileDst的一个checksum列表传给同步源，这个列表里包括了三个东西，rolling checksum(32bits)，md5 checksume(128bits)，文件块编号。


同步源端的算法

**3）checksum查找算法。** 同步源端拿到fileDst的checksum数组后，会把这个数据存到一个hash table中，用rolling checksum做hash，以便获得O(1)时间复杂度的查找性能。这个hash table是16bits的，所以，hash table的尺寸是2的16次方，对rolling checksum的hash会被散列到0 到 2^16 – 1中的某个整数值。（对于hash table，如果你不清楚，建议回去看大学时的数据结构教科书）

顺便说一下，我在网上看到很多文章说，“要对rolling checksum做排序”（比如这篇和这篇），这两篇文章都引用并翻译了原作者的这篇文章，但是他们都理解错了，不是排序，就只是把fileDst的checksum数据，按rolling checksum做存到2^16的hash table中，当然会发生碰撞，把碰撞的做成一个链表就好了。这就是原文中所说的第二步——搜索有碰撞的情况。

**4）比对算法。** 这是最关键的算法，细节如下：

4.1）取fileSrc的第一个文件块（我们假设的是512个长度），也就是从fileSrc的第1个字节到第512个字节，取出来后做rolling checksum计算。计算好的值到hash表中查。

4.2）如果查到了，说明发现在fileDst中有潜在相同的文件块，于是就再比较md5的checksum，因为rolling checksume太弱了，可能发生碰撞。于是还要算md5的128bits的checksum，这样一来，我们就有 2^-(32+128) = 2^-160的概率发生碰撞，这太小了可以忽略。如果rolling checksum和md5 checksum都相同，这说明在fileDst中有相同的块，我们需要记下这一块在fileDst下的文件编号

4.3）如果fileSrc的rolling checksum 没有在hash table中找到，那就不用算md5 checksum了。表示这一块中有不同的信息。总之，只要rolling checksum 或 md5 checksum 其中有一个在fileDst的checksum hash表中找不到匹配项，那么就会触发算法对fileSrc的rolling动作。于是，算法会住后step 1个字节，取fileSrc中字节2-513的文件块要做checksum，go to (4.1) - 现在你明白什么叫rolling checksum了吧。 

4.4）这样，我们就可以找出fileSrc相邻两次匹配中的那些文本字符，这些就是我们要往同步目标端传的文件内容了。






传送的时候需要记录分片的offset，用于接收方进行分块的拼接。

https://www.cnblogs.com/f-ck-need-u/p/7226781.html#auto_id_6



## rolling hash


该算法使用滚动哈希快速地跳过哈希不匹配的子串，只在哈希匹配时才对比整个模式串的每个字节，

该算法可以推广在用于文本搜寻单个模式串的所有匹配或在文本中搜索多个模式串的匹配。

