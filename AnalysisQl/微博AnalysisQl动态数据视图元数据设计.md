![](https://yurun-blog.oss-cn-beijing.aliyuncs.com/%E5%85%AC%E4%BC%97%E5%8F%B7/%E5%BE%AE%E5%8D%9A%E6%B5%81%E5%AA%92%E4%BD%93%E6%95%B0%E6%8D%AE%E5%B9%B3%E5%8F%B0.png)

# 前言

目前，AnalysisQl 数据视图的元数据（维度、指标、指标计算器）需要通过代码（API）或资源文件的形式硬编码，应用启动时，按照声明的顺序依次注册。这种模式下，数据视图是 **静态** 的，任何一项变更都需要重新升级发布应用服务，不利于服务快速迭代。

考虑到这种情况，AnalysisQl 在保留原有 **静态** 视图的前提下，扩展出 **动态** 视图方案，基于数据库实现元数据的存储，通过更新相应的数据库记录，即可 **实时动态** 地更新数据视图。

&nbsp;
**AnalysisQl 项目主页**：https://github.com/weibodip/analysisql。

# 元数据

数据视图元数据有 7 类：

* 数据视图信息
* 数据视图维度
* 数据视图维度值
* 数据视图指标
* 数据表信息
* 数据表维度
* 数据表指标计算器

每一类元数据对应着数据库的一张数据表，我们分别介绍。

**注**：数据库以 **MySQL** 为例进行介绍。

## 数据视图信息

数据视图信息表（aql_view_info），用于存储多个数据视图（即：主题）信息，包括：名称、别名、描述及状态信息。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|avi_topic|VARCHAR|视图名称，通常使用英文表示。|
|avi_alias|VARCHAR|视图别名，通常使用中文表示，与 avi_topic（英文名称）相对应。|
|avi_desc|VARCHAR|视图描述，通常用于说明主题附加信息。|
|avi_state|INT|视图状态，数值 **0** 表示视图处于 **禁用** 状态；数值 **1** 表示视图处于 **启用** 状态，其余数值无效。|

**注**：视图名称（avi_topic）全局唯一。

## 数据视图维度

数据视图维度表（aql_view_dimension），用于存储数据视图支持的维度（多个）信息，包括：名称、别名、描述。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|avd_topic|VARCHAR|视图名称。|
|avd_name|VARCHAR|视图维度名称，通常使用英文表示。|
|avd_alias|VARCHAR|视图维度别名，通常使用中文表示，与 avd_name（英文名称）相对应。|
|avd_desc|VARCHAR|视图维度描述，通常用于说明维度附加信息。|

**注**：视图名称（avd_topic）与视图维度名称（avd_name）组合全局唯一。

## 数据视图维度值

数据视图维度值表（aql_view_dimension_value），用于存储数据视图中各个维度对应的维度值（多个）。每一个维度的维度值支持多个版本（版本使用时间戳表示），通常使用最新版本（即时间戳最大）的维度值数据。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|topic|VARCHAR|视图名称。|
|dimension|VARCHAR|视图维度名称。|
|dtime|TIMESTAMP|视图维度值时间戳，默认当前时间（CURRENT_TIMESTAMP）。|
|dvalue|VARCHAR|视图维度值。|

**写入维度值**

数据视图维度值表中的维度值数据需要借助外部应用程序写入，可以使用定时或不定时的方式。注意，写入指定视图维度的维度值时，这些维度值的时间戳（dtime）需要保持一致，表示这些维度值归属于同一个版本。

**读取维度值**

读取指定视图维度的维度值时，需要经过以下2个步骤：

1. 根据视图名称、视图维度名称计算视图维度值的最大时间戳（最新版本）；

```
SELECT MAX(dtime) FROM %s WHERE topic = '%s' AND dimension = '%s'
```
2. 根据视图名称、视图维度名称及视图维度值的最大时间戳检索维度值；

```
SELECT DISTINCT(dvalue) FROM %s WHERE topic = '%s' AND dimension = '%s' AND dtime = '%s'
```

**注**：视图名称（topic）、视图维度名称（dimension）与视图维度值时间戳（dtime）组成联合索引。

## 数据视图指标

数据视图指标表（aql_view_metric），用于存储数据视图支持的指标（多个）信息，包括：名称、别名、描述。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|avm_topic|VARCHAR|视图名称。|
|avm_name|VARCHAR|视图指标名称，通常使用英文表示。|
|avm_alias|VARCHAR|视图指标别名，通常使用中文表示，与 avm_name（英文名称）相对应。|
|avm_desc|VARCHAR|视图指标描述，通常用于说明指标附加信息。|

**注**：视图名称（avm_topic）与视图指标名称（avm_name）组合全局唯一。

## 数据表信息

数据表信息表（aql_view_table_info），用于存储数据视图支持的数据表（多个）信息，包括：名称、时间粒度大小、时间粒度单位、保存周期、时间延迟及状态信息。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|avti_topic|VARCHAR|视图名称。|
|avti_name|VARCHAR|数据表名称，通常使用英文表示。|
|avti_data|INT|数据表数据时间粒度大小。|
|avti_unit|VARCHAR|数据表数据时间粒度单位，支持“s”（秒）、“m”（分钟）、“h”（小时）、“d”（天）、“w”（周）、“M”（月）、“q”（季度）、“y”（年）。|
|avti_period|INT|数据表数据保存周期，使用数据表时间粒度为计算单位，详情见后。|
|avti_delay|INT|数据表数据时间延迟，使用数据表时间粒度为计算单位，详情见后。|
|avti_state|INT|数据表状态，数值 0 表示数据表处于 禁用 状态；数值 1 表示数据表处于 启用 状态，其余数值无效。|

假设，时间粒度大小（avti_data）为 5，时间粒度单位（avti_unit）为 m，表示数据表数据时间粒度为 5分钟。
假设，时间粒度为 5分钟，保存周期（avti_period）为 288，表示数据表数据保存周期为 1天。
假设，时间粒度为 5分钟，时间延迟（avti_delay）为 12，表示数据表数据时间延迟为 1小时；

**注**：视图名称（avti_topic）、数据表名称（avti_name）全局唯一；


## 数据表维度

数据表维度表（aql_view_table_dimension），用于存储数据视图/数据表支持的维度（多个）。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|avtd_topic|VARCHAR|视图名称。|
|avtd_table|VARCHAR|数据表名称。|
|avtd_name|VARCHAR|数据表维度名称。|

**注**：视图名称（avtd_topic）、数据表名称（avtd_table）及数据表维度名称（avtd_name）组合全局唯一。

## 数据表指标计算器

数据表指标计算器表（aql_view_table_calculator），用于存储数据视图/数据表支持的指标计算信息，包括：名称、指标计算查询引擎（类型、链接、用户名、密码）、指标计算规则（SQL查询语句）。

|字段名称|字段类型|字段含义|
|:-:|:-:|:-|
|id|INT|主键ID|
|avtc_topic|VARCHAR|视图名称。|
|avtc_table|VARCHAR|数据表名称。|
|avtc_metric|VARCHAR|数据表指标名称。|
|avtc_type|VARCHAR|数据表指标计算时使用的查询引擎类型，支持“clickhouse”、“mysql”、“presto”。|
|avtc_url|VARCHAR|数据表指标计算时连接查询引擎的JDBC URL。|
|avtc_user|VARCHAR|数据表指标计算时连接查询引擎的用户名。|
|avtc_passwd|VARCHAR|数据表指标计算时连接查询引擎的密码。|
|avtc_sql|VARCHAR|数据表指标计算时使用的SQL查询语句，不同的查询引擎需要使用不同的SQL查询语法。|

**注**：视图名称（avtc_topic）、数据表名称（avtc_table）及数据表指标名称（avtc_metric）组合全局唯一。

# 结语

AnalysisQl 会定时扫描加载数据视图表中所有处于 **启用** 状态的视图，然后使用视图名称从其它几类元数据表中扫描加载视图相应的维度、指标、数据表等信息。如果需要更新数据视图，只需要增加或更新相应的元数据记录，下一次扫描完成之后即可生效。

