# 前言

适用于 **即席查询** 场景的开源查询引擎有很多，如：Elasticsearch、Druid、Presto、ClickHouse等；每种系统各有利弊，有的擅长检索，有的擅长统计；实践证明，**All In One** 是行不通的，最好的方式是选取若干个（考虑运维成本，建议 1 ~ 3 个），每个都对应着自身最具优势的场景。

大多数的技术分享会从系统架构、功能扩展或性能优化角度进行讨论，本文不涉及这些内容。本文以 **指标多维统计查询** 为例，讨论多个查询引擎混合应用场景下的问题思考及相应的解决方案。

# 指标多维统计查询

## 定义

什么是 **指标多维统计查询** ？按指定的条件查询指标值（一个或多个），条件包括：

* 时间范围（必选）
* 时间粒度（必选）
* 维度过滤条件（可选，类似SQL中的 **Where**）
* 维度分组条件（可选，类似SQL中的 **GroupBy**）
* 维度或指标排序条件（可选，类似SQL中的 **OrderBy**）

注意，**指标多维统计查询** 涉及4个关键元素：**时间范围**、**时间粒度**、**维度**及 **指标**（详情见后）。

## 示例

* 查询微博视频最近7天范围内，每天的PSR1（秒开率）是多少？
* 查询微博视频最近1个月内，每天的移动4G场景下各个运营商的卡顿率是多少？

# 查询引擎混合应用

多个查询引擎混合应用的场景下，会遇到哪些问题呢？我们主要从 **工程角度** 与 **业务角度** 来看下。

## 工程角度

每个查询引擎都有自己特有的API（也包含 **SQL**，虽然开源社区大多致力于提供标准的SQL支持，但实际落地情况并不尽如人意），指标查询需要 **直接连接** 查询引擎并使用相应的API进行指标值计算，开发人员和分析人员需要在多个不同的API之间来回切换，学习及维护成本较高；查询引擎版本升级或引入新的引擎会增加这些成本，系统整体的灵活性及可扩展性较差。

## 业务角度

多个查询引擎混合使用的场景下，业务指标会分散至不同的引擎中，会遇到以下问题（复杂度依次增加）：

* 某个业务指标的查询需要使用相应的查询引擎；
* 某个业务指标的查询需要使用相应的查询引擎，且计算规则较为复杂；
* 某个业务指标需要依赖多个查询引擎的联合查询；
* 某个业务指标需要依赖多个查询引擎的联合查询，且计算规则较为复杂；
    
可以看出，我们需要维护指标与查询引擎的 **对应关系**，以及指标的 **计算规则**，实际中很大程度上依赖于 **文档** （Wiki）。数据应用服务中常见的数据可视化、报表、数据接口，这些服务的实现模块均需要直接耦合这些对应关系及计算规则，而且需要重复多次。如果某个指标的对应关系或计算规则发生变化，相应的文档及相关的服务 **必须** 全部更新，才能保证数据一致性。

指标众多的场景下，上述提及的成本及复杂度会大幅增加，有过类似服务经验的同学应该深有体会。

**补充**

对应关系 与 计算规则 可能比较抽象，举一个具体的例子。假设 MySQL 数据库某数据表 DEFAULT.TEST_TABLE 的中有 2 列：A 和 B，指标 M 的值可以通过SQL语句查询得出，如：

```
SELECT
    A / B AS M
FROM
    DEFAULT.TEST_TABLE
```

数据应用服务（数据可视化、报表或数据接口）查询这个指标数据时，需要知道去什么地方查询，以及具体如何查询；上述示例需要通过远程方式连接MySQL，连接时使用到的协议类型（MySQL）、IP地址、端口号等信息就是 **对应关系**；查询时需要使用的SQL语句就是 **计算规则**。

# 指标库

针对指标众多，指标计算规则及相关信息不易维护的问题，我们提出 **指标库** 用于规范化指标管理。什么是 **指标库**？

对于某一个指标而言，可以从不同的时间范围、时间粒度及维度视角进行统计查询。换句话说，对于某一个指标而言，我们需要知道可以从哪些时间范围、哪些时间粒度、哪些维度视角进行统计查询，以及指标的计算规则是什么。一般情况下，同属于一个 **业务** 的指标会共享相同的时间范围、时间粒度及维度项，我们将类似这样的业务统称为 **主题**。每一个主题对应着一个业务或业务中的某个细分场景，如：

* 移动端视频上传性能
* PC端视频上传性能
* 移动端视频劫持性能
* CDN服务质量分析

