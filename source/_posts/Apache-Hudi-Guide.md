---
title: 【技术笔记】Apache Hudi + Sedona 地学数据湖部署
date: 2026-03-01 15:52:35
tags:
  - 技术笔记
  - Hudi
  - Spark
  - Hadoop
  - Sedona
---
本文详细记录基于 Apache Hudi 和 Sedona 的地学数据湖开发环境配置。
针对课题组服务器的开发环境，涉及一些特殊问题。
主要包括配置各种jar包、Pyspark作为开发语言可能遇到的一些麻烦。

## 环境信息

| 组件     | 版本           |
| -------- | -------------- |
| Hadoop   | 3.3.6          |
| Spark    | 3.4.3          |
| Sedona   | 1.4.1          |
| Hudi     | 1.0.0          |
| Linux    | Ubuntu 18.04.6 |
| 开发语言 | PySpark        |

## 整体部署方案

以两个节点为例，额外的节点可以参考一样的方案

- **VM_A** : Spark Master + HDFS NameNode + DataNode
- **VM_B** : Spark Worker + HDFS DataNode
- 所有服务以 `hadoop` 用户身份运行

课题组服务器的实际环境是物理机中建多个虚拟机
同一个物理机之间的网络可以用内部的IP直接连接，不用公网IP

---

## 第一阶段：基础系统配置

### 系统更新（所有节点）

```bash
sudo apt update
sudo apt upgrade -y
```

### 安装 Java

```bash
# 安装 OpenJDK 11
sudo apt install openjdk-11-jdk -y

# 验证安装
java -version
javac -version
```

### 配置 JAVA_HOME 环境变量

编辑 `~/.bashrc` 或 `/etc/environment`：

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
```

运行以下命令生效：

```bash
source ~/.bashrc  # 或 source /etc/environment
```

### 安装 Scala

实际采用了2.12版本，如果是其他版本后续的各种包需要改为对应版本

```bash
sudo apt install scala -y
scala -version
```

### 安装 Python 3 和 Pip

这里注意pyspark的版本需要和spark集群对应（也影响后续sedona的版本）

```bash
sudo apt install python3 python3-pip -y
pip3 install pyspark
```

### 配置 SSH 免密登录

**在 VM_A 上生成 SSH 密钥对：**

```bash
ssh-keygen -t rsa  # 一直按回车，不设置密码
```

**配置本地免密登录：**

```bash
ssh-copy-id hadoop@localhost
```

**配置到远程节点的免密登录：**

```bash
ssh-copy-id hadoop@192.168.1.101  # 复制 SSH 公钥到 VM_B
```

**验证免密登录：**

```bash
ssh 192.168.1.100  # 测试本地
ssh 192.168.1.101   # 测试远程
```

### 配置 /etc/hosts（所有节点）

在所有节点的 `/etc/hosts` 中添加：

```
192.168.1.100 master-node vm-a
192.168.1.101  worker-node vm-b
```

---

## 第二阶段：部署 HDFS 分布式文件系统

### 下载和安装 Hadoop

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzf hadoop-3.3.6.tar.gz
sudo mv hadoop-3.3.6 /opt/hadoop
sudo chown -R hadoop:hadoop /opt/hadoop
```

### 配置 Hadoop 环境变量（所有节点，hadoop 用户）

编辑 `~/.bashrc`：

```bash
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP_HOME}/lib/native
export HADOOP_OPTS="-Djava.library.path=${HADOOP_HOME}/lib/native"
```

### 创建 HDFS 数据目录

**在 VM_A 上：**

```bash
sudo mkdir -p /home/hadoop/hadoop_data/namenode_metadata
sudo mkdir -p /home/hadoop/hadoop_data/datanode_storage
sudo mkdir -p /home/hadoop/hadoop_tmp_data
sudo chown -R hadoop:hadoop /home/hadoop/hadoop_data
sudo chown -R hadoop:hadoop /home/hadoop/hadoop_tmp_data
```

**在 VM_B 上：**

```bash
sudo mkdir -p /home/hadoop/hadoop_data/datanode_storage
sudo mkdir -p /home/hadoop/hadoop_tmp_data
sudo chown -R hadoop:hadoop /home/hadoop/hadoop_data
sudo chown -R hadoop:hadoop /home/hadoop/hadoop_tmp_data
```

### 配置 Hadoop 配置文件（VM_A）

编辑 `/opt/hadoop/etc/hadoop/hadoop-env.sh`：

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HDFS_NAMENODE_USER="hadoop"
export HDFS_DATANODE_USER="hadoop"
export HDFS_SECONDARYNAMENODE_USER="hadoop"
```

编辑 `/opt/hadoop/etc/hadoop/core-site.xml`：

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.1.100:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:///home/hadoop/hadoop_tmp_data</value>
    </property>
</configuration>
```

