---
title: Storm HDFS Integration
layout: documentation
documentation: true
---

Storm组件和 HDFS 文件系统交互.


## Usage

以下示例将pipe（“|”）分隔的文件写入HDFS路径hdfs://localhost:54310/foo。
每1000个 tuple 之后，它将同步文件系统，使该数据对其他HDFS客户端可见。当它们达到5MB大小时，它将旋转文件。

```java
// use "|" instead of "," for field delimiter
RecordFormat format = new DelimitedRecordFormat()
        .withFieldDelimiter("|");

// sync the filesystem after every 1k tuples
SyncPolicy syncPolicy = new CountSyncPolicy(1000);

// rotate files when they reach 5MB
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);

FileNameFormat fileNameFormat = new DefaultFileNameFormat()
        .withPath("/foo/");

HdfsBolt bolt = new HdfsBolt()
        .withFsUrl("hdfs://localhost:54310")
        .withFileNameFormat(fileNameFormat)
        .withRecordFormat(format)
        .withRotationPolicy(rotationPolicy)
        .withSyncPolicy(syncPolicy);
```

### Packaging a Topology

当打包你的 topology（拓扑）代码的时候，要使用[maven-shade-plugin]() 插件，不要使用[maven-assembly-plugin]()插件.

shade 插件提供了合并 Jar manifest entries 的功能，hadoop client 可以用来做URL scheme 方案.

如果你经历了类似于下面的错误：

```
java.lang.RuntimeException: Error preparing HdfsBolt: No FileSystem for scheme: hdfs
```

这表明你的 topology jar没有正确的打包.

如果你使用maven来创建你的topology jar，你应该使用下面  `maven-shade-plugin` 配置来创建你的 topology jar:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>1.4</version>
    <configuration>
        <createDependencyReducedPom>true</createDependencyReducedPom>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass></mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>

```

### Specifying a Hadoop Version

默认情况下，storm-hdfs使用下面的Hadoop依赖.

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.2.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-hdfs</artifactId>
    <version>2.2.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

如果你使用的Hadoop版本不同，你可以移除storm-hdfs中 Hadoop依赖，并添加你自己的依赖到你的 pom中.

Hadoop客户端版本不兼容，错误如：

```
com.google.protobuf.InvalidProtocolBufferException: Protocol message contained an invalid tag (zero)
```

## Customization

### Record Formats（记录格式化）
记录格式化可以通过提供的`org.apache.storm.hdfs.format.RecordFormat`接口来控制：

```java
public interface RecordFormat extends Serializable {
    byte[] format(Tuple tuple);
}
```
提供的`org.apache.storm.hdfs.format.DelimitedRecordFormat`实现可以生成如 CSV 和 制表符分隔 的文件.
T


### File Naming
文件名称可以通过提供的`org.apache.storm.hdfs.format.FileNameFormat`接口来控制：

```java
public interface FileNameFormat extends Serializable {
    void prepare(Map conf, TopologyContext topologyContext);
    String getName(long rotation, long timeStamp);
    String getPath();
}
```

提供的 `org.apache.storm.hdfs.format.DefaultFileNameFormat` 创建的文件名称格式如下：

     {prefix}{componentId}-{taskId}-{rotationNum}-{timestamp}{extension}

例如:

     MyBolt-5-7-1390579837830.txt

默认情况下，前缀是空的，扩展标识是".txt".



### Sync Policies
同步策略允许你将 buffered data 缓冲到底层文件系统（从而client可以读取数据），通过实现`org.apache.storm.hdfs.sync.SyncPolicy` 接口：

```java
public interface SyncPolicy extends Serializable {
    boolean mark(Tuple tuple, long offset);
    void reset();
}
```
 `HdfsBolt`  会为每个要处理的 tuple 调用 `mark()`方法.返回 `true` 会触发 `HdfsBolt`执行同步/刷新，之后会调用`reset()`方法.
 
 `org.apache.storm.hdfs.sync.CountSyncPolicy`类可以简单的触发同步，当一定数量的tuple执行完成后.

### File Rotation Policies

类似于同步策略,文件反转策略允许你通过 `org.apache.storm.hdfs.rotation.FileRotation` 接口来控制数据文件反转.

```java
public interface FileRotationPolicy extends Serializable {
    boolean mark(Tuple tuple, long offset);
    void reset();
}
``` 

`org.apache.storm.hdfs.rotation.FileSizeRotationPolicy`实现允许数据文件达到指定的文件大小后，触发文件反转.

```java
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);
```

### File Rotation Actions

HDFS bolt 和 Trident State实现允许你注册任意数量的`RotationAction`s.
`RotationAction`s要做的就是提供一个hook，当文件反转后执行一些操作。例如，移动一个文件到不同的路径下，或者重命名.


```java
public interface RotationAction extends Serializable {
    void execute(FileSystem fileSystem, Path filePath) throws IOException;
}
```

Storm-HDFS 包括一个简单的操作，反转后移动一个文件：

```java
public class MoveFileAction implements RotationAction {
    private static final Logger LOG = LoggerFactory.getLogger(MoveFileAction.class);

