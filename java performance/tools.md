# Java性能调优工具箱



## 1. 操作系统的工具和分析

- 性能的起点与Java无关，所以性能测试收集操作系统的数据，至少收集CPU、内存、磁盘使用率的信息，如果程序使用网络还应该收集网络使用率。

- 如果是自动化性能测试，还需要依靠命令行工具。可以捕获输出后，分析时再将输出图形化。



### 1.1 cpu使用率

  - 性能调优的目的是在尽短的时间让CPU使用率尽可能的高。

  - linux系统使用vmstat、top

  - Mac 系统使用vm_stat

  - Java在单CPU情况下优化使减少请求响应时间，同时也使系统能够承担更多的负载，最终提高CPU使用率。

  - Java在多CPU的使用率情况下，重点考虑CPU空闲的情形，多CPU多线程的目的是通过不阻塞线程来提高CPU使用率，或者是在线程完成工作等待更多任务时降低CPU使用率，当线程被某个任务阻塞时，它就没法执行新的任务，所以会导致有任务需要执行，却没有线程执行，结果就是CPU处于空闲时间。这种情况需要加大线程池或者考虑锁或外部资源瓶颈。



**vmstat是Virtual Meomory Statistics（虚拟内存统计）的缩写，可对操作系统的虚拟内存、进程、IO读写、CPU活动等进行监视。它是对系统的整体情况进行统计，不足之处是无法对某个进程进行深入分析。 **

指令所在路径：/usr/bin/vmstat

**输出字段意义：**

**Procs**

- r: The number of processes waiting for run time.

​       等待运行的进程数。如果等待运行的进程数越多，意味着CPU非常繁忙。另外，如果该参数长期大于和等于逻辑cpu个数，则CPU资源可能存在较大的瓶颈。

- b: The number of processes in uninterruptible sleep.

​       处在非中断睡眠状态的进程数。意味着进程被阻塞。主要是指被资源阻塞的进程对列数（比如IO资源、页面调度等），当这个值较大时，需要根据应用程序来进行分析，比如数据库产品，中间件应用等。

**Memory**

- swpd: the amount of virtual memory used.

​       已使用的虚拟内存大小。如果虚拟内存使用较多，可能系统的物理内存比较吃紧，需要采取合适的方式来减少物理内存的使用。swapd不为0，并不意味物理内存吃紧，如果swapd没变化，si、so的值长期为0,这也是没有问题的     

- free: the amount of idle memory.

​       空闲的物理内存的大小

-  buff: the amount of memory used as buffers.

​       用来做buffer（缓存，主要用于块设备缓存）的内存数，单位：KB

- cache: the amount of memory used as cache.

​       用来做cache（缓存，主要用于缓存文件）的内存，单位：KB

- inact: the amount of inactive memory. (-a option)

​       inactive memory的总量

- active: the amount of active memory. (-a option)

​       active memroy的总量。

**Swap**

- si: Amount of memory swapped in from disk (/s).
​        从磁盘交换到**swap**虚拟内存的交换页数量，单位：KB/秒。如果这个值大于**0**，表示物理内存不够用或者内存泄露了  

- so: Amount of memory swapped to disk (/s).
​        从**swap**虚拟内存交换到磁盘的交换页数量，单位：KB/秒，如果这个值大于**0**，表示物理内存不够用或者内存泄露了

​       内存够用的时候，这2**个值都是**0，如果这2个值长期大于0时，系统性能会受到影响，磁盘IO和CPU资源都会被消耗。当看到空闲内存（free）很少的或接近于0时，就认为内存不够用了，这个是不正确的。不能光看这一点，还要结合si和so，如果free很少，但是si和so也很少（大多时候是0），那么不用担心，系统性能这时不会受到影响的。

​	当内存的需求大于RAM的数量，服务器启动了虚拟内存机制，通过虚拟内存，可以将RAM段移到SWAP DISK的特殊磁盘段上，这样会 出现虚拟内存的页导出和页导入现象，页导出并不能说明RAM瓶颈，虚拟内存系统经常会对内存段进行页导出，但页导入操作就表明了服务器需要更多的内存了， 页导入需要从SWAP DISK上将内存段复制回RAM，导致服务器速度变慢。

**IO**
- bi: Blocks received from a block device (blocks/s).
​     每秒从块设备接收到的块数，单位：块/秒 也就是读块设备。

- bo: Blocks sent to a block device (blocks/s).

​       每秒发送到块设备的块数，单位：块/秒  也就是写块设备。

**System**

- in: The number of interrupts per second, including the clock.

​       每秒的中断数，包括时钟中断

- cs: The number of context switches per second.

