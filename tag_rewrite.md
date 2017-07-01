## Tag Rewrite 功能

### 概览

在 Opsmind 系统内，在配置 **指标查询**、**告警规则生成**、**告警订阅匹配** 的 tag 匹配规则时可使用 Tag Rewrite 功能简化配置的编写和维护复杂度。Tag Rewrite 通过一个以关键符 `$` 开头的魔法标签实现对 tag 匹配规则的重写。

这里以 `$hosts_by_tag` 为例：

若当前共有三台机器：**host-1**, **host-2**, **host-3**

每台机器都标记了一些各自的标签（详见机器标签）：

* host-1: `zone=asia`  `service=nginx,lvs`
* host-2: `zone=asia` `service=nginx,mongodb`
* host-3: `zone=europe`

若希望单独调整拥有标签 `service=nginx` 的机器 cpu_usage 告警阈值，一种做法是配置：

```
{
  "key":"host",
  "val": ["host-1", "host-2"]
}
```

但这种做法在 `service=nginx` 的机器发生变化时，需要重新调整配置中的机器列表。

如果使用魔法标签 `$hosts_by_tag`，那么只需配置：

```
{
  "key":"host",
  "val": ["$hosts_by_tag", "service=nginx"]
}
```

这样，当 `service=nginx` 标签所附属的机器列表发生变化时候，则无需重新配置。



### 原理

Tag Rewrite 功能发生在 **指标查询**、**告警规则生成**、**告警订阅匹配** 的运行时，系统根据魔法标签的内容自动替换、修改或删除该标签最终的值列表。



### 使用位置

目前支持 Tag Rewrite 的地方为：

- 对监控项做查询时指定的匹配 tag
- 告警策略、告警微调中的 tag 配置
- 团队对告警的订阅 tag



### 已支持的魔法标签

##### $hosts_by_tag

_功能_：

通过给定机器标签（host-tag）的查询条件自动填写 host 列表。

_规格_：

```
["$hosts_by_tag", "arg1", "arg2", ...]
```

_说明_：

该魔法标签接受不定长的多个参数，参数的格式为：

```
tag1=val1,val2,...
```

每个参数可匹配一个 tag 的值空间。值空间由一个或多个字符串组成（以 `,` 分隔），其含义为：tag1=val1 或 tag1=val2 或 …，即多个值之间是 `or` 的关系。

而多个参数之间是 `and` 关系，例如：

```
["$hosts_by_tag", "tag1=val1,val2", "tag2=val3,val4"]
```

表示自动替换为：包含 `tag1=val1` 或 `tag1=val2`，且，包含 `tag2=val3` 或 `tag2=val4` 的机器列表。



### 实践案例

**环境**：当前共有三台机器：**host-1**, **host-2**, **host-3**

**目标**：此案例用于实现针对其中某一组机器配置针对指标项 cpu_usage 的告警（指标项请参考指标项管理 API）。且通过 `$hosts_by_tag` 实现自动计算告警对象。

**步骤一：创建告警策略**

该步骤是为了创建一个泛化的、非具体指向的告警模板。以针对 cpu_usage 为例，API 请求为：

```
POST /v1/policies/create
Content-Type: application/json

{
  "name": "cpu_usage_too_high",
  "metric": "cpu_usage",
  "tags": [],
  "operator": ">=",
  "for": 60,
  "threshold": 0.8,
  "desc": "policy for cpu_usage alert",
  "level": "critical",
  "slience": true
}
```



请求返回：

```
200 OK
{
  "policy_id": "7ft8ij9knijkl"
}
```

这样就拥有了一个 ID 为 `7ft8ij9knijkl` 的告警策略（模板）。其中请求中的 `"slience": true` 表示该告警策略默认不产生任何告警，只有后续基于该策略创建的更具针对性的微调策略才会产生告警。



**步骤二：通过机器标签将机器分组**

通过 API 或者 Opsmind Portal 给一批机器打上某个分组标签。

比如将 host-1，host-2 这两台打上标签：`node_id=1ax2pp5ur2rxj`

API 请求为：

```
POST /v1/hosts/tags/update
Content-Type: application/json

{
  "tag": {
    "host-1":{
      "node_id": ["1ax2pp5ur2rxj"]
    },
    "host-2":{
      "node_id": ["1ax2pp5ur2rxj"]
    }
  }
}
```

这样，host-1 和 host-2 均被打上了 "node_id=1ax2pp5ur2rxj" 这样一个标签，供后续查询使用。



**步骤三：创建告警微调**

由于配置了 `"slience": true` ，步骤一中创建的告警策略只是一个静默（不产生告警）模板，若需要针对  `node_id=1ax2pp5ur2rxj` 这一组内的机器开启告警，那么可以创建一个只作用在这一组机器上的微调：

API 请求为：

```
POST /v1/policies/7ft8ij9knijkl/adjusts/create
Content-Type: application/json

{
  "name": "node_1ax2pp5ur2rxj_cpu_usage",
  "desc": "alert adjust for hosts in node 1ax2pp5ur2rxj",
  "tags": [
    {
      "key": "host",
      "val": ["$hosts_by_tag", "node_id=1ax2pp5ur2rxj"]
    }
  ],
  "threshold": 0.8,
  "alert_tags": {
    "node_id": "1ax2pp5ur2rxj",
  }
}
```

至此，分组 `node_id=1ax2pp5ur2rxj` 中的机器将会自动按 cpu_usage 的阈值为 80% 为限进行告警了。

