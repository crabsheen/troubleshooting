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

####set LC_ALL empty and see locale
```
$ export LC_ALL=""

$ locale
LANG=en_US.utf-8
LC_CTYPE="en_US.utf-8"
LC_ALL=
```
####then set LC_CTYPE value == zh\_CN.gb2312
```
$ export LC_CTYPE="zh_CN.gb2312"

[duwei@qihe8054 /home/duwei] 17:13
$ locale
LANG=en_US.utf-8
LC_CTYPE=zh_CN.gb2312
LC_ALL=
```
####running test
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

####Conclusion
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