​       每秒的环境（上下文）切换次数。比如我们调用系统函数，就要进行上下文切换，而过多的上下文切换会浪费较多的cpu资源，这个数值应该越小越好。

**CPU**

These are percentages of total CPU time.

- us: Time spent running non-kernel code. (user time, including nice time)


​        用户CPU时间(非内核进程占用时间)（单位为百分比）。 us的值比较高时，说明用户进程消耗的CPU时间多
​    
-  sy: Time spent running kernel code. (system time)


​        系统使用的CPU时间（单位为百分比）。sy的值高时，说明系统内核消耗的CPU资源多，这并不是良性表现，我们应该检查原因。
​    
-  id: Time spent idle. Prior to Linux 2.5.41, this includes IO-wait time.

    空闲的CPU的时间(百分比)，在Linux 2.5.41之前，这部分包含IO等待时间。

- wa: Time spent waiting for IO. Prior to Linux 2.5.41, shown as zero.

​        等待IO的CPU时间，在Linux 2.5.41之前，这个值为0 .这个指标意味着CPU在等待硬盘读写操作的时间，用百分比表示。wait越大则机器io性能就越差。说明IO等待比较严重，这可能由于磁盘大量作随机访问造成，也有可能磁盘出现瓶颈（块操作）。

- st: Time stolen from a virtual machine. Prior to Linux 2.6.11, unknown.


**使用示例：**

1: 查看vmstat命令的帮助信息

```bash
man vmstat
```

2: 显示活动(active)与非活动(inactive)的内存

```bash

    #vmstat -a 2 10
    procs   -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
     r  b   swpd   free  inact active   si   so    bi    bo   in   cs us sy id wa st
     0  0 242752  56264 1294680 2365840    0    0     1    18    2    2  0  2 97  0  0
     1  0 242752  56504 1294676 2365736    0    0     0     0 1010  511  0  1 100 0  0
     0  0 242752  55844 1294716 2366616    0    0     0    16 1011  768  1  5 94  0  0
     0  0 242752  56760 1294716 2365888    0    0     0   190 1015  554  0  1 99  0  0
     0  0 242752  55472 1294744 2366636    0    0     0     0 1007  751  1  6 94  0  0
     0  0 242752  56636 1294748 2365904    0    0     0    16 1009  554  0  1 99  0  0
     0  0 242752  55844 1294772 2366656    0    0     0   178 1020  746  1  6 93  0  0
     0  0 242752  56884 1294768 2365940    0    0     0     0 1007  543  0  1 99  0  0
     1  0 242752  55208 1294816 2367220    0    0     0   206 1021  726  0  4 95  0  0
     0  0 242752  56760 1294796 2365960    0    0     0    16 1009  606  0  2 98  0  0
```

3：不加任何参数，vmstat命令只输出一条记录，这个数据是自系统上次重启之后到现在的平均数值。

```bash
    #vmstat
    procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
     r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
     0  0 242752  32496 112680 2724840    0    0     1    18    3    2  0  2 97  0  0
```

4：显示各种事件计数器表和内存统计信息，这显示不重复。

```bash
     #vmstat -s
         33011144  total memory
         32799072  used memory
         24606736  active memory
          6175700  inactive memory
           212072  free memory
            52288  buffer memory
         30158708  swap cache
         12582904  total swap
           610348  used swap
         11972556  free swap
         44159969 non-nice user cpu ticks
             8172 nice user cpu ticks
          6077972 system cpu ticks
        389217442 idle cpu ticks
         40807984 IO-wait cpu ticks
           123964 IRQ cpu ticks
           383333 softirq cpu ticks
                0 stolen cpu ticks
      10331447387 pages paged in
       2287459081 pages paged out
          1524480 pages swapped in
          1433512 pages swapped out
       2358479992 interrupts
       1876082783 CPU context switches
       1481100317 boot time
         15573677 forks
```

5：可以扩大字段长度，当内存较大时，默认长度不够完全展示内存时，会导致字段值偏移，导致查看不便

