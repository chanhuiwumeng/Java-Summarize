## 海量数据处理总结

海量数据处理六种方法：
1. 分而治之/hash映射 + HashMap统计 + 堆/快速/归并排序
2. 多层划分
3. Bloom filter/Bitmap
4. Trie树/数据库/倒排索引
5. 外排序
6. 分布式处理之Hadoop/MapReduce

## 1、分而治之/hash映射 + HashMap统计 + 堆/快速/归并排序
适用范围：类似"出现次数最多前10"、"热门前10查询"、"频率最高前100"等跟频数排序有关问题

步骤：
1. 分而治之/hash映射：由于数据量过大，内存不足于存储所有数据，所以要需要将大文件（取模映射）化成小文件，逐个解决，最后汇总。
2. HashMap统计：当大文件转化了小文件后，那么便可以统计元素的出现次数，这里除了HasHMap，还可以使用trie树/搜索二叉树/红黑树
3. 堆排序：统计完了之后便进行排序，可先在小文件采取堆排序，得到每个小文件TopN，最后归并排序，得到大文件TopN。

问题实例：
1. 海量日志数据，提取出某日访问百度次数最多的那个IP（hash映射/一致性hash + HashMap统计 + 堆排序）
2. 寻找热门查询，300万个查询字符串中统计最热门的10个查询（数据规模小，可一次性存下，直接 HashMap/trie树/搜索二叉树/红黑树统计 + 堆排序）
3. 有一个1G大小的一个文件，里面每一行是一个词，词的大小不超过16字节，内存限制大小是1M。返回频数最高的100个词（hash映射 + HashMap/trie树统计 + 堆排序/归并排序）
4. 海量数据分布在100台电脑中，想个办法高效统计出这批数据的TOP10（1、 HashMap统计 + 堆排序 2、 遍历全部 + hash映射 + HashMap统计 + 堆排序，控制同一个元素只会出现在一台电脑）
5. 有10个文件，每个文件1G，每个文件的每一行存放的都是用户的query，每个文件的query都可能重复。要求你按照query的频度排序 （1、顺序读取10个文件，hash(query)%10的结果将query写入到另外10个文件 + HashMap统计 + 堆排序/归并排序 ， 2、hash映射 + 采用分布式的架构来处理，比如MapReduce + 合并）
6. 给定a、b两个文件，各存放50亿个url，每个url各占64字节，内存限制是4G，让你找出a、b文件共同的url （hash映射 + HashSet）
7. 怎么在海量数据中找出重复次数最多的一个?（同1）
8. 上千万或上亿数据（有重复），统计其中出现次数最多的前N个数据（同2）
9. 一个文本文件，大约有一万行，每行一个词，要求统计出其中最频繁出现的前10个词，请给出思想，给出时间复杂度分析（同1，时间复杂度是O(n*lg10)）
10. 1000万字符串，其中有些是重复的，需要把重复的全部去掉，保留没有重复的字符串。请怎么设计和实现？（1、trie树/HashMap  2、LinkedHashSet）
11. 一个文本文件，找出前10个经常出现的词，但这次文件比较长，说是上亿行或十亿行，总之无法一次读入内存，问最优解（同1）
12.  100w个数中找出最大的100个数（1、局部淘汰法，O(100w*100)  2、快速排序，O(100w*100)  3、用一个含100个元素的最小堆完成，O(100w*lg100)）


## 2、多层划分
适用范围：第k大，中位数，不重复或重复的数字

基本原理及要点：因为元素范围很大，不能利用直接寻址表，所以通过多次划分，逐步确定范围，然后最后在一个可以接受的范围内进行。

问题实例：

1. 2.5亿个整数中找出不重复的整数的个数，内存空间不足以容纳这2.5亿个整数

整数个数为2^32,也就是，我们可以将这2^32个数，划分为2^8个区域(比如用单个文件代表一个区域)，然后将数据分离到不同的区域，然后不同的区域在利用bitmap（通过一个bit位来表示某个元素对应的值）就可以直接解决了

2. 5亿个int找它们的中位数

思路一：首先我们将int划分为2^16个区域，然后读取数据统计落到各个区域里的数的个数，之后我们根据统计结果就可以判断中位数落到那个区域，同时知道这个区域中的第几大数刚好是中位数。然后第二次扫描我们只统计落在这个区域中的那些数就可以了。

实际上，如果不是int是int64，我们可以经过3次这样的划分即可降低到可以接受的程度。即可以先将int64分成2^24个区域，然后确定区域的第几大数，在将该区域分成2^20个子区域，然后确定是子区域的第几大数，然后子区域里的数的个数只有2^20，就可以直接利用direct addr table进行统计了。
　　
思路二：同样需要做两遍统计，如果数据存在硬盘上，就需要读取2次。

同基数排序类似，开一个大小为65536的Int数组，第一遍读取，统计Int32的高16位的情况，也就是0-65535，都算作0,65536 - 131071都算作1。就相当于用该数除以65536。Int32 除以 65536的结果不会超过65536种情况，因此开一个长度为65536的数组计数就可以。每读取一个数，数组中对应的计数+1，考虑有负数的情况，需要将结果加32768后，记录在相应的数组内。

第一遍统计之后，遍历数组，逐个累加统计，看中位数处于哪个区间，比如处于区间k，那么0- k-1的区间里数字的数量sum应该<n/2（2.5亿）。而k+1 - 65535的计数和也<n/2，第二遍统计同上面的方法类似，但这次只统计处于区间k的情况，也就是说(x / 65536) + 32768 = k。统计只统计低16位的情况。并且利用刚才统计的sum，比如sum = 2.49亿，那么现在就是要在低16位里面找100万个数(2.5亿-2.49亿)。这次计数之后，再统计一下，看中位数所处的区间，最后将高位和低位组合一下就是结果了。

