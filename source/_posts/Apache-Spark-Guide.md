---
title: 【技术笔记】Apache Spark 和 Apache Sedona 
date: 2025-10-12 15:48:13
tags: 
  - 技术笔记
  - Spark
  - Sedona
  - GIS
---
本文记录如何快速配置 Apache Spark 和 Apache Sedona，包括环境配置和简单的示例

## 环境配置

### 前置要求

- **Java**: JDK 8 或更高版本
- **Python**: 3.7+
- **操作系统**: Linux

### 1. 安装 Java

```bash
# Linux (Ubuntu/Debian)
sudo apt install openjdk-11-jdk

# 验证安装
java -version
```

设置 `JAVA_HOME` 环境变量：

```bash
# 查找 Java 安装路径
which java              # Linux

# 添加到 ~/.bashrc
export JAVA_HOME=/path/to/java
export PATH=$JAVA_HOME/bin:$PATH
```

### 2. 安装 Spark

```bash
# 下载 Spark（选择 Hadoop 3.3 版本）
wget https://archive.apache.org/dist/spark/spark-3.5.0/spark-3.5.0-bin-hadoop3.tgz

# 解压
tar -xzf spark-3.5.0-bin-hadoop3.tgz
mv spark-3.5.0-bin-hadoop3 ~/spark

# 设置环境变量
export SPARK_HOME=~/spark
export PATH=$SPARK_HOME/bin:$PATH

# 验证安装
spark-shell --version
```

### 3. 使用 Python PySpark

```bash
# 通过 pip 安装
pip install pyspark

# 或指定版本
pip install pyspark==3.5.0
```

### 4. 安装 Apache Sedona

```bash
# 安装 Sedona Python 包
pip install apache-sedona

# 如果需要空间索引支持
pip install apache-sedona[extra]
```

### 5. 验证安装

创建测试脚本 `test_spark.py`：

```python
from pyspark.sql import SparkSession

# 创建 Spark 会话
spark = SparkSession.builder \
    .appName("test-spark") \
    .master("local[4]") \
    .getOrCreate()

# 简单测试
df = spark.range(10)
df.show()
```

运行测试：

```bash
python test_spark.py
```

### 6. 配置 Sedona（可选）

如果需要空间数据处理，创建 `test_sedona.py`：

```python
from pyspark.sql import SparkSession
import sedona.spark.sedona_spark_jars

# 使用 Sedona 配置创建 Spark 会话
spark = SparkSession.builder \
    .appName("sedona-test") \
    .master("local[4]") \
    .config("spark.jars.packages", 
            "org.apache.sedona:sedona-spark-shaded_2.12:1.5.1") \
    .config("spark.sql.extensions",
            "org.apache.sedona.sql.UDTFRegistrator") \
    .getOrCreate()

print("✓ Sedona 配置成功！")
```

---

## 简要案例

### 案例 1：基础 DataFrame 操作

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, avg

# 创建 Spark 会话
spark = SparkSession.builder.appName("spark-demo").master("local[4]").getOrCreate()

# 创建示例数据
data = [
    ("Alice", 2022, 50000),
    ("Bob", 2022, 60000),
    ("Alice", 2023, 55000),
    ("Bob", 2023, 65000),
]

columns = ["Name", "Year", "Attribute"]
df = spark.createDataFrame(data, columns)

# 基础操作
df.show()                                    # 显示数据
df.filter(col("Attribute") > 55000).show()       # 条件过滤
df.groupBy("Name").agg(avg("Attribute")).show()  # 按Name分组

# 结果转为 Pandas
result_pdf = df.toPandas()
print(result_pdf)
```

**输出**：

```
+-----+----+-----------+
| Name|Year|Attribute  |
+-----+----+-----------+
|Alice|2022|      50000|
|  Bob|2022|      60000|
|Alice|2023|      55000|
|  Bob|2023|      65000|
+-----+----+-----------+

+-----+-----------+
| Name|avg(Attribute)|
+-----+-----------+
|Alice|      52500|
|  Bob|      62500|
+-----+-----------+

     Name  Year  Attribute
0   Alice  2022       50000
1     Bob  2022       60000
2   Alice  2023       55000
3     Bob  2023       65000
```

### 案例 2：Sedona 地理空间查询

```python
from pyspark.sql import SparkSession
from sedona.sql import ST_Constructors as ST
from sedona.sql import ST_Predicates as ST_Pred
from sedona.sql import ST_Transform as ST_Trans

# 配置 Spark 会话
spark = SparkSession.builder \
    .appName("sedona-geo-demo") \
    .config("spark.jars.packages", "org.apache.sedona:sedona-spark-shaded_2.12:1.5.1") \
    .config("spark.sql.extensions", "org.apache.sedona.sql.UDTFRegistrator") \
    .getOrCreate()