```bash
    #vmstat -w 2 5
    procs -------------------memory------------------ ---swap-- -----io---- --system-- -----cpu-------
     r  b       swpd       free       buff      cache   si   so    bi    bo   in   cs  us sy  id wa st
     0  0     243852      73556     110908    2678492    0    0     1    18    3    3   0  2  97  0  0
     0  0     243852      72252     110916    2678484    0    0     0   172 1016  701   0  4  95  0  0
     0  0     243852      73556     110916    2678544    0    0     0     0 1005  636   0  2  98  0  0
     0  0     243852      72004     110916    2678540    0    0     0    16 1005  694   0  5  95  0  0
     0  0     243852      73432     110924    2678580    0    0     0   192 1015  629   0  2  98  0  0

    如下所示，由于有些字段值较大，导致出现下面偏移，不便查看。
    # vmstat  2 5
    procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
     r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
     0  0 243852  79284 110928 2678640    0    0     1    18    3    3  0  2 97  0  0
     0  0 243852  78988 110928 2678648    0    0     0     0 1006  753  0  5 95  0  0
     0  0 243852  80400 110936 2678696    0    0     0   194 1015  565  0  1 99  0  0
     0  0 243852  78352 110936 2678748    0    0     0    16 1008  680  0  5 95  0  0
     0  0 243852  79532 110936 2678748    0    0     0     0 1007  669  0  2 98  0  0shell
    [root@DB-Server ~]#
```

6:显示磁盘分区数据（disk partition statistics ）

```bash
    #vmstat -p sdc5 2 10
    sdc5          reads   read sectors  writes    requested writes
                54270570 7234336956    8939045  276196850
                54270570 7234336956    8939045  276196850
                54270570 7234336956    8939050  276196978
                54270570 7234336956    8939053  276197074
                54270574 7234337260    8939053  276197074
                54270577 7234337292    8939066  276197346
                54270622 7234339700    8939066  276197346
                54270622 7234339700    8939069  276197442
                54270859 7234342828    8939078  276197634
                54271074 7234345452    8939080  276197666
```

#### 1.1.2 Linux系统 top命令可查看系统的CPU、内存、运行时间、交换分区、执行的线程等信息。通过top命令可以有效的发现系统的缺陷出在哪里。是内存不够、CPU处理能力不够、IO读写过高…

```bash
  #top

  top - 21:31:26 up 15:16,  5 users,  load average: 0.61, 0.82, 0.75
  Tasks: 240 total,   2 running, 238 sleeping,   0 stopped,   0 zombie
  %Cpu(s): 13.7 us,  1.5 sy,  0.0 ni, 84.2 id,  0.6 wa,  0.0 hi,  0.0 si,  0.0 st
  KiB Mem :  3775264 total,   250100 free,  2495300 used,  1029864 buff/cache
  KiB Swap:  4064252 total,  2789544 free,  1274708 used.   527664 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND    
  16507 kiosk     20   0 1935284 201988  10816 R  46.8  5.4  68:11.92 plugin-con+
  15773 kiosk     20   0 1784208 497692  40776 S   4.7 13.2  37:05.32 firefox    
    408 root      20   0   36940   4116   3920 S   3.0  0.1   4:51.67 systemd-jo+
   3789 kiosk     20   0  747664  14124   4696 S   2.0  0.4   2:49.76 gnome-term+
   2404 root      20   0  439488 106688  84580 S   1.7  2.8  16:08.35 Xorg       
   2662 kiosk      9 -11  700096   5232   3032 S   1.7  0.1   5:17.25 pulseaudio
  21632 kiosk     20   0  812940 167440  30100 S   1.7  4.4  20:15.48 wps        
   2688 kiosk     20   0 2111764 218776  18580 S   1.3  5.8  20:25.33 gnome-shell
    663 root      20   0  399976   3352   2984 S   1.0  0.1   0:46.92 rsyslogd   
   7349 qemu      20   0 1697464 956932    556 S   0.7 25.3   5:03.80 qemu-kvm   
   7803 qemu      20   0 1697460 708164    544 S   0.7 18.8   4:16.74 qemu-kvm   
     18 root      20   0       0      0      0 S   0.3  0.0   0:16.94 rcuos/0    
     19 root      20   0       0      0      0 S   0.3  0.0   0:18.43 rcuos/1    
     21 root      20   0       0      0      0 S   0.3  0.0   0:19.62 rcuos/3    
    671 root      20   0  207984    160    120 S   0.3  0.0   0:01.60 abrt-watch+
   5676 root      20   0       0      0      0 S   0.3  0.0   0:00.28 kworker/u1+
      1 root      20   0  189128   2900   1432 S   0.0  0.1   0:06.11 systemd    
```

  top命令的第一行：

```bash
  top - 21:31:26 up 15:16,  5 users,  load average: 0.61, 0.82, 0.75
```

  依次对应：系统当前时间 up 系统到目前为止i运行的时间， 当前登陆系统的用户数量， load average后面的三个数字分别表示距离现在一分钟，五分钟，十五分钟的负载情况。
  这行信息与命令uptime显示的信息相同
  注意：load average数据是每隔5秒钟检查一次活跃的进程数，然后按特定算法计算出的数值。如果这个数除以逻辑CPU的数量，结果高于5的时候就表明系统在超负荷运转了。

  top命令的第二行：

  ```bash
  Tasks: 240 total,   2 running, 238 sleeping,   0 stopped,   0 zombie
  ```

  依次对应：tasks表示任务（进程），240 total则表示现在有240个进程，其中处于运行中的有2个，238个在休眠（挂起），stopped状态即停止的进程数为0，zombie状态即僵尸的进程数为0个。

  top命令的第三行，cpu状态：

