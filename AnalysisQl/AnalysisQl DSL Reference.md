# AnalysisQl

AnalysisQl是一种 **即席查询** （OLAP）的领域描述语言（DSL），应用于数据仓库、数据统计、数据分析或数据接口等业务场景，提供以下能力：

* 查询主题
    查询数据系统中有哪些分析 **主题**。
* 查询主题维度
    查询指定主题中包含有哪些 **维度项**。
* 查询主题维度值
    查询指定主题、指定维度中包含有哪些 **维度值**。
* 查询主题指标
    查询指定主题中包含有哪些 **指标项**。
* 查询主题指标值
    按主题名称、指标名称、时间范围、时间粒度、维度分组/过滤、排序等条件查询 **指标数值**，类似于SQL: "SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT ..."。

以“CDN服务质量分析”为例，

主题：CDN服务质量分析；
维度：国家、省份、城市、运营商、域名、机房、域名、业务名称等；
指标：带宽、访问量、错误量、下载速度、下载时长、回源率等；

查询指标值示例：最近1小时内，以5分钟为粒度，中国地区的每个域名的平均下载速度是多少？

## 查询语法

AnalysisQl查询语言使用JSON对象表示，

```json
{
  "type": "...",
  ...
}
```
其中，**type** 默认必须存在，其余键值对根据“type”的不同取值进行指定。

“type”取值范围：

* getTopics
    表示查询主题；
* getDimentions
    表示查询指定主题的维度项；
* getDimentionValues
    表示查询指定主题、指定维度的维度值；
* getMetrics
    表示查询指定主题的指标项；
* query
    表示以特定的查询条件，查询指定主题、指定指标的数值；


## 查询结果

AnalysisQl查询结果使用JSON对象表示，

```json
{
  "sessionId": "...",
  "code": ...,
  "msg": "...",
  "rows": [{"dimension": "str"}, {"metric": number}, ...]
}
```

sessionId: 查询ID，唯一标识一次查询；
code: 查询响应状态码，类似于HTTP状态码；
msg: 查询响应状态描述信息，与"code"相对应；如果查询执行异常，也可以是具体的异常信息；
rows: 查询结果集；使用JSON数组表示，表示多行数据；每一行数据使用JSON对象表示，表示多列数据；每一列数据使用一个键值对表示，值只能是字符串或数值类型；

**注**：后文描述查询结果时，仅涉及“rows”。

# getTopics

查询主题（多个），每一个主题包括：topic（名称）、alias（别名）、desc（描述）。

1. 语法

    ```json
    {
      "type": "getTopics"
    }
    ```

2. 示例

    查询系统中所有主题：
    ```json
    {
      "type": "getTopics"
    }
    ```

    结果：
    ```json
    [
        {
            "topic": "CDN服务质量分析",
            "alias": "CDN服务质量分析",
            "desc": "CDN服务质量，包括：Edge、阿里云、白山云、腾讯云、华为云、网宿等，分析指标：带宽、访问量、错误量、下载速度、下载时长。"
        },
        ...
    ]
    ```

# getDimentions（查询主题维度）

查询指定主题的维度项（多个），每一个维度项包括：name（名称）、alias（别名）、desc（描述）。

1. 语法

    ```json
    {
      "type": "getDimentions",
      "topic": "..."
    }
    ```

    topic：主题名称（必须）；

2. 示例

    查询主题“CDN服务质量分析”的维度项：
    ```json
    {
      "type": "getDimentions",
      "topic": "CDN服务质量分析"
    }
    ```

    结果：
    ```json
    [
        {
            "alias": "运营商",
            "name": "isp",
            "desc": null
        },
        ...
    ]
    ```

# getDimentionValues（查询主题维度值）

查询指定主题、指定维度的维度值（多个），每一个维度值包括：value（值）。