## 3、Bloom filter/Bitmap
适用范围：可以用来实现数据字典，进行数据的判重，或者集合求交集

基本原理及要点：
1. Bloom filter：利用位数组很简洁地表示一个集合，并能判断一个元素是否属于这个集合，在判断一个元素是否属于某个集合时，有可能会把不属于这个集合的元素误认为属于这个集合（false positive）。因此，Bloom 
Filter不适合那些“零错误”的应用场合。而在能容忍低错误率的应用场合下，Bloom Filter通过极少的错误换取了存储空间的极大节省。
2. Bitmap：通过一个bit数组来存储特定数据的一种数据结构；由于bit是数据的最小单位，所以这种数据结构往往是非常节省存储空间



问题实例：

1. 给你A,B两个文件，各存放50亿条URL，每条URL占用64字节，内存限制是4G，让你找出A,B文件共同的URL。如果是三个乃至n个文件呢？

根据这个问题我们来计算下内存的占用，4G=2^32大概是40亿*8大概是340亿，n=50亿，如果按出错率0.01算需要的大概是650亿个bit。现在可用的是340亿，相差并不多，这样可能会使出错率上升些。另外如果这些urlip是一一对应的，就可以转换成ip，则大大简单了。

同时，上文的第5题：给定a、b两个文件，各存放50亿个url，每个url各占64字节，内存限制是4G，让你找出a、b文件共同的url？如果允许有一定的错误率，可以使用Bloom filter，4G内存大概可以表示340亿bit。将其中一个文件中的url使用Bloom filter映射为这340亿bit，然后挨个读取另外一个文件的url，检查是否与Bloom filter，如果是，那么该url应该是共同的url（注意会有一定的错误率）。

2. 在2.5亿个整数中找出不重复的整数，注，内存不足以容纳这2.5亿个整数。

思路一：采用2-Bitmap（每个数分配2bit，00表示不存在，01表示出现一次，10表示多次，11无意义）进行，共需内存2^32 * 2 bit=1 GB内存，还可以接受。然后扫描这2.5亿个整数，查看Bitmap中相对应位，如果是00变01，01变10，10保持不变。所描完事后，查看bitmap，把对应位是01的整数输出即可。

思路二：也可采用与第1题类似的方法，进行划分小文件的方法。然后在小文件中找出不重复的整数，并排序。然后再进行归并，注意去除重复的元素。

3. 给40亿个不重复的unsigned int的整数，没排过序的，然后再给一个数，如何快速判断这个数是否在那40亿个数当中？

用位图/Bitmap的方法，申请512M的内存，一个bit位代表一个unsigned int值。读入40亿个数，设置相应的bit位，读入要查询的数，查看相应bit位是否为1，为1表示存在，为0表示不存在。

## 4、Trie树/数据库/倒排索引

### Trie树
适用范围：数据量大，重复多，但是数据种类小可以放入内存

基本原理及要点：是一种哈希树的变种，排序和保存大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。它的优点是：利用字符串的公共前缀来减少查询时间，最大限度地减少无谓的字符串比较，查询效率比哈希树高。

问题实例：
1. 寻找热门查询：查询串的重复度比较高，虽然总数是1千万，但如果除去重复后，不超过3百万个，每个不超过255字节。
2. 有10个文件，每个文件1G，每个文件的每一行都存放的是用户的query，每个文件的query都可能重复。要你按照query的频度排序。
3. 1000万字符串，其中有些是相同的(重复),需要把重复的全部去掉，保留没有重复的字符串。请问怎么设计和实现？
4. 一个文本文件，大约有一万行，每行一个词，要求统计出其中最频繁出现的前10个词。其解决方法是：用trie树统计每个词出现的次数，时间复杂度是O(n*le)（le表示单词的平准长度），然后是找出出现最频繁的前10个词。


### 数据库索引
适用范围：大数据量的增删改查

基本原理及要点：利用数据库索引（B 树、B+ 树、B* 树及R 树）的设计实现方法，对海量数据的增删改查进行处理。

### 倒排索引
适用范围：搜索引擎，关键字查询

基本原理及要点：从关键字到文档的映射过程，保存这种映射这种信息的索引

问题实例：
1. 文档检索系统，查询那些文件包含了某单词，比如常见的学术论文的关键字搜索


## 5、外排序
适用范围：大数据的排序，去重

基本原理及要点：外部排序指的是大文件的排序，即待排序的记录存储在外存储器上，待排序的文件无法一次装入内存，需要在内存和外部存储器之间进行多次数据交换，以达到排序整个文件的目的。

处理过程 ：
1. 按可用内存的大小，把外存上含有n个记录的文件分成若干个长度为L的子文件，把这些子文件依次读入内存，并利用有效的内部排序方法对它们进行排序，再将排序后得到的有序子文件重新写入外存。
2. 对这些有序子文件逐趟归并，使其逐渐由小到大，直至得到整个有序文件为止。

问题实例：
1. 如何给10^7个数据量的磁盘文件排序

## 6、 分布式处理之Hadoop/MapReduce
适用范围：数据量大，但是数据种类小可以放入内存

基本原理及要点：将数据交给不同的机器去处理，数据划分，结果归约。

问题实例：
1. 海量数据分布在100台电脑中，想个办法高效统计出这批数据的TOP10。
2. 一共有N个机器，每个机器上有N个数。每个机器最多存O(N)个数并对它们操作。如何找到N^2个数的中数(median)？

---
资料来源：[教你如何迅速秒杀掉：99%的海量数据处理面试题](https://blog.csdn.net/v_JULY_v/article/details/7382693)