# 创建地理空间数据
cities_data = [
    ("北京", "POINT(116.4074 39.9042)"),
    ("上海", "POINT(121.4737 31.2304)"),
    ("深圳", "POINT(114.0579 22.5431)"),
    ("杭州", "POINT(120.1551 30.2741)"),
]

cities_df = spark.createDataFrame(cities_data, ["城市", "坐标"])

# 创建 Geometry 列
cities_df = cities_df.withColumn("geom", ST.ST_GeomFromText("坐标"))

# 查询：找出在特定矩形范围内的城市
# 范围：115°E-122°E, 22°N-40°N
envelope = "POLYGON((115 22, 115 40, 122 40, 122 22, 115 22))"

cities_df.createOrReplaceTempView("cities")
spark.sql(f"CREATE OR REPLACE TEMP VIEW envelope AS SELECT ST_GeomFromText('{envelope}') as geom")

# 执行空间查询
result = spark.sql("""
    SELECT cities.城市, cities.坐标
    FROM cities, envelope
    WHERE ST_Contains(envelope.geom, cities.geom)
""")

result.show()
```

**输出**：

```
+----+------------------+
|城市|      坐标        |
+----+------------------+
|北京|POINT(116.4074..)|
|上海|POINT(121.4737..)|
|深圳|POINT(114.0579..)|
|杭州|POINT(120.1551..)|
+----+------------------+
```

---

## 分布式 Spark 集群部署 + 远程 Jupyter 调用方案

如果你需要部署一个分布式 Spark 集群，并通过远程 Jupyter Notebook 调用，本章提供完整的配置方案。

### 1. 集群部署（Spark Standalone 模式）

#### 1.1 在 Master 节点（VM_A）部署

**下载和安装 Spark：**

```bash
# 以 hadoop 用户登录
sudo su - hadoop

# 下载 Spark（选择与 Hadoop 3.3+ 兼容的版本）
cd ~
wget https://archive.apache.org/dist/spark/spark-3.5.0/spark-3.5.0-bin-hadoop3.tgz
tar -xzf spark-3.5.0-bin-hadoop3.tgz
sudo mv spark-3.5.0-bin-hadoop3 /opt/spark
sudo chown -R hadoop:hadoop /opt/spark
```

**配置 Spark Master：**

编辑 `/opt/spark/conf/spark-env.sh`：

```bash
cp /opt/spark/conf/spark-env.sh.template /opt/spark/conf/spark-env.sh

