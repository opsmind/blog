## OpsMind 告警回调格式



### 概览

在 OpsMind，用户可自行配置告警的回调地址与回调格式（协议），在有告警发生的时候，OpsMind 会按照用户配置的格式和地址进行调用通知。



### 回调格式（协议）

目前 OpsMind 支持的回调协议为：

* `om.native`

其中，`om.native`为 OpsMind 官方提供的告警信息格式。



#### om.native

HTTP 请求格式：

```
POST <callback_endpoint>
Content-Type: application/json
[Authorization: <BasicAuth>]

<json_body>
```


其中 `json_body` 的具体格式及说明如下：

```
{
   "start" : <unix_timestamp:int64>,         # 该告警产生的时间戳
   "policy" : <policy:struct>,               # 触发该告警的策略 或 触发该告警的微调所属策略
   "adjust" : <adjust:struct>,               # 触发该告警的微调策略(可能为空)
   "active" : <active:boolean>,              # 该告警是否仍未恢复
   "value" : <value:float64>,                # 该告警对象的当前值(仅在 Active 状态下有效)
   "notify_times" : <notify_times:int>,      # 在此次回调之前，该告警已通知过的次数(即 0 表示此次为首次通知)
   "notes" : {                               # 该告警的备注信息（可能为空）
     <note_key:string>: <note_value:string>,
     ...
   },
   "end" : <unix_timestamp:int64>,           # 告警的恢复时间(若仍处于 Active 状态，该字段为0)
   "alert_id" : <alert_id:string>,           # 该告警的唯一标识 ID
   "receiver" : [                            # 该告警需要通知的用户列表（可能为空）
      {
        "member_id" : <member_id:string>,    # 用户 ID
        "phone" : <phone_number:string>,
        "email" : <email_address:string>,
        "team_id" : <team_id:string>         # 团队 ID
      },
      ...
   ],
   "title" : <alert_title:string>,           # 该告警的标题(通常为 policy/adjust 的 name)
   "tags" : {                                # 触发该告警的对象（可能为空），例如 host=host1,service=nginx
     <tag_key:string>: <tag_value:string>,
     ...
   }
}
```


其中 `<policy:struct>` 为触发该告警的策略信息，内容为：

```
{
    "id" : <policy_id:string>,               # 该告警策略的唯一标识 ID
    "name" : <policy_name:string>,           # 该告警策略的名称
    "operator" : <operator:string>,          # 该告警策略的操作符，=/!=/>/>=/</<=
    "level" : <alert_level:string>,          # 该告警策略触发的告警级别
    "desc" : <description:string>,
    "threshold" : <threshold:float64>,       # 该告警策略的触发阈值
    "mtime" : <unix_timestamp:int64>,        # 该告警策略的最后修改时间
    "metric" : <metric_id:string>,           # 该告警策略依赖的 metric 的指标 ID
    "for" : <for:int>,                       # 触发该告警策略的持续时长
    "creator" : <creator:string>,            # 该告警策略的创建者用户 ID
    "ctime" : <unix_timestamp:int64>         # 该告警策略的创建时间
}
```


其中 `<adjust:struct>` 为触发该告警的微调信息，内容为：


```
{
    "id" : <adjust_id:string>,               # 该告警微调的唯一标识 ID
    "name" : <adjust_name:string>,           # 该告警微调的名称
    "desc" : <description:string>,
    "policy_id" : <policy_id:string>,        # 该告警微调所属的策略 ID
    "threshold" : <threshold:float64>,       # 该告警微调的触发阈值
    "creator" : <creator:string>,            # 该告警微调的创建者用户 ID
    "ctime" : <unix_timestamp:int64>,        # 该告警微调的创建时间
    "expired_at" : <expired_at:int64>,       # 该告警微调的过期时间
    "mtime" : <unix_timestamp:int64>,        # 该告警微调的最后修改时间
    "for" : <for:int>,                       # 触发该告警微调的持续时长
    "level" : <alert_level:string>           # 该告警微调触发的告警级别
}
```

