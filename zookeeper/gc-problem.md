2018-01-04T17:32:52.973+0800: 15304.228: [GC [1 CMS-initial-mark: 1572736K(1572864K)] 3259944K(3932160K), 0.6557250 secs] [Times: user=0.66 sys=0.00, real=0.66 secs] 

**Tenured GEN 区满触发CMS GC，此时YOUNG区比如S1 S0的占用为0，容量分别为262144，Eden使用为1687208，容量2097152, This is initial Marking phase of CMS where all the objects directly reachable from roots are marked and this is done with all the mutator threads stopped.**

2018-01-04T17:32:53.629+0800: 15304.883: [CMS-concurrent-mark-start]

**Start of concurrent marking phase.
In Concurrent Marking phase, threads stopped in the first phase are started again and all the objects transitively reachable from the objects marked in first phase are marked here.**

2018-01-04T17:32:54.100+0800: 15305.355: [CMS-concurrent-mark: 0.472/0.472 secs] [Times: user=2.35 sys=0.00, real=0.47 secs] 

**Concurrent marking 耗时0.472秒 cpu time and 0.472秒的wall time 包括其他进程使用的时间片或者自身等待所耗费的(比如IO)。
为什么这里user比real大？因为打日志的时间也算到user中，GC线程又是多线程执行的。 若是GC单线程的情况下，user+sys=real.**

```
关于执行时间的几个说明
       %e     (Not in tcsh.) Elapsed real time (in seconds).
       This is all elapsed time including time slices used by other processes and time the process spends blocked (for example if it is waiting for I/O to complete).
       %S     Total number of CPU-seconds that the process spent in kernel mode.
       This means executing CPU time spent in system calls within the kernel, as opposed to library code, which is still running in user-space. Like ‘user’, this is only CPU time used by the process.
       %U     Total number of CPU-seconds that the process spent in user mode.
       This is only actual CPU time used in executing the process. Other processes and time the process spends blocked do not count towards this figure.
```

2018-01-04T17:32:54.101+0800: 15305.355: [CMS-concurrent-preclean-start]

**Precleaning is also a concurrent phase. Here in this phase we look at the objects in CMS heap which got updated by promotions from young generation or new allocations or got updated by mutators while we were doing the concurrent marking in the previous concurrent marking phase. By rescanning those objects concurrently, the precleaning phase helps reduce the work in the next stop-the-world “remark” phase.**

2018-01-04T17:32:54.104+0800: 15305.358: [CMS-concurrent-preclean: 0.003/0.003 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 

这儿很快。

2018-01-04T17:32:54.104+0800: 15305.358: [CMS-concurrent-abortable-preclean-start]
2018-01-04T17:32:54.104+0800: 15305.358: [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 

在preclean之后，abortable-preclean 环节,也很快速。

2018-01-04T17:32:54.104+0800: 15305.359: [GC[YG occupancy: 1687208 K (2359296 K)] remark环节会打印young区的占用情况。

2018-01-04T17:32:54.104+0800: 15305.359: [Rescan (parallel) , 0.8217640 secs] rescanning work took 0.8217640秒。

2018-01-04T17:32:54.926+0800: 15306.181: [weak refs processing, 0.0000350 secs] weak refs processing cost 0.0000350秒。

2018-01-04T17:32:54.926+0800: 15306.181: [class unloading, 0.0010130 secs] class unloading took 0.0010130秒。

2018-01-04T17:32:54.927+0800: 15306.182: [scrub symbol table, 0.0008320 secs]

2018-01-04T17:32:54.928+0800: 15306.182: [scrub string table, 0.0002420 secs] 

[1 CMS-remark: 1572736K(1572864K)] 3259944K(3932160K), 0.8243590 secs] [Times: user=14.32 sys=0.05, real=0.83 secs] 

**Stop-the-world phase. This phase rescans any residual updated objects in CMS heap, retraces from the roots and also processes Reference objects.  Total 0.83秒完成这一步，主要耗费在`Rescan`。**

2018-01-04T17:32:54.929+0800: 15306.183: [CMS-concurrent-sweep-start]
2018-01-04T17:32:55.470+0800: 15306.725: [CMS-concurrent-sweep: 0.541/0.541 secs] [Times: user=0.54 sys=0.00, real=0.54 secs] 

**开始清理工作，Start of sweeping of dead/non-marked objects. Sweeping is concurrent phase performed with all other threads running.**

2018-01-04T17:32:55.470+0800: 15306.725: [CMS-concurrent-reset-start]
2018-01-04T17:32:55.474+0800: 15306.728: [CMS-concurrent-reset: 0.003/0.003 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 

**Reset阶段,In this phase, the CMS data structures are reinitialized so that a new cycle may begin at a later time. In this case, it took 0.003 secs.**




```
$ date;sudo -u mapp /usr/local/jdk/bin/jstat  -gc -t `pgrep -u mapp java` 1000 
Thu Jan  4 17:32:54 CST 2018
Timestamp        S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT   
        15305.8 262144.0 262144.0  0.0    0.0   2097152.0 1687208.1 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6185  3821.198 3823.919
        15306.8 262144.0 262144.0  0.0    0.0   2097152.0 1687208.1 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6185  3822.022 3824.743
        15307.8 262144.0 262144.0  0.0    0.0   2097152.0 1687208.1 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6185  3822.022 3824.743
        15308.8 262144.0 262144.0  0.0    0.0   2097152.0 1687208.1 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6186  3822.022 3824.743
        15309.8 262144.0 262144.0  0.0    0.0   2097152.0 1687208.1 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6187  3822.686 3825.407
        15310.8 262144.0 262144.0  0.0    0.0   2097152.0 1687208.1 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6187  3823.525 3826.246
        15311.8 262144.0 262144.0  0.0    0.0   2097152.0 1687803.6 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6187  3823.525 3826.246
        15312.8 262144.0 262144.0  0.0    0.0   2097152.0 1687803.6 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6187  3823.525 3826.246
        15313.8 262144.0 262144.0  0.0    0.0   2097152.0 1687803.6 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6188  3824.185 3826.906
        15314.8 262144.0 262144.0  0.0    0.0   2097152.0 1688032.7 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6189  3824.185 3826.906
        15315.8 262144.0 262144.0  0.0    0.0   2097152.0 1688032.7 1572864.0  1572736.4  98304.0 9645.0     55    2.721 6189  3825.012 3827.733
```
