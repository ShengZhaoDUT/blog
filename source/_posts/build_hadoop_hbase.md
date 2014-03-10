title: 在Hadoop上搭建HBase
date: 2014-02-19 10:53:58
categories: nosql
tags: [nosql, tech]
---

安装Hbase需要HDFS的支持，因此在搭建HBase集群首先应该搭建Hadoop集群。
两者的版本需要匹配，才能正常通信。
可以在Hadoop和Hbase的[选择]（http://hbase.apache.org/book.html#hadoop）中具体查看。


安装环境：

Hadoop-2.2.0，Hadoop Yarn稳定版

Hbase-0.96.1,与上述Hadoop版本匹配

1.安装Hadoop，在网上可以任意搜到相关教程，在配置中需要特别关注core-site.xml中fs.default.name配置项，再Hbase中会使用到该配置项。

2.安装Hbase，首先下载Hbase相关版本并解压

* 编辑/hbase/conf/hbase-env.sh
```Java
export HBASE_OPTS="$HBASE_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode"
export JAVA_HOME=/usr/lib/jvm/java-6-openjdk-amd64
export HBASE_MANAGES_ZK=true
export HBASE_HOME=自定义的Hbase的路径
```
* 编辑/hbase/conf/hbase-site.sh
```Java
<configuration>
<property>
 <name>hbase.rootdir</name>
 <value>hdfs://hadoop-master:9000/hbase(与Hadoop中HDFS的端口号相关，注意匹配)</value>
</property>
<property>
 <name>hbase.cluster.distributed</name>
 <value>true</value>
</property>
<property>
 <name>hbase.master</name>
 <value>192.168.7.115:60000</value>
</property>
<property>
 <name>hbase.zookeeper.quorum</name>
 <value>192.168.7.105,192.168.7.102,192.168.7.104（slave节点）</value>
</property>
</configuration>
```

* 编辑/hbase/conf/regionservers
```Java
192.168.7.105
192.168.7.104
192.168.7.102
```

* 然后讲所有配置拷贝到各个slave节点

* 先启动hdfs，再启动hbase
