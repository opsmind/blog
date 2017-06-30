# Plugin 配置说明(Script Runner)

## 概念
- user storage(ustg 或用户自定义存储): 存储组织机构或用户（参数private为true时）的自定义数据，详见ustg API
- local storage(localstg 或服务器存储): 存储某一台服务器的自定义数据并下发到对应的agent，详见localstg API

## 数据结构
#### script_runner
```
{
    "path": <string>, 
    "duration_second": <int64>,
    "timeout_second":<int64>
}
```
参数说明：
- path: Linux glob 路径模式匹配 例如"/tmp/dogtest/metric/*"
- duration_second: 执行周期，单位秒
- timeout_second: 超市时间，单位秒

#### ustg_marker
```
"ustg:ustg_category/ustg_name"
```
参数说明：
- ustg_category: 用户存储所属分类
- ustg_name: 用户存储唯一标识名称

## 单独服务器的Plugin配置
- 新建或更新：调用localstg upsert API 为服务器设置本地存储：
    - host: 服务器名
    - name: "plugin"
    - body: 对script_runner数组进行json序列化后的字符串
        - 例如'[{"path": "/tmp/dogtest/metric/*","duration_second": 30,"timeout_second":10}]'
    - desc: 自定义描述
    
- 删除：调用localstg delete API 删除名为"plugin"的本地存储
    - host: 服务器名
    - name: "plugin"
    
## 服务器节点的Plugin配置（并关联服务器）
- 新建或更新plugin配置：调用ustg upsert API 配置用户存储：
    - name: 节点ID（node_id）
    - category: 自定义分类
    - private: false，若为true，则该配置只有当前用户可以操作
    - meta: 用户元数据，可为空
    - body: 对script_runner的数组进行序列化后的数据，
        - 例如'[{"path": "/tmp/dogtest/metric/*","duration_second": 30,"timeout_second":10}]'
    
- 新建或更新服务器关联：调用localstg upsert API 配置服务器存储：
    - host: 服务器名
    - name: "plugin_"+节点ID(方便用户将服务器从某个节点删除时更新配置)
    - body: 对ustg_marker的数组进行序列化后的数据
        - 例如'["ustg:plugins/node_id1","ustg:plugins/node_id1"]'配置表示，该服务器从category为plugins， name为node_id1和node_id2的用户自定义存储中读取plugin数据
    - desc: 自定义描述
    
- 删除plugin配置：调用ustg delete API 删除用户存储
    - name: 节点ID（node_id）
    - category: 用户存储所属分类

- 删除服务器的plugin关联：调用ustg delete API 删除名为 "plugin_"+节点ID 的服务器存储
    - host: 服务器名
    - name: "plugin_"+节点ID

## 其他细节
- 存储数据均为规定数据结构的数组，并且序列化为对应的json格式字符串

<br><br>
