## OpsMind 系统基础监控指标


#### CPU Load 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_load_avg | gauge | host, dur | recent $dur(1/5/15) min average load | 1
dog_cpu_time | counter | cpu, host, mode | seconds the cpus spent in each $mode(user/iowait/irq/softirq/system/idle) | second

#### Memory 类

metric_name | type | tags | desc | unit | memo
------------ | ------ | ----| ----- | ----| ----
dog_mem_main_size | gauge | host, mode | $mode(buffer/cache/free/used) main memory size | byte | total = buffer + cache + free + used, avail = buffer + cache + free
dog_mem_swap_size | gauge | host, mode | $mode(avail/free) swap memory size | byte

#### Network 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ---- 
dog_netdev_bytes | counter | dev, host, dir | $dir(rx/tx) transfer bytes | byte 
dog_netdev_packets | counter | dev, host, type, dir | $dir(rx/tx) $type(drop/error/success) packets | 1 

#### Disk 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_disk_io_complete | counter | host, dev, type | total number of $type(r/w) completed successfully | 1
dog_disk_io_bytes | counter | host, dev, type | total bytes of $type(r/w) completed successfully | byte
dog_disk_io_time | counter | host, dev, type | total io time spend on $type(r/w) | ms

#### File System 类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_fs_size | gauge | host, dev, mode | $mode(used/free) space size in bytes | byte
dog_fs_files | gauge | host, dev, mode | $mode(used/free) file nodes | 1

### 其他类

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ----
dog_uptime | counter | host |  up time, in unixtime | second
dog_contexts | counter | host | total number of context switches | 1
dog_procs | gauge | host, mode | number of processes in $mode(run/block) state | 1
dog_forks | counter | host | total number of forks | 1
dog_filefd | gauge | host | allocated file descriptors | 1

### B. 服务级别

#### B.1 资源

metric_name | type | tags | desc | unit
------------ | ------ | ----| ----- | ---
dog_service_mem | gauge | host, service, type | current $type(vm/res/swap) mem size used by this service | byte
dog_service_fd | gauge | host, service | current fd numbers used by this service | 1
dog_service_cpu | counter | host, service, mode | seconds spent by this service in each $mode(sys/user) | second
dog_service_disk_io | counter | host, service, type, mode | $type(rate/bw) of disk $mode(r/w) | 1/byte
dog_service_ctx_switch | counter | host, service, type | $type(voluntary/nonvoluntary) context switch count | 1
dog_service_instances | gauge | host, service | current instance number of an service | 1
dog_service_threads | gauge | host, service | current threads count for an service | 1