# 在文件中添加：
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export SPARK_MASTER_HOST='192.168.1.100'      # Master IP
export SPARK_MASTER_PORT=7077
export SPARK_MASTER_WEBUI_PORT=8080
export SPARK_WORKER_CORES=4
export SPARK_WORKER_MEMORY=8g
export SPARK_LOCAL_IP=192.168.1.100
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
```

**配置 Workers 列表 (`/opt/spark/conf/workers`)：**

```
192.168.1.100
192.168.1.101
192.168.1.102
```

#### 1.2 在 Worker 节点分发配置

**从 Master 分发 Spark 到所有 Workers：**

```bash
# 在 Master 上执行（以 hadoop 用户）
for worker in 192.168.1.101 192.168.1.102; do
  scp -r /opt/spark/* hadoop@$worker:/opt/spark/
done

# 验证：在每个 Worker 上检查
ssh hadoop@192.168.1.101 "ls -la /opt/spark"
```

#### 1.3 启动集群

**在 Master 节点启动：**

```bash
# 启动 Master
/opt/spark/sbin/start-master.sh

# 启动所有 Workers
/opt/spark/sbin/start-workers.sh

# 验证进程
jps
# 应该看到 Master 和 Worker

# 查看 WebUI（需要 SSH 转发）
# http://192.168.1.100:8080
```

**验证集群连接：**

```bash
# 在 Master 节点测试
/opt/spark/bin/spark-shell --master spark://192.168.1.100:7077

# 在 shell 中执行
sc.parallelize(1 to 100).sum()
:quit
```

### 2. Jupyter Server 配置（Master 节点）

#### 2.1 安装 Jupyter

```bash
# 以 hadoop 用户执行
pip3 install jupyter jupyterlab ipython pyspark pandas numpy matplotlib

# 生成配置文件
jupyter notebook --generate-config

# 设置密码
jupyter notebook password
# 输入你的密码
```

#### 2.2 配置远程访问

**编辑 `~/.jupyter/jupyter_notebook_config.py`：**

```python
# 监听所有接口
c.NotebookApp.ip = '0.0.0.0'
c.NotebookApp.port = 8888
c.NotebookApp.open_browser = False

# 工作目录
c.NotebookApp.notebook_dir = '/home/hadoop/jupyter_work'

# 禁用 token（使用密码认证）
c.NotebookApp.token = ''
c.NotebookApp.disable_check_xsrf = True

# 允许远程 WebSocket 连接
c.NotebookApp.allow_remote_access = True
```

#### 2.3 启动 Jupyter

```bash
# 创建工作目录
mkdir -p ~/jupyter_work

# 使用 systemd 服务（推荐持久化方案）
sudo tee /etc/systemd/system/jupyter.service > /dev/null <<EOF
[Unit]
Description=Jupyter Notebook Server
After=network.target

[Service]
Type=simple
User=hadoop
WorkingDirectory=/home/hadoop/jupyter_work
ExecStart=/usr/local/bin/jupyter notebook --config=/home/hadoop/.jupyter/jupyter_notebook_config.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# 启用并启动
sudo systemctl daemon-reload
sudo systemctl enable jupyter
sudo systemctl start jupyter

# 查看日志
sudo journalctl -u jupyter -f
```

**或使用后台运行（临时方案）：**

```bash
cd ~/jupyter_work
nohup jupyter notebook --config ~/.jupyter/jupyter_notebook_config.py > ~/jupyter.log 2>&1 &
```

### 3. 本地远程连接配置

#### 3.1 SSH 端口转发

**在你的本地机器（Windows PowerShell）上：**

```powershell
# 方案 A：一次性转发
ssh -i D:\key `
    -L 8888:192.168.1.100:8888 `
    -L 8080:192.168.1.100:8080 `
    -L 8081:192.168.1.100:8081 `
    root@GATEWAY_IP -p 21040 -N

# 方案 B：创建持久化脚本 (forward-cluster.ps1)
$sshKey = "D:\key"
$gateway = "GATEWAY_IP"
$port = "21040"

while($true) {
    Write-Host "连接到 Spark 集群..."
    ssh -i $sshKey `
        -L 8888:192.168.1.100:8888 `
        -L 8080:192.168.1.100:8080 `
        root@$gateway -p $port -N
  
    Write-Host "连接断开，10秒后重新连接..."
    Start-Sleep -Seconds 10
}

# 运行脚本
powershell -ExecutionPolicy Bypass -File forward-cluster.ps1
```

#### 3.2 访问 Jupyter

```
http://localhost:8888
# 输入你设置的密码
```

### 4. Jupyter 中连接 Spark 集群

#### 4.1 创建 Notebook，添加以下代码

**第一个 Cell：配置 PySpark**

```python
import os
import sys

# 设置 Spark Home（如果本地没有安装 Spark，这是可选的）
# os.environ['SPARK_HOME'] = '/opt/spark'

# 配置 PySpark 连接到远程集群
spark_master_url = "spark://192.168.1.100:7077"
spark_app_name = "JupyterRemoteApp"

from pyspark.sql import SparkSession

# 创建 SparkSession 连接到集群
spark = SparkSession.builder \
    .appName(spark_app_name) \
    .master(spark_master_url) \
    .config("spark.executor.memory", "2g") \
    .config("spark.executor.cores", "2") \
    .config("spark.sql.adaptive.enabled", "true") \
    .getOrCreate()

print(f"✓ Spark 版本: {spark.version}")
print(f"✓ 已连接到集群: {spark_master_url}")
```

#### 4.2 提交任务示例

**计算密集型任务：**

```python
# 创建 RDD 并分发到集群
data = spark.sparkContext.parallelize(range(1, 1001), numPartitions=4)
result = data.map(lambda x: x ** 2).sum()

print(f"1到1000平方和: {result}")
```

**DataFrame 操作：**

```python
# 创建分布式 DataFrame
df = spark.createDataFrame([
    ("Alice", 28),
    ("Bob", 35),
    ("Charlie", 42),
], ["name", "age"])

# 执行分布式计算
df.filter(df.age > 30).show()

# 转移数据回本地
result = df.toPandas()
print(result)
```

**从 HDFS 读取数据：**

```python
# 如果集群连接了 HDFS
df = spark.read.csv("hdfs://192.168.1.100:9000/data/file.csv", header=True)
df.show()
```

### 5. 监控集群

#### 5.1 本地访问 Spark Web UI

通过浏览器访问：

```
http://localhost:8080
# 可以看到集群状态、Worker 信息、正在运行的任务
```

#### 5.2 在 Jupyter 中获取集群信息

```python
# 获取应用 ID
app_id = spark.sparkContext.applicationId
print(f"应用 ID: {app_id}")

# 获取 Spark 配置
spark_conf = spark.sparkContext.getConf().getAll()
for key, value in spark_conf:
    if 'memory' in key.lower() or 'cores' in key.lower():
        print(f"{key}: {value}")

# 获取 Executor 信息
status = spark.sparkContext.statusTracker()
print(f"总 Executor 数: {status.getExecutorInfos()}")
```


## 相关资源

- [Spark 官方文档](https://spark.apache.org/docs/latest/)
- [Sedona 官方文档](https://sedona.apache.org/)
- [PySpark API 参考](https://spark.apache.org/docs/latest/api/python/)
