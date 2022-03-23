# Linux虚拟内存（swap）调优篇-“swappiness”,“vm.dirty_background_ratio”和“vm.dirty_ratio” - 尹正杰 - 博客园
**Linux 虚拟内存 (swap) 调优篇 -“swappiness”,“vm.dirty_background_ratio” 和 “vm.dirty_ratio”**

**作者：尹正杰**

**版权声明：原创作品，谢绝转载！否则将追究法律责任。** 

　　我的 kafka 集群在上线一段时间后，发现内存使用达到峰值时系统开始使用 swap。在 swap 的过程中系统性能会有所下降，表现为较大的服务延迟。对这种情况，可以通过调节 swappiness 内核参数降低系统对 swap 的使用，从而避免不必要的 swap 对性能造成的影响。接下来，我们就一起学习一下如何调优该参数吧！

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121125032107-1875668856.png)

**一.**创建交换分区 \*\*\*\*

**1>. 什么是虚拟内存**

　　如果物理内存不够用时，可以将那些最近很少使用的页面数据（Page）置换出去，即切换到硬盘上，但是要注意的是内存文件的格式和硬盘中文件的格式是不一样的，所以这个分区必须格式化成跟内存兼容的模式不能转换成文件的格式。以便把内存的 page 直接存入这个分区，方便内存直接调用。而这个页面（page）数据对于 32 位的操作系统一个 page 大概是 4K 左右，对于 64 位操作系统这个 page 大小是可变的，4k-2M 的大小都是比较常见的。事实上到底能使用多大的页面（page）取决于 CPU 而不取决于内存哟！这就是虚拟内存的概念。在 linux 上我们称之为交换分区。记住，虚拟内存必须是一个单独的分区。

**2>. 虚拟内存能代替物理内存运行程序吗？**

　　答案是否定的，只是使用虚拟内存暂时保存数据，而不是代替物理内存运行程序。 

**3>. 虚拟内存的作用**

　　当运行某个大程序、大游戏，需要的内存超过空闲内存但小于物理内存总量时，会暂时把内存里这些数据放到磁盘上的虚拟内存里，空出物理内存运行游戏。等退出游戏后，又会把虚拟内存里的东西读出来，放回物理内存。所以，虚拟内存，并不是用来虚拟物理内存的，而是暂存数据的。如果对内存的需求大于物理内存总量，那虚拟内存设多大都不管用。电脑内存太低，根本的方法还是增加物理内存，才能流畅。虚拟内存机制上就不管用，即使管用，比物理内存低 100 倍的速度，也管不上什么实际的作用。所以，虚拟内存大了是没用的，反而白占用磁盘空间。

**4>. 交换分区常用的参数介绍**

