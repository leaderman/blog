# 前言

AnalysisQl DSL是一种 **即席查询** （OLAP）的领域特定语言（DSL），适用于数据分析、数据接口或数据可视化等需要按时间范围、时间粒度查询指标数据的场景，支持以下查询类型：

* 查询主题
    获取数据系统中有哪些 **主题**，可以简单理解为数据业务名称，如：“CDN服务质量分析”。
&nbsp;
* 查询主题维度
    获取指定 主题 中包含有哪些 **维度**，以 主题 CDN服务质量分析 为例，包含维度：国家、省份、城市、运营商、域名、机房、域名、业务名称等；其中，国家 表示可以查询指定某个国家的数据，或者查询各个国家的数据；其它维度类似。
&nbsp;
* 查询主题维度值
    获取指定 主题，指定 维度 中包含有哪些 **维度值**，以 主题 “CDN服务质量分析” 维度 “国家” 为例，包含维度值：中国、美国、日本等。
&nbsp;
* 查询主题指标
    获取指定 主题 中包含有哪些 **指标项**，以 主题 “CDN服务质量分析” 为例，包含指标：带宽、访问量、错误量、下载速度、下载时长、回源率等。
&nbsp;
* 查询主题指标值
    获取指定 主题，指定 指标、时间范围、时间粒度 的 **数值**（多个），可以额外附加若干查询条件（非必须）：维度过滤、维度分组、分组过滤、排序或限制结果行数，类似于 SQL: SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT ...。
    &nbsp;
    以 “CDN服务质量分析” 为例，查询 最近1小时内，以5分钟为时间粒度，中国地区各个省份的带宽值，带宽值需要大于 1000 MB，降序排列且最多返回 1000 行数据。
    &nbsp;
    * 最近1小时，表示 **时间范围**
    * 5分钟，表示 **时间粒度**
    * 中国地区，表示 **维度过滤**，即：WHERE 国家 = '中国'
    * 各个省份，表示 **维度分组**，即：GROUP BY 省份
    * 带宽，表示 **指标**
    * 带宽值大于 1000，表示 **分组过滤**，即：HAVING 带宽值 > 1000
    * 降序，表示 **排序**，即：ORDER BY 带宽值 DESC
    * 最多返回 1000 行数据，表示 **限制结果行数**，即：LIMIT 1000

# 查询语法

AnalysisQl DSL 查询语言使用 **JSON** 表示：

```
{
  "type": "...",
   ...
}
```
&nbsp;
其中，**type** 用于指定查询类型（必须）：
* getTopics
    查询主题。
&nbsp;
* getDimentions
    查询主题维度。
&nbsp;
* getDimentionValues
    查询主题维度值。
&nbsp;
* getMetrics
    查询指标。
&nbsp;
* query
    查询主题指标值。
    
其余键值对需要根据查询类型 type 的不同进行指定，详情见后文。

# 查询结果

AnalysisQl DSL 查询结果使用 **JSON** 表示:

```
{
  "sessionId": "...",
  "code": ...,
  "msg": "...",
  "rows": [{"dimension": "str"}, {"metric": number}, ...]
}
```
&nbsp;
* sessionId
    查询ID，全局唯一，用于标识一次查询。
&nbsp;
* code
    查询响应状态码，类似于HTTP状态码，用于标识查询结果类型（成功或失败）。
&nbsp;
* msg
    查询响应状态描述信息，与 code 相对应；如果查询异常，也可以是具体的异常信息。
&nbsp;
* rows
    查询结果集，使用JSON数组（Array）表示，表示多行数据；每一行数据使用JSON对象（Object）表示，表示多列数据；每一列数据使用一个键值对表示，值只能是字符串类型或数值类型。

**注**：后文描述查询结果时，仅涉及查询结果集 rows。

# getTopics

获取主题（多个），每一个主题包括：topic（名称）、alias（别名）、desc（描述）；其中，topic 为英文名称，alias 为中文名称，或者两者名称相同亦可。

## 语法

```
{
  "type": "getTopics"
}
```

## 示例

**查询**

获取所有主题：

```
{
  "type": "getTopics"
}
```

**结果**

```
[
  {"topic": "CDN服务质量分析", "alias": "CDN服务质量分析", "desc": "..."},
  ...
]
```

