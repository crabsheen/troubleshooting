kafka是Java应用，首先看JVM相关的两个参数设置$KAFKA_HEAP_OPTS 和 $KAFKA_JVM_PERFORMANCE_OPTS

>`$KAFKA_HEAP_OPTS` 新老节点的具体区别是：
"-Xmx1G -Xms1G" 和 "-Xmx4G -Xms4G -Xmn2G"

>`$KAFKA_JVM_PERFORMANCE_OPTS`新节点则是在原基础上增加了
"XX:MaxTenuringThreshold=15 -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=75 -Djava.awt.headless=true"   


* 其实这2种情况下，通常情况下gc表现没特别大的差异，无非是ygc(Allocation Failure)的频率会高一点，但也不碍事。

`"Allocation Failure"` means that no more space left in Eden to allocate object. So, it is normal cause of young GC.   
 
我们用的是`+UseParNewGC`   
ParNew" is a stop-the-world, copying collector which uses multiple GC threads.  
***
观察了几次CMS
2017-07-01T09:18:01.345+0800: 252823.939: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1573130K(2097152K)] 1595640K(3984640K), 0.0839292 secs] [Times: user=0.10 sys=0.01, real=0.08 secs]

**此处old区域大小2097152K，触发是在占比1573130K的时候，即满足XX:CMSInitiatingOccupancyFraction=75**

This is initial Marking phase of CMS where all the objects directly reachable from roots are marked and this is done with all the mutator threads stopped,
**所有mutator线程会stop,但是没关系,initial-mark还是很快的.**
	
相比较而言，耗时稍长的阶段发生在Remark.  
2017-07-01T09:18:07.141+0800: 252829.735: [Rescan (parallel) , 0.0131579 secs]2017-07-01T09:18:07.154+0800: 252829.748: [weak refs processing, 0.0332428 secs]2017-07-01T09:18:07.187+0800: 252829.781: [class unloading, 0.5822911 secs]2017-07-01T09:18:07.769 +0800: 252830.363: [scrub symbol table, 0.0639340 secs]2017-07-01T09:18:07.833+0800: 252830.427: [scrub string table, 0.0012848 secs][1 CMS-remark: 1574881K(2097152K)] 1597219K(3984640K), 0.7652786 secs] [Times: user=1.76 sys=0.00, real=0.77 secs]

`Rescan 以及 weak refs processing ,unloading 以及 scrub symbol table 以及scrub string table 都是包含在remark期间的。`

此处引用CMS collector的说明
> The CMS collector pauses an application twice during a concurrent collection cycle. The first pause is to mark as live the objects directly reachable from the roots (for example, object references from application thread stacks and registers, static objects and so on) and from elsewhere in the heap (for example, the young generation).`This first pause is referred to as the initial mark pause.` The second pause comes at the end of the concurrent tracing phase and finds objects that were missed by the concurrent tracing due to updates by the application threads of references in an object after the CMS collector had finished tracing that object. `This second pause is referred to as the remark pause.`

暂时线索不多，继续看看其他节点，有进一步发现   
* 2017-07-10T20:46:28.127+0800: 1071730.721: [GC (Allocation Failure) 2017-07-10T20:46:28.128+0800: 1071730.722: [ParNew: 1693977K->66440K(1887488K), 11.6090617 secs] 2518463K->891558K(3984640K), 11.6140182 secs] [Tim
es: user=301.34 sys=0.00, real=11.61 secs]   
ygc耗时超过11秒，由于zookeeper.session.timeout.ms的默认值是`6000ms`。

于是发生了Kafka与Zookeeper断连重新做re-elect。 
  
* cat /var/www/html/kafka_2.9.2-0.8.2.2/logs/controller.log   
[2017-07-10 20:46:41,213] INFO [SessionExpirationListener on 723], ZK expired; shut down all controller components and try to re-elect (kafka.controller.KafkaController$SessionExpirationListener)

抽看以往日志都有类似情况发生，如下   
* [2017-07-05 19:38:33,543] INFO [SessionExpirationListener on 723], ZK expired; shut down all controller components and try to re-elect (kafka.controller.KafkaController$SessionExpirationListener)

* 2017-07-05T19:38:25.340+0800: 635647.934: [GC (Allocation Failure) 2017-07-05T19:38:25.340+0800: 635647.934: [ParNew: 1720318K->71275K(1887488K), 7.0692471 secs] 2309708K->661485K(3984640K), 7.0694741 secs] [Times:
user=185.20 sys=0.00, real=7.07 secs]


这时候感觉就不像是只在cms gc下才会发生的恶劣状况了，怀疑其他的可能，比如OS上。

