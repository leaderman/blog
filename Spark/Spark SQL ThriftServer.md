# Spark SQL ThriftServer

## 启动命令

默认情况下，Spark 日志目录 **SPARK_LOG_DIR** 指向 **SPARK_HOME/logs**，如因权限访问控制，可以通过显示设置环境变量 **SPARK_LOG_DIR**，将日志目录指向其它路径，如：

```text
export SPARK_LOG_DIR=/tmp/spark_client_logs
```

**启动**

```text
/data0/spark/spark-2.2.1-bin/sbin/start-thriftserver.sh \
--name spark_sql_thriftserver \
--master yarn \
--deploy-mode client \
--driver-cores 3 \
--driver-memory 3g \
--executor-cores 8 \
--executor-memory 20g \
--num-executors 30 \
--hiveconf hive.server2.thrift.port=10000
```

其中，**deploy-mode** 用于指定部署模式，ThriftServer仅支持 **client**；**hive.server2.thrift.port** 用于指定实例端口。

## 终止命令

```text
/data0/spark/spark-2.2.1-bin/sbin/stop-thriftserver.sh
```

## Beeline

```text
beeline> !connect jdbc:hive2://{HOSTNAME}:{PORT}
Connecting to jdbc:hive2://{HOSTNAME}:{PORT}
Enter username for jdbc:hive2://{HOSTNAME}:{PORT}:{USERNAME}
Enter password for jdbc:hive2://{HOSTNAME}:{PORT}:{PORT}
```

其中，**{HOSTNAME}** 为ThriftServer实例域名或IP地址， **{PORT}** 为ThriftServer实例端口，**{USERNAME}** 为用户名，**{PASSWORD}** 为密码。

## 公平调度

默认情况下，Spark应用内部的多个 **Job** 使用 **FIFO** 模式进行调度，可能导致多租户场景下计算时延较高，可以启用 **FAIR** 调度模式（公平调度），使多个 **Job** 的 **Tasks** 之间使用 **round bobin**（轮询）的方式分配资源。

使用公平调度模式需要经过以下2步：

1. 开启公平调度器；

```text
--conf spark.scheduler.mode=FAIR
```

2. 配置若干个资源池（Pool），以及每个资源池的调度模式（schedulingMode，FIFO或FAIR）、权重（weight）和最小资源量（minShare）；

资源池需要使用单独的文件进行配置：

```text
/usr/home/weibo_rd_dip/fairscheduler.xml
```

```text
<allocations>
  <pool name="default">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>10</minShare>
  </pool>
  <pool name="pro">
    <schedulingMode>FIFO</schedulingMode>
    <weight>10</weight>
    <minShare>10</minShare>
  </pool>
</allocations>
```

**fairscheduler.xml** 配置两个资源池：default和pro。其中，

**default**

调度模式：FAIR，公平调度；
权重：1；
最小资源量：10，CPU 10 Cores；

**pro**

调度模式：FIFO，先进先出，排队；
权重：2，相较于default，可以使用2倍的资源；
最小资源量：10，CPU 10 Cores；

&nbsp;
注意，公平调度器可以有多个资源池，且每个资源池使用不同的调度模式。

使用Beeline时，应用默认提交至default，可以使用如下命令切换资源池：

```text
SET spark.sql.thriftserver.scheduler.pool=pro;
```

启动命令示例：

```text
export SPARK_LOG_DIR=/tmp/spark_client_logs

/data0/spark/spark-2.2.1-bin/sbin/start-thriftserver.sh \
--name spark_sql_thriftserver \
--master yarn \
--deploy-mode client \
--driver-cores 180 \
--driver-memory 3g \
--executor-cores 8 \
--executor-memory 30g \
--num-executors 30 \
--hiveconf hive.server2.thrift.port=10000 \
--conf spark.scheduler.mode=FAIR \
--conf spark.scheduler.allocation.file=/usr/home/weibo_rd_dip/fairscheduler.xml
```