# getDimentions

获取指定 主题 的维度（多个），每一个维度包括：name（名称）、alias（别名）、desc（描述）；其中，name 为英文名称，alias 为中文名称，或者两者名称相同亦可。

## 语法

```
{
  "type": "getDimentions",
  "topic": "..."
}
```
&nbsp;
* topic
    主题名称，必须。

## 示例

**查询**

获取主题 CDN服务质量分析 的维度：

```
{
  "type": "getDimentions",
  "topic": "CDN服务质量分析"
}
```

**结果**

```
[
  {"name": "isp", "alias": "运营商", "desc": null},
  ...
]
```

# getDimentionValues

获取指定 主题，指定 维度 的维度值（多个），每一个维度值是一个字符串。

## 语法

```
{
  "type": "getDimentionValues",
  "topic": "...",
  "dimension": "..."
}
```
&nbsp;
* topic
    主题名称，必须。
&nbsp;
* dimension
    维度名称，必须。

## 示例

**查询**

获取主题 CDN服务质量分析，维度 isp（运营商） 的维度值：

```
{
  "type": "getDimensionValues",
  "topic": "CDN服务质量分析",
  "dimension": "isp"
}
```

**结果**

```json
[
  {"value": "移动"},
  {"value": "联通"},
  ...
]
```

# getMetrics

获取指定 主题 的指标（多个），每一个指标包括：name（名称）、alias（别名）、desc（描述）、rule（计算规则）；其中，name 为英文名称，alias 为中文名称，或者两者名称相同亦可。rule 比较特殊，如果指标的计算通过 SQL 实现，则取值为相应的 SQL 语句，其余情况均为 custom，表示指标的计算需要通过 非SQL 实现（通常是代码）。

## 语法

```
{
  "type": "getMetrics",
  "topic": "..."
}
```
&nbsp;
* topic
    主题名称，必须。

## 示例

**查询**

获取主题 CDN服务质量分析 的指标：

```
{
  "type": "getMetrics",
  "topic": "CDN服务质量分析"
}
```

**结果**

```json
[
  {
    "name": "download_time_avg",
    "alias": "下载时间（s）",
    "desc": null,
    "rule": "SELECT SUM(request_time) / SUM(request_num) AS $METRIC FROM ..."
    }, 
    ...
]
```

# query

query 是最为复杂的查询类型，涉及的内容较多，我们先逐个介绍 query 的各个组成部分，然后再介绍 query。

## interval

interval 用于指定查询的时间范围，即：起始时间与终止时间。

### 语法

```
{
  "start": "...",
  "end": "..."
}
```
&nbsp;
* start
    起始时间，闭区间，格式：yyyy-MM-dd HH:mm:ss，必须。
    
* end
    终止时间，闭区间，格式：yyyy-MM-dd HH:mm:ss，必须。

### 示例

指定时间范围：[2020-03-16 00:00:00, 2020-03-16 23:59:59]：

```
{
  "start": "2020-03-16 00:00:00",
  "end": "2020-03-16 23:59:59"
}
```

## granularity

granularity 用于指定查询的时间粒度，比如：每5分钟、每1小时、每1天的数据。

### 语法

```
{
  "data": ...,
  "unit": "..."
}
```

&nbsp;
* data
    数值，整型，必须。
* unit
    单位，枚举型，包括：s（秒）、m（分钟）、h（小时）、d（天）、w（周）、M（月）、y（年），必须。

### 示例

指定时间粒度为5分钟：

```
{
  "data": 5,
  "unit": "m"
}
```

## where

where 通过表达式指定维度过滤条件，支持关系表达式与逻辑表达式。

### 关系表达式

关系表达式分为3类：比较表达式、包含表达式和正则表达式。

#### 比较表达式

比较表达式用于指定维度（值）与特定值的比较关系是否成立，包括：等于、不等于、大于、小于、大于或等于、小于或等于。

##### 语法

```json
{
  "operator": "...",
  "name": "...",
  "value": "..."
}
```

&nbsp;
* operator
    运算符，eq（等于）、ne（不等于）、gt（大于）、lt（小于）、ge（大于或等于）、le（小于或等于），必须。
* name
    维度名称，必须。
* value
    值，字符串，必须。

##### 示例

指定 国家（country） 为 “中国”：