    private String destination;

    public MoveFileAction withDestination(String destDir){
        destination = destDir;
        return this;
    }

    @Override
    public void execute(FileSystem fileSystem, Path filePath) throws IOException {
        Path destPath = new Path(destination, filePath.getName());
        LOG.info("Moving file {} to {}", filePath, destPath);
        boolean success = fileSystem.rename(filePath, destPath);
        return;
    }
}
```

如果你使用 Trident，并且是有序的文件，你可以像下面这样使用：

```java
        HdfsState.Options seqOpts = new HdfsState.SequenceFileOptions()
                .withFileNameFormat(fileNameFormat)
                .withSequenceFormat(new DefaultSequenceFormat("key", "data"))
                .withRotationPolicy(rotationPolicy)
                .withFsUrl("hdfs://localhost:54310")
                .addRotationAction(new MoveFileAction().withDestination("/dest2/"));
```


## Support for HDFS Sequence Files

`org.apache.storm.hdfs.bolt.SequenceFileBolt`类允许你写入storm data 到连续的HDFS文件中：

```java
        // sync the filesystem after every 1k tuples
        SyncPolicy syncPolicy = new CountSyncPolicy(1000);

        // rotate files when they reach 5MB
        FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);

        FileNameFormat fileNameFormat = new DefaultFileNameFormat()
                .withExtension(".seq")
                .withPath("/data/");

        // create sequence format instance.
        DefaultSequenceFormat format = new DefaultSequenceFormat("timestamp", "sentence");

        SequenceFileBolt bolt = new SequenceFileBolt()
                .withFsUrl("hdfs://localhost:54310")
                .withFileNameFormat(fileNameFormat)
                .withSequenceFormat(format)
                .withRotationPolicy(rotationPolicy)
                .withSyncPolicy(syncPolicy)
                .withCompressionType(SequenceFile.CompressionType.RECORD)
                .withCompressionCodec("deflate");
```

`SequenceFileBolt` 需要你提供一个 `org.apache.storm.hdfs.bolt.format.SequenceFormat`，用来映射 tuples到 key/value pairs。


```java
public interface SequenceFormat extends Serializable {
    Class keyClass();
    Class valueClass();