无意逮到一次cms gc耗时稍长的，这是个突破点。
![](/var/folders/d_/mfrr4ww10d18vsmwnplt6s9r0000gn/T/com.evernote.Evernote/com.evernote.Evernote/WebKitDnD.J1aVys/2EB66D06-27A1-4CAB-B720-021B1238FE28.png)   
耗时将近4秒的remark.

看了下OS层面的表现，这时候基本猜到到什么原因了。
![
](/var/folders/d_/mfrr4ww10d18vsmwnplt6s9r0000gn/T/com.evernote.Evernote/com.evernote.Evernote/WebKitDnD.7udhmE/E933EF7E-5F16-4798-88F1-DF55C9B1CE57.png)
majflt/s 的意思是Number of major faults the system has made per second, those which have required loading a memory page from disk.   
注意看，这是一个从disk loading 数据进入 memory的过程，我们可以想象这个时间要多久，再仔细看发现这个读取量平时是很低的。

那么为什么会有这样的情况发生？根本原因在于应JVM使用到了Swap.  
***
## 一些进阶,引申知识
- 注意到class unloading的过程是最慢的，为什么呢？   
因为JVM Metaspace空间是最不容易被经常访问的空间，也是内存不足时最应该被放入Swap区域的。  
 
- 用jmap -clstats `$javapid`去查看class loader statistics但是屡次都把进程弄死了，怎么办？   
别着急，用kill给它发送一个`-CONT`信号， 顾名思义这是continue的意思，即告诉Java让它继续run起来，不需要重启服务即可继续运行，毕竟重启kafka可是要很久的。。。   
这非常有用，有兴趣的同学可以看下kill的各种信号及交互，比如NGINX的USR1 TERM等

- 能看到class loading 和unloading的统计数据么？   
$ jstat -class `$javapid`   
Loaded  Bytes  Unloaded  Bytes     Time   
  4070  8595.1      205   302.9      52.82   
  最后一列Time的意思是class loading和unloading的耗时。
  
- Metaspace到底长什么样的？内部结构？以及如何申请释放？   
Java Hotspot VM explicitly manages the space used for metadata. `Space is requested from the OS and then divided into chunks.` A class loader allocates space for metadata from its chunks (a chunk is bound to a specific class loader). `When classes are unloaded for a class loader, its chunks are recycled` for reuse or returned to the OS. `Metadata uses space allocated by mmap, not by malloc.`

- 如何关闭Swap，有没有危险？`我先说非常危险，除非你深知你的应用情况以及OS情况，然后说如何操作`   
通过`/etc/fstab`找到swap分区信息   
UUID=d4a9fd97-55da-4bab-a078-2e86edc9aa2c swap                    swap    defaults        0 0   
然后 `ls -l /dev/disk/by-uuid`   
total 0   
lrwxrwxrwx 1 root root 10 Apr 11  2016 1618ce3c-740b-477b-af05-ae7c442ad533 -> ../../sda3   
lrwxrwxrwx 1 root root 10 Apr 11  2016 74e016f0-bbef-4f4f-b7d8-458eecc49b5e -> ../../sda1   
`lrwxrwxrwx 1 root root 10 Apr 11  2016 d4a9fd97-55da-4bab-a078-2e86edc9aa2c -> ../../sda2`   
lrwxrwxrwx 1 root root 10 Apr 28  2016 fc9a6e6f-6d3e-48fe-8bc8-4291a3ecb9ea -> ../../sdb1   
确认Swap属于/dev/sda2，执行`swapoff /dev/sda2`即可关闭。   
#### 严重注意:确保有足够的free memory可以容纳当前整个Swap的使用量，关闭期间应用性能会受影响，实际上关闭Swap是个慢慢将磁盘数据读入内存的过程，磁盘IO压力非常大。我有个数据，700M的Swap刷到内存耗时45秒。

