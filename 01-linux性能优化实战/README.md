## Linux 性能优化实战

[Linux 性能优化实战 by 倪朋飞](https://time.geekbang.org/column/intro/140)

#### [01-如何学习Linux性能优化？](https://time.geekbang.org/column/article/69346)

> 笔记

* 概念
    * 性能优化的一般步骤（和性能测试的一般步骤很像）
        * 选择性能指标
        * 设置性能目标
        * 选择性能测试(调试)工具
        * 性能测试和分析
        * 性能优化
        * 性能监控和预警
    * 性能指标
        * 应用指标
            * QPS/RT 和其他
                * C 并发数 N 请求总数 T 总耗时
                * QPS = N / T
                * RT = T / N * C
                * QPS = C / RT 
        * 系统指标
            * cpu/内存/磁盘io/网络io

> 金句

**学习的重点是，建立整体系统性能的全局观**

我对这句话体会颇深，对于线上问题，如果一开始就钻到细节，而不是全局分析，容易走入死胡同，浪费时间；
正确的方法是，先分析网络拓扑，搞清楚服务链路，整体查看调用情况，并根据监控数据对比分析（一定要对比），
找到异常点，再针对它深入分析。这其实就是一种全局观，它要求我们具备一种对问题的综合认知能力，对涉及
性能或功能的方方面面有一个全局多维度的掌握。对于性能优化，我想也是这样，如果你只了解一个工具，一种指标，
那么你就只能或只会想到一个方法，实际上性能问题大多数是综合问题，仅靠"一把铁锤"是无法解决问题的，把这句
话当做学习的目标吧：建立整体系统性能的全局观。

> 问题

1. 你们是怎么做性能测试或压力测试的？一般流程有哪些？
2. 一般的应用性能指标有哪些？它是怎么统计的?

#### [02-基础篇：到底应该怎么理解“平均负载”?](https://time.geekbang.org/column/article/69618)

> 笔记

* 概念
    * 平均负载 = 平均活跃进程数（R和D状态的进程）
    * 进程状态: [状态机](./img/process_state_machine.png)
    * cpu逻辑核心数(processor) = cpu物理个数(physic id) * 单个cpu核心数（core id) * 单核线程数 

* 工具
    * 压测工具
        * stress/stress-ng -> 模拟cpu/io/多线程压力
    * 性能分析工具
        * mpstat -> 分析cpu每个逻辑核心的时间占用
        * pidstat -> 分析进程级别的cpu时间占用

> 金句

* 你真的知道每列的含义吗

