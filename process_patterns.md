# process config 指南

## 概览

OpsMind™ 的 agent 在抓取进程信息时会根据一组配置来决定进程的名字、进程协议分析、进程资源数据收集、进程日志分析、进程日志回传等行为，这组配置在 OpsMind™ 中叫作 `process config` ，`process config` 存放在 OpsMind™ 的服务端，并自动下发到每个 agent 上。一般情况下，我们不需要配置 `process config` ，这样OpsMind™ 会使用系统默认的 `process config`，系统默认的 `process config` 在大多数时候可以工作得很好；在另外一些时候，我们会主动配置 `process config` 以改变系统默认的行为，这会使 OpsMind™ 更加适合被监控的环境，比如我们希望分析或回传某些进程的日志，或是将 java、python 等进程名字扩展为更具业务意义的字符串。通过主动配置 `process config`，我们还可以使用 docker name 或 k8s pod name 这样的信息去命名一个进程。

这篇文档会说明 `process config` 位置和结构、参数，并提供几个样例。

## 位置和结构

`process config` 是一段 json，它存放在 ustg 的  `process_config` 分类下，可以使用 `duck ustg` 相关命令管理。在 ustg 的 `process_cofnig` 分类下，名字为 `default` 的配置会被下发到所有 agent，除非某台服务器有特殊的配置。ustg 的 `process_config` 分类下，名字为 `host:<HOSTNAME>`(如 `host:web1`) 的配置为针对某台服务器的特殊配置，特殊配置会覆盖全局配置（` default` ）。不管是全局配置还是特殊配置，都有一样的结构，这个结构描述如下：

```
{
  "process_patterns": [
    {
      "search_in": <string>,
      "regexp": <string>,
      "process_config": {
        "name": <string>,
        "list": <string>,
        "protocol": <string>,
        "res_metric": <string>,
        "service": <string>,
        "slog_file_patterns": [
          {
            "path_regexp": <string>,
            "typ": <string>,
            "groks": [
              {
                "name": <string>,
                "match": <string>,
                "labels": {
                  <string>: <string>,
                  ...
                },
                "value": <string>,
                "aggr": <string>
              },
              ...
            ]
          },
          ...
        ]
      }
    },
    ...
  ]
}
```

可以使用 duck 命令行工具管理 `process_patterns`：

* `duck ustg read process_config default`：读取当前 `process_patterns` 配置
* `duck ustg upsert -c process_config -n default -j -f your_process_patterns.json`：将配置更新到 OpsMind™ 服务端。





## 参数

### process_patterns

process config 的最上级结构，是一个数组。process config 基于“匹配”工作，`process_patterns` 定义了多个“进程模板”，agent 处理在服务器上发现的每一个进程，对任何一个进程，agent 会按顺序匹配每一个“进程模板”（匹配逻辑见 `search_in` 和 `regexp` 两个字段），如果有任何一个“进程模板”匹配成功，则匹配过程停止，该进程使用匹配到的模板进行下一步处理。

### search_in

定义一个待匹配字符串，agent 会对 `search_in` 使用 `regexp` 匹配（match），如果匹配成功，则使用当前的“进程模板”，如果匹配未成功，则按 `process_patterns` 的列表顺序尝试匹配下一个“进程模板”。

在 `search_in` 中，支持使用一些变量，agent 会根据当前处理进程的上下文信息将 `search_in` 中的变量替换为实际的值，`search_in` 支持的变量有：

* `${exe}`：进程的二进制路径，如 `/path/to/nginx` ，一般用于取得进程的名字。
* `${pwd`：进程的工作目录，如`/home/user/app`，相同二进制的进程，如果希望有不同的名字，可使用工作目录区分。
* `${cmdline}`：进程运行的命令行参数，不同参数间以`"\0"`分隔，如：`"python\0-m\0SimpleHTTPServer\08080"`。
* `${comm}`：进程的comm。
* `${user}`：运行进程的用户名。
* `${endpoints}`：进程的监听地址，多个监听地址以`,`分隔，如：`127.0.0.1:83,0.0.0.0:443`，这个变量可用于一个进程是否为网络服务的判断，端口信息也可用于网络协议的识别。
* `${pid}`：进程的pid。
* `${uid}`：运行进程的用户id。
* `${docker_image}`：进程所属 docker 的 image 名字。如果进程不在 docker 中运行，这个变量会被替换为空字符串（`""`）。
* `${env-X}`：进程的环境变量，`X`为环境变量的名字，如 `${env-PATH}` 会被替换为 `/bin/:/usr/local/bin`，如果指定的环境变量不存在，这个变量会被替换为空字符串（`""`）。
* `${docker_label-X}`：进程所在 docker 的 label，`X`为 label 的名字，如 `${docker_label-io.kubernetes.container.name}` 会被替换为 `some_pod`。如果进程不在 docker 中运行，或指定的 label 名字不存在，这个变量会被替换为空字符串（`""`）。这个变量可用于使用 docker 或 k8s 的信息命名或区分进程。
* `${jvm_stat-X}`：进程所在 jvm 中的 stat 信息，`X`为 stat 中的名字（key），如 `${jvm_stat-sun.rt.javaCommand}` 会被替换为 `some_java_command`。如果进程不在 jvm 中运行，或指定的 stat 名字不存在，这个变量会被替换为空字符串（`""`）。这个变量可用于使用 jvm 的信息命名或区分进程。所有 jvm stat 列表可使用 `jinfo <pid>` 获得。

