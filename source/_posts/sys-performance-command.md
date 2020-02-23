---
title: 查看系统性能的命令
date: 2019-04-21 13:43:39
tags: operating system
---
- 查看cpu信息
cat /proc/cpuinfo

- vmstat
是个linux下的命令，查看系统状态
可以加两个参数指定采样的间隔时间个总的采样次数，如：vmstat 2 10，为两秒一次，共显示十次
显示的值有：procs, memory, swap, io, system, cpu几个部分
procs： 
r：等待运行的进程数
b：阻塞的进程数
memory：
swpd：虚拟内存swap已使用的大小
free：空闲的物理内存的空闲
buff：用于buffers的内存大小，包括file meta和设备交互的数据
cache：用于cache的内存大小，包括文件数据和mmap调用后的内存映射信息。和buff的区别是buffer和设备交互，从设备读取或写到设备（flush），但是cache是在文件系统内部交互的。
swap：
si：从磁盘到swap的内存数据，每秒
so：从swap到磁盘的内存数据，每秒
IO：
bi：从块设备（如磁盘）进入到系统的块数目（blocks/s）
bo：从系统输出到块设备的块数目（blocks/s）
system：
in: interrupt，每秒的中断数
cs: context switch，每秒的上下文切换数
CPU：
us: user time, 用户CPU时间
sy: system time，系统CPU时间
id: idle，空闲的CPU时间,在2.5.61前包括了io等待时间
wa: waiting for io ，CPU等待IO的时间
st: stolen from virtual machine

- top
    开头是统计区，统计了系统，进程，内存，swap的信息
    load average：系统1min，5min和15min内的负载情况。一般要除以CPU核数，最佳值是1。
    进程内显示的是各个区域在CPU使用时的占比，除了us，sy等外
    ni：改变优先级的进程占CPU使用率的比
    hi：hard interrupt硬中断
    si：soft interrupt
  
    进程状态监控中：
    PR：进程优先级
    NI：nice，负值表高优先级，正值表低优先级
    VIRT：进程使用内存总量，包括虚拟内存和物理内存，单位都是KB
    RES：进程使用的物理内存大小
    SHR：共享内存大小
    S：进程状态。R：运行，S：睡眠，D，不可中断的睡眠，T：停止，Z：僵尸进程。
    
    使用技巧：
    按1，查看多核运行情况
    shift + p/m按cpu和内存占用排序

- ip -s link
也是一个linux下的命令，link表示显示链路层信息，-s表示statstic

- df -h
查看磁盘

- free -h
查看内存
