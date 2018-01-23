### Kafka Request Packet Analysis Tools -------- Base on V0.8.2.2
  
**最初出发点是为了找到谁在消费磁盘的哪些冷数据。**

High-Level API Consumer的消费进度比较好掌握，比如Offset，logSize，Lag。

```
used to view specific groupid's consumer progress.
bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper $IP:2181 --group groupid
```

Low-Level Consumer 适用于自主控制消费行为，SimpleConsumer是个例子 [https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)

Low-Level Consumer 比如消费进度由消费者自主维护，所以kafka.tools.ConsumerOffsetChecker查看会报异常`KeeperErrorCode = NoNode`。

针对问题出发点，临时起了2个思路
>	blktrace blkio 跟踪OS 读写事件 找到被频繁读的block，inode 再利用inode找到被读的File。

>	client request协议交互数据包里面有offset topic等信息，通过解析数据包之后找到对应 log及index file。

从定位准确性程度，实施操作复杂度，占用系统资源开销来看，这里选择第二种。

###话不多，操刀开始干吧。

最低成本地制造 `消息写入/消息读取` 场景并对此做数据分析。

```
消息写入。
bin/kafka-console-producer.sh --broker-list xxx:9092 --topic yyy

消息读取，print-offset用于输出时打印消息序号，fetchsize表示拉取大小，offset表示从哪个位置开始消费。
bin/kafka-run-class.sh kafka.tools.SimpleConsumerShell --broker-list xxx:9092 --clientId group --topic yyy --partition n --print-offset --fetchsize 1048575 --offset 12345
```

