# Request

Request是用于封装AnalysisQl DSL的顶层抽象类，如下：

```java
public abstract class Request implements Serializable {
  protected String sessionId;
  protected String type;
  
  public Request(String type) {
    this.type = type;
  }
}
```

&nbsp;
sessionId：唯一标识一次请求；
type：标识请求类型；

与AnalysisQl DSL相对应，Request支持以下5种实现类型：

* getTopics（GetTopicsRequest）
* getDimensions（GetDimensionsRequest）
* getDimensionValues（GetDimensionValuesRequest）
* getMetrics（GetMetricsRequest）
* query（QueryRequest）

![Request](https://yurun-blog.oss-cn-beijing.aliyuncs.com/analysisql/request.png)

## GetTopicsRequest

获取主题请求（getTopics），如下：

```java
public class GetTopicsRequest extends Request {
  public GetTopicsRequest() {
    super(Request.GET_TOPICS);
  }
}
```

例如，获取主题请求，new GetTopicsRequest()。

## GetDimensionsRequest

获取指定主题维度项请求（getDimensions），如下：

```java
public class GetDimensionsRequest extends Request {
  private String topic;
  public GetDimensionsRequest() {
    super(Request.GET_DIMENSIONS);
  }
  public GetDimensionsRequest(String topic) {
    this();
    this.topic = topic;
  }
}
```

&nbsp;
topic：主题名称；

例如，获取主题“CDN服务质量分析”的维度项请求，new GetDimensionsRequest("CDN服务质量分析")。

## GetDimensionValuesRequest

获取指定主题、指定维度的维度值请求（getDimensionValues），如下：

```java
public class GetDimensionValuesRequest extends Request {
  private String topic;
  private String dimension;
  private Filter where;

  public GetDimensionValuesRequest() {
    super(Request.GET_DIMENSION_VALUES);
  }

  public GetDimensionValuesRequest(String topic, String dimension) {
    this(topic, dimension, null);
  }

  public GetDimensionValuesRequest(String topic, String dimension, Filter where) {
    this();

    this.topic = topic;
    this.dimension = dimension;
    this.where = where;
  }
}
```

&nbsp;
topic：主题名称；
dimension：维度名称；
where：级联维度信息；

维度值的级联请求是一种比较特殊的情况，以中国的省份/城市为例，获取城市名称时，有些场景下并不需要中国全部的城市名称，而是某一个省份的城市名称，就可以通过where指定具体的省份。

例如，获取主题“CDN服务质量分析”、维度“isp”的维度值请求，new GetDimensionValuesRequest("CDN服务质量分析", "isp")。

**注** ：Filter描述见后。

## GetMetricsRequest

获取指定主题的指标项请求（getMetrics），如下：

```java
public class GetMetricsRequest extends Request {
  private String topic;

  public GetMetricsRequest() {
    super(Request.GET_METRICS);
  }

  /**
   * Initialize a instance with topic.
   *
   * @param topic topic
   */
  public GetMetricsRequest(String topic) {
    this();

    this.topic = topic;
  }

  public String getTopic() {
    return topic;
  }

  public void setTopic(String topic) {
    this.topic = topic;
  }
}
```

&nbsp;
topic：主题名称；

例如，获取主题“CDN服务质量分析”的指标项，new GetMetricsRequest("CDN服务质量分析")。

## QueryRequest

查询请求（query），如下：

```java
public class QueryRequest extends Request {
  private String topic;
  private Interval interval;
  private Granularity granularity;
  private String metric;
  private Filter where;
  private String[] groups;
  private Filter having;
  private Order[] orders;
  private int limit = -1;

  public QueryRequest() {
    super(Request.QUERY);
  }

  public QueryRequest(
      String topic,
      Interval interval,
      Granularity granularity,
      String metric,
      Filter where,
      String[] groups,
      Filter having,
      Order[] orders,
      int limit) {
    this.topic = topic;
    this.interval = interval;
    this.granularity = granularity;
    this.metric = metric;
    this.where = where;
    this.groups = groups;
    this.having = having;
    this.orders = orders;
    this.limit = limit;
  }
}
```

### topic

主题名称。

***

### interval

**Interval** 类型，表示时间范围，如下：

```java
/** Interval. */
public class Interval implements Serializable {
  private Date start;
  private Date end;

  public Interval(Date start, Date end) {
    this.start = start;
    this.end = end;
  }
}
```

start：起始时间，闭区间；
end：结束时间，闭区间；

***

### granularity

**Granularity** 类型，表示时间粒度，如下：

```java
public class Granularity implements Serializable {
  private int data;
  private Unit unit;

  public Granularity(int data, Unit unit) {
    this.data = data;
    this.unit = unit;
  }

  public enum Unit {
    s,
    m,
    h,
    d,
    w,
    M,
    q,
    y
  }
}
```

data：时间数值；
unit：时间单位；枚举类型（Unit），s（秒）、m（分钟）、h（小时）、d（天）、w（周）、M（月）、q（季度）、y（年）;

例如：5分钟，new Granularity(5, Unit.m)。

***

### metric

指标名称。

***

### where

**Filter** 类型，表示维度过滤条件，类似于SQL where语句。

**注** ：Filter描述见后。

***

### groups

维度分组。

例如，使用国家（country）、省份（province）分组统计指标数值，new String[]{"国家", "省份"}。

***

### having

**Filter** 类型，表示分组过滤条件，类似于SQL having语句，可以过滤分组维度及指标数值。

**注** ：Filter描述见后。
***

### orders

**Order** 类型，表示排序条件（多个），类似于SQL order by 语句，可以对“timeBucket”、分组维度及指标数值进行升序或降序排列，如下：

```java
public class Order implements Serializable {
  private String name;
  private Sort sort;

  public Order() {}

  public Order(String name, Sort sort) {
    this.name = name;
    this.sort = sort;
  }
  
  public enum Sort {
    asc,
    desc
  }
}
```

&nbsp;
name：排序字段名称，可以是“timeBucket”、分组维度名称或指标名称之一；
sort：升序或降序；枚举类型（Sort），asc（升序）、desc（降序）；

例如，使用国家（country）按字典序降序排列，new Order[]{new Order("country", Sort.desc)}。

***

### limit

结果集行数大小。

# Filter Expression

过滤表达式（Filter Expression），类似于SQL WHERE/HAVING语句，例如：

* 国家（country）为“中国”，省份（province）为“山西”

    country = '中国' AND province = '山西'
&nbsp;
* 省份（province）为“山西”、“河北”或“广东”

    province = '山西' OR province = '河北' OR province = '广东'
    
    province IN ['山西'， ‘河北’, '广东']
&nbsp;
* 省份（province）不为“山西”、“河北”或“广东”

    province != '山西' AND province != '河北' AND province ！= '广东'
    
    province NOT IN [山西'， ‘河北’, '广东']
&nbsp;
* 秒开率（psr1）大于或等于90%

    psr1 >= 0.9
&nbsp;
* 域名（domain）包含“weibo.com”

    domain LIKE '%weibo.com%'

&nbsp;
**Filter** 是过滤表达式的顶层抽象父类，如下：

```java
public abstract class Filter implements Serializable {
  protected String operator;

  public Filter() {}

  public Filter(String operator) {
    this.operator = operator;
  }
}
```

&nbsp;
operator：标识过滤表达式类型；

&nbsp;
过滤表达式的实现包含以下4大类：

* 关系表达式（Relational Expression）
* 包含表达式（In Expression）
* 正则表达式（Regex Expression）
* 逻辑表达式（Logical Expression）

![Filter](https://yurun-blog.oss-cn-beijing.aliyuncs.com/analysisql/filter.png)

## Relational Expression

关系表达式由3部分组成：

* 维度名称或指标名称
* 运算符（operator）
* 值

例如，国家（country）为“中国”，关系表达式为“country = '中国'”，"country"为维度名称，“=”为运算符，“中国”为值。

运算符支持以下6种类型：

* 等于（eq）
* 不等于（ne）
* 大于（gt）
* 小于（lt）
* 大于或等于（ge）
* 小于或等于（le）

值支持3种类型：

* string（字符串）
* long（整型）
* double（浮点型）

### RelationalFilter

**RelationalFilter** 是关系表达式的抽象父类，如下：

```java
public abstract class RelationalFilter<T> extends Filter {
 
  protected String name;
  protected String type;
  protected T value;

  public RelationalFilter() {}

  public RelationalFilter(String operator, String name, String type, T value) {
    super(operator);

    this.name = name;
    this.type = type;
    this.value = value;
  }
}
```

&nbsp;
name：维度名称或指标名称；
type：值类型标识，string、long或double；
value：值，泛型；

### EqFilter

**EqFilter** 继承自 **RelationalFilter** ，是关系表达式 **等于（eq）** 的抽象父类，如下：

```java
public abstract class EqFilter<T> extends RelationalFilter<T> {
  public EqFilter() {}

  public EqFilter(String name, String type, T value) {
    super(Filter.EQ, name, type, value);
  }
```

#### StringEqFilter

**StringEqFilter** 继承自 **EqFilter** ，是关系表达式 **等于（eq）** 的 **字符串类型** 实现类，如下：

```java
public class StringEqFilter extends EqFilter<String> {
  public StringEqFilter() {}

  public StringEqFilter(String name, String value) {
    super(name, RelationalFilter.STRING, value);
  }
}
```

&nbsp;
例如，国家（country）等于“中国”，new StringEqFilter("country", "中国")。

#### LongEqFilter

**LongEqFilter** 继承自 **EqFilter** ，是关系表达式 **等于（eq）** 的 **整型** 实现类，如下：

```java
public class LongEqFilter extends EqFilter<Long> {
  public LongEqFilter() {}

  public LongEqFilter(String name, long value) {
    super(name, RelationalFilter.LONG, value);
  }
}
```

&nbsp;
例如，访问量（access）等于10000，new LongEqFilter("access", 10000)。

#### DoubleEqFilter

**DoubleEqFilter** 继承自 **EqFilter** ，是关系表达式 **等于（eq）** 的 **浮点型** 实现类，如下：

```java
public class DoubleEqFilter extends EqFilter<Double> {
  public DoubleEqFilter() {}

  public DoubleEqFilter(String name, double value) {
    super(name, RelationalFilter.DOUBLE, value);
  }
}
```

&nbsp;
例如，秒开率（psr1）等于90%，new DoubleEqFilter("psr1", .09)。

### NeFilter

**NeFilter** 继承自 **RelationalFilter** ，是关系表达式 **不等于（ne）** 的抽象父类，其余内容与 **EqFilter** 雷同，不再赘述。

### GtFilter

**GtFilter** 继承自 **RelationalFilter** ，是关系表达式 **大于（gt）** 的抽象父类，其余内容与 **EqFilter** 雷同，不再赘述。

### LtFilter

**LtFilter** 继承自 **RelationalFilter** ，是关系表达式 **小于（lt）** 的抽象父类，其余内容与 **EqFilter** 雷同，不再赘述。

### GeFilter

**GeFilter** 继承自 **RelationalFilter** ，是关系表达式 **大于或等于（ge）** 的抽象父类，其余内容与 **EqFilter** 雷同，不再赘述。

### LeFilter

**LeFilter** 继承自 **RelationalFilter** ，是关系表达式 **小于或等于（le）** 的抽象父类，其余内容与 **EqFilter** 雷同，不再赘述。

## In Expression

包含表达式由3部分组成：

* 维度名称或指标名称
* in（operator）
* 值列表（数组）

例如，国家（country）为“中国”、“美国”或“日本”，关系表达式为“country in ('中国', '美国', '日本')”，"country"为维度名称，“in”为运算符，“('中国', '美国', '日本')”为值列表。

值支持3种类型：

* string（字符串数组）
* long（整型数组）
* double（浮点型数组）

**InFilter** 继承自 **Filter** ，如下：

```java
public abstract class InFilter<T> extends Filter {
  
  protected String name;
  protected String type;
  protected T[] values;

  public InFilter() {}

  public InFilter(String name, String type, T[] values) {
    super(Filter.IN);

    this.name = name;
    this.type = type;
    this.values = values;
  }
}
```

&nbsp;
name：维度名称或指标名称；
type：值类型；
values：值列表，泛型；

### StringInFilter

**StringInFilter** 继承自 **InFilter** ，是关系表达式 **包含（in）** 的 **字符串类型** 实现类，如下：

```java
public class StringInFilter extends InFilter<String> {
  public StringInFilter() {}

  public StringInFilter(String name, String[] values) {
    super(name, InFilter.STRING, values);
  }
}
```

&nbsp;
例如，国家（country）为“中国”、“美国”或“日本”，new StringInFilter("country", new String[]{"中国", "美国", "日本"})。

### LongInFilter

**LongInFilter** 继承自 **InFilter** ，是关系表达式 **包含（in）** 的 **整型** 实现类，其余内容与 **StringInFilter** 雷同，不再赘述。

### DoubleInFilter

**DoubleInFilter** 继承自 **InFilter** ，是关系表达式 **包含（in）** 的 **浮点型** 实现类，其余内容与 **StringInFilter** 雷同，不再赘述。

## Regex Expression

正则表达式由3部分组成：

* 维度名称
* regex（operator）
* 正则字符串

例如，域名（domain）包含“weibo.com”，关系表达式为“domain regex '^.*weibo.com.*$'”，"domain"为维度名称，“regex”为运算符，“^.*weibo.com.*$”为正则字符串。

**RegexFilter** 继承自 **Filter** ，是关系表达式 **正则（regex）** 的实现类，如下：

```java
public class RegexFilter extends Filter {
  private String name;
  private String pattern;

  public RegexFilter() {}

  public RegexFilter(String name, String pattern) {
    super(Filter.REGEX);

    this.name = name;
    this.pattern = pattern;
  }
}
```

&nbsp;
name：维度名称；
pattern：正则字符串；

例如，域名（domain）包含“weibo.com”，new RegexFilter("domain", "^.*weibo.com.*$")。

## Logical Expression

逻辑表达式用于连接一个或多个表达式（关系表达式、包含表达式、正则表达式、逻辑表达式），支持以下3种类型：

* and（与）
* or（或）
* not（非）

### AndFilter

**AndFilter** 继承自 **Filter**，是关系表达式 **与（and）** 的实现类，如下：

```java
public class AndFilter extends Filter {
  private List<Filter> filters;

  public AndFilter() {
    super(Filter.AND);
  }

  public AndFilter(List<Filter> filters) {
    this();

    this.filters = filters;
  }
}
```

&nbsp;

**AndFilter** 用于表示两个或两个以上表达式之间的 **与** 关系，例如：new AndFilter(Arrays.asList(filter1, filter2, ...))。

### OrFilter

**OrFilter** 继承自 **Filter**，是关系表达式 **或（or）** 的实现类，如下：

```java
public class OrFilter extends Filter {
  private List<Filter> filters;

  public OrFilter() {
    super(Filter.OR);
  }

  public OrFilter(List<Filter> filters) {
    this();

    this.filters = filters;
  }
}
```

&nbsp;

**OrFilter** 用于表示两个或两个以上表达式之间的 **或** 关系，例如：new OrFilter(Arrays.asList(filter1, filter2, ...))。

### NotFilter

**NotFilter** 继承自 **Filter**，是关系表达式 **非（not）** 的实现类，如下：

```java
public class NotFilter extends Filter {
  private Filter filter;

  public NotFilter() {
    super(Filter.NOT);
  }
  
  public NotFilter(Filter filter) {
    this();

    this.filter = filter;
  }
}
```

&nbsp;

**NotFilter** 用于表示对一个表达式的 **非** 关系（取反），例如：new NotFilter(filter)。