&nbsp;
每一个 **主题** 需要包含以下信息：

* 主题名称

    顾名思义，业务名称或业务中的某个细分场景名称。
&nbsp;
* 时间范围

    可支持的数据查询时间范围（[起始时间，结束时间]）；相当于数据的存储周期，过期数据会被清除。
&nbsp;
* 时间粒度
    
    可支持的数据查询最小时间粒度，相当于数据的聚合程度，如：1 秒，5 分钟，1 小时 或 1 天 之类的。
    假设时间粒度是 5 分钟，表示最小可以支持 每5分钟 一个数据点的查询请求，同时也可以支持 每1小时 或 每1天 之类的查询请求，但不可以支持 每1秒 或 每1分钟 之类的查询请求。
&nbsp;
* 维度

    可以支持的数据查询视角，通常会有多个，如：省份、运营商、设备 之类的。以 运营商 为例，表示可以查看指定运营商或各个运营商的数据。
&nbsp;
* 维度值

    维度可以支持的查询值，通常会有多个。以维度 运营商 为例，维度值会有移动、联通、电信之类的。
&nbsp;
* 指标

    可以支持查询的指标，通常会有多个，如：秒开率、卡顿率 之类的。
&nbsp;
* 指标计算规则

    某个指标的计算公式。以指标 卡顿率 为例，计算公式：卡顿次数 / 总次数。

&nbsp;
若干个 **主题** 的 **集合**，我们称之为 **指标库**。**指标库** 的内容可以按照上述描述的格式记录到专用的文档中，统一更新发版，用于公司内部数据共享。

# 查询语言

数据应用服务（数据可视化、报表、数据接口）的核心逻辑是查询指标数据，并不是使用 对应关系 和 计算规则 完成指标计算，且存在重复直接耦合的问题。为此，我们在 **指标库** 的基础之上，针对 **指标多维统计查询** 场景设计了一种领域特定语言（DSL）。也就是说，数据应用服务查询指标数据时直接使用统一的查询语言，不再存在直接耦合 对应关系 和 计算规则 的情况，后续 对应关系（查询引擎升级或迁移）或 计算规则 的 **变更** 对于数据应用服务也是 **透明** 的。

查询语言是基于 **JSON** 实现的。

## getTopics

查询指标库中有哪些主题（多个）。

```
 {
   "type": "getTopics"
 }
```

## getDimentions

查询指定主题中有哪些维度项（多个）。

```
 {
   "type": "getDimentions",
   "topic": "..."
 }
```

其中，topic 为主题名称。

## getDimentionValues

查询指定主题、指定维度中有哪些维度值（多个）。

```
 {
   "type": "getDimentionValues",
   "topic": "...",
   "dimension": "..."
 }
```

其中，topic 用于指定主题名称， dimension 用于指定维度名称。

## getMetrics

查询指定主题中有哪些指标项（多个）。

```
 {
   "type": "getMetrics",
   "topic": "..."
 }
```

其中，topic 用于指定主题名称。

## query

查询（指标数据），是查询语言中最为复杂的类型。

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

### topic

主题名称（必填项）。

### interval

时间范围（必填项）。

```
 {
     "start": "...",
     "end": "..."
 }
```

其中，start/end 均为闭区间，格式：yyyy-MM-dd HH:mm:ss。

### granularity

时间粒度，默认为无穷大。

```
 {
     "data": ...,
     "unit": "..."
 }
```

其中，data 为数值（整型），unit 为单位，如：s(秒)、m（分钟）、h（小时）、d（天）、w（周）、M（月）、y（年）。

### metric

指标名称（必填项）。

### where

维度过滤，对应于SQL语句中的 **Where**，默认为空，支持比较表达式与逻辑表达式。

#### 比较表达式

比较表达式用于表示数值比较、包含或不包含，以及正则匹配。

##### eq/ne/gt/lt/ge/le

```
 {
     "operator": "...",
     "name": "...",
     "value": "..."
 }
```

其中，operator 为 eq（等于）、ne（不等于）、gt（大于）、lt（小于）、ge（大于或等于）或 le（小于或等于），name 为维度名称，value 为维度值。

##### in

```
 {
     "operator": "in",
     "name": "...",
     "values": [...]
 }
```

其中，operator 为 in（包含），name 为维度名称，values 为维度值列表（JSON数组）。

##### regex

```
  {
     "operator": "regex",
     "name": "...",
     "pattern": "..."
 }
```

