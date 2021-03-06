---
layout: post
title: Linux系统性能剖析工具（一）
---

最近抽空翻了System Performance这本书，内容主要是讲**如何使用工具**去查找Linux系统的性能瓶颈。前面我特意把**“如何使用工具“**这几个字加粗了，除了这就是这本书的核心之外，还有是因为这里面其实含有两层意思：一是字面的意思——怎么使用这些工具以及通过这些个工具能够获取到哪些信息；二是关于工具使用的时间和场合，应该在什么时候使用什么样的工具。我觉得第二点尤为重要，Linux系统经过这么些年的开源演化，伴随着也产生了很多优秀的系统剖析工具，不过对于需要做软件优化的工程师——特别是新人来说，可能就会懵逼了。所以把这些工具梳理出来，系统化叙述成册以便索引是很重要的。

这里我把书中提到的工具并结合一些例子做简单总结，因为篇幅关系及循序渐进的需要，我会分成2~3篇文章陆陆续续的发出来。在此篇里面，主要关注一些基本的命令的用法，如uptime, top, iostat, vmstat等。把它们集中在这里，一个是因为这些命令都比较简单，二是确实非常的常用，可能出问题了第一个想到的就是这些命令了。比如可能首先会用uptime看一下整个系统最近的负载，如果load很高的话然后就会用top去看看哪些任务占用的CPU资源，然后再去检查某一个进程等等。

对于每个命令，我会先贴上在我的机器上执行的输出，然后加以注释，如果有需要的话也会举例加以说明。就此开始吧：  

### uptime  

	b20yang@ubuntu$ uptime   
	23:49:23 up 79 days,  5:13,  2 users,  load average: 0.00, 0.01, 0.05  

最后三个数字分别表示过去1分钟，5分钟，15分钟系统的负载，可以依此判断系统负载最近的变化趋势。  

### top/htop  

	b20yang@ubuntu$ top
	%Cpu(s):  0.2 us,  0.3 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
	KiB Mem:   3880708 total,  3711540 used,   169168 free,  1347968 buffers
	KiB Swap:  2042876 total,   593940 used,  1448936 free.   834140 cached Mem
	
	  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
	 1839 b20yang   20   0 1690320 236636  17060 S   0.7  6.1 453:30.65 compiz
	    1 root      20   0  117196   4452   3148 S   0.0  0.1   1:30.42 systemd
	    2 root      20   0       0      0      0 S   0.0  0.0   0:00.52 kthreadd

相比uptime，top会显示更多的信息，包括进程级别CPU LOAD、交换分区和物理内存的使用率等。可以通过top找到哪些进程占用了比较多的系统资源。那么如果通过top找到了cpu load很高的进程之后，那接下来应该怎么办呢？接下来就应该去分析这个进程的code execution path，找出为什么用了这么多的资源。 htop是top的高级版本。  

### mpstat  

	b20yang@ubuntu$ mpstat -P ALL 1
	02:24:11 PM  CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
	02:24:12 PM  all    0.00    0.00    0.07    0.00    0.00    0.00    0.00   99.93    618.00
	02:24:12 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00      0.00
	02:24:12 PM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00      0.00
	02:24:12 PM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00 4294967288.00
	02:24:12 PM    3    0.00    0.00    1.98    0.00    0.00    0.00    0.00   98.02      0.00
	02:24:12 PM    4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00      0.00
	02:24:12 PM    5    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00      0.00
	02:24:12 PM    6    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00      0.00
	02:24:12 PM    7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00      0.00
	02:24:12 PM    8    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00      0.00
	02:24:12 PM    9    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00      0.00
	02:24:12 PM   10    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00      0.00
	02:24:12 PM   11    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00      0.00


用这个命令可以很方便的看到CPU各个核心之间负载是否均衡。  


### iostat
	b20yang@ubuntu$ iostat 1
	avg-cpu:  %user   %nice %system %iowait  %steal   %idle
	           0.00    0.00    0.00    0.00    0.00  100.00
	
	Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
	sda               0.00         0.00         0.00          0          0
	sdb               0.00         0.00         0.00          0          0

iostat主要用来检查系统输入输出设备的使用状态，比如磁盘的tps等。如果我们在一个终端执行**“dd bs=1014 count=1024k if=/dev/zero of=/tmp/tmp.bin”**，同时在另外一个终端上面执行命令iostat 1，那么可以看到sda上面的kB_wrtn/s急剧上涨。  

	b20yang@ubuntu$ iostat 1
	avg-cpu:  %user   %nice %system %iowait  %steal   %idle
	           0.51    0.00    0.00   95.96    0.00    3.54
	
	Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
	sda              73.00         4.00     73728.00          4      73728
	

### vmstat

	b20yang@ubuntu$ vmstat 1
	procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
	 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
	 0  0 613484 1236068 382688 750232    0    0     0     0   66  166  0  0 99  0  0
	 0  0 613484 1236068 382688 750232    0    0     0     0   64  178  0  0 100  0  0
	 0  0 613480 1236068 382688 750232   12    0    12     0   61  158  1  0 100  0  0
	 0  0 613480 1236068 382688 750232    0    0     0     0   52  126  0  1 100  0  0

vmstat主要用来检查系统内存，交换分区、磁盘块设备等状态。类似的，也可以看看命令**“dd bs=1014 count=1024k if=/dev/zero of=/tmp/tmp.bin”**是如何影响vmstat的。  

	b20yang@ubuntu$ vmstat 1
	procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
	 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
	 3  2 611476 1163584 389952 822688    0    0     5    16    1    1  0  0 99  0  0
	 0  1 611476 1078792 389956 907204    0    0     4 58816  268  352  1  1 25 73  0
	 1  3 611476 1006452 389964 979660    0    0     0 49240  227  267  2  9 15 75  0
	 1  2 611476 965156 389976 1020360    0    0     4 122084  215  293  1  4  0 95  0
	 1  1 611476 828852 389976 1153044    0    0     0 81920  293  414  0  1 17 82  0
	 0  1 611476 748700 389984 1230996    0    0     8 140064  245  276  2  7 31 60  0
	 0  2 611476 687552 389988 1289972    0    0     4 131072  190  271  1  7 10 82  0
	 0  2 611476 687392 389992 1290024    0    0     4 131072  120 8305  1  2  0 98  0