示例：如一个 `search_in` 字符串为 `"exe:${exe},user:${user}"`，则 agent 处理 nginx 时会将该字符串替换为 `"exe:/usr/sbin/nginx,user:www-data"`，替换后的字符串会被 `regexp` 匹配。

### regexp

定义一个正则表达式，agent 会使用这个正则表达式去匹配 `search_in` 字符串，匹配的结果会作为该“进程模板”对当前进程的匹配结果，正则表达式中的 Capturing Group 可被 `name` 字段使用。

示例：假设 `name` 字段为 `"$1($2)"`（参见 `name` 字段文档）， 如一个 `regexp` 为 `"exe:.*/(.+)\\,user:(.+)"`，则对于不同 `search_in` 有如下结果的进程命名：

* `"exe:/usr/sbin/nginx /etc/nginx,user:www-data"`：`nginx(www-data)`
* `"exe:/usr/sbin/nginx /etc/nginx,user:root"`：`nginx(root)`

### process_config

在“进程模板”中，除去匹配相关的 `search_in` 和 `regexp` 字段，都为进程配置信息，这些信息都在 `process_config` 结构中表示。agent 根据 `process_config` 中的信息处理每一个进程，具体的处理方式见下列字段描述。

### name

指定一个字符串，agent 会依此字符串命名进程。`name` 字段支持获取 `regexp` 字段中的 Capturing Group，用 `$1`、`$2`... 表示（参加 `regexp` 示例）。

### list

指定一个为 `always` 或 `none` 的字符串，表示该进程是否在进程列表中出现，默认为 `always`。

### protocol

指定一个字符串，标识进程的网络协议，agent 将根据标识的协议进行网络包分析，支持的协议有：

* `none`：无协议
* `http`：http 协议
* `resp`：RESP 协议，一般为 redis 所用
* `mysql`：mysql 协议
* `mongo`：mongodb 协议

### res_metric

指定一个为 `full` 或 `none` 的字符串，表示 agent 是否收集该进程的资源数据，默认为 `full`。

### service

指定一个字符串，标识进程所代表的服务名。（该字段在之后版本中将废弃，服务名将自动与进程名一致）

### slog_file_patterns

agent 带有日志收集和分析功能，这些功能通过 `slog_file_patterns` 配置。

`slog_file_patterns` 是一个数组，agent 会将进程打开的日志文件路径与该进程所属模板中的`slog_file_patterns` 中所配置的“日志模板”匹配，如任何一个“日志模板”匹配成功，则停止匹配，使用当前“日志模板”处理该日志文件。

### path_regexp

指定一个正则表达式，agent 使用该正则表达式匹配进程打开日志文件的绝对路径，匹配结果作为该“日志模板”的匹配结果。

### typ

指定一个字符串，标识该“日志模板”的日志类型。该字符串会作为一个元信息填加在每条日志上。

### groks

agent 可以通过 `groks` 字段的配置实现日志分析，并自动将分析结果以监控指标方式存储。`groks` 字段为一个数组，数组中可定义多种分析方式，不同分析方式间为并列关系。

### name

指定一个字符串，表示监控指标的名字。监控指标将以和自定义监控指标一样的方式回传、存储和查询，所以监控指标的名字不能与自定义监控指标中的名字相同。在存储时，监控指标名字会被加上 `_exported_` 前缀。

### match

指定一个 [Grok](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) 表达式。该“日志模板”处理的每行日志都会使用该表达式分析，并将指定了名字的grok pattern提取，供 `labels` 和 `value` 字段使用。如：`"%{IP:ip} - - \\[%{HTTPDATE}\\] %{QUOTEDSTRING} %{INT:code} %{INT:dur} %{QUOTEDSTRING} %{QUOTEDSTRING}"`。