```bash
  %Cpu(s): 13.7 us,  1.5 sy,  0.0 ni, 84.2 id,  0.6 wa,  0.0 hi,  0.0 si,  0.0 st
```

  依次对应：
  us:user 用户空间占用cpu的百分比
  sy:system 内核空间占用cpu的百分比
  ni:niced 改变过优先级的进程占用cpu的百分比
  空闲cpu百分比
  wa:IO wait IO等待占用cpu的百分比
  hi:Hardware IRQ 硬中断 占用cpu的百分比
  si:software 软中断 占用cpu的百分比
  st:被hypervisor偷去的时间

  top命令第四行，内存状态：

```bash
  KiB Mem :  3775264 total,   250100 free,  2495300 used,  1029864 buff/cache
```

  依次对应：物理内存总量（3.7G),空闲内存总量（2.5G),使用中的内存总量（2.4G),缓冲内存量
  第四行中使用中的内存总量（used）指的是现在系统内核控制的内存数，空闲内存总量（free）是内核还未纳入其管控范围的数量。纳入内核管理的内存不见得都在使用中，还包括过去使用过的现在可以被重复利用的内存，内核并不把这些可被重新使用的内存交还到free中去，因此在linux上free内存会越来越少，但不用为此担心
  top命令第五行，swap交换分区：

```bash
  KiB Swap:  4064252 total,  2789544 free,  1274708 used.   527664 avail Mem
```

  依次对应：交换区总量（4G），空闲交换区总量（2.7G),使用的交换区总量（1.2G），可用交换取总量

  对于内存监控，在top里我们要时刻监控第五行swap交换分区的used，如果这个数值在不断的变化，说明内核在不断进行内存和swap的数据交换，这是真正的内存不够用了。

  top命令第六行是空行

  top命令第七行，各进程的监控：

```bash
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND    
```

  依次对应：
  PID — 进程id
  USER — 进程所有者
  PR — 进程优先级
  NI — nice值。负值表示高优先级，正值表示低优先级
  VIRT — 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
  RES — 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
  SHR — 共享内存大小，单位kb
  S — 进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程
  %CPU — 上次更新到现在的CPU时间占用百分比
  %MEM — 进程使用的物理内存百分比
  TIME+ — 进程使用的CPU时间总计，单位1/100秒
  COMMAND — 进程名称（命令名/命令行）

  二.交互命令
  1.1 ‘h’ 帮助
  top命令进入视图后，键入h会显示如下界面，显示交互命令的帮助菜单

  ```bash
  Help for Interactive Commands - procps-ng version 3.3.10
Window 1:Def: Cumulative mode Off.  System: Delay 3.0 secs; Secure mode Off.

  Z,B,E,e   Global: 'Z' colors; 'B' bold; 'E'/'e' summary/task memory scale
  l,t,m     Toggle Summary: 'l' load avg; 't' task/cpu stats; 'm' memory info
  0,1,2,3,I Toggle: '0' zeros; '1/2/3' cpus or numa node views; 'I' Irix mode
  f,F,X     Fields: 'f'/'F' add/remove/order/sort; 'X' increase fixed-width

  L,&,<,> . Locate: 'L'/'&' find/again; Move sort column: '<'/'>' left/right
  R,H,V,J . Toggle: 'R' Sort; 'H' Threads; 'V' Forest view; 'J' Num justify
  c,i,S,j . Toggle: 'c' Cmd name/line; 'i' Idle; 'S' Time; 'j' Str justify
  x,y     . Toggle highlights: 'x' sort field; 'y' running tasks
  z,b     . Toggle: 'z' color/mono; 'b' bold/reverse (only if 'x' or 'y')
  u,U,o,O . Filter by: 'u'/'U' effective/any user; 'o'/'O' other criteria
  n,#,^O  . Set: 'n'/'#' max tasks displayed; Show: Ctrl+'O' other filter(s)
  C,...   . Toggle scroll coordinates msg for: up,down,left,right,home,end

  k,r       Manipulate tasks: 'k' kill; 'r' renice
  d or s    Set update interval
  W,Y       Write configuration file 'W'; Inspect other output 'Y'
  q         Quit
          ( commands shown with '.' require a visible task display window )
Press 'h' or '?' for help with Windows,
Type 'q' or <Esc> to continue
  ```

  1.2 敲ENTER或者 SPACE键: 刷新显示

  1.3 A’: 切换交替显示模式
  top命令视图下，键入‘A‘显示如下：

