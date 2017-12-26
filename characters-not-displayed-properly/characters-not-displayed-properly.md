### About characters-not-displayed-properly

We run into problems when using `String.getBytes()` to convert String to Bytes in Java.


If charset is not specify when application is starting,then will use OS Platform's default charset.

So let's see something about how and why a locale would be used.

```
The values of locale categories are determined by a precedence order; the first condition met below determines the value:

If the LC_ALL environment variable is defined and is not null, the value of LC_ALL is used.
If the LC_* environment variable ( LC_COLLATE, LC_CTYPE, LC_MESSAGES, LC_MONETARY, LC_NUMERIC, LC_TIME) is defined and is not null, the value of the environment variable is used to initialise the category that corresponds to the environment variable.
If the LANG environment variable is defined and is not null, the value of the LANG environment variable is used.
If the LANG environment variable is not set or is set to the empty string, the implementation-dependent default locale is used.

```

ok，we do some test and verify.

**set LC_ALL empty and see locale**
```
$ export LC_ALL=""

$ locale
LANG=en_US.utf-8
LC_CTYPE="en_US.utf-8"
LC_ALL=
```
**then set LC_CTYPE value == zh\_CN.gb2312**
```
$ export LC_CTYPE="zh_CN.gb2312"

[duwei@qihe8054 /home/duwei] 17:13
$ locale
LANG=en_US.utf-8
LC_CTYPE=zh_CN.gb2312
LC_ALL=
```
**running test**
```
public static void main(String[] args)
  {
    System.out.println(System.getProperty("file.encoding")+" " +Locale.getDefault());
    
    while (true)
        {

        }

  }

Used to print the file.encoding and locale setting.
[duwei@qihe8054 /home/duwei] 17:13
$ export CLASSPATH=.CLASSPATH;java getpro
GB2312 en_US

Jinfo view properties.
[duwei@qihe8054 /home/duwei] 17:13
$ jinfo -sysprops `pgrep -u duwei java` 2>/dev/null |grep 'file.encoding '
file.encoding = GB2312
```

**Conclusion**

when `LC_ALL` is empty,program get properties from `LC_CTYPE`.



Later I setting all `LC_*` to empty,and it will getting properties from `LANG`.

```
[duwei@qihe8054 /home/duwei] 17:22
$ locale
LANG=zh_CN.gbk
LC_CTYPE=
LC_NUMERIC=
LC_TIME=
LC_COLLATE=
LC_MONETARY=
LC_MESSAGES=
LC_PAPER=
LC_NAME=
LC_ADDRESS=
LC_TELEPHONE=
LC_MEASUREMENT=
LC_IDENTIFICATION=
LC_ALL=

[duwei@qihe8054 /home/duwei] 17:22
$ export CLASSPATH=.CLASSPATH;java getpro
GBK zh_CN
```
***
***
###此前在蘑菇街评价应用上遇到的乱码问题排查

先看10.11.8.166上评价服务启动环境`cat /proc/$pid/environ` 或者直接看

```
jinfo -sysprops 的 system properties参数
(jinfo还可以动态调vm参数的，keep safe，在某些情况下使用会让JVM run into STOP state)
```

```
$ /usr/local/jdk/bin/jinfo -sysprops 14891|grep file.encoding
Attaching to process ID 14891, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.65-b04
file.encoding.pkg = sun.io
file.encoding = ANSI_X3.4-1968
```
发现乱码的机器带的**file.encoding是ANSI_X3.4-1968**。
![乱码](https://s11.mogucdn.com/mlcdn/c024f5/171226_5gf062gh7ld7ik20983bgi0134l40_968x478.png)
发现其他机器是正常中文日志并且评价内容写到DB也是OK的，基本排除代码问题。

为了进一步确定请求来源数据没有乱码，对来源抓包解码分析，tcpdump -XX 将数据包内容显示成 hex and ASCII 两者对应。
![](http://s8.mogucdn.com/new1/v1/fxihe/210efc0beb2afa550172a58796b198f6/A14a0f1751b2000402.png)
将comment评价字段的十六进制转换成文本 convert hex to string
![](http://s6.mogucdn.com/new1/v1/fxihe/67773944502f82019c9b0077ca2fe707/A186bb2751b2000802.png)
基本判断来源也是OK的。

最后是无意在一位同事的mac上发现的，问题根源是由于本地ssh客户端设置并且带过去的环境变量，读的是`/etc/ssh/ssh_config`，其中`SendEnv LANG`，然后他再重启应用便继续传染过去。
![](http://s8.mogucdn.com/new1/v1/fxihe/0ff247f0ab51767e27d7a1fe4433c490/A137ae2751b2000402.png)

打开ssh的verbose模式，很明显的就会发现登录主机时带过去的LANG 被set到登录的服务器上。