1. 语法

    ```json
    {
      "type": "getDimentionValues",
      "topic": "...",
      "dimension": "...",
      "where": {...}
    }
    ```

    topic：主题名称（必须）；
    dimension：维度名称（必须）；
    where：维度过滤条件，用于维度级联查询，暂不支持（非必须）；

2. 示例

    查询主题“CDN服务质量分析”、维度“isp”的维度值：
    ```json
    {
        "type": "getDimensionValues",
        "topic": "CDN服务质量分析",
        "dimension": "isp"
    }
    ```

    结果：
    ```json
    [
        {
            "value": "移动"
        },
        ....
    ]
    ```

# getMetrics（查询主题指标）

查询指定主题的指标项（多个），每一个指标项包括：name（名称）、alias（别名）、desc（描述）、rule（计算规则）。其中，rule默认值为“custom”；如果指标计算基于SQL实现，rule则为相应的SQL语句。

1. 语法

    ```json
    {
      "type": "getMetrics",
      "topic": "..."
    }
    ```

    topic：主题名称（必须）；

2. 示例

    查询主题“CDN服务质量分析”的指标项：
    ```json
    {
      "type": "getMetrics",
      "topic": "CDN服务质量分析"
    }
    ```

    结果：
    ```json
    [
        {
            "alias": "下载时间（s）",
            "name": "download_time_avg",
            "rule": "SELECT SUM(request_time) / SUM(request_num) AS $METRIC FROM cdn-all_cdn_staging WHERE $WHERE AND regexp_like(httpcode, \"^[^45].*$\") GROUP BY $GROUPS HAVING $HAVING AND download_time_avg != Infinity AND download_time_avg != NaN AND download_time_avg >= 0.0 ORDER BY $ORDERS LIMIT $LIMIT",
            "desc": null
        }, 
        ...
    ]
    ```

# query

查询指标值。