言外之意，我们通常只知道某列或少数几列的含义，似懂非懂，对工具掌握不全面，容易陷入"铁锤人"思维。例如cpu使用率，
需要区分不同场景下的使用率，例如用户态，内核态(上下文切换等），iowait(D状态进程等），软硬中断的使用率等，如果只把
usr的消耗作为cpu的使用率，就很可能走偏误判，无法找到问题根源。

> 发现问题

1. 进程包含哪些状态，状态之间如何转换？
2. 什么是平均负载？一般怎么分析该指标？
3. 如何排查导致负载升高的进程？

> 分享精华

#### [03 | 基础篇：经常说的 CPU 上下文切换是什么意思？（上）](https://time.geekbang.org/column/article/69859)

> 笔记

* 上下文 -> 任务执行所依赖的资源
    * 内核态上下文
        * 指令 -> 寄存器/程序计数器(PC是寄存器的一种，存储指令及其执行位置）
        * 内核堆栈等
    * 用户态上下文
        * 虚拟内存 -> 页表和TLB
        * 用户栈
        * 全局变量等

* 上下文切换 -> 保存（保存到内核管理的内存中）旧任务上下文，加载新任务上下文
    * 进程间切换 -> 调度两个不同进程的线程
    * 线程间切换 -> 调度进程内的不同线程
    * 中断切换 -> 响应硬件异步信号（例如点击鼠标，收到一个网络包等）
    * 内核模式切换 -> 系统调用， 进程内在用户态(ring3)和内核态(ring0)之间切换
    * 协程切换 -> 在用户态完成切换（其他模式再内核完成切换）

**ps: 开销排名 进程间切换 > 线程间切换(同进程）> 中断切换 > 内核模式切换/协程切换, 切换时间算为cpu sys 时间**
**ps: 进程内切换无需切换虚拟内存等进程级别的资源**

* 进程切换的原因
    * 非自愿
        * cpu时间片耗尽
    * 自愿
        * 资源不足，例如内存不足
        * 主动sleep
        * 有更高优先级的进程出现
        * 中断
    
> 金句

**线程是调度的基本单位，进程是资源拥有的基本单位**

理解这句话才能更好的理解上下文切换，实际上进程间的切换本质上是线程在两个不同
的进程之间发生调度。对于内核，它没有虚拟内存的概念，因此也就没有进程的说法，
都是基于线程进行调度的，例如内核的软中断线程，负责处理软中断。

> 问题

1. 什么是cpu的上下文切换？为什么说上下文切换会产生性能开销?
2. 进程在什么情况下会被挂起？说说进程调度的策略有哪些？
3. 为什么说golang的协程比java的线程模型开销更小，能否从上下文切换的角度分析？

#### [04 | 基础篇：经常说的 CPU 上下文切换是什么意思？（下）](https://time.geekbang.org/column/article/70077)

> 笔记

* 概念
    * cpu上下文切换过多的现象
        * 平均负载升高
        * cpu sys 升高
        * cpu 切换次数变多
            * 通过vmstat或pidstat可查看

    * cpu上下文切换次数受什么影响
        * cpu本身的性能
        * 系统负载
        
        
* 工具
    * vmstat -> 查看系统整体资源使用(cpu/内存/上下文切换）
        * vmstat 5 -> 5s钟输出一次
            * in -> 中断次数
            * cs -> 上下文切换总次数
            * r -> 就绪进程数
            * b -> 不可中断睡眠进程数（相当于D状态）
        * vmstat 输出第一行是系统启动以来的平均值, 接下来才是实时值

    * sysbench -> 压力测试工具
        * 模拟多线程任务
          * sysbench --threads=10 --max-time=300 threads run
        * stress -c 是模拟多进程，它会引起进程上下文切换
    
    * pidstat -> 分析进程资源使用
        * pidstat -wut 5
            * -w -> 显示上下文切换
                * cswch/s -> 自愿切换(资源不足）
                * nvcswch/s -> 非资源切换（cpu时间片用完）
            * -u -> 显示cpu
            * -t -> 显示线程级别的上下文切换
                * 否则只显示进程级别切换，无法体现线程上下文切换的情况, 与vmstat统计的次数对不上

    * /proc/interrupt -> 查看系统中断
        * cat /proc/interrupts  | sort -nr -k 2 -> 按切换次数排序
        * RES -> 处理器间中断 (调度器唤醒cpu时触发）
            * 进程竞争激烈时，除了导致上下文切换增加，同时也会引起RES的明显增加(比平稳情况下增加的更多）
            * 现象是vmstat查出来 in 和 cs变多
            * 单核CPU可能没有RES

> 最佳实践

* 用zabbix等工具监控cpu的上下文切换和中断时间，只有明显变化(指数级）才去关注分析它

> 问题

* 怎么查看cpu上下文切换次数？多少次是不正常的呢？
* cpu上下文切换次数过多会导致cpu使用率变高吗？

#### [05 | 基础篇：某个应用的CPU使用率居然达到100%，我该怎么办？](https://time.geekbang.org/column/article/70476)

> 笔记

* 概念
    * cpu使用率
        * 时间分类
            * usr/nice/sys/idle/iowait/irq/softirq/steal/gurst/guest_nice
                * 和cat /proc/stat 显示的列一一对应
        * 计算公式
            * （cpu时间2 - cpu时间1）/ (cpu总时间2 - cpu总时间1)
            *  cpu时间 = cpu中断次数(jiffies) * 单次时间(1 / CPU_HZ) = jiffies / CPU_HZ
                * jiffies -> cpu总节拍数, 通过 /proc/stat 查看
                * 例如200 jiffies = 200 / 100 = 2s 
        * 节拍率
            * CONFIG_HZ -> 内核节拍率
                * grep 'CONFIG_HZ' /boot/config-$(uname -r) 查看
            * CPU_HZ -> 用户空间节拍率,固定值100
                * 表示1s时间中断100次，平均一次0.01s（作为cpu时间单位）

* 工具
    * top -> 查看cpu使用率和进程cpu使用率
        * 按1查看各个cpu耗时
    * pidstat -> 查看cpu使用率
    * perf -> 分析cpu性能瓶颈
        * perf top -> 查看热点函数
            * Overhead -> 热点比例
            * Shared -> 函数共享对象，内核/进程/动态链接库/内核模块名
            * Object -> 共享对象类型
                * [.] -> 进程/动态链接库
                * [k] -> 内核
            * Symbol -> 函数/指令名，找不到映射则显示16进制(函数地址）
        * perf record -g -p -> 历史采样
            * -g 采集调用链
            * -p 指定进程
        * perf report -> 针对record生成的perf.data进行分析
            * perf report 解析函数名依赖它所在函数库，如果lib在容器内，需要进入容器执行perf
    * ab -> http性能测试工具
        * ab -c 10 -n 10000 http://127.0.0.1:8080/
    * docker 
        * docker cp perf.data phpfpm:/tmp -> 容器文件复制
        * docker exec -it phpfpm:/tmp bash -> 访问容器
    * 其他
        * cat /etc/issue 查看系统
        * apt-get update && apt-get install -y linux-perf linux-tools procps -> debian系统安装perf_4.9


> 金句

**我希望你在按照步骤操作之前，先不要查看源码（避免先入为主），而是把它当成一个黑盒来分析**

这里本质上说，排查问题时，不要一开始就陷入细节，避免走入死胡同；一定要具有全局观，从宏观趋势去发现异常，再深入分析。

> 问题

1. 什么是cpu使用率，有哪些使用率，分别是怎么计算的?
2. cpu使用率和平均负载的区别是什么？
3. 如何排查cpu用户进程使用率过高的问题？怎么找到引起cpu飙升的代码点?

#### [06 | 案例篇：系统的 CPU 使用率很高，但为啥却找不到高 CPU 的应用?](https://time.geekbang.org/column/article/70822)

> 笔记

* 概念
    * 短时进程 -> 表现为pid很短暂，经常发生变化
        * 被其他父进程调用(exec())，生命很短的进程
        * 不断重启的进程

ps: 由于短时进程pid飘忽不定，很难通过pidstat或ps等进程维度的工具排查问题

* 工具
    * top -> 查看指定状态的进程
        * o -> 开启filter
        * S=R -> 过滤状态为R的进程
    * perf -> 排查cpu热点进程（包括短时进程）
        * perf record -g
        * perf report
            * 如果进程run in container, 依赖的库在容器内，需要回到容器内执行该命令
    * pstree
        * pstree | grep stress -> 排查父子进程
        * 查看是否是父进程反复创建短时进程导致
    * [execsnoop](https://github.com/brendangregg/perf-tools/blob/master/execsnoop)
        * 监控exec()调用

> 金句

**对于这类问题，有没有更好的方法监控呢**

execsnoop这类针对特定场景的工具是怎么来的？就是有人去思考，去尝试吗，去找到一种更高效的
问题解决方案，才被设计开发出来。在日常工作中，无论是需求开发还是问题处理，是否仅仅满足
于现有的处理方式，有没有更好更快的方法，能否减少重复劳动，减少排查的时间？小小的思考，
带来的是时间效率的提升，最终落实为成本的节省。


> 问题

1. 是否遇到过应用不断重启的情况？怎么发现和排查这类问题？
2. 如何查看指定状态的进程？


#### [07 | 案例篇：系统中出现大量不可中断进程和僵尸进程怎么办？（上）](https://time.geekbang.org/column/article/71064)

> 笔记

* 概念
    * 进程状态
	* R -> running/runnable -> 就绪队列中的进程
	* D -> disk sleep -> 不可中断的睡眠进程 -> 正在与硬件例如磁盘进行交互
	    * D 进程通常导致iowait升高, 平均负载升高
	* Z -> zoombie -> 已经结束但是没有被（父进程)清理资源的进程
	    * 未优雅关闭子进程的父进程会导致zoombie的子进程
	* S -> interrupt sleep -> 可中断睡眠 -> 等待某个条件/事件被暂时挂起
	* I -> idle -> 用在不可中断的<b>内核</b>线程上
	    * I 不会导致平均负载的升高
	* T -> 暂停或跟踪
	    * stopped -> 向进程发生sigstop信号 -> 可通过sigcont 恢复
	    * traced -> 使用调试器(gdb)进入端点时
	* X -> dead -> 死亡进程
	    * 不会在ps或top指标中出现
    * 进程组
    	* ps -axjf -> 查看pid, pgid和sid
	    * pid -> 进程id
	    * pgid -> 进程组id
		* 状态显示+，例如S+
	    * sid -> 会话id -> 通过ssh连接server，生成一个会话
		* 状态显示s，例如Ss+
    * 磁盘与分区
    	* df -h -> 查看分区和挂载
		* fdisk -l -> 查看物理磁盘
	    	* 利用cgroup对磁盘进行iops/bps限制时，需要指定物理磁盘


* 工具
    * docker 
	* docker run --privileged --name app --device /dev/vda1:/dev/vda1 --device-write-iops /dev/vda1:3 --device-read-iops /dev/vda1:3 -itd feisky/app:iowait
	    * --privileged -> 授予docker root 权限
	    * --device -> 挂载设备
	    * --device-write-iops -> cgroup限制磁盘写iops
	    * --device-read-iops -> cgroup限制磁盘读iops

> 问题

* 进程有哪些状态，能否画出进程的状态机流转图？
* 说说D和S进程的区别？
	
#### [08 | 案例篇：系统中出现大量不可中断进程和僵尸进程怎么办？（下）](https://time.geekbang.org/column/article/71382)

> 笔记

* 工具
    * dstat -> 综合查看cpu/io指标
        * dstat 1 10 -> 注意对比io读写量和cpu wait时间
    * pidstat
    	* pidstat -d -> 查看进程io读写速率
    * strace
    	* trace -p -> 追踪进程的系统调用（排查io热点）
    * pstree
    	* pstree -aps [Z进程 pid] -> 查看父子进程关系(用于发现僵尸进程的父进程)
    * perf
        * perf record -> 排查负载热点

> 金句

**iowait升高不代表io有性能瓶颈**

基于指标的分析有一个前提，即一定要充分了解工具指标的含义和计算方法，搞清楚它的使用场景。而且，
为了避免单指标分析可能产生误判，我们最好是采用多个指标相互分析佐证。例如cpu的iowait使用率，它
能够反映磁盘io的压力情况，但同时我们也要结合进程状态（是否有大量D进程），排查磁盘相关的其他工具
(例如dstat, pidstat, vmstat，iostat)等，综合的来看，可能iowait升高是进程本身就是io密集型
，但系统io未达到瓶颈。

> 问题

1. 僵尸进程是怎么产生的？如何排查僵尸进程?
2. 如何排查io热点进程及其对应热点代码？

#### [09 | 基础篇：怎么理解Linux软中断？](https://time.geekbang.org/column/article/71868)

> 笔记

* 查看cpu软中断使用率的方式
    * top 
    * mpstat -P ALL
    * dstat 1 3

**ps: pidstat和vmstat都无法查看软中断使用率**

* 什么是中断
    * 系统响应硬件设备请求的一种机制 -> 异步事件处理机制 -> 提高系统并发处理能力
        * 如何避免硬件时间过长导致其他中断丢失？
            * 上半部分 -> 硬中断 -> cat /proc/interrupts
                * 快速执行
                * 中断其他进程
                * 发送软中断信号
            * 下半部分(异步) -> 软中断 -> cat /proc/softirqs
                * 延迟执行
                * 以内核方式执行 -> ksoftirqd/CPU[编号] -> ps aux | grep softirq
                * 软中断不一定依赖于硬中断，也可以由内核发送特定事件单独触发
    * 以网卡数据包接收为例
        * 网卡收到数据 -> 发送硬中断请求
            * 内核调用中断处理程序：读网卡数据到内存
            * 发送软中断信号
        * 软中断线程从内存读数据，根据网络协议栈解析数据
    * 中断是否会引起性能问题
        * 中断过多 -> 中断会导致中断上下文切换 
            * 浪费cpu时间
        * 中断时间过长 
            * 分配给用户进程的有效时间减少

> 金句

**中断其实是一种异步的事件处理机制，可以提高系统的并发处理能力**

异步是一种有效的提高系统吞吐量的方法，不仅体现在系统内核的处理方式上，应用层面我们
也经常采用这种模式。例如CQRS等设计模式，消息中间件等异步解耦的组件，本质上都是采用
异步的方式，让关键业务快速响应，再通过其他线程或其他服务延时处理对时间不敏感的部分。

> 问题

1. 什么是中断，如何查看进程的中断cpu使用率? 
2. 以网卡接收数据为例，说说该过程发生了哪些中断？

#### [10 | 案例篇：系统的软中断CPU使用率升高，我该怎么办？](https://time.geekbang.org/column/article/72147)

> 笔记

* 软中断有哪些常见类型？
    * 网络收发 -> NET_TX/NET_RX
    * 定时-> TIMER
    * 调度-> SCHED
    * RCU锁 -> RCU 
        * 一种限制和包含共享资源的锁机制

* 如何排查软中断使用率增加的问题
    * top -> 发现si% 增加
        * ksoftirqd/1或ksoftirqd/0 使用率增加
    * watch -d cat /proc/softirqs 
        * 观察不同类型软中断变化速度 -> NET_RX变化速度较快
            * 说明网络包接收中断变多，可能有网络问题
            * 表现为远程访问服务变卡，ping时长增加并有丢包现象
    * sar -n DEV 1 -> 查看网卡(pps/bbs) -> eth0的rxpck/s(pps)增加
        * pps -> 每s接收/发送网络帧数
        * bps -> 每s接收/发送网络字节数
        * 每个包的大小 = bps * 1000byte / pps 
            * 如果 < 100 byte 通常认为是小包
    * tcpdump -i eth0 -n tcp port 80 -> 抓包
        * -i 指定网卡
        * -n 指定协议
        * port 指定接收端口
        * 结果：
            * Flags [S] -> sync 包
            * 172.18.155.128.http > 183.238.179.102.50979 
                * 数据包来源和目标
    * 结论 
        * 网络卡顿(ping延时丢包)是因为sync flood 攻击导致
            * ping -> top -> softirqs -> sar -> tcpdump

* 如何模拟sync flood 攻击
    * 使用hping3发送网络包
        * hping3 -S -p 80 -i u100 192.168.0.30
            * -S -> 发送sync包
            * -p -> 指定服务端口
            * -i -> 指定发送频率，调整它控制请求压力
                * u100 -> 100us一次
    * mac如何安装hping3

```
brew uninstall hping
cd ~
mkdir hping
cd hping
git clone https://github.com/antirez/hping.git
brew install tcl-tk
brew install libpcap
./configure
make
sudo make install
hping

```
> 金句

**所以我们直接查看文件内容，得到的只是累积中断次数，对这里的问题并没有直接参考意义**

分析问题的关键是"对比"，好比是做试验也要设置实验组和对比组，我们可以和过去的历史数据
对比，和其他指标进行对比，和今天的其他时段对比，和以前的同一时段对比等等，对比分析比
直觉分析更靠谱，更科学。

> 问题

1. 一般是怎么排查网络问题的，例如网络变卡时，怎么排查原因？
2. 请说说什么是软中断，系统有哪些常见的软中断类型，软中断升高问题如何排查？

#### [11 | 套路篇：如何迅速分析出系统CPU的瓶颈在哪里？](https://time.geekbang.org/column/article/72685)

> 笔记

* 如何衡量cpu的性能？
    * cpu使用率 -> usr/sys/iowait/si/hi/steal/guest
        * top/vmstat/dstat/mpstat -> 系统整体
        * pidstat/ps -> 进程
        * -> perf 查找热点代码
    * 平均负载 -> R和D进程数
        * top/uptime
        * vmstat -> 查看R和D的数量
    * 上下文切换 -> 进程/线程/中断上下文切换 -> 自愿/非资源切换
        * vmstat/pidstat
    * 缓存命中率 -> L1/L2/L3 cache

* 记录几个扩展的命令
    * htop 
        * 颜色区分进程状态
        * 显示进程启动命令
    * atop
        * 全面监控cpu,内存,磁盘和网络
    * sar
        * 监控cpu,内存,磁盘和网络等

```
(0) sar 5 5     //  CPU和IOWAIT统计状态 
(1) sar -b 5 5        // IO传送速率
(2) sar -B 5 5        // 页交换速率
(3) sar -c 5 5        // 进程创建的速率
(4) sar -d 5 5        // 块设备的活跃信息
(5) sar -n DEV 5 5    // 网路设备的状态信息
(6) sar -n SOCK 5 5   // SOCK的使用情况
(7) sar -n ALL 5 5    // 所有的网络状态信息
(8) sar -P ALL 5 5    // 每颗CPU的使用状态信息和IOWAIT统计状态 
(9) sar -q 5 5        // 队列的长度（等待运行的进程数）和负载的状态
(10) sar -r 5 5       // 内存和swap空间使用情况
(11) sar -R 5 5       // 内存的统计信息（内存页的分配和释放、系统每秒作为BUFFER使用内存页、每秒被cache到的内存页）
(12) sar -u 5 5       // CPU的使用情况和IOWAIT信息（同默认监控）
(13) sar -v 5 5       // inode, file and other kernel tablesd的状态信息
(14) sar -w 5 5       // 每秒上下文交换的数目
(15) sar -W 5 5       // SWAP交换的统计信息(监控状态同iostat 的si so)
(16) sar -x 2906 5 5  // 显示指定进程(2906)的统计信息，信息包括：进程造成的错误、用户级和系统级用户CPU的占用情况、运行在哪颗CPU上
(17) sar -y 5 5       // TTY设备的活动状态
(18) 将输出到文件(-o)和读取记录信息(-f)

```
    
> 金句

**"虽然 CPU 的性能指标比较多，但要知道，既然都是描述系统的 CPU 性能，它们就不会是完全孤立的，很多指标间都有一定的关联"**

这句话的关键词是“联系”，要学会用联系的思维去理解和分析现象，特别是对于复杂多变的问题，仅仅依靠单一的，孤立的指标或方法是
无法搞定的。例如说，cpu上下文切换次数速度增加，关联的看，cpu的sys也会增加；负载升高，可能iowait，R进程数也会升高等，通过
指标之间的连续来从多个角度分析，验证我们的猜想，这就是所谓的综合分析能力。

> 问题

1. 怎么衡量cpu的性能，主要参考哪些指标？
2. 如何查看这些指标，怎么从现象一步步深入到代码的定位？
