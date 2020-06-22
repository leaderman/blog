# 前言

本文以 **Flume 1.9.0** 源码为例，介绍 **TaildirSource** 实现原理及核心代码。

# TaildirSource

TaildirSource，可以支持对多个匹配文件执行 **tailf** 操作；其中，匹配规则可以通过文件名称或路径中的正则表达式表示。整体流程可以划分为以下4个步骤：

1. 配置（configure）
2. 启动（start）
3. 运行（process）
4. 停止（stop）

## 配置（configure）
    
配置过程主要完成配置属性到实例变量的解析。

## 启动（start）

启动过程主要完成以下3个重要实例的初始化：

* reader(ReliableTaildirEventReader)，读取器，“tailf”核心实现，详情见后。
* idleFileChecker(ScheduledExecutorService)，专用线程，以指定的时间周期定时关闭 **空闲** 文件，详情见后。
* positionWriter(ScheduledExecutorService)，专用线程，以指定的时间周期定时将文件读取位置写入指定文件，详情见后。

## 运行（process）

外部线程不断轮询调用process()，读取各个匹配文件新近写入的数据行（tailf）。

## 停止（stop）

* 终止专用线程（idleFileChecker、positionWriter）
* 写出各个文件已读取完成的位置（最近一次）
* 关闭读取器（内部会同时关闭多个匹配文件已打开的输入流）

# 启动（start）

## reader(ReliableTaildirEventReader)

读取器初始化，源码：

```java
    try {
      reader = new ReliableTaildirEventReader.Builder()
          .filePaths(filePaths)
          .headerTable(headerTable)
          .positionFilePath(positionFilePath)
          .skipToEnd(skipToEnd)
          .addByteOffset(byteOffsetHeader)
          .cachePatternMatching(cachePatternMatching)
          .annotateFileName(fileHeader)
          .fileNameHeader(fileHeaderKey)
          .build();
    } catch (IOException e) {
      throw new FlumeException("Error instantiating ReliableTaildirEventReader", e);
    }
```

**ReliableTaildirEventReader** 使用建造者（Builder）模式，build()会调用ReliableTaildirEventReader的内部构造函数完成实例初始化：

```java
    public ReliableTaildirEventReader build() throws IOException {
      return new ReliableTaildirEventReader(filePaths, headerTable, positionFilePath, skipToEnd,
                                            addByteOffset, cachePatternMatching,
                                            annotateFileName, fileNameHeader);
    }
```

```java
  private ReliableTaildirEventReader(Map<String, String> filePaths,
      Table<String, String, String> headerTable, String positionFilePath,
      boolean skipToEnd, boolean addByteOffset, boolean cachePatternMatching,
      boolean annotateFileName, String fileNameHeader) throws IOException {
    ...
    List<TaildirMatcher> taildirCache = Lists.newArrayList();
    for (Entry<String, String> e : filePaths.entrySet()) {
      taildirCache.add(new TaildirMatcher(e.getKey(), e.getValue(), cachePatternMatching));
    }
    
    this.taildirCache = taildirCache;
    ...
    updateTailFiles(skipToEnd);

    ...
    loadPositionFile(positionFilePath);
  }
```

ReliableTaildirEventReader构造函数执行过程中，有3个重要部分需要特殊说明：
1. taildirCache
2. updateTailFiles
3. loadPositionFile。

### taildirCache

**taildirCache** 是一个集合，存储着多个 **TaildirMatcher**。

```java
    List<TaildirMatcher> taildirCache = Lists.newArrayList();
    for (Entry<String, String> e : filePaths.entrySet()) {
      taildirCache.add(new TaildirMatcher(e.getKey(), e.getValue(), cachePatternMatching));
    }
```

每一个 **TaildirMatcher** 对应着配置文件中的一个文件组（file group），如：

```text
a1.sources.r1.filegroups = f1 f2
a1.sources.r1.filegroups.f1 = /var/log/test1/example.log
a1.sources.r1.filegroups.f2 = /var/log/test2/.*log.*

a1.sources.r1.cachePatternMatching = true
```

配置属性含义参考：https://flume.apache.org/FlumeUserGuide.html#taildir-source。

对应着：

```java
new TaildirMatcher("f1", "/var/log/test1/example.log", true)
new TaildirMatcher("f2", "/var/log/test2/.*log.*", true)
```

**TaildirMatcher** 构造函数参数：

key：文件组名称；
value：文件匹配规则；
cachePatternMatching：是否使用缓存；