编辑 `/opt/hadoop/etc/hadoop/hdfs-site.xml`：

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///home/hadoop/hadoop_data/namenode_metadata</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///home/hadoop/hadoop_data/datanode_storage</value>
    </property>
    <property>
        <name>dfs.namenode.http-address</name>
        <value>192.168.1.100:9870</value>
    </property>
    <property>
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>
</configuration>
```

编辑 `/opt/hadoop/etc/hadoop/workers`：

```
192.168.1.100
192.168.1.101
```

### 分发 Hadoop 配置到 VM_B

```bash
# 在 VM_A 上，hadoop 用户身份执行
scp -r /opt/hadoop/etc/hadoop/* hadoop@192.168.1.101:/opt/hadoop/etc/hadoop/
```

### 格式化 HDFS（仅 VM_A，一次性）

```bash
# 在 VM_A 上，hadoop 用户身份执行
/opt/hadoop/bin/hdfs namenode -format
```

### 启动 HDFS 集群

```bash
# 在 VM_A 上，hadoop 用户身份执行
/opt/hadoop/sbin/start-dfs.sh
```

### 验证 HDFS

**检查进程：**

```bash
jps  # 应看到 NameNode 和 DataNode
```

**访问 Web UI：**访问 `http://192.168.1.100:9870`

**测试文件操作：**

```bash
/opt/hadoop/bin/hdfs dfs -mkdir /test_dir
/opt/hadoop/bin/hdfs dfs -ls /
/opt/hadoop/bin/hdfs dfs -put /opt/hadoop/README.txt /test_dir/
/opt/hadoop/bin/hdfs dfs -cat /test_dir/README.txt
```

---

## 第三阶段：部署 Apache Spark

### 下载和安装 Spark（VM_A）

```bash
wget https://dlcdn.apache.org/spark/spark-3.4.3/spark-3.4.3-bin-hadoop3.tgz
tar -xzf spark-3.4.3-bin-hadoop3.tgz
sudo mv spark-3.4.3-bin-hadoop3 /opt/spark
sudo chown -R hadoop:hadoop /opt/spark
```

### 配置 Spark 环境变量（所有节点）

编辑 `~/.bashrc`：

```bash
export SPARK_HOME=/opt/spark
export PATH=$SPARK_HOME/bin:$SPARK_HOME/sbin:$PATH
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export PYSPARK_PYTHON=python3
```

### 配置 Spark（VM_A）

复制模板文件：

```bash
cd /opt/spark/conf
cp spark-env.sh.template spark-env.sh
cp workers.template workers
```

编辑 `spark-env.sh`：

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export SPARK_MASTER_HOST='192.168.1.100'
export SPARK_MASTER_PORT=7077
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
```

编辑 `workers`：

```
192.168.1.100
192.168.1.101
```

### 分发 Spark 到 VM_B

```bash
# 在 VM_A 上，hadoop 用户身份执行
scp -r /opt/spark/* hadoop@192.168.1.101:/opt/spark/
```

### 启动 Spark 集群

```bash
# 在 VM_A 上，hadoop 用户身份执行
/opt/spark/sbin/start-master.sh
/opt/spark/sbin/start-workers.sh
```

### 验证 Spark

**访问 Web UI：**访问 `http://192.168.1.100:8080`

**检查进程：**

```bash
jps  # 应看到 Master 和 Worker
```

**测试 PySpark：**

```bash
pyspark --master spark://192.168.1.100:7077
```

在 PySpark shell 中：

```python
data = list(range(1000))
rdd = spark.sparkContext.parallelize(data)
print(rdd.count())  # 应输出 1000
spark.stop()
```

---

## 第四阶段：配置 Apache Hudi 和 Sedona

这部分是手动配置依赖包的做法，需要自行确保各个包的版本匹配

### 下载依赖包

需要以下 JAR 包：

```
sedona-spark-shaded-3.4_2.12-1.4.1.jar
hudi-spark3.4-bundle_2.12-1.0.0.jar
geotools-wrapper-1.4.0-28.2.jar
```

建议放在 Spark home 目录或统一的包管理位置。

### 启动 PySpark with Hudi & Sedona

```bash
pyspark --master spark://192.168.1.100:7077 \
  --packages org.apache.hudi:hudi-spark3.4-bundle_2.12:1.0.0 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
  --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
  --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'
```

### 或使用本地 JAR

```bash
pyspark --master spark://192.168.1.100:7077 \
  --jars /path/to/hudi-spark3.4-bundle_2.12-1.0.0.jar,/path/to/sedona-spark-shaded-3.4_2.12-1.4.1.jar \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
  --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
  --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'
```

### 提交 Spark 作业

```bash
spark-submit --master spark://192.168.1.100:7077 \
  --packages org.apache.hudi:hudi-spark3.4-bundle_2.12:1.0.0 \
  --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
  your_script.py
```

---

# 环境相关使用方案

## WebUI相关

在校园网内的集群，外部可能不能直接访问HDFS和Spark的WebUI

相对简单的方案是ssh转发端口到服务器配置过密钥的设备本地

在本地cmd执行

```bash
ssh -i D:\key -L 8080:192.168.1.100:8080 root@GATEWAY_IP -p 21040

ssh -i D:\key -L 9870:192.168.1.100:9870  root@GATEWAY_IP -p 21040
```

之后可以通过本地端口访问

hdfs的WebUI可以通过可视化的界面删除一些实验后不需要的大数据文件

## 集群重启

Hadoop：/opt/hadoop/sbin

Spark: /opt/spark/sbin

在里面找相关的脚本即可

比如关闭Spark集群可以分别执行stop-workers.sh和stop-master.sh

## GeoMesa

论文中做了GeoMesa的对比试验，相关环境配置也有一定麻烦

Hadoop和Spark集群基础不变

GeoMesa对于一些jar包的版本依赖不同
最终用了下面两个包可以对上目前的Hadoop和Spark环境

/root/geomesa-gt-spark-runtime_2.12-5.1.0.jar

/root/geomesa-fs-spark-runtime_2.12-5.1.0.jar

另外注意python虚拟环境中的版本也要对上，Sedona可能要用1.8.1（Hudi环境用的1.4.1）

## Log相关

spark集群运行的时候会有很多log，建议都采取下述方法打印到log file里方便查阅，也不会在terminal产生过多输出

```bash
spark-submit ......  >> log_route/log_file_name.log 2>&1 
```