其中，operator 为 regex（正则匹配），name 为维度名称，pattern 为正则表达式。


#### 逻辑表达式

逻辑表达式用于表示一个或多个表达式之间的 与（And）、或（Or）、非（Not）关系。

##### and

```
 {
     "operator": "and",
     "filters": [...]
 }
```

其中，operator 为 and（与），filters 为两个或两个以上的比较表达式或逻辑表达式。

##### or

```
 {
     "operator": "or",
     "filters": [...]
 }
```

其中，operator 为 or（或），filters 为两个或两个以上的比较表达式或逻辑表达式。

##### not

```
 {
     "operator": "not",
     "filter": {...}
 }
```

其中，operator 为 not（非），filter 为一个比较表达式或逻辑表达式。

### groups

维度分组，类似于SQL语句中的 **GroupBy**，默认为空，可以指定多个维度项（JSON数组）。

```
 [
     "...",
      ...
 ]
```

### having

分组过滤，类似于SQL语句中的 **Having**，默认为空，语法同 **where**。分组过滤表达式中的 name 可以是维度名称，也可以是指标名称。

### orders

排序，类似于SQL语句中的 **OrderBy**，默认为空，可以指定多个排序项。

```
 [
     {
         "name": "...",
         "sort": "..."
     },
     ...
 ]
```

其中，name 为维度名称或指标名称，sort 为 asc（升序）或 desc（降序）。

### limit

限制结果集行数，类似于SQL语句中的 **Limit**，默认为空。

### 示例参考

**查询指标**

主题：CDN服务质量分析； 
时间范围：["2020-03-16 00:00:00", "2020-03-16 23:59:59"]; 
时间粒度：1小时； 
指标：下载速度（download_speed_avg）； 
维度过滤：域名（domain）包含有字符串“weibo”； 
维度分组：运营商（isp）； 
分组过滤：下载速度（download_speed_avg）大于2000 Kb/s； 
排序：下载速度降序（download_speed_avg）； 
结果集：最多100行；

&nbsp;
**查询语言**

```
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

# 查询代理

 查询语言只是一种描述性的协议规范，并不能完成实际的查询过程。我们需要一种 **服务**，可以接收并执行这种查询语言，过程如下：
 
 1. 解析查询语言，获取需要查询的指标；
 2. 查找指标对应的查询引擎及计算规则；
 3. 使用查询语言设定的查询条件，连接查询引擎，根据计算规则调用API，执行指标的实际计算过程；
 4. 返回结果。

这就是我们即将要重点介绍的 **查询代理**。
 
## 客户端

**AnalysisQl** 本质是一套Java API，需要嵌入到Web或RPC容器中使用。

```
DefaultConnector connector = new DefaultConnector();

...

AnalysisQl analysisQl = new AnalysisQl(connector);
```

AnalysisQl 的实现是线程安全的，一个Web或RPC容器进程中创建一个 AnalysisQl 实例即可，实例创建完成即表示查询代理服务启动完成。

查询语言的执行都需要通过 AnalysisQl.request 来完成，如下：

```
Response response = analysisQl.request(dsl);
```

相当于，AnalysisQl 是查询代理的客户端。

&nbsp;
**代码参考**

https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/AnalysisQl.java

## 解析器

查询语言是基于 **JSON** 实现的，不方便直接在Java语言中使用，需要将其 **解析** 成相应的 **JavaBean**，如下：

```
    Parser parser = new Parser(connector);

    Request request = parser.parse(dsl);
```

**Parser** 是解析器，**Request** 是查询类型的顶层抽象类，查询语言中的每一种查询类型都有其对应的实现类：

* GetTopicsRequest（getTopics）
* GetDimensionsRequest（getDimentions）
* GetDimensionValuesRequest（getDimentionValues）
* GetMetricsRequest（getMetrics）
* QueryRequest（query）

&nbsp;
**代码参考**

https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/dsl/request/GetTopicsRequest.java
https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/dsl/request/GetDimensionsRequest.java
https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/dsl/request/GetDimensionValuesRequest.java
https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/dsl/request/GetMetricsRequest.java
https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/dsl/request/QueryRequest.java

其中，QueryRequest 关于 **Filter**（过滤）的实现较为有意思，有兴趣的同学可以参考 https://github.com/weibodip/analysisql/tree/master/core/src/main/java/com/weibo/dip/analysisql/dsl/filter。

## 连接器

连接器用于定义查询代理支持的接口协议，对应着查询语言的查询类型：

```
public interface Connector {
  Response getTopics(GetTopicsRequest request);

