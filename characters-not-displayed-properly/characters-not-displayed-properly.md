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
![乱码](https://camo.githubusercontent.com/514b85f11fda7f9c990133172fd0197dd486d8fb/68747470733a2f2f7331312e6d6f677563646e2e636f6d2f6d6c63646e2f6330323466352f3137313232365f3567663036326768376c6437696b3230393833626769303133346c34305f393638783437382e706e67)
发现其他机器是正常中文日志并且评价内容写到DB也是OK的，基本排除代码问题。

为了进一步确定请求来源数据没有乱码，对来源抓包解码分析，tcpdump -XX 将数据包内容显示成 hex and ASCII 两者对应。
![](https://camo.githubusercontent.com/ac016fc1bf5233d774ac80ff228ae91cc899e2b8/687474703a2f2f73382e6d6f677563646e2e636f6d2f6e6577312f76312f66786968652f32313065666330626562326166613535303137326135383739366231393866362f4131346130663137353162323030303430322e706e67)
将comment评价字段的十六进制转换成文本 convert hex to string
![](https://camo.githubusercontent.com/4cd9b18c34ac3b6a2286c63494c867a50e22b73a/687474703a2f2f73362e6d6f677563646e2e636f6d2f6e6577312f76312f66786968652f36373737333934343530326638323031396339623030373763613266653730372f4131383662623237353162323030303830322e706e67)
基本判断来源也是OK的。

最后是无意在一位同事的mac上发现的，问题根源是由于本地ssh客户端设置并且带过去的环境变量，读的是`/etc/ssh/ssh_config`，其中`SendEnv LANG`，然后他再重启应用便继续传染过去。
![](https://camo.githubusercontent.com/e5c6387c11a1cef32bfc285753d3c650260bbc34/687474703a2f2f73382e6d6f677563646e2e636f6d2f6e6577312f76312f66786968652f30666632343766306162353137363765323764376131666534343333633439302f4131333761653237353162323030303430322e706e67)

打开ssh的verbose模式，很明显的就会发现登录主机时带过去的LANG 被set到登录的服务器上。

