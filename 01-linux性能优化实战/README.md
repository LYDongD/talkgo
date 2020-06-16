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

#### [12 | 套路篇：CPU 性能优化的几个思路](https://time.geekbang.org/column/article/73151)

> 笔记

* 性能优化的一般流程
    * 选择性能指标
        * 应用指标
        * 系统指标
    * 确定性能目标 -> 根据历史数据和预期
        * 例如根据历史订单峰值，估计双11高峰期的预期峰值，确定性能目标
    * 性能测试(压测） -> 当前系统是否满足性能目标
        * 记录优化前的性能指标
            * 例如在不同场景下，设置不同的并发数，压测系统
                * 记录tps/rt等应用指标
                * 记录cpu使用率，平均负载，内存，磁盘和网络等系统指标
    * 性能优化 
        * 应用层
        * 系统层
    * 性能测试(评估） -> 优化后是否达到性能目标
        * 对比优化前后性能指标

* 以cpu为例的性能优化
    * 应用层
        * 编译器优化
            * jvm升级，参数调优等
        * 算法优化
            * 降低算法时间复杂度
        * 异步/非阻塞
            * 轮询 -> 事件通知
            * bio -> nio等
        * 多线程替代多进程
            * 甚至用多协程替代多线程
        * 缓存
            * 例如内存缓存和分布式缓存
    * 系统层
        * 资源隔离
            * cgroup现在进程资源/容器化
        * 优先级调整
            * 降低非核心进程的优先级
        * cpu绑定/独占
            * 让核心进程独享资源
        * 中断负载均衡 -> 充分利用cpu并行能力
        * NUMA
 
* 性能相关的两个指导原则
    * 避免过早优化
    * 利用28法则，只优化那20%的导致80%性能下降的问题


> 金句

**并不是所有的性能问题都值得优化**

站在企业的角度，任何占用人力资源的任务都是有成本的，性能优化也不例外。事实上，不是所有问题都
值得优化，在企业发展初期，有些问题是可以忽略的，应该把资源投入到最关键的地方，例如业务功能开发
等，等到用户量上来后，再针对性的进行处理。切记，不要为了优化而优化。

> 问题

1. 遇到过哪些性能问题？后来是怎么优化的?
2. 如何做性能测试，怎么保证测试不影响生产？

#### [13 | 答疑（一）：无法模拟出 RES 中断的问题，怎么办？](https://time.geekbang.org/column/article/73381)

> 笔记

* 性能工具当中的指标是如何计算出来的？
    * 以cpu使用率为例
        * 数据来源：/proc/stat
        * 周期性采样/proc/stat下不同场景的cpu时间计算
    * 例如cpu核心数
        * 数据来源: /proc/cpuinfo
    * 如果无法查询出来期望的指标
        * 升级版本
        * 采用别的工具
        * 去/proc 下查询

* 如何判断磁盘类型
    * lsblk -d
        * 查看参数rota -> 是否可旋转
            * HDD -> ro = 1 -> 可旋转
            * SSD -> ro = 0 -> 不可旋转
    * 磁盘类型影响io性能
        * 相同io压力下，不同类型磁盘表现不同
            * 是否导致iowait的升高

> 金句

**在碰到直观上解释不了的现象时，要第一时间去查命令手册**

文档是解决问题的最佳实践，基本上能够覆盖80%的问题。因此养成查询文档的直觉具有事半功倍的效果。
那么，如何提供文档查询的效率就很关键了，一方面要善于使用文档查询工具，快速检索；另一方面需要
对文档本身的组织架构清楚，在大脑里面形成一个索引，这要求至少把文档泛读一遍，形成模块化记忆。
对于linux工具的使用和指标的理解，基本上都能从man文档中找到答案，包括如何使用参数，输出指标
的含义等等。

> 问题

1. top命令中cpu的使用率是怎么计算的呢？你知道哪些常见指标的计算方法？
2. 遇到问题一般是怎么解决的？如何高效使用文档？


#### [14 | 答疑（二）：如何用perf工具分析Java程序？](https://time.geekbang.org/column/article/73738)

> 笔记

* 如何用perf分析Java进程？
    * 使用perf-map-agent 生成/tmp/perf-<pid>.map
    		* perf会去临时文件/tmp/perf-<pid>.map查找指定进程的符号表
    * 如何使用perf-map-agent
		* 如何使用curl下载github压缩包
			* curl -L -O https://github.com/jvm-profiling-tools/perf-map-agent/archive/master.zip 
		* 如何安装cmake并保证版本>2.8
			* yum install -y cmake
		* 编译构建
	    		* cmake .
				* CMake Error: your CXX compiler: "CMAKE_CXX_COMPILER-NOTFOUND" was not found.   Please set CMAKE_CXX_COMPILER to a valid compiler path or name.
		    		* yum install gcc-c++
		* 生成perf-java套件的link
			* bin/create-links-in <somedir>
				* perf-java-flames   
				* perf-java-record-stack  
				* perf-java-report-stack  
				* perf-java-top
	* 分析es进程
	    	* 使用perf-java-top
			* 错误: 找不到或无法加载主类 net.virtualvoid.perf.AttachOnce
		    	* 不支持java12?
		    	* 调整JAVA_HOME和es对齐，也无法解决该问题
* 推荐书目
    * 《性能之巅：洞悉系统、企业与云计算》

> 金句

**抓住主线不动摇，先从最基本的原理开始，掌握性能分析的思路，然后再逐步深入，探究细节，不要试图一口吃成个大胖子**

遇到问题时，尤其是复杂问题时，通常会感到很痛苦，这个时候不要着急，越是焦虑越无法解决问题。
正确的做法是分解问题，降低复杂度，然后逐个解决，搞清楚每个基础概念和原理，尽可能收集多的信息
，最后综合下来很可能就找到了解决问题的思路

#### [15 | 基础篇：Linux内存是怎么工作的？](https://time.geekbang.org/column/article/74272)

> 笔记

* 什么是内存映射？
    * 虚拟内存地址到物理内存地址的映射机制
        * 页表 -> 存储在cpu的内存管理单元MMU/TLB
            * 通过MMU通过TLB缓存页表
            * 页表
                * 页大小 -> 4kb一页
		* 如果页表使用线性结构，整个虚拟地址空间需要占用大量的页表项
                    * 如何避免页表项过多
                        * 多级页表 -> 本质上是个多叉树
			    * 仅保留使用中的页，因此可以节省多数未使用的页表项
			    * 一级 -> 二级 -> 三级 -> 偏移量 => 物理地址
				* 转换成线性结构: 1级 * 2级 * 3级
                        * 大页 -> 2MB、1GB
		    * 如何加速页表的访问
			* 使用块表 -> TLB
			    * 在cpu中缓存部分页表，缓存命中率较高的情况下可以加速访问
                * 什么时候刷新页表
                    * 缺页异常 -> 找不到地址映射
                        * 内核分配物理内存，刷新页表
    * 什么是虚拟内存?
        * 进程独享的内存地址空间
            * 内核空间 -> 内核态可访问
            * 用户空间 -> 用户态可访问
        * 地址空间 -> 32位/64位范围
            * 虚拟内存 >> 物理内存
            * 分段
                * 只读 -> 常量，代码
                * 数据 -> 全局变量
                * 堆 -> 动态分配内存
                * 文件映射 -> 动态库，共享内存(进程共享）
                * 栈 -> 局部变量，函数调用上下文

* 如何分配和回收内存？
    * 动态分配和释放
        * 堆 -> 适合小对象
            * brk()/free() 系统调用
            * 可重复使用，释放后不会立即归还系统
            * 容易导致内存碎片
        * 文件映射 -> 适合大对象
            * mmap()/unmap() 系统调用
            * 每次调用都会触发缺页异常
            * 释放后立刻归还系统
        * 物理内存分配管理机制
            * 页 -> 按页分配
            * SLAB -> 适合小对象
            * 其他
        * java的内存分配和回收
            * 交给jvm自动管理
   * 系统回收
        * 回收缓存(LRU)
        * swap -> 交换不常用的内存到磁盘
            * 可能导致性能严重下降，例如ES等对性能要求严格的服务会禁用swap
        * OOM -> oom_score 打分机制
            * 内存高，cpu占用低的进程oom_score较大
            * 手动打分 -> echo -16 > /proc/$(pidof sshd)/oom_adj
            * 查看OOM日志 -> dmesg |grep -E 'kill|oom|out of memory'

ps: 内存分配是懒加载模式，只有发生缺页异常才会实际分配物理内存

* 内存指标
    * free
        * total -> 总内存
        * used -> 已使用内存，包括share
        * free -> 未使用内存(排除了buff/cache/share
        * share -> 共享内存(共享 + 程序代码段 + 动态链接库）
        * buff/cache -> buffers(磁盘缓存) + cached(页缓存）+ SReclaim(可回收Slab)
            * slab -> 管理内核中的小块内存
        * available -> 可用内存
    * top
        * VIRT -> 虚拟内存（申请过的，即使没有实际分配物理内存）
            * 内核线程该指标=0
        * RES -> 常驻内存 (实际使用的物理内存，不包括swap)
        * SHR -> 共享内存(共享 + 程序代码段 + 动态链接库)
        * %MEM -> 物理内存占用 / 总内存

> 问题

1. 什么是缺页异常？如何理解linux系统中的虚拟内存?
2. free命令中的avalible可用内存指的是什么？包括buffer和cache吗?

#### [16 | 基础篇：怎么理解内存中的Buffer和Cache？](https://time.geekbang.org/column/article/74633)

> 笔记

* 怎么理解free指令中的buff/cache指标
    * 数据从哪里获取? 
        * buffer -> /proc/meminfo -> Buffers
            * Buffers -> 磁盘缓存
        * cache -> /proc/meminfo -> cached + SReclaimable
            * cached -> 页缓存
            * SReclaimable -> 可重用的slab内存

* 模拟文件写和磁盘写
    * 文件写 -> 查看buff和cache的变化
        * 清缓存：echo 3 > /proc/sys/vm/drop_caches
            * 为什么是3？
                * 查看proc文档 -> 3可以同时清理：pagecache,  dentries  and  inodes
        * 写文件: dd if=/dev/urandom of=/tmp/file bs=1M count=500
        * 查看buff/cache变化: vmstat 1
·       * -> 发现cache随着bo的增加明显增加
   * 磁盘写 
        * 清缓存：echo 3 > /proc/sys/vm/drop_caches
        * 写块设备: dd if=/dev/urandom of=/dev/sdb1 bs=1M count=2048
        * 查看buff/cache变化: vmstat 1
·       * -> 期望buff明细增加

ps: 现代的libux系统，写文件的时候不会经过两次cache，即不会走cache后又经过buffer


* 模拟文件读和磁盘读
    * 文件读
        * dd if=/tmp/file of=/dev/null
        * -> bi增加，同时cache也明细增加
    * 磁盘读
        * dd if=/dev/vda1 of=/dev/null
        * -> bi增加，同时buff也明细增加

* 如何统计进程的物理内存使用量？
    * top -> 取RES相加？
        * RES可能包括share,buff和cache，这些有可能是进程共享的，相加会出现重复
    * ps -> 取RSS相加
        * 可能的问题同top
    * /proc/<pid>/smaps -> 取Pss相加
        * pss = 私有的 + 共享的属于当前进程的部分
        * 例如share被5个进程共享，共500k, 私有100k
            * pss = 100 + 500 / 5 = 200k

* /dev/null,/dev/vdb1, 和/tmp/file 三种文件类型有什么区别?
    * 使用ls -l 查看，并通过 info coreutils 'ls invocation' 查看ls文档
        * d -> 目录
        * c -> 字符文件
        * b -> 设备块文件
        * n -> 网络文件
        * 

> 金句

**你一定要养成查文档的习惯，并学会解读这些性能指标的详细含义**

要学会调整自己的第一反应，遇到问题首先不是google，先想想，能否通过查阅文档
获取答案，文档实际上已经可以解决80%的问题；当然，前提是对文档以及它的用法
有一定了解，以便于我们快速检索定位目标问题。

#### [17 | 案例篇：如何利用系统缓存优化程序的运行效率?](https://time.geekbang.org/column/article/75242)

> 笔记

* 如何在centos环境下安装bcc工具套件
    * 如何查看内核版本？
        * uname -r -> 版本3.1
            * 内核版本 < 4.1 时无法安装bcc
    * 如何升级内核版本？
        * 更新yum源并下载最新的rpm包
            * rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
            * rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
            * yum --enablerepo=elrepo-kernel install kernel-ml-devel kernel-ml
        * 查看当前系统所有内核版本
            * rpm -qa | grep -i kernel
        * 切换新内核
            * cat /boot/grub2/grub.cfg | grep menuentry -> 查看内核完整名称
            * grub2-set-default 'CentOS Linux (5.7.1-1.el7.elrepo.x86_64) 7 (Core)' -> 切换内核
            * grub2-editenv list ->  查看当前内核
        * 重启
            * reboot
        * 确认
            * uname -r -> 5.7.1-1.el7.elrepo.x86_64
                * 内核已经升级到5.7
    * 安装bbc-tools
        * yum install -y bcc-tools
        * export PATH=$PATH:/usr/share/bcc/tools -> 添加环境变量
        * source ~/.bash_profile
        * 使用cachestat和cachetop
            * cachestat -> 查看系统整体的缓存命中率
            * cachetop -> 查看进程级别的缓存命中率

* 如何查看指定文件的缓存大小
    * 安装pcstat
        * 下载go 
            * curl -L -O https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
            * tar -xvf go1.14.4.linux-amd64.tar.gz
        * 添加环境变量
            * export GOPATH=~/go
            * export PATH=~/go/bin:$PATH
            * source ~/.bash_profile
        * 安装pcstat
            * 设置公共代理 -> 否则无法下载golang.org包
                * export GO111MODULE=on
                * export GOPROXY=https://goproxy.io
            * 下载包
                * go get golang.org/x/sys/unix
                * go get github.com/tobert/pcstat/pcstat
   * 使用pcstat
        * pcstat /bin/ls -> 查看ls文件的缓存    
            * echo 3 > /proc/sys/vm/drop_caches
            * pcstat /bin/ls -> 此时cache = 0
            * ls
            * pcstat /bin/ls -> 此时cache > 0
    
* 使用dd工具观察读文件时cache的变化
    * 先生成一个512M的文件
        * dd if=/dev/vda1 of=/tmp/file bs=1M count=512
    * 清空缓存
        * echo 3 > /proc/sys/vm/drop_caches
    * 读取文件,并用cachetop观察
        * dd if=/tmp/file of=/dev/null bs=1M  
            * 读速率：109M/s
            * dd READ_HIT: 50% -> 由于是第一次读，仅命中一半(元数据）
        * 再次读: dd if=/tmp/file of=/dev/null bs=1M
            * 读速率: 3.4 GB/s
            * dd READ_HIT: 100% -> 完全命中缓存
        * 查看文件缓存
            * pcstat /tmp/file
                * Percent = 100 -> 完全缓存 
                * Cached = 131072 -> 缓存页大小

* 如何发现应用io操作时是否绕开了缓存
    * cachetop -> 查看进程对应的HITS和READ_HIT
        * READ_HIT 是否100%
        * HITS 大小是否和读取量匹配
            * 页(4kb) * hits 
    * strace -> 查看系统调用
        * 是否出现O_DIRECT等direct io 选项

> 金句

**在案例开始前，你应该习惯性地先问自己一个问题，你想要做成某件事情，结果应该怎么评估**

这实际上是一种结果导向的思维，你可以想象，没有办法评估的结果，可能是多么糟糕的一个结果，
就像你无法评估系统发版后是否正常运行，这就相当于一个定时炸弹，随时引爆。如何评估，本质上
是提前确定一些量化的手段和工具，通过什么方式获取什么指标可以证明什么结果。它是高效的，关注
它能让我们在开发或实验过程中阶段性的建议我们的成果。

这里分享3句什么时候都应该优先思考的问题：

理性：

1. 没有它会怎么样？-> 逆向思维
2. 依据是什么？-> 批判思维
3. 评估指标有哪些? -> 结果思维
        
#### [18 | 案例篇：内存泄漏了，我该如何定位和处理？](https://time.geekbang.org/column/article/75670)

> 笔记

* 内存不足可能引发的问题
    * 新的进程无法启动
    * OOM导致进程被杀死
    * 触发swap机制(未关闭swap的情况下），导致性能下降

* 内存问题
    * 访问了非法的内存
    * 内存泄露导致内存消耗殆尽
        * 堆/文件映射段动态分配的内存未正确回收
        * 数据/只读/栈的段不会导致内存泄露

* 如何查找内存分配热点进程和代码
    * vmstat 3
        * 查看free, buffer和cache
            * buffer和cache不变的情况下，free持续下降
                * 说明有进程一直在申请内存,且当前没有释放
    * memleak -> bcc-tools 工具
        * memleak -a 
            * Exception: Failed to compile BPF text
        * memleak -a -p 21379 -> 前提是需要知道热点进程
            * 打印内存热点代码

> 线上实践

* 服务内存饱和
    * 现象
        * 物理内存分配6G（pod)
        * jvm heap 4G，metaspace 512M
        * 看k8s监控指标内存使用率接近100%
            * 堆实际物理占用只有20%-50%
    * 指标理解
        * 监控指标计算公式：
            * container_memory_usage_bytes = cache+rss+swap
        * 堆外被占用的实际上是cache部分，即页缓存
    * 结论
        * cache可以被回收，rss实际占用2G，内存远远没有饱和

* 服务young gc 时间相对较长
    * 现象
        * 服务堆分配4G，默认younggc最大占60% = 2.5G
        * 采用G1回收器
        * 看k8s监控，current和avg值平均在500ms，是其他应用的5-10倍
    * 分析
        * 分析gc日志，young gc 平均耗时实际上只有18ms，与监控值不符
        * 指标含义：1min内累计young gc时间
            * 指标值 = n次/min * t ms/单次
            * 平均耗时不高的情况下，说明gc次数较多
        * 查看gc次数: > 100次/min，是其他应用的10-100倍
        * 堆使用率在50%左右, young区峰值0.5G,未到2.5G
        * 对比堆使用率相当的另外一个服务，比较jvm参数
    * 结论
        * g1参数maxGcPauseMills=200 可能设置过小, 调大为800
            * 待验证

#### [19 | 案例篇：为什么系统的Swap变高了（上）](https://time.geekbang.org/column/article/75797)

> 笔记

* 系统应对内存不足时的机制有哪些？
    * 回收内存
        * 回收buffer/cache等可回收内存
            * 回收文件页
                * 脏页怎么回收？
                    * 同步磁盘后释放
                        * fsync 系统调用
                        * pdflush定时刷新
        * swap换出进程堆内存
            * 回收匿名页
    * 系统怎么选择回收内存的方式?
        * 是否开启swap
            * 关闭 -> 不会进行swap
            * swappiness -> 值越大，swap越积极
               * /proc/sys/vm/swappiness -> 设置swappniess
    * OOM
        * 根据oom_score杀死内存消耗大，cpu消耗小的进程
    
* swap如何交换内存？什么时候触发swap?
    * 交换
        * 换出 -> 将内存写入磁盘并释放内存
        * 换人 -> 从磁盘读入内存并释放磁盘
    * 时机
        * 分配内存时，内存不可用 -> 直接内存回收
        * 内核线程定期检测回收 -> kswapd0(比较阈值回收）
            * 查看阈值
                * cat /proc/zoneinfo -> cat /proc/zoneinfo | grep -i normal -A 15
                    * pages_min -> 最小阈值
                    * pages_low -> 低阈值
                    * pages_high -> 高阈值
                    * pages_free -> 剩余内存
                * 当free < low 时，触发swap，使free回到high以上
            * 调整阈值
                * /proc/sys/vm/min_free_kbytes
                    * low = min * 5 / 4
                    * high = min * 3 / 2
    * 如何查看是否开启swap机制？
        * free -h 显示swap.total = 0 
            * 说明swap未开启

    * 为什么free显示内存充足的情况下仍然发生swap?
        * NUMA架构下node本地内存不足触发swap
            * 查看有几个node及其可用内存
                * numactl --hardware
                    * normal -> 普通内存区
                    * DMA -> 直接内存访问区
                    * MOVABLE -> 伪内存区
            * 查看node的内存区和阈值
                * cat /proc/zoneinfo | grep -i normal -A 15
                * 当free < low时会触发
                    * 调整NUMA回收策略
                        * echo [数值] > /proc/sys/vm/zone_reclaim_mode
                            * 0 -> 从其他node获取空闲内存或从本地内存获取
                            * 2 -> 刷新脏页回收内存
                            * 4 -> swap方式回收内存           
    * elasticsearch对swap机制的限制
        * 避免jvm 堆被换出，gc时重新换入导致性能下降（耗时数分钟）
            * 关闭系统swap机制
                * swapoff -a -> 运行时关闭
                * /etc/fstab -> 永久关闭
                    * 注释swap相关的行
            * es 服务增加内存锁
                * vim config/elasticsearch.yml -> bootstrap.memory_lock: true
                    * GET _nodes?filter_path=**.mlockall -> 查看是否锁定内存
            * 降低系统swap倾向
                * vim /proc/sys/vm/swappiness -> vm.swappiness=1
                * 该方法不能完全避免swap发生

> 问题

1. 内存不足时，系统通过哪些机制回收内存？
2. jvm应用是否开启内存交换机制？可能有什么问题？如何关闭swap?

#### [20 | 案例篇：为什么系统的Swap变高了？（下）](https://time.geekbang.org/column/article/75973)

> 笔记

* 如何开启swap
    * fallocate -l 1G /mnt/swapfile -> 预分配1G的空间给指定swap文件
    * chmod 600 /mnt/swapfile -> 禁止非root读写swap
    * mkswap /mnt/swapfile -> 创建分区文件
    * swapon /mnt/swapfile -> 开启swap
        * swapon: /mnt/swapfile: swapon failed: Invalid argument (centos7)
            * 猜想：fallocate机制导致swap分配失败 -> 用dd写入文件
        * dd if=/dev/zero of=/mnt/swapfile bs=1G count=1
            * 重复上诉过程
    * free -h 查看swap是否已经分配成功

* 如何模拟内存吃紧的场景?
    * 如何降低free?
        * dd -> 磁盘读写，free -> buffer 
        * dd if=/dev/vda1 of=/dev/null bs=1G count=20
            * 磁盘读写会消耗内存作为buffer
    * 如何观察内存变化?
        * sar
            * sar -r -S 1
                * %memused -> 内存使用率
                * %swpused -> 交换区使用率
        * 发现内存使用率和buffer升高，但是swapused并未升高
            * swap机制未触发
    * 如何调整swap机制触发优先级
        * echo 60 > /proc/sys/vm/swappiness 
        * 查看swap阈值相对变化
            * watch -d grep -A 15 -i normal /proc/zoneinfo
        * 执行 dd if=/dev/vda1 of=/dev/null bs=1G count=20
            * 观察/proc/zoneinfo 发现free在low和hight之间循环变化
            * 观察 sar 输出，swapused升高，说明swap机制触发
            * 说明当free<low时触发了swap并使free恢复到hight以上的水平
    * 如何查看swap使用较多的进程
        * 进程的swap使用情况 -> /proc/*/status
            * smem --sort swap
    * 如何关闭swap
        * swapoff -a
    
* swap机制的一些说明
    * 即使开启swap，也未必会使用它，至少需要调整2个地方
        * vm.swappniess 优先级是否调高
        * /proc/zoneinfo 中free 是否 < low
            * 可通过 /proc/sys/vm/min_free_kbytes 调整
    * swap中的free包括可用内存+cache(文件页），但是不包括buffer
        * 磁盘io导致buffer升高依然会触发swap

> 最佳实践

参考es的配置，jvm应用应该禁用或尽可能避免swap的发生

* 系统层面关闭swap
* 系统层面调低swap倾向(vm.swappniess)
* 进程锁定内存 -> mlock()或mlockall()
* jvm堆初始和最大值一致，避免内存扩展时内存不足发生swap换入

#### [21 | 套路篇：如何“快准狠”找到系统内存的问题？](https://time.geekbang.org/column/article/76460)

> 笔记

* 和内存有关的指标有哪些？
    * 系统指标 -> top/vmstat/sar
	* total -> 总内存
	* used -> 使用内存
	* free -> 剩余内存
	* available -> 可用内存
	    * available包括已使用的可回收内存(buff/cache), 一定比free大
	* buffer -> 磁盘缓存 
	* cache -> 文件页缓存
	    * 缓存命中率 -> cachetop/cachestat
	    * 缓存使用量 -> pcstat
	* slabs -> 可回收slab内存 ?
	    * 大小通常算在cache内，例如free看到的cache包括reclaimable slab
	* swap -> 交换内存 
	    * 换入速度 -> sar -W
	    * 换出速度 -> sar -W
    * 进程指标 -> smem/pidstat
	* vss -> 虚拟内存
	    * 已经申请的虚拟内存，即使还没有被分配物理内存(缺页异常）
	* rss -> 常驻内存
	* uss -> 独占内存
	* pss -> 按比例分配共享内存后的物理内存
	* share -> 共享内存
	* swap -> 交换内存
	* 缺页异常 ? -> pidstat -r 
	    * 次缺页异常 -> 从物理内存中分配
	    * 主缺页异常 -> 从磁盘中分配，例如swap换入
	* mem% -> 内存使用率 = rss/total
	* share -> 共享内存
    
* 和内存相关的问题有哪些？
    * oom 
	* memleak 查找内存热点代码
	* 调整oom_score避免核心应用被杀死
	* 资源隔离 -> cgroup
    * swap导致性能下降
	* smem找出swap热点进程
	    * 调整swap倾向或关闭swap
    * 大量缺页异常
	* pidstat 查看缺页异常热点进程

> 感想

理解性能问题需要时刻记住两个关键词：指标和关联。性能分析本质上是指标分析，先搞清楚该性能是通过
哪些指标衡量的，然后怎么去获取这些指标，包括实时和历史的数据；最后才是分析，定位，找到异常的指标
并确定瓶颈。如果说性能问题解决分为定位瓶颈和性能优化，那么定位即找出异常的指标，优化即评估指标是
否达到预期。而在分析的时候，不能孤立的看待问题，要把指标关联起来看，单个指标要关联历史和现在，对
比分析，关注其变化趋势，才能快速定位。

#### [22 | 答疑（三）：文件系统与磁盘的区别是什么？](https://time.geekbang.org/column/article/76675)

> 笔记

* 回收内存的三种策略
    * 文件页回收 -> buffer/cache的回收
	    * LRU -> 回收不活跃(inactive)的文件页
	    * 查看活跃/不活跃内存页
	        * cat /proc/meminfo | grep -i active
        * 如何查看进程buff/cache的动态变化？
            * 只能查看系统层面的动态变化，无法查看进程级别
                * cachetop -> 查看进程级别的缓存命中率
                * pidstat/ps -> 查看进程的虚拟内存，物理内存和内存使用率
    * 匿名页回收 -> swap
	    * LRU
        * 可以查看进程级别的swap使用情况
            * smem
    * OOM -> 根据oom_score 
	    * 内存计算
	        * 进程申请的虚拟内存 + used < free 则会触发
		    * 可通过overcommit机制避免oom
	* 查看OOM日志
	    * dmesg -T | grep -i "Out of memory"

* 如何计算进程消耗的总物理内存？
    * 累计指标Pss -> 独占物理内存(uss) + 按比例分配后的共享内存
        * smem 累计Pss
            * smem | awk '{total+=$7}; END{printf "%d\n", total}'
        * 提取/proc/[1-9]*/smaps
            * grep Pss /proc/[1-9]*/smaps | awk '{total+=$2}; END {printf "%d kB\n", total }'

> 金句

**比如，在需要大量内存的场景中，你就可以考虑用栈内存、内存池、HugePage 等方法，来优化内存的分配和管理**

回到jvm应用上面，我们如何监控内存泄露问题呢？频繁gc或gc后内存释放不明显，是否就意味着内存泄露？如何排查
内存泄露的热点代码？这些是接下来需要去进一步探讨的问题。


#### [23 | 基础篇：Linux 文件系统是怎么工作的？](https://time.geekbang.org/column/article/76876)

> 笔记

* 磁盘与文件系统的区别是什么？
    * 磁盘是持久化存储的媒介
    * 文件系统是磁盘之上对文件进行组织管理的一种机制
	* 树状结构
	    * 目录项（dentry) -> 目录项之间相互关联形成目录树(文件系统结构）
		* 文件名/父子目录节点/索引节点指针
		* 不进行持久化，内核维护的数据结构
	    * 索引节点(inode) -> 记录文件的元数据 -> 文件的唯一标识
		* 文件名/访问权限/文件大小/创建日期等
		* 持久化 -> 磁盘的索引区
		* 缓存 -> inode cache 

* 如何设置硬链接并查看他们的索引节点？
    * ln a.txt a_2.txt -> 创建a.txt的硬链接a_2.txt
	* ls -il 查看 -> 第一列为inode
	* inode相同
    * 硬链接只是创建文件的dentry，它们的inode指针相同
    * 软链接创建出来一个附属文件，它们的inode指针不同

* 文件系统是如何规划磁盘的？
    * 超级块 -> 存储文件系统的状态
    * 索引节点区 -> 存储索引节点
    * 数据块区 -> 存储文件数据
	* 逻辑块为单位
	    * 1 逻辑块 = 8 扇区 = 8 * 512B = 4kb

* 文件系统有哪些分类
    * 磁盘文件系统 -> 数据存储到磁盘
	    * Ext4, XFS
    * 内存文件系统 -> 不持久化(虚拟）
	    * /proc,/sys
    * 网络文件系统 -> 可访问其他机器数据
	    * NFS,SMB

* 如何访问文件系统中的文件
    * 通过VFS提高的统一接口以挂载目录的方式访问底层不同类型的文件系统
	* 文件系统需要挂载到VFS的指定目录
	    * 磁盘文件系统 -> /
	    * 内存文件系统 -> /proc, /sys等
	    * 网络文件系统(NFS) -> /nfs 等
	* VFS统一接口是通过系统调用（函数）的方式提供
	    * 例如cat
		* open()/read()/write()
    * IO
	* 是否使用标准库的缓存(注意不是cache/buffer)
	    * 缓冲io和非缓冲io
	* 是否使用page cache
	    * 直接IO -> 不使用page cache，直接跟文件系统交互
	    * 非直接IO -> 使用page cache
	    * 裸IO -> 跳过文件系统直接和磁盘交互
	* 是否阻塞IO线程
	    * 阻塞IO和非阻塞IO
	* 是否等待IO结果
	    * 同步和异步IO

* 如何评估文件系统的性能？
    * 容量 -> 文件数据或索引节点容量不足都会导致性能问题 
        * 文件数据容量 -> df -h
        * 索引节点容量 -> df -ih
            * 小文件过多可能导致inode容量耗尽
    * 缓存
        * 页缓存
            * page cache
        * 索引节点/目录项缓存
            * slab -> /proc/slabinfo
                * cat /proc/slabinfo | grep -E '^#|dentry|inode' -> 查看slab占用情况
                * slabtop -> 分析slab缓存占用百分比

* find / -name xxx 对缓存使用情况分析
    * 先清空缓存 -> echo 3 > /proc/sys/vm/drop_caches
    * 执行vmstat 观察buffer和cache的变化
    * 执行slabtop 观察dentry和inode_cache的变化
    * 执行find / -name xxx ，对照观察vmstat
        * 指标是否升高？哪个指标升高
            * vmstat指标：buffer 和 cache 都有升高 -> 说明查找时写入缓存
                * 注意，cache = 页缓存 + reclaimableSlab(目录和索引节点缓存）
            * slabtop指标：
                * proc_inode_cache 增加，内存文件系统索引节点缓存增加
                    * 因为会扫描/proc目录
                * ext4_inode_cache 增加（增加明显），本地文件系统所有节点缓存增加
                * dentry 增加（增加明细）
                * inode_cache 增加
    * 再次执行 find / -name xxx
        * 执行非常快，说明利用了缓存

#### [24 | 基础篇：Linux 磁盘I/O是怎么工作的（上）](https://time.geekbang.org/column/article/77010)

> 笔记

* 如何评估磁盘的性能
    * 磁盘类型 -> fdisk -l
       * 固态硬盘 -> sectors 
            * 读写单位是页，通常是4kb
       * 机械硬盘 -> 包含head,tracks和cylinder 
            * 读写单位是扇, 512B, 文件系统以8扇为一个逻辑块进行管理 = 4k
    * 磁盘架构
        * 独立磁盘
            * 例如/dev/vda，可进行分区
        * 磁盘阵列 -> 将多块磁盘合并成一个逻辑磁盘(RAID)
            * 冗余
            * 提供读写性能
        * 网络磁盘
            * NFS/SMB/ISCSI
            * 例如作为日志文件收集目录，服务发现目录等

* IO栈中的通用块层
    * 为什么叫做块层?
        * 磁盘实际上是作为一个块设备来进行管理的，以块为单位读写数据
    * 通用块层的作用？
        * 解耦，类似于VFS, 提供标准接口，实现上下层解耦，可以灵活扩展
            * 上层：文件系统/用户进程
            * 下层：磁盘
        * io调度，提高性能
            * NONE -> 虚拟机io，不进行调度
            * NOOP -> FIFO
                * 适合SSD
            * CFQ -> 进程隔离，每个进程一个队列
            * DeadLine -> 读写隔离，分别设置读写队列
                * 适合数据库等IO压力较大的场景

> 感想

对比VFS和通用块层的设计思想，在应用架构的设计中，也可以借鉴这种模式。应用往往也是分层或
最后通过分层的方式来管理，不同分层之间需要进行交互，为了保证层的自由扩展，就需要解耦。解耦
最通用的做法是增加一个抽象层，它是稳定的，可以隔离经常变化的层。
