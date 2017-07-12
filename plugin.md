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
        - 例如'["ustg:plugins/node_id1","ustg:plugins/node_id2"]'配置表示，该服务器从category为plugins， name为node_id1和node_id2的用户自定义存储中读取plugin数据
    - desc: 自定义描述
    
- 删除plugin配置：调用ustg delete API 删除用户存储
    - name: 节点ID（node_id）
    - category: 用户存储所属分类

- 删除服务器的plugin关联：调用ustg delete API 删除名为 "plugin_"+节点ID 的服务器存储
    - host: 服务器名
    - name: "plugin_"+节点ID

## 其他细节
- 存储数据均为规定数据结构的数组，并且序列化为对应的json格式字符串
- script_runner数据结构中，duration_second和timeout_second均可以不指定，此时agent会自动检测所要执行文件的文件名，对形如<number>_xxx.sh的文件名进行解析，设置<number>为执行周期或超时时间
- 如果通过其他方式解析不出duration_second和timeout_second，系统默认均为30秒
- agent执行时，超时时间不大于执行周期

## 实际使用案例

**案例一： 配置独立服务器**
- 服务器 host1
- 60秒运行一次该服务器下/a/b/c/d.sh脚本，并设置超时时间20秒
- 30秒运行一次该服务器下/a/b/e文件夹下脚本，并设置超时时间20秒
 
```
POST /v1/localstg/upsert
Content-Type: application/json
{
  "host": "host1",
  "name": "plugin",
  "body": '[{"path": "/a/b/c/d.sh","duration_second": 60,"timeout_second":20}，{"path": "/a/b/e/*","duration_second": 30,"timeout_second":10}]',
  "desc": "any thing can describe"
}

```
这样，host1便配置了plugin，按照配置独立执行脚本。后续可以调用对应的API进行查询、修改和删除

**案例二：配置节点**
- 节点 node1 
- 为该节点配置plugin，60秒运行一次节点下所有服务器的/a/b/c/d.sh脚本，并设置超时时间20秒
 
```
POST /v1/ustg/category/plugin/name/node1/upsert
Content-Type: application/json
{
  private: false,
  meta: "",
  body: '[{"path": "/a/b/c/d.sh","duration_second": 60,"timeout_second":20}]'
}

```

- node1下有三台服务器： host1 host2 host3， 让该节点下的三台机器都执行配置

```
POST /v1/localstg/upsert
Content-Type: application/json
{
  "host": "host1",
  "name": "plugin_node1",
  "body": '["ustg:plugin/node1"],
  "desc": "any thing can describe"
}

POST /v1/localstg/upsert
Content-Type: application/json
{
  "host": "host2",
  "name": "plugin_node1",
  "body": '["ustg:plugin/node1"],
  "desc": "any thing can describe"
}

POST /v1/localstg/upsert
Content-Type: application/json
{
  "host": "host3",
  "name": "plugin_node1",
  "body": '["ustg:plugin/node1"],
  "desc": "any thing can describe"
}

```

**案例三：更改节点配置**
- 前提： 案例二
- 更改配置，增加30秒运行一次节点下服务器的/a/b/e文件夹下的脚本

```
POST /v1/ustg/category/plugin/name/node1/upsert
Content-Type: application/json
{
  private: false,
  meta: "",
  body: '[{"path": "/a/b/c/d.sh","duration_second": 60,"timeout_second":20}，{"path": "/a/b/e/*","duration_second": 30,"timeout_second":20}]'
}

```

- 更改之后，节点node1下三台服务器的plugin配置将自动更新

**案例四：为节点增加机器**
- 前提： 案例二
- 节点增加新机器host4,让host4执行该节点的plugin配置

```
POST /v1/localstg/upsert
Content-Type: application/json
{
  "host": "host4",
  "name": "plugin_node1",
  "body": '["ustg:plugin/node1"],
  "desc": "any thing can describe"
}
```

- 更改之后，节点node1下三台服务器的plugin配置将自动更新

**案例五：删除节点机器**
- 前提： 案例二
- 让host1删除该节点node1的plugin配置

```
POST /v1/localstg/delete
Content-Type: application/json
{
  "host": "host1",
  "name": "plugin_node1"
}
```

<br><br>
