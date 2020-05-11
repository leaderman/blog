# Hive UDF使用资源文件及动态更新方案

## 背景

**注：** 本文中的“函数”等同于UDF，默认情况下特指永久函数。

Hive 0.13版本开始支持自定义永久函数（Permanent Function）,可以将函数注册到Hive Metastore，通过Hive/Beeline/Spark SQL可以直接引用，不需要类似于临时函数（Temporary Function） ，每次使用时均需要显式声明创建的过程。

函数创建语句：

```sql
CREATE FUNCTION [db_name.]function_name AS class_name
  [USING JAR|FILE|ARCHIVE 'file_uri' [, JAR|FILE|ARCHIVE 'file_uri'] ];
```

函数删除语句：

```sql
DROP FUNCTION [IF EXISTS] function_name;
```

&nbsp;
假设业务场景：自定义函数的计算逻辑过程中，需要使用 **数据**，且 **数据** 存在需要动态（定时或临时）更新的情况。

1. 数据量较小

    数据量较小，且需要动态更新的场景下，可以将数据存储至类似于MySQL的数据引擎中，数据需要更新时，直接更新MySQL中的数据即可；函数初始化时，可以将MySQL中的数据一次性加载到内存中，用于函数后续计算逻辑使用。
 
    复杂一点的情况，可以考虑给MySQL中的每一条数据附加一个时间戳，以每次数据的更新时间为准；也可以写入新的数据时，使用一个较旧的时间戳，数据写入完成之后，统一调整为新的时间戳；函数初始化时，加载具有最新时间戳的数据。
    
    **注：** 为什么需要一次性完成数据加载？Hive UDF运行于MapReduce或Spark分布式计算应用中，如果UDF处理每一行数据均需要访问MySQL，海量数据场景下，将会导致同一时刻有大量的并发连接，很容易压垮MySQL。
    
2. 数据量较大
    
    数据量较大，且需要动态更新的场景下，不适用于将数据存储至类似于MySQL的数据引擎，有以下2方面的考虑：
    
    a. 数据加载时间较长；
    b. 数据使用内存空间较大；
    
    这种情况下，建议使用 **资源文件**，即预先将数据写入文件（文本文件或索引文件），将文件存储至HDFS；函数初始化时，可以直接读取HDFS中的文件，也可以将HDFS中的文件下载至本地后再读取。
    
    **注：** 索引文件可以应用于数据使用内存空间较大的场景下，如：将数据生成Lucene索引文件并上传至HDFS，函数使用时将索引文件下载至本地，基于磁盘即可快速检索数据，无需装入内存。
    
    资源文件更新策略参见下文。

## 资源文件动态更新

HDFS中的文件仅支持追加（Append）操作，不支持更新（Update）操作，因此无法通过直接修改HDFS中已有的资源文件内容完成更新。假设HDFS中资源文件的存储目录为“/tmp/resource/”，资源文件名称为“resource.data”，以天为粒度进行更新，方案如下：

1. 使用当天的最新数据，创建资源文件，名称：resource.data；
2. 将资源文件resource.data上传至HDFS，名称：/tmp/resource/resource.data；
3. 将已上传至HDFS中的资源文件 **/tmp/resource/resource.data** 重命名为 **/tmp/resource/resource.data.yyyy_MM_dd**；其中，**yyyy_MM_dd** 为当天日期；
4. 函数初始化时，根据资源文件的日期后缀，读取或下载最新的资源文件；

资源文件存储目录示例如下：

```text
/tmp/resource/resource.data.2020_05_01
/tmp/resource/resource.data.2020_05_02
/tmp/resource/resource.data.2020_05_03
...
```

**注：** 如果直接上传带有最新时间戳的资源文件，文件较大情况下，耗时可能较长，资源文件未上传完成之前，可能会被正在运行的函数读取或下载，因文件不完整引发异常；HDFS中的重命名为原子操作，使用重命名的方式可以避免这种问题。

## Hive UDF Jar 动态更新

Hive UDF目前仅支持创建（Create）/删除（Drop），不支持更新（Update）。