  Response getDimensions(GetDimensionsRequest request);

  Response getDimensionValues(GetDimensionValuesRequest request);

  Response getMetrics(GetMetricsRequest request);

  Response query(QueryRequest request);
}
```


&nbsp;
**为什么需要设计连接器？** 

理论上，查询语言是标准的，但查询代理的实现可以是多种多样的。类似于 Java JDBC，统一抽象数据库的操作接口，连接数据库时需要使用相应类型的驱动。我们使用连接器，用于桥接查询客户端与查询代理的具体实现，保证系统的可扩展性；同时提供系统默认的连接器 DefaultConnector，满足大部分场景的使用需求。

每一种查询类型应调用连接器的哪一个接口，是通过 Request.type 判断的，如下：

```
   switch (request.getType()) {
      case Request.GET_TOPICS:
        return connector.getTopics((GetTopicsRequest) request);

      case Request.GET_DIMENSIONS:
        return connector.getDimensions((GetDimensionsRequest) request);

      case Request.GET_DIMENSION_VALUES:
        return connector.getDimensionValues((GetDimensionValuesRequest) request);

      case Request.GET_METRICS:
        return connector.getMetrics((GetMetricsRequest) request);

      case Request.QUERY:
        return connector.query((QueryRequest) request);
      default:
        throw new UnsupportedOperationException();
    }
```

&nbsp;
**代码参考**

https://github.com/weibodip/analysisql/blob/master/core/src/main/java/com/weibo/dip/analysisql/connector/Connector.java
https://github.com/weibodip/analysisql/blob/master/view/src/main/java/com/weibo/dip/analysis/view/DefaultConnector.java

## 数据视图

数据视图 **View** 是一个抽象类，对应于指标库中的主题，每一个主题均需要提供相应的数据视图实现类，并于查询代理初始化之前注册到连接器中。实现时需要设置以下信息：

```
protected String topic;

protected List<Dimension> dimensions;

protected List<Metric> metrics;

protected List<Table> tables;

protected PolicyRouter router;
```

&nbsp;
topic：主题名称；
dimensions：维度项，每一个 Dimension 表示一个维度；
metrics：指标项，每一个 Metric 表示一个维度；
tables：数据表，每一个 Table 表示一张数据表，详细见后；
router：策略路由，详情见后；

&nbsp;
以 **CDN服务质量分析** 为例，数据视图实现类如下：

```
public class CdnServiceQualityView extends DefaultView {

  /** Initialize a instance. */
  public CdnServiceQualityView() {
    super(
        "CDN服务质量分析", // 主题名称
        ...);

    addDimension("country", "国家"); // 添加维度项
    addDimension("business", "业务");
    addDimension("city", "城市");

    addMetric("error_num", "错误量（万）"); // 添加指标项
    addMetric("request_num", "访问量（亿）");
    addMetric("download_time_avg", "下载时间（s）");

    addTable(new CdnServiceQualityTable(this)); // 添加数据表
    addTable(new CdnNetflowBillingTable(this));
  }
}
```

**注**：DefaultView 是 View 的默认实现类。

CdnServiceQualityView 构建函数中直接完成主题、维度项、指标项及数据表的设置，也可以在 CdnServiceQualityView 实例创建完成之后，通过相应的实例方法（addDimension/addMetric/addTable）设置。

然后，注册到连接器：

```
connector.register(new CdnServiceQualityView());
```

连接器中保存着所有已注册的数据主题：

```
  protected Map<String, Metadata> metadatas = new HashMap<>();

  public void register(Metadata metadata) {
    metadatas.put(metadata.getTopic(), metadata);
  }
