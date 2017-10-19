## OpsMind 系统基础监控指标

### A. 机器级别

#### CPU Load 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_load_avg | gauge | host, dur | 最近 $dur(1/5/15) 分钟内的平均负载值 | 1
dog_cpu_time | counter | cpu, host, mode | 从机器开始运行起，所有 CPU 处于 $mode(user/iowait/irq/softirq/system/idle) 模式下的总时长 | second

#### Memory 类

metric_name | type | tags | desc | unit | memo
------------ | ------ | ----| ----- | ----| ----
dog_mem_main_size | gauge | host, mode | 机器主内存中，处于 $mode(buffer/cache/free/used) 模式下的内存大小 | byte | total = buffer + cache + free + used, avail = buffer + cache + free
dog_mem_swap_size | gauge | host, mode | 机器交换内存中，处于 $mode(used/free) 模式下的大小 | byte

#### Network 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ---- 
dog_netdev_bytes | counter | dev, host, dir | 网卡设备在 $dir(rx/tx) 方向上累计传输的流量大小 | byte 
dog_netdev_packets | counter | dev, host, type, dir | 网卡设备在 $dir(rx/tx) 方向上累计捕获到 $type(drop/error/success) 类型的包数量 | 1 

#### Disk 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_disk_io_complete | counter | host, dev, type | 磁盘设备在 $type(r/w) 模式下累计完成的 IO 次数 | 1
dog_disk_io_bytes | counter | host, dev, type | 磁盘设备在 $type(r/w) 模式下累计完成的 IO 吞吐总量 | byte
dog_disk_io_time | counter | host, dev, type | 磁盘设备在 $type(r/w/all) 模式下累计消耗的总时长 | ms

#### File System 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_fs_size | gauge | host, dev, mode | 文件系统中，处于 $mode(used/free) 状态下的磁盘大小 | byte
dog_fs_files | gauge | host, dev, mode | 文件系统中，处于 $mode(used/free) 状态下的 inode 数量 | 1

### 其他类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_uptime | counter | host |  机器自运行起经过的时长 | second
dog_contexts | counter | host | 操作系统内核自运行起累计发生的总上下文切换次数 | 1
dog_procs | gauge | host, mode | 操作系统中处于 $mode(run/block) 状态下的进程数量 | 1

### B. 服务级别

#### B.1 资源

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ---
dog_service_mem | gauge | host, service, type | 服务占用 $type(vm/rss/swap) 类型的内存大小 | byte
dog_service_fd | gauge | host, service | 服务占用的总句柄数量 | 1
dog_service_cpu | gauge | host, service, mode | 服务平均每秒（墙上时间）内占用 $mode(sys/user) 模式下的总 CPU 时间片长度 | second
dog_service_disk_io | gauge | host, service, type, mode | 服务平均每秒内对磁盘进行 $mode(r/w) 操作的统计，$type(rate) 表示 IO 次数，$type(bw) 表示吞吐量  | 1/byte
dog_service_ctx_switch | gauge | host, service, type | 服务平均每秒内引发操作系统进行 $type(voluntary/involuntary) 模式的上下文切换次数 | 1
dog_service_instances | gauge | host, service | 服务正在运行的实例数量（非进程数量） | 1
dog_service_threads | gauge | host, service | 服务正在运行的线程数量 | 1

**说明**

1. 服务级别的资源指标一律会将该服务下的所有子进程纳入统计
2. 服务的实例数量并非该服务的进程数量，实例与实例之间无进程父子关系