```bash
1:Def - 01:05:09 up 9 min,  2 users,  load average: 0.00, 0.03, 0.05
Tasks:  87 total,   1 running,  86 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3914860 total,  3656516 free,    86008 used,   172336 buff/cache
KiB Swap:   524284 total,   524284 free,        0 used.  3611676 avail Mem

1  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 10366 root      20   0  148128   1984   1432 R   0.3  0.1   0:00.07 top
     1 root      20   0  123240   3772   2576 S   0.0  0.1   0:01.10 systemd
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd
     3 root      20   0       0      0      0 S   0.0  0.0   0:00.13 ksoftirqd/0
     5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
2  PID  PPID     TIME+  %CPU %MEM  PR  NI S    VIRT    RES   UID COMMAND
 10366 10343   0:00.07   0.3  0.1  20   0 R  148128   1984     0 top
 10343 10341   0:00.01   0.0  0.1  20   0 S  115436   2032     0 bash
 10341 10178   0:00.05   0.0  0.2  20   0 S  152544   5876     0 sshd
 10337     1   0:00.00   0.0  0.0  20   0 S  121168    748     0 anacron
 10324     2   0:00.00   0.0  0.0  20   0 S       0      0     0 kworker/0:0
3  PID %MEM    VIRT    RES   CODE    DATA    SHR nMaj nDRT  %CPU COMMAND
   636  0.4  555644  16388      4  304324   5800   53    0   0.0 tuned
 10341  0.2  152544   5876    800     944   4572    0    0   0.0 sshd
 10178  0.1  108160   4136    800     760   3160    0    0   0.0 sshd
     1  0.1  123240   3772   1408   83088   2576   35    0   0.0 systemd
   343  0.1   39068   3288    320     360   2968    7    0   0.0 systemd-journal
4  PID  PPID   UID USER     RUSER    TTY          TIME+  %CPU %MEM S COMMAND
     1     0     0 root     root     ?          0:01.10   0.0  0.1 S systemd
     2     0     0 root     root     ?          0:00.00   0.0  0.0 S kthreadd
     3     2     0 root     root     ?          0:00.13   0.0  0.0 S ksoftirqd/0
     5     2     0 root     root     ?          0:00.00   0.0  0.0 S kworker/0:0H
     6     2     0 root     root     ?          0:00.01   0.0  0.0 S kworker/u4:0
```



显示4个窗口：Def （默认字段组）
  Job （任务字段组）
  Mem （内存字段组）
  Usr （用户字段组）
  四组字段共有一个独立的可配置的概括区域和它自己的可配置任务区域。4个窗口中只有一个窗口是当前窗口。当前窗口的名称显示在左上方。（注：只有当前窗口才会接受你键盘交互命令）
  我们可以用’a’和’w’在4个 窗口间切换。’a’移到后一个窗口，’w’移到前一个窗口。用’g’命令你可以输入一个数字来选择当前窗口。
  在键入‘A‘后在键入‘a‘的显示如下：