也就是说，**taildirCache** 实质上就是一个文件组集合，每一个文件组（TaildirMatcher）包含组名称、文件匹配规则、是否使用缓存。**TaildirSource** 使用 **taildirCache**（主要是文件匹配规则） 用于判断需要 **tailf** 哪些文件。

#### TaildirMatcher

TaildirMatcher 的核心作用就是通过设定的文件匹配规则计算出符合条件的文件列表，会涉及3个方法：
* getMatchingFilesNoCache
* sortByLastModifiedTime
* getMatchingFiles

##### getMatchingFilesNoCache

使用文件匹配规则，从父目录中过滤符合条件的文件（List<File>）。

```java
  private List<File> getMatchingFilesNoCache() {
    List<File> result = Lists.newArrayList();
    try (DirectoryStream<Path> stream = Files.newDirectoryStream(parentDir.toPath(), fileFilter)) {
      for (Path entry : stream) {
        result.add(entry.toFile());
      }
    } catch (IOException e) {
      logger.error("I/O exception occurred while listing parent directory. " +
                   "Files already matched will be returned. " + parentDir.toPath(), e);
    }
    return result;
  }
```

以文件匹配规则 **/var/log/test2/.\*log.\*** 为例，父目录（parentDir）为**/var/log/test2/** ，过滤器（fileFilter）实际是一个文件名称的正则（regex，\*.log.\*）匹配：

```java
    String regex = f.getName();
    final PathMatcher matcher = FS.getPathMatcher("regex:" + regex);
    this.fileFilter = new DirectoryStream.Filter<Path>() {
      @Override
      public boolean accept(Path entry) throws IOException {
        return matcher.matches(entry.getFileName()) && !Files.isDirectory(entry);
      }
    };
```

##### sortByLastModifiedTime

根据文件最后修改时间对文件进行排序（升序）。

```java
  private static List<File> sortByLastModifiedTime(List<File> files) {
    final HashMap<File, Long> lastModificationTimes = new HashMap<File, Long>(files.size());
    for (File f: files) {
      lastModificationTimes.put(f, f.lastModified());
    }
    Collections.sort(files, new Comparator<File>() {
      @Override
      public int compare(File o1, File o2) {
        return lastModificationTimes.get(o1).compareTo(lastModificationTimes.get(o2));
      }
    });

    return files;
  }
```


##### getMatchingFiles

**getMatchingFiles** 的本质逻辑比较简单：使用文件匹配规则，从父目录中过滤符合条件的文件，并按照最后修改时间升序排序之后，返回即可。因为需要考虑文件数目较多且需要频繁调用的场景，文件的扫描及过滤可能会遇到性能瓶颈。方法的实现过程引入缓存，会涉及缓存的更新机制，方法的实现略显复杂。

```java
List<File> getMatchingFiles() {
    long now = TimeUnit.SECONDS.toMillis(
        TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis()));
    long currentParentDirMTime = parentDir.lastModified();
    List<File> result;

    // calculate matched files if
    // - we don't want to use cache (recalculate every time) OR
    // - directory was clearly updated after the last check OR
    // - last mtime change wasn't already checked for sure
    //   (system clock hasn't passed that second yet)
    if (!cachePatternMatching ||
        lastSeenParentDirMTime < currentParentDirMTime ||
        !(currentParentDirMTime < lastCheckedTime)) {
      lastMatchedFiles = sortByLastModifiedTime(getMatchingFilesNoCache());
      lastSeenParentDirMTime = currentParentDirMTime;
      lastCheckedTime = now;
    }

    return lastMatchedFiles;
  }
```

**lastMatchedFiles** 中保存着上一次扫描过滤的匹配文件，也就是缓存；大多数时候，目录中的文件迭代（创建新文件或删除新文件）速度相对较慢，扫描时希望使用缓存的中的数据直接返回即可，避免文件系统操作；只有在触发以下3个条件之一时才会触发更新：

1. 不使用缓存；

    通过配置属性 **cachePatternMatching** 指定是否使用缓存，默认值为true，即使用缓存。
    
    ```
    !cachePatternMatching
    ```

2. 当前父目录的最后修改时间（currentParentDirMTime）大于系统上一次记录的最近修改时间（lastSeenParentDirMTime）；

    ```
    long currentParentDirMTime = parentDir.lastModified();
    ...
    lastSeenParentDirMTime < currentParentDirMTime
    ```
    
    即：目录中的文件明确发生过变化。
    
    这里有一个问题需要思考，**lastSeenParentDirMTime < currentParentDirMTime** 可以确切地说明目录中的文件发生过变化；但是目录中的文件发生变化，**lastSeenParentDirMTime < currentParentDirMTime** 一定会成立么？答案：不一定！
    
    假设操作系统以 **秒** 为最小时间粒度去更新文件最后修改时间，如果在1s内，目录内的文件发生变化；getMatchingFiles() 的2次调用也发生在这1s内，那么 getMatchingFiles() 第2次调用时会出现 **currentParentDirMTime == lastSeenParentDirMTime** 的情况。虽然此时 **lastSeenParentDirMTime < currentParentDirMTime** 不成立，但目录中的文件实际是发生过变化的，这种情况需要及时更新缓存。
    
    为解决这个问题，引入第3个判断条件。
    
3. 当前父目录的最后修改时间（currentParentDirMTime）不小于系统上一次记录的最后判断时间（lastCheckedTime）；

    ```
    !(currentParentDirMTime < lastCheckedTime)
    ```
    
    如果父目录的最后修改时间（currentParentDirMTime）不是显式地小于系统上一次记录的最后判断时间（lastCheckedTime），即：lastCheckedTime <= currentParentDirMTime，则不能明确判断时间区间[lastCheckedTime， currentParentDirMTime]内是否有文件发生过变化，为确保安全起见，强制更新缓存。
    
**注**：文件（目录）最后修改时间与系统判断时间来源是不一样的。

### updateTailFiles

**updateTailFiles** 的每一次调用，都会使用 **taildirCache** 中的文件匹配规则，创建或更新文件及其对应的文件对象信息，并返回文件列表（inode）。

文件及其对应的文件对象保存在一个名为 **tailFiles** 的变量中，定义如下：

```java
  private Map<Long, TailFile> tailFiles = Maps.newHashMap();
```

其中，键 为文件inode， 值 为文件对象。

##### 思考

目前的代码版本中，**tailFiles** 是没有任何 **remove** 类操作的；也就是说，如果文件匹配规则会导致不断有新的日志文件需要 **tailf**（不是固定数目的若干个文件），长时间运行的场景下 **tailFiles** 中的元素数目会不断增长，可能触发 **OOM**。

##### 为什么使用 inode？

文件系统中，每一个文件都会对应一个inode，即使文件名称发生变化（rename/mv），该文件的inode仍保持不变。

例如存在一个应用，会将日志输出至“/var/log/app.log”，每小时会迭代生成（rotate）一个新的日志文件，且以小时为日志名称后缀，如：

```text
/var/log/
    app.log
    app.log.2020061700
    app.log.2020061701
    app.log.2020061702
    ....
```

日志文件迭代生成的过程：

1. 应用暂停日志写入；
2. 重命名日志文件，如：app.log -> app.log.2020061703;
3. 创建新的日志文件app.log；
4. 恢复日志写入；

应用日志的匹配规则：/var/log/app.log，即仅匹配文件名称为 **app.log** 的日志文件。

一次读取（tailf）处理完成之后，会记录 **app.log** 的读取位置 **n**。假设，下一次读取尚未开始之前，应用持续有新的日志写入到 **app.log**，且满足日志文件迭代条件，**app.log** 被应用重命名为 **app.log.2020061703**，并创建新的日志文件 app.log 继续写入日志。

下一次读取开始之后：

使用 **文件名称或路径**，**app.log.2020061703** 文件名称不符合匹配规则被过滤，重命名之前写入的文本行会因此全部丢失；**app.log** 虽然可以正常匹配出，但此时 **app.log** 已经是一个新生成的文件，原有系统中记录的 **app.log** 读取位置 **n** 实际已失效（不是同一个文件）。此时，继续使用 **n** 作为 **app.log**的读取位置：

如果读取位置 **n** 小于或等于 **app.log** 的实际长度，直接从 **n** 开始读取，不会出现程序异常，但会导致丢失[0, n)区间的数据，也可能会读取到不完整的文件行（**n** 不一定正好是行的起始处）；

如果读取位置 **n** 大于 **app.log** 的实际长度，直接从 **n** 开始读取则会引发程序异常（通常会提前判断并切换至文件开头读取，即：重置 n 为 0）；

使用 **inode**，**app.log.2020061703** 文件名称虽然发生变化，但对应的 **inode** 却保持不变，系统中记录的读取位置 **n** 可以继续使用，结合文件的实际长度可判断是否有新的数据写入需要读取；新创建的 **app.log** 对应的inode，系统中找不到相关记录，可以安全地认为是一个新生成的文件，直接从 **0** 开始读取即可。

可见，使用文件名称或路径用于生成文件列表或记录，会有丢失数据的风险，应使用 **inode**。

#### TailFile

文件对象使用 **TailFile** 表示，存储着文件的数据（输入）流对象（RandomAccessFile）、路径、inode、读取位置（pos）、上一次更新时间（lastUpdated），以及文件读取过程中需要的缓存区等，如下：

```java
  private RandomAccessFile raf;
  private final String path;
  private final long inode;
  private long pos;
  private long lastUpdated;
  private boolean needTail;
  private final Map<String, String> headers;
  private byte[] buffer;
  private byte[] oldBuffer;
  private int bufferPos;
  private long lineReadPos;
```

#### updateTailFiles工作过程

1. 更新时间戳；

    ```
    updateTime = System.currentTimeMillis();
    ```
    
    每一次调用updateTailFiles()，都会先更新一下时间戳；这个时间戳还会被应用于文件对象（TailFile）记录最后更新时间（TailFile.setLastUpdated()）。
    
2. 创建集合，用于存储需要读取数据的文件（inode）；

    ```
    List<Long> updatedInodes = Lists.newArrayList();
    ```

3. 循环遍历文件组集合（taildirCache）：

    ```
    for (TaildirMatcher taildir : taildirCache) {
        ...      
    }
    ```

4. 循环遍历某个文件组（TaildirMatcher）的文件列表：

    ```
    for (File f : taildir.getMatchingFiles()) {
        ...
    }
    ```
    
    文件组的文件列表通过TaildirMatcher.getMatchingFiles()获取。
    
    对于每一个文件，创建或更新文件对象（TailFile）；

    * 获取文件对应的 **inode**:
    
        ```
        long inode;
        ...
        inode = getInode(f);        
        ```
    * 通过 **inode** 获取对应的文件对象：
    
        ```
        TailFile tf = tailFiles.get(inode);
        ```
    * 如果文件对象为空（null）或者文件对象中的路径与当前文件路径不同；
    
        什么时候会出现 **文件对象中的路径与当前文件路径不同** 的情况？如前所述，**tailFiles** 是没有删除机制的，**inode** 之前对应的文件已被系统删除，且系统生成新的文件时使用了相同的 **inode**（相当于 **inode** 被系统回收再利用）。
        
        这两种情况均表明发现一个新的文件，创建新的文件对象：
        
        ```
        if (tf == null || !tf.getPath().equals(f.getAbsolutePath())) {
          long startPos = skipToEnd ? f.length() : 0;
          tf = openFile(f, headers, inode, startPos);
        }
        ```
        
        其中，文件读取起始位置的设置取决于skipToEnd（默认值为false，表示从文件起始处开始读取)。
        
        文件对象中有一个变量 **needTail**，用于标识该文件是否需要 **tailf**，创建新的文件对象时，该值默认为true。
    
    * 如果文件对象不为空（Null），且文件对象中的路径与当前文件路径相同；
        
        判断文件是否有更新：
        ```
        boolean updated = tf.getLastUpdated() < f.lastModified() || tf.getPos() != f.length();
        ```
        
        文件更新需要满足以下2个条件之一：
        * 文件对象记录的最后更新时间小于文件的最后修改时间；
        * 文件对象记录的已读取位置与文件的长度不相等；
        
        如前所述， **tailFiles** 是没有删除机制的，**inode** 复用可能需要更新相应文件对象内部的状态：
        
        ```
        if (tf.getRaf() == null) {
          tf = openFile(f, headers, inode, tf.getPos());
        }
        if (f.length() < tf.getPos()) {
          logger.info("Pos " + tf.getPos() + " is larger than file size! "
            + "Restarting from pos 0, file: " + tf.getPath() + ", inode: " + inode);
          tf.updatePos(tf.getPath(), inode, 0);
        }
        ```
        
        设置文件对象是否需要 “tailf”，取决于 **updated**：
        
        ```
        tf.setNeedTail(updated);
        ```
    * 更新 **tailFiles**；
    
        ```
        tailFiles.put(inode, tf);
        ```
    * 将 **inode** 加入文件列表；
    
        ```
        updatedInodes.add(inode);
        ```


### loadPositionFile

文件读取位置会被定时的写入一个指定的文件（position file）持久化保存，Flume异常终止或重新启动之后，可以通过该文件的内容，还原文件读取位置：

```
TailFile tf = tailFiles.get(inode);
if (tf != null && tf.updatePos(path, inode, pos)) {
    tailFiles.put(inode, tf);
}
```

其中， **TailFile.updatePos()** 会重置读取位置相关信息：

```
  public boolean updatePos(String path, long inode, long pos) throws IOException {
    if (this.inode == inode && this.path.equals(path)) {
      setPos(pos);
      updateFilePos(pos);
      logger.info("Updated position, file: " + path + ", inode: " + inode + ", pos: " + pos);
      return true;
    }
    return false;
  }
```

```
  public void updateFilePos(long pos) throws IOException {
    raf.seek(pos);
    lineReadPos = pos;
    bufferPos = NEED_READING;
    oldBuffer = new byte[0];
  }
```

## idleFileChecker(ScheduledExecutorService)

**idleFileChecker**，专用线程，以指定的时间周期定时关闭 **空闲** 文件：

```
    idleFileChecker = Executors.newSingleThreadScheduledExecutor(
        new ThreadFactoryBuilder().setNameFormat("idleFileChecker").build());
    idleFileChecker.scheduleWithFixedDelay(new idleFileCheckerRunnable(),
        idleTimeout, checkIdleInterval, TimeUnit.MILLISECONDS);
```

其中，**checkIdleInterval** 用于指定运行周期； **idleTimeout** 用于指定文件空闲超时时间。

工作线程 **idleFileCheckerRunnable**：

```
  private class idleFileCheckerRunnable implements Runnable {
    @Override
    public void run() {
      try {
        long now = System.currentTimeMillis();
        for (TailFile tf : reader.getTailFiles().values()) {
          if (tf.getLastUpdated() + idleTimeout < now && tf.getRaf() != null) {
            idleInodes.add(tf.getInode());
          }
        }
      } catch (Throwable t) {
        logger.error("Uncaught exception in IdleFileChecker thread", t);
        sourceCounter.incrementGenericProcessingFail();
      }
    }
  }
```

空闲文件需要同时满足以下2个条件：

1. （文件对象最后更新时间 + 空闲超时时间）小于 当前时间；

    ```
    tf.getLastUpdated() + idleTimeout < now
    ```

2. 文件对象中的文件输入流已关闭；

    ```
    tf.getRaf() != null
    ```
    
满足条件的空闲文件会被加入一个特定的集合 **idleInodes**，**closeTailFiles()** 也会用到该集合：

```
private List<Long> idleInodes = new CopyOnWriteArrayList<Long>();
```

```
idleInodes.add(tf.getInode());
```

### 思考

**idleFileChecker** 线程仅仅是将空闲文件加入到一个线程安全的集合 **idleInodes**，实际的空闲文件关闭过程是在 **TaildirSource.process()** 每一次调用的末尾进行的：

```
@Override
  public Status process() {
    ...
      closeTailFiles();
    ...
  }
```

```
  private void closeTailFiles() throws IOException, InterruptedException {
    for (long inode : idleInodes) {
      TailFile tf = reader.getTailFiles().get(inode);
      if (tf.getRaf() != null) { // when file has not closed yet
        tailFileProcess(tf, false);
        tf.close();
        logger.info("Closed file: " + tf.getPath() + ", inode: " + inode + ", pos: " + tf.getPos());
      }
    }
    idleInodes.clear();
  }
```

也就是说，空闲文件（idleInodes）的收集（add）与清理（close/clear）是分布于不同的线程中的，而且两者之间没有任何的协调机制。

考虑这样的一种情况：

1. closeTailFiles() 正在运行中；
2. idleFileCheckerRunnable 正好也在不断地将空闲文件加入到 idleInodes；
    
    **idleInodes** 使用 **CopyOnWriteArrayList** 实现，是线程安全的，上述2种操作同时进行完全是允许的，但是根据 CopyOnWriteArrayList 的特性，新写入的空闲文件是不会被 **closeTailFiles** 中的 **for** 循环感知到的；相当于不会对这些新写入的空闲文件执行 close。
    
3. closeTailFiles() 运行到最后一步，执行 idleInodes.clear()；

    这一步会将 2. 中新写入的空闲文件一并删除。
    
如果上述情况以一定的概率不断发生，类似于 2. 中的空闲文件会不会永远也得不到关闭？

## positionWriter(ScheduledExecutorService)

**positionWriter**，专用线程，以指定的时间周期定时将文件读取位置写入指定文件：

```
    positionWriter = Executors.newSingleThreadScheduledExecutor(
        new ThreadFactoryBuilder().setNameFormat("positionWriter").build());
    positionWriter.scheduleWithFixedDelay(new PositionWriterRunnable(),
        writePosInitDelay, writePosInterval, TimeUnit.MILLISECONDS);
```

其中，**writePosInterval** 用于指定写入周期。

工作线程 **PositionWriterRunnable**：

```
  private class PositionWriterRunnable implements Runnable {
    @Override
    public void run() {
      writePosition();
    }
  }
```

**writePosition**：

```
  private List<Long> existingInodes = new CopyOnWriteArrayList<Long>();
```

```
  private void writePosition() {
      ...
      writer = new FileWriter(file);
      if (!existingInodes.isEmpty()) {
        String json = toPosInfoJson();
        writer.write(json);
      }
      ...
  }
```

**existingInodes** 用于表示当前的文件列表，是 **TaildirSource.process()** 运行时负责 **清空** 并 **写入** 数据的。

由上面的代码可以看出，如果 **positionWriter** 运行时，**existingInodes** 中正好保存有文件列表，则会将这些文件的inode、文件路径、读取位置生成一行 **JSON Array** 字符串（toPosInfoJson）写入到指定文件中，生成过程：

```
  private String toPosInfoJson() {
    @SuppressWarnings("rawtypes")
    List<Map> posInfos = Lists.newArrayList();
    for (Long inode : existingInodes) {
      TailFile tf = reader.getTailFiles().get(inode);
      posInfos.add(ImmutableMap.of("inode", inode, "pos", tf.getPos(), "file", tf.getPath()));
    }
    return new Gson().toJson(posInfos);
  }
```

### 思考

**existingInodes** 与 **idleInodes** 存在类似的问题（**writePosition()** 运行时 **existingInodes** 可能永远为空），不再赘述。而且，**positionWriter** 使用定时调度运行的方式，也不能保证及时将文件读取位置持久化。如果调度间隔中应用异常终止且重启之后，数据有重复被读取的可能性。

# 运行（process）

process()由外部线程以循环的方式调用，主要完成以下3个过程：

1. 获取所有文件列表；

    ```
    existingInodes.clear();
    existingInodes.addAll(reader.updateTailFiles());
    ```
    
    文件列表是通过 ReliableTaildirEventReader.updateTailFiles() 获取，实现逻辑参考前文所述。
    
2. 遍历文件列表；

    ```
    for (long inode : existingInodes) {
      TailFile tf = reader.getTailFiles().get(inode);
      if (tf.needTail()) {
        boolean hasMoreLines = tailFileProcess(tf, true);
        if (hasMoreLines) {
          status = Status.READY;
        }
      }
    }
    ```
    
    对于其中的每一个文件，如果判断需要读取（tailf），则调用 tailFileProcess()，详情见后。
    
    tailFileProcess() 返回值为布尔类型，用于标识某个文件是否读取到内容。如果文件列表遍历完成之后，所有的文件都没有读取到内容，**status** 会被设置为 **BACKOFF**，并返回给外部线程，让其 **休息** 一会儿。
 
3. 关闭空闲文件；

    ```
    closeTailFiles();
    ```
    
    closeTailFiles() 实现逻辑参考前文所述。
    
## tailFileProcess

**tailFileProcess()** 用于完成对某个文件的 **tailf** 操作；如果可以读取到内容，返回 **true**；否则，返回 **false**。

整体是一个循环读取的过程：

```
  private boolean tailFileProcess(TailFile tf, boolean backoffWithoutNL)
      throws IOException, InterruptedException {
    long batchCount = 0;
    while (true) {
      ...
      if (events.size() < batchSize) {
        logger.debug("The events taken from " + tf.getPath() + " is less than " + batchSize);
        return false;
      }
      if (++batchCount >= maxBatchCount) {
        logger.debug("The batches read from the same file is larger than " + maxBatchCount );
        return true;
      }
    }
  }
```

每次读取一定行数（batchSize）的内容，然后继续下一次读取，直至累积读取的行数（batchCount）高于系统设置（maxBatchCount），会结束这个文件的读取过程，并返回 true：

```
if (++batchCount >= maxBatchCount) {
  logger.debug("The batches read from the same file is larger than " + maxBatchCount );
  return true;
}
```

期间可能会遇到读取不到内容或读取内容行数太少，会提前结束这个文件的读取过程，并返回 false:

```
if (events.size() < batchSize) {
  logger.debug("The events taken from " + tf.getPath() + " is less than " + batchSize);
  return false;
}
```

对于循环中的每一次读取：

```
reader.setCurrentFile(tf);
List<Event> events = reader.readEvents(batchSize, backoffWithoutNL);
if (events.isEmpty()) {
  return false;
}
...
  reader.commit();
...
```

读取过程涉及3个函数：ReliableTaildirEventReader.setCurrentFile()、ReliableTaildirEventReader.readEvents() 和 ReliableTaildirEventReader.commit()。

1. ReliableTaildirEventReader.setCurrentFile()

    ```
    reader.setCurrentFile(tf);
    ```
    
    设置当前文件的读取， **tf** 就是文件对应的文件对象 **TailFile**。

2. ReliableTaildirEventReader.readEvents()

    readEvents 完成对某个文件一次性最多读取 batchSize 行数据的过程。函数的实现依赖于 **TailFile** 的若干个实例方法，核心调用顺序如下：
    * TailFile.readEvents(int numEvents, boolean backoffWithoutNL, boolean addByteOffset)
    * TailFile.readEvent(boolean backoffWithoutNL, boolean addByteOffset)
    * TailFile.readLine()
    
    简单理解，**readEvents** 用于完成循环读取 **Event** 的逻辑；**readEvent** 用于完成一个 **Event** 的读取；其中，**Event** 是 **Flume** 的概念，相对于一行数据而言，多一些元数据（如：Headers）；**readLine** 用于完成一行数据的读取。
    
    这里重点介绍 **readLine** 的实现过程。
    
    **readLine** 使用2个字节数组：
    
    ```
    private byte[] buffer;
    private byte[] oldBuffer;
    ```
    
    其中，**buffer** 用于读取新数据，相当于读取过程中的一个缓冲区，每次最多读取 **buffer.length** 个字节；**oldBuffer** 用于合并存储已读取到的数据（**buffer**）。
    
    循环执行以下步骤：
    
    a. 判断 **buffer** 中是否有剩余数据；
    
    ```
    if (bufferPos == NEED_READING) {
      ...
    }
    ```
    
    **bufferPos** 用于表示字节数组 **buffer** 下一个可以读取字节数据的游标；值为 **NEED_READING(-1)** 时，表示字节数组中无数据可读取，即数组为空。
    
    如果 **buffer** 中无剩余数据，则执行 b；否则，执行 c；
    
    b. 判断文件是否有新的数据可以读取；
    
    b1. 如果文件有新的数据，则执行读取：
        
    ```
    if (raf.getFilePointer() < raf.length()) {
      readFile();
    }
    ```
        
    读取过程由 **readFile()** 负责：
        
    ```
      private void readFile() throws IOException {
        if ((raf.length() - raf.getFilePointer()) < BUFFER_SIZE) {
          buffer = new byte[(int) (raf.length() - raf.getFilePointer())];
        } else {
          buffer = new byte[BUFFER_SIZE];
        }
        raf.read(buffer, 0, buffer.length);
        bufferPos = 0;
      }
    ```
        
    读取过程会使用一个字节数组 **buffer**，即每次最多读取 buffer.length 个字节：
        
    ```
    raf.read(buffer, 0, buffer.length);
    ```
    
    b2. 如果文件没有新的数据，结束循环，并返回 **lineResult（LineResult，表示一行数据）**；
    
    这一步有一种特殊情况：**oldBuffer** 数据不为空；这表示此时已经读取到一行的部分数据（存储于 **oldBuffer** 中），但尚未读取完成。代码的处理逻辑是将这部分数据直接形成一行数据，且清空 **oldBuffer**：
    
    ```
    if (oldBuffer.length > 0) {
      lineResult = new LineResult(false, oldBuffer);
      oldBuffer = new byte[0];
      setLineReadPos(lineReadPos + lineResult.line.length);
    }
    ```
    
    注意，创建 **LineResult** 实例时，构造函数参数 **lineSepInclude** 为 false，表示这并不是一个完整的数据行。至于是否接收这个不完整的数据，则交由上层处理：
    
    ```
    if (backoffWithoutNL && !line.lineSepInclude) {
      logger.info("Backing off in file without newline: "
          + path + ", inode: " + inode + ", pos: " + raf.getFilePointer());
      updateFilePos(posTmp);
      return null;
    }
    ```
    
    由参数 **backoffWithoutNL**（默认为true，不接受不完整的数据行）与 **line.lineSepInclude**（标识数据行是否完整） 共同决定是否接受不完整的数据行。
    
    也就是说，**oldBuffer** 中数据为空时，返回 null；否则，返回 lineResult 实例。
    
    b1 或 b2 执行完成之后，继续 c；
        
    c. 判断字节数组 **buffer** 中是否包含 **换行符（(byte) 10）**，如果不包含，执行 d；否则，执行 e；
    
    d. 将字节数组 **buffer** 中的数据 **合并** 至 字节数组 **oldBuffer**，清空字节数组 **buffer**，跳转至 a；
        
    ```
    oldBuffer = concatByteArrays(oldBuffer, 0, oldBuffer.length,
                         buffer, bufferPos, buffer.length - bufferPos);
    bufferPos = NEED_READING;
    ```
        
    e. 判断字节数组 **buffer** 中是否包含 **换行符** 是一个循环的过程；
    
    ```
    for (int i = bufferPos; i < buffer.length; i++) {
      if (buffer[i] == BYTE_NL) {
        ...
        
        break;
      }
    }
    ```
    
    找到 **换行符** 位于字节数组中的游标：
    
    ```
    if (buffer[i] == BYTE_NL) {
      ...
      
      break;
    }
    ```
    
    将 **buffer** 中 **换行符** 之前的数据与**oldBuffer** 中的数据 **合并** ，创建 **LineResult** 实例，即一行数据：
    
    ```
    int oldLen = oldBuffer.length;
    // Don't copy last byte(NEW_LINE)
    int lineLen = i - bufferPos;
    
    ...

    lineResult = new LineResult(true,
        concatByteArrays(oldBuffer, 0, oldLen, buffer, bufferPos, lineLen));
    setLineReadPos(lineReadPos + (oldBuffer.length + (i - bufferPos + 1)));
    ```
    
    其中，**setLineReadPos()** 用于记录当前文件下一次读取时的起始游标。
    
    清空 **oldBuffer**；
    
    ```
    oldBuffer = new byte[0];
    ```
    
    更新 **bufferPos**；
    
    ```
    if (i + 1 < buffer.length) {
      bufferPos = i + 1;
    } else {
      bufferPos = NEED_READING;
    }
    ```
    
    注意，**buffer** 中可能会有剩余数据，即： **换行符**之后至字节数组末尾的数据。这部分数据会在下一次 **readLine** 时被合并至 **oldBuffer**。
    
    最后，返回 **lineResult**（结束循环）。
    
3. ReliableTaildirEventReader.commit()

    ReliableTaildirEventReader.readEvents()运行完成之后，获取一批数据行 **events**：
    
    ```
    List<Event> events = reader.readEvents(batchSize, backoffWithoutNL);
    if (events.isEmpty()) {
      return false;
    }
    ```
    
    如果 **events** 不为空，则将 **events** 交付至 **Channel**（Channel不在本文讨论范围内）；如果交互过程无异常，则执行 **commit（提交）**，表示文件指定游标之前的数据已安全的读取并交付完成：
    
    ```
    try {
      getChannelProcessor().processEventBatch(events);
      reader.commit();
    } catch (ChannelException ex) {
      ...
    }
    ```
    
    **commit** 代码实现很简单：
    
    ```
    long pos = currentFile.getLineReadPos();
    
    currentFile.setPos(pos);
    currentFile.setLastUpdated(updateTime);
    ```
    
    将文件下一个读取游标（pos）及更新时间戳（updateTime）保存至文件对象（currentFile）。如前方所述，**positionWriter** 会定时将 **pos** 持久化到指定文件中。
    
# 停止（stop）

1. 终止专用线程（idleFileChecker、positionWriter）

    ```
    ExecutorService[] services = {idleFileChecker, positionWriter};
    for (ExecutorService service : services) {
      service.shutdown();
      if (!service.awaitTermination(1, TimeUnit.SECONDS)) {
        service.shutdownNow();
      }
    }
    ```
    
2. 写出各个文件已读取完成的位置（最近一次）

    ```
    writePosition();
    ```
    
    **TaildirSource** 停止前，显式调用 **writePosition()**，确保将各个文件已读取完成的位置持久化。
    
3. 关闭读取器（内部会同时关闭多个匹配文件已打开的输入流）

    ```
    reader.close();
    ```
    
    ```
    for (TailFile tf : tailFiles.values()) {
      if (tf.getRaf() != null) tf.getRaf().close();
    }
    ```
    
# 结语

**TaildirSource** 的实现逻辑并不是简单的文本文件读取，类似于 inode 的使用、目录文件列表缓存、文件更新状态判断等方法是非常值得借鉴的。当然，本文中提及的 tailFiles、idleFileChecker、positionWriter等相关问题也是需要继续思考的。


