# Hive UDF使用资源文件及动态更新方案--后记

在 [Hive UDF使用资源文件及动态更新方案](https://www.dosomething.xyz/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93/Hive%20UDF%E4%BD%BF%E7%94%A8%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6%E5%8F%8A%E5%8A%A8%E6%80%81%E6%9B%B4%E6%96%B0%E6%96%B9%E6%A1%88.html) 一文中，针对UDF动态更新的问题，提出解决方案：UDF仅使用业务接口，初始化时动态从位于HDFS的Jar文件中加载业务接口实现类；其中，业务接口及实现类与UDF一一对应。

通常情况下，业务接口仅包含一个方法（Method），方法的定义也比较简单，支持传入若干参数及一个返回值即可。实践过程中，逐渐发现为每一个UDF提供相应的业务接口/实现类的设计有点 **冗余**。开始思考是否可以设计一个公用的业务接口？

不同的业务接口本质上有什么不同呢？这里的不同主要是针对业务接口中的方法来定义的，如下：

1. 方法名称不同；
2. 方法参数个数、类型不同；
3. 方法返回值类型不同；

公用的业务接口使用相同的方法名称是没有问题的，因为具体的实现类不同，方法的计算逻辑也就不同；方法参数及返回值的类型，我们可以使用 **Object** 替代；如此来看，业务接口的不同仅仅体现在方法参数个数不同这一点上面。

UDF是通过我们常说的 **函数** 的形式在Hive/Spark SQL的场景下使用的，根据我们的经验可以知道，实际使用时函数的参数个数不会太多，我们假设：10。

**PS：** 如果对 **10** 这个假设有疑问，大家试想一下见过多少个SQL 函数的参数个数是大于10的呢？

于是，我们设计了UdfMethod，用于替换业务接口，

```java
package com.weibo.dip.udf.common;

import com.google.common.base.Preconditions;
import java.io.IOException;
import java.io.InputStream;
import java.io.SequenceInputStream;
import java.util.List;
import java.util.Objects;
import java.util.Vector;
import org.apache.commons.io.IOUtils;
import org.apache.commons.lang.ArrayUtils;
import org.apache.commons.lang.CharEncoding;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * UdfMethod.
 *
 * @author yurun
 */
public abstract class UdfMethod {
  private static final Logger LOGGER = LoggerFactory.getLogger(UdfMethod.class);

  public Object m0() {
    return null;
  }

  public Object m1(Object p1) {
    return p1;
  }

  public Object m2(Object p1, Object p2) {
    return null;
  }

  public Object m3(Object p1, Object p2, Object p3) {
    return null;
  }

  public Object m4(Object p1, Object p2, Object p3, Object p4) {
    return null;
  }

  public Object m5(Object p1, Object p2, Object p3, Object p4, Object p5) {
    return null;
  }

  public Object m6(Object p1, Object p2, Object p3, Object p4, Object p5, Object p6) {
    return null;
  }

  public Object m7(Object p1, Object p2, Object p3, Object p4, Object p5, Object p6, Object p7) {
    return null;
  }

  public Object m8(
      Object p1, Object p2, Object p3, Object p4, Object p5, Object p6, Object p7, Object p8) {
    return null;
  }

  public Object m9(
      Object p1,
      Object p2,
      Object p3,
      Object p4,
      Object p5,
      Object p6,
      Object p7,
      Object p8,
      Object p9) {
    return null;
  }

  public Object m10(
      Object p1,
      Object p2,
      Object p3,
      Object p4,
      Object p5,
      Object p6,
      Object p7,
      Object p8,
      Object p9,
      Object p10) {
    return null;
  }
}

```

UdfMethod是一个抽象类（抽象类主要用于表示不能直接使用，需要相应的实现类），包含10个方法（方法全部包含默认实现）：m0、m1、...、m10；方法名称中的数字用于表示参数个数，且方法参数及返回值类型均使用 **Object** 表示。

实际使用时仅需要根据具体情况（根据参数个数决定），重写实现类中相应参数个数的方法即可。UdfMethod使用及动态加载过程与业务接口相同。