```bash
2:Job - 01:14:58 up 19 min,  2 users,  load average: 0.00, 0.01, 0.05
Tasks:  87 total,   1 running,  86 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.2 sy,  0.0 ni, 99.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3914860 total,  3656080 free,    86320 used,   172460 buff/cache
KiB Swap:   524284 total,   524284 free,        0 used.  3611280 avail Mem

1  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
     1 root      20   0  123240   3776   2580 S   0.0  0.1   0:01.10 systemd
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd
     3 root      20   0       0      0      0 S   0.0  0.0   0:00.13 ksoftirqd/0
     5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
     6 root      20   0       0      0      0 S   0.0  0.0   0:00.01 kworker/u4:0
2  PID  PPID     TIME+  %CPU %MEM  PR  NI S    VIRT    RES   UID COMMAND
 10369 10343   0:00.25   0.0  0.1  20   0 R  148128   1980     0 top
 10343 10341   0:00.01   0.0  0.1  20   0 S  115436   2032     0 bash
 10341 10178   0:00.08   0.0  0.2  20   0 S  152544   5876     0 sshd
 10337     1   0:00.00   0.0  0.0  20   0 S  121168    748     0 anacron
 10324     2   0:00.00   0.0  0.0  20   0 S       0      0     0 kworker/0:0
3  PID %MEM    VIRT    RES   CODE    DATA    SHR nMaj nDRT  %CPU COMMAND
   636  0.4  555644  16388      4  304324   5800   53    0   0.0 tuned
 10341  0.2  152544   5876    800     944   4572    0    0   0.0 sshd
 10178  0.1  108160   4136    800     760   3160    0    0   0.0 sshd
     1  0.1  123240   3776   1408   83088   2580   35    0   0.0 systemd
   343  0.1   39068   3304    320     360   2984    7    0   0.0 systemd-journal
4  PID  PPID   UID USER     RUSER    TTY          TIME+  %CPU %MEM S COMMAND
     1     0     0 root     root     ?          0:01.10   0.0  0.1 S systemd
     2     0     0 root     root     ?          0:00.00   0.0  0.0 S kthreadd
     3     2     0 root     root     ?          0:00.13   0.0  0.0 S ksoftirqd/0
     5     2     0 root     root     ?          0:00.00   0.0  0.0 S kworker/0:0H
     6     2     0 root     root     ?          0:00.01   0.0  0.0 S kworker/u4:0
```


  1.4 ‘B’: 触发粗体显示
  一些重要信息会以加粗字体显示。这个命令可以切换粗体显示

  1.5 ‘d’ 或‘s’: 设置显示的刷新间隔
  当键下’d’或’s’时，你将被提示输入一个值（以秒为单位），它会以设置的值作为刷新间隔。如果你这里输入了6.0，top将会每秒刷新


  1.6 ‘l’、‘t’、‘m’: 切换负载、任务、内存信息的显示

  1.7 ‘f’: 字段管理
  用于选择你想要显示的字段。用’*’标记的是已选择的。

```bash
Fields Management for window 2:Job, whose current sort field is PID
   Navigate with Up/Dn, Right selects for move then <Enter> or Left commits,
   'd' or <Space> toggles display, 's' sets sort.  Use 'q' or <Esc> to end!

* PID     = Process Id             P       = Last Used Cpu (SMP)
* PPID    = Parent Process pid     TIME    = CPU Time
* TIME+   = CPU Time, hundredths   CODE    = Code Size (KiB)
* %CPU    = CPU Usage              DATA    = Data+Stack (KiB)
* %MEM    = Memory Usage (RES)     nMaj    = Major Page Faults
  USER    = Effective User Name    nMin    = Minor Page Faults
* PR      = Priority               nDRT    = Dirty Pages Count
* NI      = Nice Value             WCHAN   = Sleeping in Function
* S       = Process Status         Flags   = Task Flags <sched.h>
* VIRT    = Virtual Image (KiB)    CGROUPS = Control Groups
* RES     = Resident Size (KiB)    SUPGIDS = Supp Groups IDs
  SHR     = Shared Memory (KiB)    SUPGRPS = Supp Groups Names
  SWAP    = Swapped Size (KiB)     TGID    = Thread Group Id
* UID     = Effective User Id      ENVIRON = Environment vars
* COMMAND = Command Name/Line      vMj     = Major Faults delta
  RUID    = Real User Id           vMn     = Minor Faults delta
  RUSER   = Real User Name         USED    = Res+Swap Size (KiB)
  SUID    = Saved User Id          nsIPC   = IPC namespace Inode
  SUSER   = Saved User Name        nsMNT   = MNT namespace Inode
  GID     = Group Id               nsNET   = NET namespace Inode
  GROUP   = Group Name             nsPID   = PID namespace Inode
  PGRP    = Process Group Id       nsUSER  = USER namespace Inode
  TTY     = Controlling Tty        nsUTS   = UTS namespace Inode
  TPGID   = Tty Process Grp Id
  SID     = Session Id
  nTH     = Number of Threads
```

  上下光标键在字段内导航，左光标键可以选择字段，回车或右光标键确认。

  按’<’移动已排序的字段到左边，’>’则移动到右边。

  1.8 ‘R’: 反向排序

  1.9 ‘c’: 触发命令
  切换是否显示进程启动时的完整路径和程序名。