`match` 支持[常见 Grok pattern](https://github.com/vjeantet/grok/blob/master/patterns.go)，如果需要扩展，可将 Grok pattern 文件放置在 `/var/lib/om_grok_patterns/` 目录。

### labels

指定一系列 key/value 对，key 和 value 都为字符串。key 为监控指标中的label name，value 为监控指标中的 label value，value 支持 [golang template](https://golang.org/pkg/text/template/) 。如：`"code": "{.code}"` 或 `"static": "abc"`。

### value

以字符串方式指定监控指标的值，支持 [golang template](https://golang.org/pkg/text/template/)。`value` 字符串必须可以转换为 float 数字。如：`"value": "{.dur}"` 或 `"value": "42.0"`。

### aggr

以字符串方式指定一个聚合方法。在一个上报周期（约30秒）内，同一个监控指标下相同 labels 的数值会按指定聚合方法聚合运算，支持的聚合方法有：

* `sum`：求和
* `avg`：求平均
* `count`：计数
* `last`：取最后一个值
* `rsum`：求和后除以时间（单位为秒）
* `ravg`：求平均后除以时间（单位为秒）
* `rcount`：计数后除以时间（单位为秒）
* `rlast`：最后一个值除以时间（单位为秒）

## 示例

### 基础配置

基础配置也是 `process_patterns` 的默认配置，不论是否配置了 `process_patterns` ，也不论配置了几个，下面两个“进程模板”都会无条件被 agent 使用（作为最低优先级）。

```
{
  "process_patterns": [
    {
      "search_in": "endpoints:${endpoints},exe:${exe}",
      "regexp": "endpoints:.+\\,exe:.*/redis-server",
      "process_config": {
        "name": "redis",
        "list": "always",
        "protocol": "resp",
        "res_metric": "full",
    },
    {
      "search_in": "endpoints:${endpoints},exe:${exe}",
      "regexp": "endpoints:.+\\,exe:.*/([^\0]*).*",
      "process_config": {
        "name": "$1",
        "list": "always",
        "protocol": "http",
        "res_metric": "full",
    },
    {
      "search_in": "endpoints:${endpoints},exe:${exe}",
      "regexp": "endpoints:\,exe:.*/([^\0]*).*",
      "process_config": {
        "name": "$1",
        "list": "always",
        "protocol": "none",
        "res_metric": "none",
    }
  ]
}
```

### 取 jvm 和 k8s pod name

```
"process_patterns": [
        {
            "search_in": "endpoints:${endpoints},exe:${exe},jvm:${jvm_stat-sun.rt.javaCommand}",
            "regexp": "endpoints:.+\\,exe:.*/(.+)\\,jvm:(\\S+).*",
            "process_config": {
                "name": "$1($2)",
                "res_metric": "full",
                "list": "always",
                "protocol": "http",
                "slog_file_patterns": [
                    {
                        "path_regexp": ".+\\.log.*",
                        "typ": "java_logs"
                    }
                ]
            }
        },
        {
            "regexp": "endpoints:.+\\,exe:.*/([^\\0]*)\\,container:(.+)",
            "search_in": "endpoints:${endpoints},exe:${exe},container:${docker_label-io.kubernetes.container.name}",
            "process_config": {
                "name": "$1($2)",
                "res_metric": "full",
                "list": "always",
                "protocol": "http",
                "slog_file_patterns": [
                    {
                        "path_regexp": ".+\\.log$",
                        "typ": "common_logs"
                    }
                ]
            }
        }
    ]
}
```

### 日志处理

```
"process_patterns": [
        {
            "regexp": ".*nginx.*",
            "search_in": "${exe}",
            "process_config": {
                "name": "nginx-xxx",
                "res_metric": "full",
                "list": "always",
                "protocol": "http",
                "slog_file_patterns": [
                    {
                        "path_regexp": ".*\\.log$",
                        "typ": "common_logs",
                        "groks": [
                            {
                                "name": "test_metric",
                                "match": "%{IP:ip} - - \\[%{HTTPDATE}\\] %{QUOTEDSTRING} %{INT:code} %{INT:dur} %{QUOTEDSTRING} %{QUOTEDSTRING}",
                                "labels": {
                                    "ip": "{{.ip}}",
                                    "code": "{{.code}}"
                                },
                                "value": "1",
                                "aggr":"rsum"
                            },
                            {
                                "name": "test_metric2",
                                "match": "%{IP:ip} - - \\[%{HTTPDATE}\\] %{QUOTEDSTRING} %{INT:code} %{INT:dur} %{QUOTEDSTRING} %{QUOTEDSTRING}",
                                "labels": {
                                    "ip": "{{.ip}}",
                                    "dur": "{{.dur}}"
                                },
                                "value": "{{.code}}",
                                "aggr":"sum"
                            }
                        ]
                    }
                ]
            }
        }
    ]
}
```
