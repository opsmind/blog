## OpsMind 对接 Dana 日志内容规格

### 请求包：
```
{
    project: <string>,
    did: <string>,
    ouid: <string>,
    timestamp: int64,
    event: <string>,
    properties: {
        "file: <string>,
        "pid: <string>,
        "service: <string>
    }
}
```

* `project` : OpsMind 日志收集规则中指定的日志类型，如nginx syslog common_logs等
* `did` : 服务器 hostname
* `ouid` : 服务器 hostname
* `timestamp` : 毫秒单位的 unix epoch
* `event` : 原始日志
* `properties.file` : 日志文件路径
* `properties.pid` : 产生该日志的进程的 pid
* `properties.service` : 产生该日志的服务名
