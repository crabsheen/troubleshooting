### Server disk iops getting worse and find out why?
Runing fio disk performance testing on 10.17.107.35

```
fio -name iops -rw=read -bs=4k -runtime=60 -iodepth 32 -filename /dev/sdb -ioengine libaio -direct=1
```
The result is really bad,from beginning,the read bandwidth started from 350M per second,after ten seconds later,it getting down to only dozens of M per second.

(跑磁盘读写测试发现磁盘IO性能抖动非常厉害，一开始350M/s，10秒之后迅速降低到几十兆每秒。)

There's my thinking and examine steps below.

1. compare to other server which has same hardware configuration(disk type,raid controller),and running same testing case above,the result is good,disk performance lasted stable.
（对比相同硬件配置服务器(磁盘型号，raid卡)，跑同样的fio测试用例发现IO反应无异常，持续稳定。所以怀疑10.17.107.35这台服务器本身问题。）

2. I'm doubt about raid controller may has something wrong,the server has two Virtual Drive:
`one is two drives and raid level is raid1,the another 12 drivers(3TB) made to raid5` 
I try to run fio testing on sda(the raid1)，the result is ok,So rule out raid controller‘s problem. then doubt about may be some bad disk in sdb.
(怀疑raid卡问题，然后对sda做测试，发现fio测试用例也是很稳定，排除raid卡问题。然而raid5结构的sdb就是始终抖动，所以怀疑是sdb内部个别磁盘问题，sdb是由10多块盘3TB规格磁盘组成。)

3. **Using `/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL` to view detail info about all drivers.During the output i've find it's a little pause when display info about `slot number:7`.That's very important discovery.  **       
![](https://camo.githubusercontent.com/ae7b14a7344e0ab8f63a4ae79ec0ee8087cc2096/687474703a2f2f73362e6d6f677563646e2e636f6d2f6e6577312f76312f66786968652f37373838313665613266313532646135626632306234663266386563313533372f4131616434303937353162323030303830322e706e67)        
(继续跑fio测试然后用megacli64 -PDList -aALL去看看，这个指令是显示所有disk的详细信息,反复跑了几次观察下来，发现总是在展示slot number:7的时候有短暂的停顿，你看我都输入4个ffff了，所以开始怀疑这块盘有问题。)

4. Using SMART tools check disk's state `smartctl -H --device=megaraid,1-n /dev/sdb`, all result is `SMART Health Status: OK`.
(用SMART去查看磁盘的情况，smartctl -H --device=megaraid,1-n /dev/sdb 依次看sdb内部每一块盘的情况，结果都是OK的。SMART Health Status: OK)

5. Finally I find the essence problem,through disk's Error counter log.
`for i in 1 2 7 8;do echo $i;smartctl -l error -d megaraid,$i /dev/sdb;done;`
![](https://camo.githubusercontent.com/6a877e6c210ecbf2ca0a402b605b9db8ebf47c4e/687474703a2f2f73362e6d6f677563646e2e636f6d2f6e6577312f76312f66786968652f35663831663966356131323136396263616533303162353233633866653337612f4131313335343937353162323030303430322e706e67)  
Searching by Google and there's extremely race about that `Error counter log`,Only one article[https://discuss.zendesk.com/hc/en-us/articles/201900786-Using-smartctl-to-Troubleshoot-Disk-Performance-on-DCA](https://discuss.zendesk.com/hc/en-us/articles/201900786-Using-smartctl-to-Troubleshoot-Disk-Performance-on-DCA) has more detail information about the `Error counter log` description, include the meanings of each field.  (特地抽查几块盘看磁盘读写错误累计数，基本就确定了slot number:7这块存在问题。
for i in 1 2 7 8;do echo $i;smartctl -l error -d megaraid,$i /dev/sdb;done;查了很多资料关于Error counter log这部分，没啥详细的说明。最后这份解释非常棒，解释了每个字段的含义的可能的问题。)

6. After manufacturer’s staff replaced the bad disk and rebuilding completely,I running again same fio testing is ok.