创建一个名称为“udf_map_test”的函数：

```sql
CREATE FUNCTION udf_map_test AS 'com.weibo.dip.hubble.mis.udf.UdfMapTest' USING JAR 'viewfs://c9/user_ext/weibo_rd_dip/udf/jar/udf_map_test.jar';
```

该函数依赖位于HDFS的Jar文件：viewfs://c9/user_ext/weibo_rd_dip/udf/jar/udf_map_test.jar。

函数的计算逻辑需要更新时，可以修改函数相关的代码，重新编译、打包，然后可以选择以下2种方式之一进行更新：

1. 删除函数，删除HDFS Jar文件，重新上传新的Jar文件至HDFS，创建函数；
2. 删除HDFS Jar文件，重新上传新的Jar文件（两者路径名称须保持一致）；

上述2种方式或者其它类似方式，本质目的在于替换HDFS Jar文件。一般情况下，更新过程需要的时间较短，但 **替换** 过程并不是 **原子** 操作；如果更新过程中，恰好（概率大小取决于集群规模及应用数量）有计算应用需要使用该函数，则会引发异常。

更好的方案是借鉴资源文件动态更新的策略。Hive UDF Jar中仅包含业务接口，不包含业务接口的实现类；业务接口的实现类位于额外的一个Jar中，这个Jar的存储方式类似于资源文件动态更新，也使用时间戳后缀的命名方式；Hive UDF初始化时，根据类名称及Jar名称前缀从HDFS中选取最新的Jar，并从中加载具体的业务接口实现类。

业务接口与函数定义相对应，在函数的输入参数个数及类型、返回结果类型不发生变化的情况化下，业务接口无须变更，业务接口生命周期相对稳固。函数的计算逻辑发生变化时，仅需要修改业务接口实现类，然后重新编译、打包，并按照最新的时间戳上传即可（过程参考资源文件更新）。

首先，需要一个兼容HDFS的类加载器，代码如下：

```java
import com.google.common.base.Preconditions;
import java.net.URL;
import java.net.URLClassLoader;
import org.apache.commons.lang.ArrayUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.FsUrlStreamHandlerFactory;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.fs.PathFilter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * UdfClassLoader.
 *
 * @author yurun
 */
public class UdfClassLoader extends URLClassLoader {
  private static final Logger LOGGER = LoggerFactory.getLogger(UdfClassLoader.class);

  static {
    try {
      URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());
    } catch (Error e) {
      if (!e.getMessage().equals("factory already defined")) {
        throw new ExceptionInInitializerError(e);
      }
    }
  }

  private String directory;
  private String file;

  /**
   * Initialize a instance.
   *
   * @param directory hdfs directory path
   * @param file hdfs jar name prefix
   */
  public UdfClassLoader(String directory, String file) {
    super(new URL[] {}, UdfClassLoader.class.getClassLoader());

    try {
      Path hdfsDirectory = new Path(directory);

      FileSystem fs = hdfsDirectory.getFileSystem(new Configuration());

      FileStatus[] statuses =
          fs.listStatus(
              hdfsDirectory,
              new PathFilter() {
                @Override
                public boolean accept(Path path) {
                  return path.getName().startsWith(file);
                }
              });
      Preconditions.checkState(
          ArrayUtils.isNotEmpty(statuses),
          String.format("File %s not exist in directory %s", file, directory));

      Path jarPath = statuses[statuses.length - 1].getPath();
      LOGGER.info("Directory: {}, File: {}, Jar: {}", directory, file, jarPath.getName());

      addURL(jarPath.toUri().toURL());
    } catch (Exception e) {
      throw new ExceptionInInitializerError(e);
    }
  }
}
```

**UdfClassLoader** 继承自 **URLClassLoader**，其中，

```java
URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());
```

FsUrlStreamHandlerFactory，设置 **URLClassLoader** 兼容HDFS数据访问协议。

```java
  private String directory;
  private String file;
```

**directory** 和 **file** 用于定义业务接口实现类所属的Jar文件位于HDFS的存储路径和文件名称（前缀）。