```bash
top - 01:25:09 up 29 min,  2 users,  load average: 0.00, 0.01, 0.05
Tasks:  87 total,   1 running,  86 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  0.0 sy,  0.0 ni, 99.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3914860 total,  3655948 free,    86436 used,   172476 buff/cache
KiB Swap:   524284 total,   524284 free,        0 used.  3611156 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    3 root      20   0       0      0      0 S   0.0  0.0   0:00.13 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    6 root      20   0       0      0      0 S   0.0  0.0   0:00.01 kworker/u4:0
    7 root      rt   0       0      0      0 S   0.0  0.0   0:00.42 migration/0
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
    9 root      20   0       0      0      0 S   0.0  0.0   0:00.14 rcu_sched
   10 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 lru-add-drain
   11 root      rt   0       0      0      0 S   0.0  0.0   0:00.01 watchdog/0
   12 root      rt   0       0      0      0 S   0.0  0.0   0:00.01 watchdog/1
   13 root      rt   0       0      0      0 S   0.0  0.0   0:00.03 migration/1
   14 root      20   0       0      0      0 S   0.0  0.0   0:00.00 ksoftirqd/1
   15 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kworker/1:0
   16 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/1:0H
   18 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kdevtmpfs
   19 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 netns
   20 root      20   0       0      0      0 S   0.0  0.0   0:00.00 khungtaskd
   21 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 writeback
   22 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kintegrityd
   23 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 bioset
   24 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 bioset
   25 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 bioset
   26 root       0 -20       0      0      0 S   0.0  0.0   0:00.01 kblockd
   27 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 md
```

  键入‘c’后显示：

```bash
top - 01:23:44 up 28 min,  2 users,  load average: 0.00, 0.01, 0.05
Tasks:  87 total,   1 running,  86 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  0.0 sy,  0.0 ni, 99.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3914860 total,  3656196 free,    86192 used,   172472 buff/cache
KiB Swap:   524284 total,   524284 free,        0 used.  3611404 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    6 root      20   0       0      0      0 S   0.0  0.0   0:00.01 [kworker/u4:0]
    7 root      rt   0       0      0      0 S   0.0  0.0   0:00.42 [migration/0]
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 [rcu_bh]
    9 root      20   0       0      0      0 S   0.0  0.0   0:00.14 [rcu_sched]
   10 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [lru-add-drain]
   11 root      rt   0       0      0      0 S   0.0  0.0   0:00.01 [watchdog/0]
   12 root      rt   0       0      0      0 S   0.0  0.0   0:00.01 [watchdog/1]
   13 root      rt   0       0      0      0 S   0.0  0.0   0:00.03 [migration/1]
   14 root      20   0       0      0      0 S   0.0  0.0   0:00.00 [ksoftirqd/1]
   15 root      20   0       0      0      0 S   0.0  0.0   0:00.00 [kworker/1:0]
   16 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [kworker/1:0H]
   18 root      20   0       0      0      0 S   0.0  0.0   0:00.00 [kdevtmpfs]
   19 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [netns]
   20 root      20   0       0      0      0 S   0.0  0.0   0:00.00 [khungtaskd]
   21 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [writeback]
   22 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [kintegrityd]
   23 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [bioset]
   24 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [bioset]
   25 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [bioset]
   26 root       0 -20       0      0      0 S   0.0  0.0   0:00.01 [kblockd]
   27 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [md]
   28 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 [edac-poller]
   29 root      20   0       0      0      0 S   0.0  0.0   0:04.48 [kworker/1:1]
```

  2.0 ‘i’: 空闲任务

```bash
top - 01:25:48 up 30 min,  2 users,  load average: 0.00, 0.01, 0.05
Tasks:  87 total,   1 running,  86 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3914860 total,  3655452 free,    86908 used,   172500 buff/cache
KiB Swap:   524284 total,   524284 free,        0 used.  3610664 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
```

  切换显示空闲任务


### 1.2 cpu运行队列

  - 监控运行队列对于分辨系统是否满负载也有重大意义，当运行队列长度是虚拟处理器的4倍或更多时说明系统响应已经非常迟缓了。

  - 操作系统都可以监控线程数，Unix系统称为运行队列(run queue),windows系统使用typeperf，linux vmstat。

  - windows使用该命令：

    ```bash
    typeperf -si 1 "\System\Processor Queue Length"
    ```

- linux vmstat输出的第一行是队列长度

  ```bash
  procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
   r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
   1  0      0 3656064    872 171620    0    0    29    10   55   53  1  0 99  0  0
  ```
### 1.3 内存使用率

- Windows系统使用 该命令：

```bash
  typeperf -si 1 "\System\Processor Queue Length"
```

- Linux 可以使用vmstat或者top查看


### 1.4 监控锁竞争上下文切换

- Linux系统使用sysstat包的pidstat查看

```bash

pidstat  -w -I -p 1 5
Linux 4.9.125-linuxkit (372caddfa2a9) 	07/12/19 	_x86_64_	(4 CPU)

21:37:55      UID       PID   cswch/s nvcswch/s  Command
21:38:00        0         1      0.00      0.00  run.sh
21:38:05        0         1      0.00      0.00  run.sh
21:38:10        0         1      0.00      0.00  run.sh

```

### 1.5 磁盘使用率