1. 语法

    ```json
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

    1.1 topic

    主题名称（必须）。

    1.2 interval

    时间范围（必须），语法：

    ```json
    {
        "start": "...",
        "end": "..."
    }
    ```
    start/end均为闭区间，格式：yyyy-MM-dd HH:mm:ss。

    例如，时间范围：[2020-03-16 00:00:00, 2020-03-16 23:59:59]，如下：

    ```json
    {
        "start": "2020-03-16 00:00:00",
        "end": "2020-03-16 23:59:59"
    }
    ```

    1.3 granularity

    时间粒度（必须），语法：

    ```json
    {
        "data": ...,
        "unit": "..."
    }
    ```

    data： 数值（整型）；
    unit: 单位，s(秒)、m（分钟）、h（小时）、d（天）、w（周）、M（月）、y（年）;

    例如，时间粒度：5分钟，如下：

    ```json
    {
        "data": 5,
        "unit": "m"
    }
    ```

    1.4 metric

    指标名称（必须）。

    1.5 where

    维度过滤（非必须）， 支持比较表达式与逻辑表达式。

    1.5.1 比较表达式

    a. eq/ne/gt/lt/ge/le

    ```json
    {
        "operator": "...",
        "name": "...",
        "value": "..."
    }
    ```

    operator：eq（等于）、ne（不等于）、gt（大于）、lt（小于）、ge（大于或等于）、le（小于或等于）；
    name：维度名称；
    value：维度值；

    例如，国家（country）为“中国”，如下：

    ```json
    {
        "operator": "eq",
        "name": "country",
        "value": "中国"
    }
    ```
    b. in

    ```json
    {
        "operator": "in",
        "name": "...",
        "values": [...]
    }
    ```

    operator：in（包含）；
    name：维度名称；
    values：维度值列表，使用JSON数据表示；

    例如，国家（country）为“中国”、“美国”或“日本”，如下：

    ```json
    {
        "operator": "in",
        "name": "country",
        "values": ["中国", "美国", "日本"]
    }
    ```
    c. regex

    ```json
    {
        "operator": "regex",
        "name": "...",
        "pattern": "..."
    }
    ```

    operator：regex（正则匹配）；
    name：维度名称；
    pattern：正则表达式；

    例如，域名（domain）包含有字符串“weibo”，如下：

    ```json
    {
        "operator": "regex",
        "name": "domain",
        "pattern": "^.*weibo.*$"
    }
    ```

    1.5.2 逻辑表达式

    a. and

    ```json
    {
        "operator": "and",
        "filters": [...]
    }
    ```

    operator：and（与）；
    filters：两个或两个以上的比较表达式或逻辑表达式；比较表达式和逻辑表达式可以混合使用；逻辑表达式可以嵌套使用；

    例如：国家（country）为“中国”、“美国”或“日本”，**且** 域名（domain）包含有字符串“weibo”，如下：

    ```json
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

    b. or

    ```json
    {
        "operator": "or",
        "filters": [...]
    }
    ```

    operator：or（或）；
    filters：两个或两个以上的比较表达式或逻辑表达式；比较表达式和逻辑表达式可以混合使用；逻辑表达式可以嵌套使用；

    例如：国家（country）为“中国”、“美国”或“日本”，**或** 域名（domain）包含有字符串“weibo”，如下：

    ```json
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

    c. not

    ```json
    {
        "operator": "not",
        "filter": {...}
    }
    ```

    operator：not（非）；
    filter：一个比较表达式或逻辑表达式；逻辑表达式可以嵌套使用；

    例如：国家（country） **不** 为“中国”、“美国”或“日本”，如下：

    ```json
    {
        "operator": "not",
        "filter": {
            "operator": "in",
            "name": "country",
            "values": ["中国", "美国", "日本"]
        }
    }
    ```
    &nbsp;
    **注** ：维度过滤表达式中的“name”只能是维度。

    1.6 groups

    维度分组（非必须），可以指定多个维度值，语法：

    ```json
    [
        "...",
        ...
    ]
    ```

    例如：使用国家（country）、域名（domain）进行分组统计，如下：

    ```json
    [
        "country",
        "domain"    
    ]
    ```

    1.7 having

    分组过滤（非必须），语法同where。

    &nbsp;
    **注** ：分组过滤表达式中的“name”只能是分组（groups）中的维度名称，或是指标名称（metric）。其中，过滤指标（值）时，过滤表达式中的“value”须是数值类型。

    例如：下载速度（download_speed_avg）大于或等于1000 Kb/s，如下：

    ```json
    {
        "operator": "gt",
        "name": "download_speed_avg",
        "value": 1000
    }
    ```

    1.8 orders

    排序（非必须），可以指定多个排序条件，语法：

    ```json
    [
        {
            "name": "...",
            "sort": "..."
        },
        ...
    ]
    ```

    name：“timeBucket”，或是分组（groups）中的维度名称，或是指标名称（metric）；
    sort：asc（升序）或desc（降序）；

    例如：下载速度（download_speed_avg）降序排列，如下：

    ```json
    [
        {
            "name": "download_speed_avg",
            "sort": "desc"
        }
    ]
    ```

    **注** ：“timeBucket” 为指定时间范围（interval）根据时间粒度（granularity）分桶之后的列名称。

    1.9 limit

    结果集行数（非必须）。

2. 示例

    查询条件，如下：

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

    结果：

    ```json
    [
        {
            "timeBucket": "2020-03-16 00:00:00",
            "download_speed_avg": 12219.801698511234,
            "isp": "联通"
        }, 
        {
            "timeBucket": "2020-03-16 00:00:00",
            "download_speed_avg": 12099.871752209983,
            "isp": "长城宽带"
        }, 
        {
            "timeBucket": "2020-03-16 00:00:00",
            "download_speed_avg": 10846.016188287193,
            "isp": "移动"
        },
        ...
    ]
    ```
    
    &nbsp;
    **注** ：这里的下载速度（download_speed_avg）为每1小时的平均值，也可以是最大值、最小值或累加值，取决于指标的具体实现。