```java
      Path hdfsDirectory = new Path(directory);

      FileSystem fs = hdfsDirectory.getFileSystem(new Configuration());

      ......

      Path jarPath = statuses[statuses.length - 1].getPath();
      LOGGER.info("Directory: {}, File: {}, Jar: {}", directory, file, jarPath.getName());

      addURL(jarPath.toUri().toURL());
```

检索具有最新时间戳的Jar文件，并加入 **URLClassLoader** 的类搜索路径。

然后，定义、创建业务接口及其实现类；

```java
public interface RelationService extends UdfService {
  String get(String key);
}
```

```java
public class RelationServiceImpl implements RelationService {
  private static final Logger LOGGER = LoggerFactory.getLogger(RelationServiceImpl.class);

  @Override
  public String get(String key) {
    ...

    return ...;
  }
}
```

接着，创建Hive UDF；

```java
public class UdfMapDynamicLoadClassFromHdfsTest extends GenericUDF {
  private static final Logger LOGGER =
      LoggerFactory.getLogger(UdfMapDynamicLoadClassFromHdfsTest.class);

  private static final RelationService RELATION_SERVICE;

  static {
    try {
      UdfClassLoader udfClassLoader =
          new UdfClassLoader(
              "viewfs://c9/user_ext/weibo_rd_dip/udf/jar",
              "udf_map_dynamic_load_class_from_hdfs_test_impl");

      RELATION_SERVICE =
          (RelationService)
              udfClassLoader
                  .loadClass("com.weibo.dip.hubble.mis.udf.example.dynamic.RelationServiceImpl")
                  .newInstance();
    } catch (Exception e) {
      throw new ExceptionInInitializerError(e);
    }
  }

  @Override
  public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
    ......
  }

  @Override
  public Object evaluate(DeferredObject[] arguments) throws HiveException {
    String key = (String) converter.convert(arguments[0].get());

    RELATION_SERVICE.get(key);

    return ...;
  }

  @Override
  public String getDisplayString(String[] children) {
    ......
  }

  @Override
  public void close() throws IOException {
    ......
  }
}
```

注意，在UDF实现类的初始化过程中，完成业务接口的声明，以及业务接口实现类的加载及实例创建。业务接口 **RELATION_SERVICE** 使用静态实例的形式，主要是考虑Hive UDF的序列化及线程安全，这里不深入讨论。

```java
private static final RelationService RELATION_SERVICE;

  static {
    try {
      UdfClassLoader udfClassLoader =
          new UdfClassLoader(
              "viewfs://c9/user_ext/weibo_rd_dip/udf/jar",
              "udf_map_dynamic_load_class_from_hdfs_test_impl.jar");

      RELATION_SERVICE =
          (RelationService)
              udfClassLoader
                  .loadClass("com.weibo.dip.hubble.mis.udf.example.dynamic.RelationServiceImpl")
                  .newInstance();
    } catch (Exception e) {
      throw new ExceptionInInitializerError(e);
    }
  }
```

其中，“viewfs://c9/user_ext/weibo_rd_dip/udf/jar”表示HDFS存储Jar文件的目录，“udf_map_dynamic_load_class_from_hdfs_test_impl.jar”表示业务接口实现类的Jar文件名称，“com.weibo.dip.hubble.mis.udf.example.dynamic.RelationServiceImpl”表示业务接口实现类的名称。UDF的实现时仅需要简单调度业务接口即可：

```java
RELATION_SERVICE.get(key);
```

最后，编译、打包及上传。

打包时，UDF、业务接口以及类加载器需要打包为一个Jar文件，业务接口实现类需要打包为另一个Jar文件。除函数首次创建之外，计算逻辑变更仅需要重新编译、打包及上传业务接口实现类即可。

## 结束语

本文中描述的Hive UDF资源文件、计算逻辑动态更新方案，可以较好地满足复杂场景下业务变更且不会引发异常的需求，但一定程度上也增加了Hive UDF实现及打包的复杂度，可以根据实际的的需求或应用场景决定是否使用。