![](https://common.cnblogs.com/images/copycode.gif)

交换分区：
      mkswap 格式化为虚拟内存 -L label 指定卷标
      swapon 启动虚拟内存 -a 启动所有的虚拟分区 -p：指定优先级
      swapoff 关闭虚拟内存
      更多参数请参考 man mkswap

![](https://common.cnblogs.com/images/copycode.gif)

**5>. 案例实操 - 创建交换分区的步骤**

![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)

![](https://common.cnblogs.com/images/copycode.gif)

\[root@yinzhengjie ~]# fdisk /dev/sdb  #对第二块硬盘进行分区调整

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): p  #查看当前分区情况

Disk /dev/sdb: 10.7 GB, 10737418240 bytes 255 heads, 63 sectors/track, 1305 cylinders
Units \\= cylinders of 16065 \* 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x8614a108 Device Boot      Start         End      Blocks   Id  System /dev/sdb1               1         132     1060258+  83 Linux /dev/sdb2             133         264     1060290   83 Linux /dev/sdb3             265         526     2104515   83 Linux /dev/sdb4             527        1305     6257317+   5 Extended /dev/sdb5             527         919     3156741   83 Linux /dev/sdb6             920        1181     2104483+  83 Linux    #我想讲第 6 个分区弄成交换分区。

Command (m for help): t  #调整分区 ID
Partition number (1-6): 6 #选择分区编号为 6
Hex code (type L to list codes): L   #查看分区类型所对应的 ID 号，我们发现 “82” 就是交换分区的编号 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris 1  FAT12           39  Plan 9          82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      3c  PartitionMagic  83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       40  Venix 80286     84  OS/2 hidden C:  c6  DRDOS/sec (FAT-
 4  FAT16 &lt;32M      41  PPC PReP Boot   85 Linux extended  c7  Syrinx 5  Extended        42  SFS             86  NTFS volume set da  Non-FS data 6  FAT16           4d  QNX4.x          87  NTFS volume set db  CP/M / CTOS / . 7  HPFS/NTFS       4e  QNX4.x 2nd part 88 Linux plaintext de  Dell Utility 8  AIX             4f  QNX4.x 3rd part 8e  Linux LVM       df BootIt 9  AIX bootable    50  OnTrack DM      93 Amoeba          e1  DOS access
 a  OS/2 Boot Manag 51  OnTrack DM6 Aux 94  Amoeba BBT      e3  DOS R/O
 b  W95 FAT32 52  CP/M            9f  BSD/OS          e4  SpeedStor
 c  W95 FAT32 (LBA) 53 OnTrack DM6 Aux a0  IBM Thinkpad hi eb  BeOS fs
 e  W95 FAT16 (LBA) 54 OnTrackDM6      a5  FreeBSD         ee  GPT
 f  W95 Ext'd (LBA) 55  EZ-Drive        a6  OpenBSD         ef  EFI (FAT-12/16/
10  OPUS            56  Golden Bow      a7  NeXTSTEP        f0  Linux/PA-RISC b 11 Hidden FAT12    5c  Priam Edisk     a8  Darwin UFS      f1  SpeedStor 12  Compaq diagnost 61 SpeedStor       a9  NetBSD          f4  SpeedStor 14  Hidden FAT16 &lt;3 63 GNU HURD or Sys ab  Darwin boot     f2  DOS secondary 16  Hidden FAT16    64  Novell Netware  af  HFS / HFS+ fb  VMware VMFS 17  Hidden HPFS/NTF 65 Novell Netware  b7  BSDI fs         fc  VMware VMKCORE 18  AST SmartSleep  70 DiskSecure Mult b8  BSDI swap       fd  Linux raid auto
1b  Hidden W95 FAT3 75  PC/IX           bb  Boot Wizard hid fe  LANstep
1c  Hidden W95 FAT3 80 Old Minix       be  Solaris boot    ff  BBT
1e  Hidden W95 FAT1
Hex code (type L to list codes): 82 #设置该分区的标号
Changed system type of partition 6 to 82 (Linux swap / Solaris)

Command (m for help): P   #查看当前分区情况

Disk /dev/sdb: 10.7 GB, 10737418240 bytes 255 heads, 63 sectors/track, 1305 cylinders
Units \\= cylinders of 16065 \* 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x8614a108 Device Boot      Start         End      Blocks   Id  System /dev/sdb1               1         132     1060258+  83 Linux /dev/sdb2             133         264     1060290   83 Linux /dev/sdb3             265         526     2104515   83 Linux /dev/sdb4             527        1305     6257317+   5 Extended /dev/sdb5             527         919     3156741   83 Linux /dev/sdb6             920        1181     2104483+  82  Linux swap / Solaris

Command (m for help): W  #保存并退出
The partition table has been altered! Calling ioctl() to re-read partition table.
Syncing disks.
\[root@yinzhengjie ~]#
\[root@yinzhengjie ~]# fdisk -l /dev/sdb  #查看分区信息

Disk /dev/sdb: 10.7 GB, 10737418240 bytes 255 heads, 63 sectors/track, 1305 cylinders
Units \\= cylinders of 16065 \* 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x8614a108 Device Boot      Start         End      Blocks   Id  System /dev/sdb1               1         132     1060258+  83 Linux /dev/sdb2             133         264     1060290   83 Linux /dev/sdb3             265         526     2104515   83 Linux /dev/sdb4             527        1305     6257317+   5 Extended /dev/sdb5             527         919     3156741   83 Linux /dev/sdb6             920        1181     2104483+  82  Linux swap / Solaris
\[root@yinzhengjie ~]# 

![](https://common.cnblogs.com/images/copycode.gif)

![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)

\[root@yinzhengjie ~]# kpartx -af /dev/sdb
\[root@yinzhengjie ~]# partx -a /dev/sdb   #重读分区表信息，其实也可以不用敲击这些命令的如果你是一块新硬盘的话。
BLKPG: Device or resource busy
error adding partition 1 BLKPG: Device or resource busy
error adding partition 2 BLKPG: Device or resource busy
error adding partition 3 BLKPG: Device or resource busy
error adding partition 4 BLKPG: Device or resource busy
error adding partition 5 BLKPG: Device or resource busy
error adding partition 6 \[root@yinzhengjie ~]# 

用 partx 重读一下分区表，避免系统未识别最新分区信息。

![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)

\[root@yinzhengjie ~]# mkswap /dev/sdb6  #将分区格式化成 swap 格式
Setting up swapspace version 1, size = 2104476 KiB
no label, UUID\\=41687bb2-c775-489c-9b32-1e4be73c233b   #看见没有，这里是 “no label”，是因为我没有定义卷标名。
\[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# mkswap -L myswap /dev/sdb6   #用 - L 参数定义一个卷标名。
Setting up swapspace version 1, size = 2104476 KiB
LABEL\\=myswap, UUID=0553b99a-ee75-4476-8eda-70c591206467  #看见没，“LABEL=myswap” 这就是我定义的卷标名称。
\[root@yinzhengjie ~]# 

用 mkswap 定义卷标名称

![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)

\[root@yinzhengjie ~]# cat /proc/meminfo | grep "^S" #查看当前交换分区大小
SwapCached: 0 kB
SwapTotal: 2031612 kB  #目前交换分区大小为 2G
SwapFree: 2031612 kB  #表示空闲交换分区大小
Shmem: 1444 kB
Slab: 75020 kB
SReclaimable: 15620 kB
SUnreclaim: 59400 kB
\[root@yinzhengjie ~]#
\[root@yinzhengjie ~]# swapon /dev/sdb6   #启用我们已经格式化好的交换分区 “/dev/sdb”
\[root@yinzhengjie ~]# cat /proc/meminfo | grep "^S" #再次查看当前交换分区大小
SwapCached: 0 kB
SwapTotal: 4136088 kB   #我们发现交换分区大小变大了 2G
SwapFree: 4136088 kB
Shmem: 1444 kB
Slab: 75148 kB
SReclaimable: 15656 kB
SUnreclaim: 59492 kB
\[root@yinzhengjie ~]#
\[root@yinzhengjie ~]# swapoff /dev/sdb6  #关闭交换分区 “/dev/sdb”
\[root@yinzhengjie ~]#
\[root@yinzhengjie ~]# cat /proc/meminfo | grep "^S" #验证是否关闭成功
SwapCached: 0 kB
SwapTotal: 2031612 kB   #发现的确是少了 2G 的空间
SwapFree: 2031612 kB
Shmem: 1444 kB
Slab: 75068 kB
SReclaimable: 15636 kB
SUnreclaim: 59432 kB
\[root@yinzhengjie ~]#

swapon 和 swapoff 的用法展示

**6>.Linux 清除 swap 方法**

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121120154605-1759337448.png)

**想要了解更多关于文件系统的知识，详情请参考：[https://www.cnblogs.com/yinzhengjie/p/6840563.html ](https://www.cnblogs.com/yinzhengjie/p/6840563.html)。** 

**7>.swap 分区使用说明**

![](https://common.cnblogs.com/images/copycode.gif)

　　我本人并不推荐大家使用 swap 分区，因为它会降低服务器性能。

　　早期由于工业原因内存相对较贵，因此很多软件在设计之初都考虑尽可能的使用磁盘来代替内存，但是磁盘的 I/O 性能要和内存的 I/O 性能完全是天壤之别。比如在大数据领域 Hadoop 的一个 MapReduce 组件，该计算框架就尽可能使用磁盘，这是导致它计算速度很慢的一个原因，这也是为什么后来 Spark 和 Flink 崛起埋下伏笔。

　　因此，在生产环境中我们应该尽量禁用虚拟内存，比如阿里云的服务器默认就是禁用虚拟内存的，我在生产环境中也是直接禁用虚拟内存的。但有的服务器内存相对较小，比如 8G，担心程序发生 OOM. 于是为了留一手才被迫使用虚拟内存。

　　如果非要使用虚拟内存建议参考以下几点:
　　　　(1) 尽可能使用较块的设备，比如固态硬盘;
　　　　(2) 如果使用磁盘建议使用扇区靠外的分区，这样在查找数据时速度相对较块;
　　　　(3) 生产环境中虚拟磁盘不建议超过 8G;

![](https://common.cnblogs.com/images/copycode.gif)

**二.\*\***swappiness 参数在内存与交换分区之间优化作用 \*\*

\***\*swappiness 的值的大小对如何使用 swap 分区是有着很大的联系的。先前，人们建议把 vm.swapiness 设置为 0，它意味着 “除非发生内存益处，否则不要进行内存交换”。直到 Linux 内核 3.5-rcl 版本发布，这个值的意义才发生了变化。这个变化被一直到其他的发行版本上，包括 RedHat 企业版内核 2.6.32-303。在发生变化之后，0 意味着 “在任何情况下都不要发生交换”。所以现在建议把这个值设置为 1。swappiness＝100 的时候表示积极的使用 swap 分区，并且把内存上的数据及时的搬运到 swap 空间里面。\*\***

**1>.linux 的 swappiness 参数的默认设置为 60（\[root@yinzhengjie ~]# cat /proc/sys/vm/swappiness）**

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121122941442-865187019.png)

　　也就是说，你的内存在使用到 100-60=40% 的时候，就开始出现有交换分区的使用。大家知道，内存的速度会比磁盘快很多，这样子会加大系统 io，同时造的成大量页的换进换出，严重影响系统的性能，所以我们在操作系统层面，要尽可能使用内存，对该参数进行调整。

**2>. 临时调整 swappiness 的方法 \*\***（\[root@yinzhengjie ~]# sysctl vm.swappiness=1）\*\*![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121123319207-939516521.png)

**3>.**永久调整 swappiness 的方法 \***\*（\[root@yinzhengjie ~]#  echo "vm.swappiness=1" >> /etc/sysctl.conf）\*\***

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121124119024-739849685.png)

　　在 linux 中，可以通过修改 swappiness 内核参数，降低系统对 swap 的使用，从而提高系统的性能。简单地说这个参数定义了系统对 swap 的使用倾向，默认值为 60，值越大表示越倾向于使用 swap。不推荐设为 0，因为这样做会对 3.5 以上的 kernel 禁止对 swap 的使用，我推荐打击设置一个较小对值，比如 1，它只是最大限度地降低了使用 swap 的可能性。

**4>. 查看 swappiness 参数的当前设置**

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121125405769-1922198703.png)

**三.\*\***使用 vm.dirty_ratio 和 vm.dirty_background_ratio 更好的 Linux 磁盘缓存和性能 \*\* 

**1>. 脏页对概念**

　　脏页是 linux[内核](https://baike.baidu.com/item/%E5%86%85%E6%A0%B8)中的概念，因为[硬盘](https://baike.baidu.com/item/%E7%A1%AC%E7%9B%98)的读写速度远赶不上内存的速度，系统就把读写比较频繁的数据事先放到[内存](https://baike.baidu.com/item/%E5%86%85%E5%AD%98)中，以提高读写速度，这就叫高速缓存，linux 是以页作为[高速缓存](https://baike.baidu.com/item/%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98)的单位，当进程修改了高速缓存里的数据时，该页就被内核标记为脏页，内核将会在合适的时间把脏页的数据写到磁盘中去，以保持高速缓存中的数据和磁盘中的数据是一致的。

**2>. 相关参数解释**

![](https://common.cnblogs.com/images/copycode.gif)

\[root@yinzhengjie ~]# sysctl -a | grep dirty
vm.dirty_background_bytes \\= 0 vm.dirty_background_ratio \\= 10 vm.dirty_bytes \\= 0 vm.dirty_expire_centisecs \\= 3000 vm.dirty_ratio \\= 30 vm.dirty_writeback_centisecs \\= 500 \[root@yinzhengjie ~]# 

vm.dirty_background_ratio :
    是内存可以填充 “脏数据” 的百分比。这些 “脏数据” 在稍后是会写入磁盘的，pdflush/flush/kdmflush 这些后台进程会稍后清理脏数据。举一个例子，我有 32G 内存，那么有 3.2G 的内存可以待着内存里，超过 3.2G 的话就会有后来进程来清理它。

vm.dirty_ratio：
    是绝对的脏数据限制，内存里的脏数据百分比不能超过这个值。如果脏数据超过这个数量，新的 IO 请求将会被阻挡，直到脏数据被写进磁盘。这是造成 IO 卡顿的重要原因，但这也是保证内存中不会存在过量脏数据的保护机制。

vm.dirty_background_bytes 和 vm.dirty_bytes 是
    指定这些参数的另一种方法。如果设置\_bytes 版本，则\_ratio 版本将变为 0，反之亦然。

vm.dirty_expire_centisecs ：
    指定脏数据能存活的时间。在这里它的值是 30 秒。当 pdflush/flush/kdmflush 进行起来时，它会检查是否有数据超过这个时限，如果有则会把它异步地写到磁盘中。毕竟数据在内存里待太久也会有丢失风险。

vm.dirty_writeback_centisecs：
    指定多长时间 pdflush/flush/kdmflush 这些进程会起来一次。

![](https://common.cnblogs.com/images/copycode.gif)

**以上说明饮用自：[https://blog.csdn.net/csCrazybing/article/details/78127308](https://blog.csdn.net/csCrazybing/article/details/78127308)**

**3>. 调整内核对脏页对处理方式可以让我们从中获益**

 　　脏页会被冲刷到磁盘上，调整内核对脏页的处理方式可以让我们从中获益。Kafka 依赖 I/O 性能为生产者提供了快速的响应。这就是为什么日志片段一般要保存在快速磁盘上，不管是单个快速磁盘（如 SSD）还是具有 NVRAM 缓存的磁盘子系统（如 RAID）。这样一来，在后台刷新进程将脏页写入磁盘之前，可以减少脏页的数量，这个可以通过 vm.dirty_backgroud_ratio 设置为小于 10 的值来实现。改值指的是系统内存的百分比，大部分情况下设置为 5 就可以来。它不应该被设置为 0，因为那样会促使内核频繁地刷新页面，从而降低内核为底层设备的磁盘写入提供缓冲的能力。

　　通过设置 vm.dirty_ratio 参数可以增加被内核进程刷新到磁盘之前的脏页数量，可以将它设置为大于 20 的值（这也是系统内存的百分比），这个值可设置的范围很广，60～80 是个比较合理的区间。不过调整这个参数会带来一些风险，包括未刷新磁盘操作的数量和同步刷新引起的长时间 I/O 等待。如果篡改参数设置了较高的值，建议启用 Kafka 的复制功能，避免因系统崩溃造成数据丢失。

　　为了给这些参数设置合适的值，最好是在 Kafka 集群运行期间检查脏页的数量，不管是在生产环境还是在模拟环境。可以在 “/proc/vmstat” 文件里查看当前脏页的数量。

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121141842719-1963176191.png)

**4>. 减少 Cache（虚拟机的典型应用）**

![](https://common.cnblogs.com/images/copycode.gif)

你可以针对要做的事情，来制定一个合适的值。
在一些情况下，我们有快速的磁盘子系统，它们有自带的带备用电池的 NVRAM caches，这时候把数据放在操作系统层面就显得相对高风险了。所以我们希望系统更及时地往磁盘写数据。
可以在 / etc/sysctl.conf 中加入下面两行，并执行 "sysctl -p" vm.dirty_background_ratio \\= 5 vm.dirty_ratio \\= 10 这是虚拟机的典型应用。不建议将它设置成 0，毕竟有点后台 IO 可以提升一些程序的性能。

![](https://common.cnblogs.com/images/copycode.gif)

**5>. 增加 Cache（适合数据并不是很重要的场景，要求读写的效率想到高的场景）**

![](https://common.cnblogs.com/images/copycode.gif)

在一些场景中增加 Cache 是有好处的。例如，数据不重要丢了也没关系，而且有程序重复地读写一个文件。允许更多的 cache，你可以更多地在内存上进行读写，提高速度。

vm.dirty_background_ratio \\= 50 vm.dirty_ratio \\= 80 有时候还会提高 vm.dirty_expire_centisecs 这个参数的值，来允许脏数据更长时间地停留。 

![](https://common.cnblogs.com/images/copycode.gif)

**6>. 增减兼有（如果你部署 kafka 集群的话，我推荐使用这个方案，在《Kafka 权威指南》一书中，也有相关的记载哟！）**

![](https://common.cnblogs.com/images/copycode.gif)

有时候系统需要应对突如其来的高峰数据，它可能会拖慢磁盘。（比如说，每个小时开始时进行的批量操作等）
这个时候需要容许更多的脏数据存到内存，让后台进程慢慢地通过异步方式将数据写到磁盘当中。

vm.dirty_background_ratio \\= 5 vm.dirty_ratio \\= 80 这个时候，后台进行在脏数据达到 5% 时就开始异步清理，但在 80% 之前系统不会强制同步写磁盘。这样可以使 IO 变得更加平滑。

![](https://common.cnblogs.com/images/copycode.gif)

**7>. 案例实操 - 调整内核对脏页的处理方式**

![](https://common.cnblogs.com/images/copycode.gif)

\[root@yinzhengjie ~]# sysctl -a | grep vm.dirty  
sysctl: reading key "net.ipv6.conf.all.stable_secret"sysctl: reading key"net.ipv6.conf.default.stable_secret"sysctl: reading key"net.ipv6.conf.ens160.stable_secret"sysctl: reading key"net.ipv6.conf.lo.stable_secret" vm.dirty_background_bytes \\= 0 vm.dirty_background_ratio \\= 10 vm.dirty_bytes \\= 0 vm.dirty_expire_centisecs \\= 3000 vm.dirty_ratio \\= 30 vm.dirty_writeback_centisecs \\= 500 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# cat /etc/sysctl.conf  | grep -v ^# 
vm.swappiness\\=1 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# echo "vm.dirty_background_ratio=5" >> /etc/sysctl.conf
\[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# echo "vm.dirty_ratio=80" >> /etc/sysctl.conf
\[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# cat /etc/sysctl.conf  | grep -v ^#  
vm.swappiness\\=1 vm.dirty_background_ratio\\=5 vm.dirty_ratio\\=80 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# sysctl -p
vm.swappiness \\= 1 vm.dirty_background_ratio \\= 5 vm.dirty_ratio \\= 80 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# sysctl -a | grep vm.dirty  
sysctl: reading key "net.ipv6.conf.all.stable_secret"sysctl: reading key"net.ipv6.conf.default.stable_secret"sysctl: reading key"net.ipv6.conf.ens160.stable_secret"sysctl: reading key"net.ipv6.conf.lo.stable_secret" vm.dirty_background_bytes \\= 0 vm.dirty_background_ratio \\= 5 vm.dirty_bytes \\= 0 vm.dirty_expire_centisecs \\= 3000 vm.dirty_ratio \\= 80 vm.dirty_writeback_centisecs \\= 500 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# sysctl -q vm.dirty_background_ratio  
vm.dirty_background_ratio \\= 5 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# sysctl -q vm.dirty_ratio
vm.dirty_ratio \\= 80 \[root@yinzhengjie ~]# 
\[root@yinzhengjie ~]# 

![](https://common.cnblogs.com/images/copycode.gif)

![](https://img2018.cnblogs.com/blog/795254/201811/795254-20181121145002826-151732081.png)

当你的才华还撑不起你的野心的时候，你就应该静下心来学习。当你的能力还驾驭不了你的目标的时候，你就应该沉下心来历练。问问自己，想要怎样的人生。 欢迎加入基础架构自动化运维：598432640，大数据 SRE 进阶之路：959042252，DevOps 进阶之路：526991186 
 [https://www.cnblogs.com/yinzhengjie/p/9994207.html](https://www.cnblogs.com/yinzhengjie/p/9994207.html)
