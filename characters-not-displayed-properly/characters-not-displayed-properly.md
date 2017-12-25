About characters-not-displayed-properly

Java代码中的一个转换获取数组用的是String.getBytes()，如果启动时代码内部未强制指定的话就会使用OS platform's default charset，有兴趣可看下javadoc
但是java是如何获取拿到这个编码value的呢？
它是按照一定的顺序来拿：