```
{
  "operator": "eq",
  "name": "country",
  "value": "中国"
}
```

指定 国家（country）不为 ”中国“：

```
{
  "operator": "ne",
  "name": "country",
  "value": "中国"
}
```

#### 包含表达式

包含表达式用于指定维度（值）与特定值（多个）的包含关系是否成立。

##### 语法

```
{
  "operator": "in",
  "name": "...",
  "values": [...]
}
```

&nbsp;
* operator
    运算符，值为 in，必须。
* name
    维度名称，必须。
* values
    值（多个），字符串，必须。

##### 示例

指定 国家（country）为 “中国”、“美国”或“日本”：

```
{
  "operator": "in",
  "name": "country",
  "values": ["中国", "美国", "日本"]
}
```

#### 正则表达式

正则表达式用于指定维度（值）与特定正则表达式的匹配关系是否成立。

##### 语法

```
{
  "operator": "regex",
  "name": "...",
  "pattern": "..."
}
```

&nbsp;
* operator
    运算符，值为 regex，必须。
* name
    维度名称，必须。
* pattern
    正则表达式（Java），字符串，必须。

##### 示例

指定 域名（domain）包含字符串 “weibo”：

```
{
  "operator": "regex",
  "name": "domain",
  "pattern": "^.*weibo.*$"
}
```

### 逻辑表达式

逻辑表达式用于指定若干个表达式（关系表达式或逻辑表达式）之间的 与/或/非 关系。

#### 与表达式

与表达式用于指定两个或两个以上的表达式（关系表达式或逻辑表达式）之间的 与 关系。

##### 语法

```
{
  "operator": "and",
  "filters": [...]
}
```

&nbsp;
* operator
    运算符，值为 and，必须。
* filters
    两个或两个以上的表达式（关系表达式或逻辑表达式），必须。

##### 示例

指定 国家（country）为 “中国”、“美国”或“日本”，且 域名（domain）包含有字符串 “weibo”：

```
{
  "operator": "and",
  "filters": [
    {
      "operator": "in",
      "name": "country",
      "values": ["中国", "美国", "日本"]
    },
    {
        "operator": "regex",
        "name": "domain",
        "pattern": "^.*weibo.*$"
    }
  ]
}
```

#### 或表达式

或表达式用于指定两个或两个以上的表达式（关系表达式或逻辑表达式）之间的 或 关系。

##### 语法

```
{
  "operator": "or",
  "filters": [...]
}
```

&nbsp;
* operator
    运算符，值为 or，必须。
* filters
    两个或两个以上的表达式（关系表达式或逻辑表达式），必须。

##### 示例

指定 国家（country）为 “中国”、“美国”或“日本”，或 域名（domain）包含有字符串 “weibo”：

```
{
  "operator": "or",
  "filters": [
    {
      "operator": "in",
      "name": "country",
      "values": ["中国", "美国", "日本"]
    },
    {
        "operator": "regex",
        "name": "domain",
        "pattern": "^.*weibo.*$"
    }
  ]
}
```

#### 非表达式

非表达式用于指定表达式（关系表达式或逻辑表达式）的 非 关系。

##### 语法

```
{
  "operator": "not",
  "filter": {...}
}
```

&nbsp;
* operator
    运算符，值为 not，必须。
* filter
    一个表达式（关系表达式或逻辑表达式），必须。

##### 示例

指定 国家（country） 不为 “中国”、“美国”或“日本”：

```
{
  "operator": "not",
  "filter": {
    "operator": "in",
    "name": "country",
    "values": ["中国", "美国", "日本"]
  }
}
```

## groups

groups 用于指定维度分组条件，支持多个维度。

### 语法

```
["...", ...]
```

&nbsp;
维度名称数组。

### 示例

指定使用 国家（country）、域名（domain）进行分组统计：

```json
["country", "domain"]
```

## having

having 通过表达式的指定分组过滤条件。

### 语法

语法同 **where**；其中，关系表达式中的 **name** 为维度名称时，**value** 为字符串类型；**name** 为指标名称时，**value** 为数值类型。

### 示例

指定 下载速度（download_speed_avg）大于或等于 1000

```
{
  "operator": "gt",
  "name": "download_speed_avg",
  "value": 1000
}
```