```

**注**：Metadata 是 View 的抽象父类。

连接器中多数协议方法的实现本质就是根据主题名称（topic）查找到相应的数据主题实例，调用其实例方法完成查询请求。

&nbsp;
**为什么需要设计数据视图？**
&nbsp;

假设数据表 A 以 5分钟 的时间粒度，存储着 m 个维度、n 个指标的数据，每5分钟的数据量（即：数据行数）的最大值理论上取决于各个维度的维度值数目；如果 m 个维度的维度值数目依次为 N1、N2、...、Nm，那么每5分钟数据量为 N1 * N2 * ... * Nm（即多个维度的维度值数目的笛卡尔乘积）；每日的数据最为 288 * N1 * N2 * ... * Nm。

我们列举几个常用的查询：

1. 查询最近 1小时内，以 5分钟 为时间粒度，某个维度或某几个维度组合的指标数据；
2. 查询最近 7天内，以 1小时 为时间粒度，某个维度或某几个维度组合的指标数据；
3. 查询最近 1月内，以 1天 为时间粒度，某个维度或某几个维度组合的指标数据；

可以看出，数据表 A 是可以完全支撑上述查询需求的。那么，问题在于哪里？问题在于查询的 **响应时间**，响应时间很大程度上取决于查询时需要扫描的 **数据量**。在我们的场景中，很多业务按数据表 A 的方式存储数据，每天的数据量会达到数百亿级别，较长时间范围内（跨天/跨周/跨月）的指标数据查询，响应是比较缓慢的。

我们经常使用的优化方案：

* 创建数据表 B，以 5分钟 为时间粒度，存储数据表 A 中部分维度的指标数据；

    维度数目的减少，表示着数据表 B 相对于 数据表 A 的数据是减少的。以上述 查询1 为例，如果需要查询的维度正好存在于数据表 B，那么使用数据表 B 相比于 数据表 A，需要扫描的数据量更少，响应时间更快。
&nbsp;
* 创建数据表 C，以 1天 为时间粒度，存储数据表 A 聚合之合的指标数据；
    时间粒度的增大，表示着数据表 C 相对于 数据表 A 的数据量是减少的。以上述 查询3 为例，使用数据表 C 相比于 数据表 A，需要扫描的数据量更少，响应时间更快。

&nbsp;
扩展一下思路：可以根据实际的业务场景，在基础数据表（如:数据表A）之上，有效组合不同的时间粒度、不同的维度组合额外创建出若干张数据表，用于优化查询的响应时间。

也就是说，指标库中的某一个主题可以对应着多张数据表，这些数据表可以有不同的时间粒度、不同的维度组合，甚至不同的指标项，存储于不同的查询引擎，或者被多个主题所共享。这样的 **自由组合** 从工程角度看是非常灵活的；但从数据服务角度看，会带来2个问题：

* 多张数据表之间的信息是 **混乱** 的，缺乏一致的数据口径；
* 查询指标数据时应该使用哪张数据表？

因此，我们设计 **数据视图**，显示声明包含的维度、指标信息，以及相关的数据表（参考数据视图定义）。数据视图 如何解决查询指标时使用的数据表问题，参见后文。

**代码参考**

https://github.com/weibodip/analysisql/blob/master/view/src/main/java/com/weibo/dip/analysis/view/View.java

## 数据表


如前文所述，同一个主题（数据视图）中可以包含多张数据表，且这些数据表之间可能会有不同的时间粒度、维度项、指标项等，数据表（Table）的设计需要能够明确显示声明这些信息。

```
protected String topic;

protected List<Dimension> dimensions;

protected List<Metric> metrics;

protected Map<String, MetricCalculator> calculators;

private Granularity granularity;
private int period;
private int delay;
```

&nbsp;
topic：数据表属于的主题名称；
dimensions：数据表支持的维度项；
metrics：数据表支持的指标项；
calculators：数据表支持的指标项对应的 **指标计算器** （参见后文）；
granularity：数据表存储数据的时间粒度；
period：数据表存储数据的周期，也就是数据表可以查询的时间范围；
delay：数据表数据的延迟时间，也就是说相对于当前时间，这个延迟范围内的数据是查询不到的；

以 **CDN服务质量分析** 的数据表为例，数据表实现类如下：

```
public class CdnServiceQualityTable extends Table {
 