针对以上场景做数据包分析，具体见右边注释。
![](https://github.com/crabsheen/troubleshooting/blob/master/Kafka-Request-Packet-Analysis-Tools/kafka%20protocol.png?raw=true)

关于交互协议强烈建议直接看说明文档 [https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)

摘录一段FetchRequest的Format。

```  
FetchRequest => ReplicaId MaxWaitTime MinBytes [TopicName [Partition FetchOffset MaxBytes]]
  ReplicaId => int32
  MaxWaitTime => int32
  MinBytes => int32
  TopicName => string
  Partition => int32
  FetchOffset => int64
  MaxBytes => int32
```
`FetchRequest`又是一个完整的`RequestMessage`的必要成分，具体看下面。

```  
RequestMessage => ApiKey ApiVersion CorrelationId ClientId RequestMessage
  ApiKey => int16
  ApiVersion => int16
  CorrelationId => int32
  ClientId => string
  RequestMessage => MetadataRequest | ProduceRequest | FetchRequest | OffsetRequest | OffsetCommitRequest | OffsetFetchRequest
```
搞清楚了的具体协议格式，继续说下一步。

先抓包。

```  
为什么过滤条件用 tcp[36:4] == 0x00010000
tcp[36:4]表示从包的37位置开始，长度4个字节的内容段。（IP头部，TCP头部长度等其实是有关的）

这部分刚好是2个字节的ApiKey和2个字节的ApiVersion，0001代表请求类型为FetchRequest。
tcpdump -i bond0 dst port 9092 and tcp[36:4] == 0x00010000 -nn -A -w /tmp/kafka.pcap
```
附: Api Keys And Current Versions


| API name        | ApiKey Value  |
| --------------- |:-------------:|
| ProduceRequest  | 0 		        |
| FetchRequest    | 1             | 
| OffsetRequest   | 2             |
| MetadataRequest | 3             |

然后做过滤等预处理。

```  
打印出client ip 和 具体的数据包，新增过滤条件是排除节点间的复制。
tshark -r kafka.pcap -T fields -e ip.src -e data -R "not data.data contains 52:65:70:6c:69:63:61:46:65:74:63:68:65:72:54:68:72:65:61:64"
ccc.ddd.aaa.bbb    000000480001000000129fd40002687affffffff0000006400000001000000010016687a2d707572652d61636d5f65746c5f6578706f7365000000010000000a000000001b4b2ea600100000
ccc.ddd.aaa.ccc    0000005c000100000001f78a001f61645f7265616c74696d655f666561747572655f31302e31392e32322e3338ffffffff000000640000000100000001000d61636d5f6578706f73655f7631000000010000000b000000004266bb7800100000

为什么是52:65:70:6c:69:63:61:46:65:74:63:68:65:72:54:68:72:65:61:64？

[duwei@xxxx /home/duwei]
$ echo "52 65 70 6c 69 63 61 46 65 74 63 68 65 72 54 68 72 65 61 64"|xxd -r -p
ReplicaFetcherThread

这段文字代表着复制线程clientid，需要排除掉，从中找出普通client的请求。
```
附: tshark 输出，过滤字段说明，非常detail，见 [https://www.wireshark.org/docs/dfref/t/tcp.html](https://www.wireshark.org/docs/dfref/t/tcp.html)

最后具体分析包体，根据FetchRequest的Format进行分段拆分做解析。(就是Offset是从哪里开始数到哪里结束之类的。。)

```
写了段awk来完成它，主要是针对 字符串 和 进制转换的操作，只要搞清楚请求体结构就行。
cat kafka.awk
        #covert hex to string
        function hex2str(n)
        {
                s = "";
                for(i=1; i<=length(n)-1; i+=2)
                s = s sprintf("%c", strtonum("0x" substr(n, i, 2)));
                return s
        }

        {
                #Get FetchRequest data
                FetchRequest=substr($2,29,length($2)-28-7-16-8-8);

                #Split FetchRequest data into Array,"ffffff" is ReplicaID 代表-1 还看到个ffff fffe 代表-2
                split(FetchRequest,Array,"fffffff[fe]");

                #client ip
                printf $1;

                #FetchOffset    strtonum():Examine str and return its numeric value.
                printf("\tFetchOffset:%d",strtonum("0x"substr($2,length($2)-7-16,16)));

                #Partition
                #printf("\tPartition:%d",strtonum("0x"substr($2,length($2)-7-16-8,8)));

                #TopicName
                printf("\tTopicName:%-50s",hex2str(substr(Array[2],29))"-"strtonum("0x"substr($2,length($2)-7-16-8,8)));

                #MaxBytes,MaxWaitTime,ClientId
                #print "\tMaxBytes:"substr($2,length($2)-7) "\tMaxWaitTime:"substr(Array[2],0,8) "\tClientId:"hex2str(Array[1])
                print "\tClientId:"hex2str(Array[1])
        }


我们来运行一下:
[duwei@xxxx /home/duwei]
$ sudo /usr/sbin/tshark -i bond0 -f "dst host ccc.ddd.aaa.aaa and dst port 9092 and tcp[36:4] == 0x00010000" -c 100 -n -T fields -e ip.src -e data -R "not data.data contains 52:65:70:6c:69:63:61:46:65:74:63:68:65:72:54:68:72:65:61:64" 2>/dev/null |awk -f /home/duwei/kafka.awk
ccc.ddd.aaa.bbb    FetchOffset:316216553   Partition:8     TopicName:mario_etl_network_s_v1                                ClientId:im_req_code
ccc.ddd.ccc.bbb    FetchOffset:721772963   Partition:6     TopicName:mario_etl_network_n                                   ClientId:mdata_network_n_base_cql
ccc.ddd.ddd.fff    FetchOffset:748096840   Partition:10    TopicName:mario_etl_mwp_http_network                            ClientId:im_mdata_biz_request_group       
        
上面的例子是抓取数据包并实时分析，主要字段有来源IP，FetchOffset，Partition分区号，TopicName，ClientId
```






**20171129 Update 对以上抓包分析增加Lag差值(获取具体Topic下具体Partition的Logsize然后做减法) 更容易判断是哪些ClientId落后更多。**

首先弄清楚需要哪些信息，这儿需要拿到具体Topic下具体Partition的最大Offset，OK，这里要用到Offset Request。
参考[https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-OffsetFetchRequest](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-OffsetFetchRequest)

来，我们先看个API结构，这是V0版本的`ListOffsetRequest`。
```
// v0
ListOffsetRequest => ReplicaId [TopicName [Partition Time MaxNumberOfOffsets]]
  ReplicaId => int32
  TopicName => string
  Partition => int32
  Time => int64
  MaxNumberOfOffsets => int32
```
来实地分析一下包
![](https://github.com/crabsheen/troubleshooting/blob/master/Kafka-Request-Packet-Analysis-Tools/180123_85bb2ab3jkajl7kh1b7b6jja1cki6_2854x300.png?raw=true)

```
看最后一行 从后面往前看
00:00:00:80    MaxNumberOfOffsets => int32    最大请求offset数量，就是输出最后面的多少数量的offset
ff:ff:ff:ff:ff:ff:ff:ff    Time => int64    超时时间
00:00:00:0a    Partition => int32    分区ID
61:63:6d:5f:65:78:70:6f:73:65:5f:76:31    TopicName => string    Topic名字
00:0d 代表Topic名字的长度大小，十六进制。
ff:ff:ff:ff    ReplicaId => int32 
00:35 代表整个数据包长度大小，十六进制。

抓包分析基于kafka.tools.GetOffsetShell

/home/data/kafka_2.9.2-0.8.2.2/bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list  *.*.*.*:9092 --offsets 2  --topic acm_expose_v1 --time -1
acm_expose_v1:0:29432412164,29430988440
acm_expose_v1:1:29432477585,29430971331

顺带扩展说一下这个工具吧，这工具有3个事要做，1是获取Topic的分区分布信息，2是根据获取到的信息分别对不同分区所在server发起请求offset，3汇总这些数据病展示。

```

再来，我们继续看协议结构，这是V0版本的`OffsetResponse`，即返回数据。

```
// v0
OffsetResponse => [TopicName [PartitionOffsets]]
  PartitionOffsets => Partition ErrorCode [Offset]
  Partition => int32
  ErrorCode => int16
  Offset => int64
  
下面例子是构造ListOffsetRequest然后拿到该topic下该分区的offset信息。
[duwei@xxxx /home/duwei] 17:55
$ (echo -ne "\x00\x00\x00\x35\x00\x02\x00\x00\x00\x00\x00\x00\x00\x00\xff\xff\xff\xff\x00\x00\x00\x01\x00\x0d\x61\x63\x6d\x5f\x65\x78\x70\x6f\x73\x65\x5f\x76\x31\x00\x00\x00\x01\x00\x00\x00\x0b\xff\xff\xff\xff\xff\xff\xff\xff\x00\x00\x00\x01";sleep 1)|nc x.x.x.x 9092|xxd -p -c 1024
0000002d0000000000000001000d61636d5f6578706f73655f7631000000010000000b000000000001000000009380f1e7 

返回结果的最后int64长度的000000009380f1e7就是所谓的offset值，转化成十进制就很直观了。

```

好了，下面上新代码，在前者基础上改的。
```
具体的理解可以看注释。
$ cat kafka.awk.new
        #covert hex to string
        function hex2str(n)
        {
                s = "";
                for(i=1; i<=length(n)-1; i+=2)
                s = s sprintf("%c", strtonum("0x" substr(n, i, 2)));
                return s
        }

        function get_last_offset(packet,FetchOffset)
        {
                ##调用system执行系统命令
                system("/home/duwei/Lag.sh "packet" "FetchOffset"");
        }

        {
                #Get FetchRequest data
                FetchRequest=substr($2,29,length($2)-28-7-16-8-8);

                #Split FetchRequest data into Array,"ffffff" is ReplicaID 代表-1 还看到个ffff fffe 代表-2
                split(FetchRequest,Array,"fffffff[fe]");

                #client IP
                printf $1;

                #FetchOffset    strtonum():Examine str and return its numeric value.
                FetchOffset = sprintf("%d",strtonum("0x"substr($2,length($2)-7-16,16)));
                printf("\tFetchOffset:%d",FetchOffset);

                #Partition
                #printf("\tPartition:%d",strtonum("0x"substr($2,length($2)-7-16-8,8)));

                #TopicName
                printf("\tTopicName:%-50s",hex2str(substr(Array[2],29))"-"strtonum("0x"substr($2,length($2)-7-16-8,8)));

                ##这儿想构造一个请求offset的数据包,主要是固定格式+Topicname,固定40字节,Topicname可以换算下,下面几个格式化的代表数据长度,topic长度十六进制,topic内容的十六进制.
                ##substr($2,length($2)-7-16-8,8) 是分区ID
                ##length(hex2str(substr(Array[2],29)))+40 是请求数据包内容长度
                ##length(hex2str(substr(Array[2],29))) 是topic的长度
                ##substr(Array[2],29) 是topic的十六进制文本
                ##sprintf 赋值.
                packet = sprintf("0000%04x00020000000000000000ffffffff00000001%04x%s0000001"substr($2,length($2)-7-16-8,8)"ffffffffffffffff00000001",length(hex2str(substr(Array[2],29)))+40,length(hex2str(substr(Array[2],29))),substr(Array[2],29));
                get_last_offset(packet,FetchOffset);

                #MaxBytes,MaxWaitTime,ClientId
                #print "\tMaxBytes:"substr($2,length($2)-7) "\tMaxWaitTime:"substr(Array[2],0,8) "\tClientId:"hex2str(Array[1])
                print "\tClientId:"hex2str(Array[1])
        }
```

```
该脚本主要用于获取某topic下指定分区的最大offset值 然后再计算一个Lag，快速定位是谁在落后。这个东西最好的地方在于无论是否low high level都一览无遗。
$ cat Lag.sh 
#!/bin/bash

#LogSize - FetchOffset equals Lag

FetchRequest=`echo "$1"|sed 's/../\\\x&/g'`;
FetchOffset=$2;

(echo -ne "$FetchRequest";sleep 0.1)|nc localhost 9092|xxd -p -c 102400|awk -v FetchOffset="$FetchOffset" '{printf("LogSize:%d\tLag:%-10d",strtonum("0x"substr($0,length($0)-15,16)),strtonum("0x"substr($0,length($0)-15,16))-FetchOffset)}';
```

上个例子大伙看下，观察到10几万20多万的落后，没有读到磁盘的行为。
具体命令：

`
sudo /usr/sbin/tshark -i bond0 -f "dst host x.x.x.x and dst port 9092 and tcp[36:4] == 0x00010000" -c 100 -n -T fields -e ip.src -e data -R "not data.data contains 52:65:70:6c:69:63:61:46:65:74:63:68:65:72:54:68:72:65:61:64" 2>/dev/null |awk -f /home/duwei/kafka.awk.new
`
![](https://raw.githubusercontent.com/crabsheen/troubleshooting/master/Kafka-Request-Packet-Analysis-Tools/command.png)

下面捕捉到拉老数据的行为了，并且用`bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper x.x.x.x:2181 --group funnel`是看不到的
![](https://github.com/crabsheen/troubleshooting/blob/master/Kafka-Request-Packet-Analysis-Tools/find.png?raw=true)

###总结
该工具可以升级为实时分析当前各种请求类型（支持多种ApiKey及对应的数据字段解析）。