## orders

orders 指定排序条件，支持多个。

### 语法

```json
[
  {
    "name": "...",
    "sort": "..."
  },
  ...
]
```

&nbsp;
* name
    维度名称、指标名称或timeBucket（保留关键字，表示时间粒度），必须。
* sort
    升序（asc）或降序（desc），必须。

### 示例

指定 下载速度（download_speed_avg）按降序排列

```json
[
  {
    "name": "download_speed_avg",
    "sort": "desc"
  }
]
```

## limit

limit 指定查询结果返回的最大行数。

### 语法

整数，必须。

### 示例

指定查询结果返回的最大行数为1000

```
1000
```

## query

获取指定 主题，指定 指标、时间范围、时间粒度 的 数值（多个），可以额外附加若干查询条件。

### 语法

```
{
  "type": "query",
  "topic": "...",
  "interval": {...},
  "granularity": {...},
  "metric": "...",
  "where": {...},
  "groups": [...],
  "having": {...},
  "orders": [...],
  "limit": ...
}
```

&nbsp;
* topic
    主题名称，必须。
* interval
    时间范围，参考 **interval**，必须。
* granularity
    时间粒度，参考 **granularity** 必须。
* metric
    指标名称，必须。
* where
    维度过滤条件，参考 **where** 非必须。
* groups
    维度分组条件，参考 **groups**，非必须。
* having
    分组过滤条件，参考 **having** 非必须。
* orders
    排序条件，参考 **orders** 非必须。
* limit
    限制结果最大行数，参考 **limit** 非必须。

### 示例

**查询**

假设查询条件如下：

主题：CDN服务质量分析；
时间范围：["2020-03-16 00:00:00", "2020-03-16 23:59:59"];
时间粒度：1小时；
指标：下载速度（download_speed_avg）；
维度过滤：域名（domain）包含有字符串“weibo”；
维度分组：运营商（isp）；
分组过滤：下载速度（download_speed_avg）大于2000 Kb/s；
排序：下载速度降序（download_speed_avg）；
结果集：最多100行；

```json
{
  "type": "query",
  "topic": "CDN服务质量分析",
  "interval": {
    "start": "2020-03-16 00:00:00",
    "end": "2020-03-16 23:59:59"
  },
  "granularity": {
    "data": 1,
    "unit": "h"
  },
  "metric": "download_speed_avg",
  "where": {
    "operator": "regex",
    "name": "domain",
    "pattern": "^.*weibo.*$"
  },
  "groups": [
    "isp"
  ], 
  "having": {
    "operator": "gt",
    "name": "download_speed_avg",
    "value": 2000
  },
  "orders": [
    {
      "name": "download_speed_avg",
      "sort": "desc"
    }
  ], 
  "limit": 100
}
```

**结果**

```json
[
  {
    "timeBucket": "2020-03-16 00:00:00",
    "download_speed_avg": 2000.801698511234,
    "isp": "联通"
  }, 
  {
    "timeBucket": "2020-03-16 00:00:00",
    "download_speed_avg": 3000.871752209983,
    "isp": "长城宽带"
  }, 
  {
    "timeBucket": "2020-03-16 00:00:00",
    "download_speed_avg": 4000.016188287193,
    "isp": "移动"
  },
  ...
    {
    "timeBucket": "2020-03-16 01:00:00",
    "download_speed_avg": 5000.801698511234,
    "isp": "联通"
  }, 
  {
    "timeBucket": "2020-03-16 01:00:00",
    "download_speed_avg": 6000.871752209983,
    "isp": "长城宽带"
  }, 
  {
    "timeBucket": "2020-03-16 01:00:00",
    "download_speed_avg": 7000.016188287193,
    "isp": "移动"
  },
  ...
]
```

# 结语

AnalysisQl DSL 最大的价值是提供了一种统一的查询语言，实现数据应用服务（数据可视化、报表、数据接口）和查询引擎、指标计算规则的解耦，降低数据的开发和维护成本。本文描述的内容并不能完全覆盖即席查询的全部应用场景，仅供大家借鉴参考。

&nbsp;
&nbsp;
&nbsp;
![](https://yurun-blog.oss-cn-beijing.aliyuncs.com/%E5%85%AC%E4%BC%97%E5%8F%B7/card.png)