  public CdnServiceQualityTable(View view) {
    super(view, "clickhouse-cdn-all_cdn_staging", new Granularity(5, Unit.m), 103680, 36); // 设置时间粒度、存储周期及延迟时间

    addDimension("country"); // 声明支持的维度项
    addDimension("business");
    addDimension("city");

   
    addCalculator("error_num", new HubbleClickHouseCalculator("analysisql/cdn/cdn-error_num.sql")); // 声明支持的指标项的同时，提供指标相应的指标计算器
    addCalculator(
        "request_num", new HubbleClickHouseCalculator("analysisql/cdn/cdn-request_num.sql"));
    addCalculator(
        "download_time_avg",
        new HubbleClickHouseCalculator("analysisql/cdn/cdn-download_time_avg.sql"));
  }
}
```

&nbsp;
**代码参考**

https://github.com/weibodip/analysisql/blob/master/view/src/main/java/com/weibo/dip/analysis/view/Table.java

## 策略路由

数据视图（View）中包含多张数据表（Table），查询指标数据时，我们需要根据一定的 **规则** 从这些数据表中选取出 **查询代价最小**（响应时间最快） 的数据表用于指标计算，**规则** 的具体实现就是 **策略路由**。

### 过滤

过滤就是 **排除** 不能支持查询的数据表，考虑以下4个因素：

1. 指标

    数据表必须包含查询的指标项；
2. 维度

    数据表必须包含查询涉及的维度项（多个）；
3. 时间粒度

    数据表数据存储的时间粒度必须小于或等于查询指定的时间粒度，否则无法聚合数据；
4. 时间范围

    数据表在查询指定的时间范围内存在数据（根据数据表存储周期及延迟时间计算）；
    
满足上述4个条件的数据表即可以进入下一阶段。

### 排序

排序规则：

* 数据表的时间粒度越大，查询代价越小；
* 数据表的维度数目越少，查询代价越小；

经过过滤、排序之后，位于第1个位置的数据表即是 **最优** 的数据表。目前，策略路由（PolicyRouter）的实现是直接内置于数据视图，尚不支持自定义扩展。

&nbsp;
**代码参考**

https://github.com/weibodip/analysisql/blob/master/view/src/main/java/com/weibo/dip/analysis/view/PolicyRouter.java

## 指标计算器

指标计算器（MetricCalculator）用于根据查询请求完成指标的计算。也就是说，前文中提及的 对应关系 和 计算规则 均包含在指标计算器中，指标计算器需要连接查询引擎，使用相应的API或SQL，根据计算规则完成指标的计算过程，并返回结果。数据表的各个指标均需要提供相应的指标计算器实现器，并注册到数据表中。

```
public interface MetricCalculator {
  List<Row> calculate(QueryRequest request) throws Exception;
}
```

考虑到常用的查询引擎（如：MySQL、Presto、ClickHouse）大多支持以 **SQL** 的方式查询数据，为加速指标计算器的实现效率，系统支持以 SQL模板 的方式定义指标的计算规则，并提供多种 **模板引擎**， 可将 查询条件 与 SQL模板 整合转换特定查询引擎的SQL语句（不同查询引擎的SQL实现存在一定差异）。

以 ClickHouse 为例，某指标计算规则可以这样定义：

```
SELECT
    $COLUMNS, SUM(netflow) * 8 / 1000 / 1000 / 1000 AS $METRIC
FROM
    cdn.edge_scheduler_bandwith
WHERE
    $WHERE
GROUP BY
    $GROUPS
HAVING
    $HAVING
    AND isNaN($METRIC) == 0
    AND isInfinite($METRIC) == 0
    AND $METRIC >= 0.0
ORDER BY
    $ORDERS
LIMIT
    $LIMIT
```
&nbsp;
**\$COLUMNS**、**\$WHERE**、**\$GROUPBY**、**\$HAVING**、**\$ORDERBY**、**\$LIMIT** 均是 **模板引擎** 支持的自定义变量（不同的模板引擎支持的变量种类存在一定差异）。**模板引擎** 会将查询语言中的过滤表达式、维度分组、分组过滤、排序及结果集行数转换为这些自定义变更对应的值，并输出完整的SQL语句。

指标计算器直接使用转换之后的SQL语句，连接查询引擎查询数据即可。

综上所述，查询代理核心工作流程如下：

1. 解析查询语言；
2. 使用主题名称查找数据视图；
3. 使用策略路由选取最优数据表；
4. 使用指标名称查找最优数据表的指标计算器；
5. 计算指标并返回结果；

数据应用服务仅需要知道查询代理服务地址（域名）、端口号，使用查询语言查询需要的指标数据即可。

# 结语

本文介绍的指标库、查询语言（DSL）、查询代理是我们团队自主研发的OLAP服务，在微博视频性能数据分析中取得很好地应用效果。通过技术优化的方式，在有限的计算资源范围内得到不错的性能表现，大幅降低数据接口、可视化及监控服务的开发成本。 同时，我们团队也在准备项目开源（https://github.com/weibodip/analysisql ）的准备工作，有兴趣的同学可关注交流。

&nbsp;
&nbsp;
&nbsp;
![avatar](https://yurun-blog.oss-cn-beijing.aliyuncs.com/%E5%85%AC%E4%BC%97%E5%8F%B7/card.png)