-  Linux系统使用iostat

```bash

  iostat  -xm 5
  Linux 4.9.125-linuxkit (372caddfa2a9) 	07/12/19 	_x86_64_	(4 CPU)
  
  avg-cpu:  %user   %nice %system %iowait  %steal   %idle
             0.21    0.00    0.32    0.03    0.00   99.44
  
  Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
  sda               0.00     2.16    0.84    2.14     0.03     4.89  3381.68     0.01    4.45    0.42    6.04   0.51   0.15
  scd0              0.00     0.00    0.19    0.00     0.01     0.00   135.40     0.00    0.57    0.57    0.00   0.45   0.01
  scd1              0.00     0.00    0.00    0.00     0.00     0.00    51.00     0.00    0.00    0.00    0.00   0.00   0.00
  scd2              0.00     0.00    0.27    0.00     0.02     0.00   189.19     0.00    0.66    0.66    0.00   0.54   0.01
  
```

### 1.6 网络使用率

- Linux使用nicstat

```bash

 nicstat 5
    Time      Int   rKB/s   wKB/s   rPk/s   wPk/s    rAvs    wAvs %Util    Sat
17:48:21     eth0    5.56    0.14    4.20    1.68  1357.2   83.94  0.00   0.00
17:48:26     eth0    0.03    0.07    0.40    0.40   66.00   182.0  0.00   0.00
17:48:31     eth0    0.01    0.04    0.20    0.20   66.00   182.0  0.00   0.00
17:48:36     eth0    0.01    0.04    0.20    0.20   66.00   182.0  0.00   0.00

Time	#抽样结束的时间  
Int	#网卡名 
rKB/s,InKB	#每秒读的千字节数(received) 
wKB/s,OutKB	#每秒写的千字节数(transmitted) 
rMbps,RdMbps	#每秒读的百万字节数K(received) 
wMbps,WrMbps	#每秒写的百万字节数M(transmitted) 
rPk/s,InSeg,InDG	#每秒读的数据包 
wPk/s,OutSeg,OutDG #每秒写的数据包 
rAvs	#平均读的数据包大小
wAvs	#平均写的数据包大小 
%Util	#接口的利用率百分比 
Sat	#每秒的错误数，接口接近饱和的一个指标

```

## 2. JAVA监控工具和分析

- 这里只做个简单介绍，后续会把各个工具的使用案例完善好。

### 2.1 jcmd

- 用于打印Java进程所涉及的基本类、线程和VM信息

- jcmd 是JDK1.7之后推出的，实现了jmap大部分功能，官方也建议使用jcmd替代jmap

```bash

  jcmd -l #查看所有java 进程
  jcmd process_id help <command> #查看特定命令语法或所有语法
  
  VM.native_memory #堆外内存分析
  ManagementAgent.stop #关闭监控代理
  ManagementAgent.start_local #启动本地jmx、jdp监控代理
  ManagementAgent.start #启动远程jmx、jdp监控代理
  VM.classloader_stats #jvm类加载状态
  GC.rotate_log #gc滚动日志
  Thread.print #打印线程堆栈信息
  GC.class_stats #gc类状态
  GC.class_histogram #gc类统计信息
  GC.heap_dump #生成Java堆转储快照文件
  GC.finalizer_info #显示在F-Queue中等待Finalizer线程执行finalize方法的对象
  GC.heap_info #显示堆信息
  GC.run_finalization #执行System.runFinalization()
  GC.run #执行System.gc()
  VM.uptime #运行时长
  VM.dynlibs #显示所有加载的动态库
  VM.flags #调优标志
  VM.system_properties #系统属性
  VM.command_line #jvm命令行
  VM.version #jvm版本

```

### 2.2 jconsole

-  提供JVM的图形视图，包括线程使用、类的使用和GC活动。

### 2.3 jhat

- 读取内存堆转储。

### 2.4 jmap


- 提供堆转储和其它JVM内存使用的信息。

### 2.5 jinfo

- 查看JVM的系统属性，可以动态设置一些系统属性。

### 2.6 jstack

- 转储java进程的栈信息。

### 2.7 jstat

- 提供GC和类装载活动的信息。

### 2.8 visualVM

- 监视JVM的GUI工具，是jconsole的加强版，用于分析运行的应用。分析JVM堆转储。

### 2.9 BTrace

- 动态跟踪工具,工作原理是通过 instrument + asm 来对正在运行的java程序中的class类进行动态增强。默认对正在运行的程序是只读的，也可以使用unsafe模式绕过。
- Btrace可以单独运行，也可以通过visualVM插件的形式运行。
- 类似还有byteman，个人感觉没Btrace灵活