    Writable key(Tuple tuple);
    Writable value(Tuple tuple);
}
```

## Trident API

storm-hdfs 还包括一个 Trident `state` 实现，用于写入数据到HDFS，API类似于 bolts.

 ```java
         Fields hdfsFields = new Fields("field1", "field2");

         FileNameFormat fileNameFormat = new DefaultFileNameFormat()
                 .withPath("/trident")
                 .withPrefix("trident")
                 .withExtension(".txt");

         RecordFormat recordFormat = new DelimitedRecordFormat()
                 .withFields(hdfsFields);

         FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, FileSizeRotationPolicy.Units.MB);

        HdfsState.Options options = new HdfsState.HdfsFileOptions()
                .withFileNameFormat(fileNameFormat)
                .withRecordFormat(recordFormat)
                .withRotationPolicy(rotationPolicy)
                .withFsUrl("hdfs://localhost:54310");

         StateFactory factory = new HdfsStateFactory().withOptions(options);

         TridentState state = stream
                 .partitionPersist(factory, hdfsFields, new HdfsUpdater(), new Fields());
 ```

要使用序列文件`State`实现，请使用`HdfsState.SequenceFileOptions`：

 ```java
        HdfsState.Options seqOpts = new HdfsState.SequenceFileOptions()
                .withFileNameFormat(fileNameFormat)
                .withSequenceFormat(new DefaultSequenceFormat("key", "data"))
                .withRotationPolicy(rotationPolicy)
                .withFsUrl("hdfs://localhost:54310")
                .addRotationAction(new MoveFileAction().toDestination("/dest2/"));
```

##Working with Secure HDFS

如果您的 topology（拓扑）将与安全的HDFS进行交互，则您的 bolts/states 需要通过NameNode进行身份验证。我们
目前有2个选项支持：


### Using HDFS delegation tokens 
您的管理员可以配置nimbus来代表拓扑提交者用户自动获取授权令牌。
nimbus需要从以下配置开始：

nimbus.autocredential.plugins.classes : ["org.apache.storm.hdfs.common.security.AutoHDFS"] 
nimbus.credential.renewers.classes : ["org.apache.storm.hdfs.common.security.AutoHDFS"] 
hdfs.keytab.file: "/path/to/keytab/on/nimbus" (hdfs 超级管理员可以代理其他用户.)
hdfs.kerberos.principal: "superuser@EXAMPLE.com" 
nimbus.credential.renewers.freq.secs : 82800 (23 小时, hdfs tokens 需要每24个小时更新一次.)
topology.hdfs.uri:"hdfs://host:port" (可选的配置, 默认情况下，我们会在core-site.xml 文件中指定 "fs.defaultFS" 属性)

你的topology 配置应该包括：
topology.auto-credentials :["org.apache.storm.hdfs.common.security.AutoHDFS"] 

如果nimbus没有上述配置，您需要添加它，然后重新启动它。确保hadoop配置
文件（core-site.xml和hdfs-site.xml）以及具有所有依赖项的storm-hdfs jar都存在于nimbus的类路径中。
Nimbus将使用配置文件中指定的 keytab 和主体对 Namenode 进行身份验证。从那时起每一个
topology 提交，nimbus将模拟拓扑提交者用户并代表代理令牌
topology 提交者用户。如果通过将topology.auto-credentials设置为AutoHDFS启动 topology（拓扑），nimbus将推送
将所有的工作人员的代理令牌用于您的 topology（拓扑），并且hdfs bolt / state将使用namenode进行身份验证
这些令牌。

由于nimbus模拟topology（拓扑）提交者用户，您需要确保hdfs.kerberos.principal中指定的用户
具有代表其他用户获取令牌的权限。要实现这一点，您需要遵循配置指导
列在此链接上：
http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/Superusers.html

你可以看这里如何配置安全的HDFS: http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SecureMode.html.

### Using keytabs on all worker hosts
如果您已将hdfs用户的 keytab 文件分发给所有潜在的worker ，那么可以使用此方法。你应该指定一个
使用HdfsBolt / State.withconfigKey（“somekey”）方法的hdfs配置密钥，该密钥的值映射应具有以下2个属性:

hdfs.keytab.file: "/path/to/keytab/"
hdfs.kerberos.principal: "user@EXAMPLE.com"

在workers 上，bolt/Ttrident-staet code 将使用配置中提供的主体的keytab文件进行认证
Namenode。这种方法很危险，因为您需要确保所有 worker 的keytab文件位于同一位置，您需要
在集群中启动新主机时记住这一点.

