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

言外之意，我们通常只知道某列或少数几列的含义，似懂非懂，对工具掌握不全面，容易陷入"铁锤人"思维。

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
的进程之间发生调度。

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

基于指标的分析，前提是我们要充分了解工具指标的含义和计算方法，它是否准确，是否适用于当前的性能场景。
为了避免单指标误判，我们最好是采用多指标多工具分析法，相互佐证。

> 问题

1. 僵尸进程是怎么产生的？如何排查僵尸进程?
2. 如何排查io热点进程及其对应热点代码？