- 因class unloading而进入的误区，查问题往往会进入其他未知领域的。   
发现一个关于class unloading慢的解决办法，提及了large page的[http://www.oracle.com/technetwork/java/javase/tech/largememory-jsp-137182.html](www.oracle.com/technetwork/java/javase/tech/largememory-jsp-137182.html)   
为什么用large page可以缓解class unloading慢？TLB是个page 转换 cache   
By using bigger page size, a single TLB entry can represent larger memory range. There will be less pressure on TLB and memory-intensive applications may have better performance.
但是JVM开启large page支持很麻烦，在jdk 7u60和8以后是默认禁用的。
首先要设置`-XX:+UseLargePages`其实还要设置`-XX:+UseLargePagesInMetaspace`才能让MetaSpace使用large page，庆幸没去尝试。

## Swap深入
- 查看当前服务器swap使用情况？   
$ `cat /proc/swaps`   
Filename                                Type            Size    Used    Priority   
/dev/sda2                               partition       16383992        2680972 -1   
- 看看都是谁在用swap？   
top 然后按f 再按p 回车。
![](/var/folders/d_/mfrr4ww10d18vsmwnplt6s9r0000gn/T/com.evernote.Evernote/com.evernote.Evernote/WebKitDnD.trxwVj/F77A4673-5F83-44A5-BC69-E202957340A4.png)
如上图其中一个java用了1.1g的swap   
- 具体进程的维度看swap使用量   
`cat /proc/$javapid/status`   
其中的`VmSwap:`字段，这个值和top中看到的是一致的。   
此外还可以看`/proc/$javapid/smaps`   
信息量巨大，可以看到应用的全部内存空间，包括动态库，heap，脏页，正在打开的文件句柄等等。 
![heap](/var/folders/d_/mfrr4ww10d18vsmwnplt6s9r0000gn/T/com.evernote.Evernote/com.evernote.Evernote/WebKitDnD.JRC4US/C3803131-E05A-4EAD-9C59-18158E197833.png)  
可以用这个算下swap用量，同样是一致的。
```   
$ grep Swap /proc/17414/smaps|awk 'BEGIN {sum=0}{sum+=$2} END{print sum}'
832884
```

- 还是很想知道java把什么数据放swap，准备用blk工具跟踪。   
首先挂debugfs，`sudo mount -t debugfs none /sys/kernel/debug/`   
然后跟踪读事件的情况 `/usr/bin/blktrace -a read -w 30 -d /dev/sda2` 
```  
  sda2 
  CPU  0:                   50 events,        3 KiB data
  CPU  1:                    9 events,        1 KiB data
  CPU  2:                  165 events,        8 KiB data
  CPU  3:                    0 events,        0 KiB data
  CPU  4:                  311 events,       15 KiB data
  CPU  5:                   19 events,        1 KiB data
  CPU  6:                   40 events,        2 KiB data
  CPU  7:                    0 events,        0 KiB data
  CPU  8:                   90 events,        5 KiB data
  CPU  9:                    0 events,        0 KiB data
  CPU 10:                  115 events,        6 KiB data
  CPU 11:                    0 events,        0 KiB data
  CPU 12:                    0 events,        0 KiB data
  CPU 13:                    0 events,        0 KiB data
  CPU 14:                   29 events,        2 KiB data
  CPU 15:                    0 events,        0 KiB data
  CPU 16:                   25 events,        2 KiB data
  CPU 17:                    0 events,        0 KiB data
  CPU 18:                    2 events,        1 KiB data
  CPU 19:                    9 events,        1 KiB data
  CPU 20:                    0 events,        0 KiB data
  CPU 21:                    9 events,        1 KiB data
  CPU 22:                  113 events,        6 KiB data
  CPU 23:                    0 events,        0 KiB data
  Total:                   986 events (dropped 0),       47 KiB data
```
转换为可读性较强的文本，`blkparse sda2.blktrace.* > readable.log`   
关于该文本字段解释参见[https://www.cse.unsw.edu.au/~aaronc/iosched/doc/blktrace.html](https://www.cse.unsw.edu.au/~aaronc/iosched/doc/blktrace.html)   
$ grep 44939 readable.log |more   
  8,0    8        6    26.451956364 `44939`  A   R 11149504 + 8 <- (8,2) 10737856   
  8,0    8        6    26.451956364 `44939`  A   R 11149504 + 8 <- (8,2) 10737856   
  8,0    8        6    26.451956364 `44939`  A   R 11149504 + 8 <- (8,2) 10737856   
  8,0    8        6    26.451956364 `44939`  A   R 11149504 + 8 <- (8,2) 10737856   
  8,0    8        6    26.451956364 `44939`  A   R 11149504 + 8 <- (8,2) 10737856   
  8,0    8        6    26.451956364 `44939`  A   R 11149504 + 8 <- (8,2) 10737856      
第一列是设备标识号 major/minor  8,2是swap区域，见/proc/partitions，major表示设备类型，minor表示该设备类型上不同分区。(linux很多都是有major,minor的概念)    
插入TT截图  
readable.log中高亮部分`44939` 是java的其中1个Child thread。   
```   
$ pstree -p 43178|grep 44939  
            |-{java}(44939)
``` 
- 还有关于swap文件格式的问题就不展开了，至此我们可以看到从系统层面能捕捉到java的确在操作swap。

