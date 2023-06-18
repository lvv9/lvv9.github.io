# 搭建Hadoop集群
1. 准备ubuntu环境
```shell
docker pull ubuntu
```
2. 启动容器
3. 下载hadoop，这里为hadoop-3.3.1-aarch64.tar.gz，官方的部署说明可以见 [这里](https://hadoop.apache.org/docs/r3.3.1/hadoop-project-dist/hadoop-common/ClusterSetup.html)
4. 复制hadoop到docker容器中
```shell
docker cp hadoop-3.3.1-aarch64.tar.gz hadoop_template:/root
```
5. 登录容器并解压、重命名
```shell
tar xvf hadoop-3.3.1-aarch64.tar.gz -C /usr/local/
mv /usr/local/hadoop-3.3.1/ /usr/local/hadoop
```
6. 安装hadoop依赖
```shell
apt update
apt install openjdk-8-jdk openssh-server
```
7. 配置hadoop运行时环境变量及参数
etc/hadoop/hadoop-env.sh
```shell
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-arm64
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
```
及etc/hadoop/core-site.xml
```xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://hadoop1:9000</value>
        </property>
</configuration>
```
etc/hadoop/hdfs-site.xml (optional)
```xml
<configuration>
        <property>
                <name>dfs.permissions.enabled</name>
                <value>false</value>
        </property>
</configuration>
```
etc/hadoop/yarn-site.xml
```xml
<configuration>

<!-- Site specific YARN configuration properties -->
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop1</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>

</configuration>
```
etc/hadoop/mapred-site.xml
```xml
<configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
        <property>
                <name>mapreduce.application.classpath</name>
                <value>
                        /usr/local/hadoop/etc/hadoop,
                        /usr/local/hadoop/share/hadoop/common/*,
                        /usr/local/hadoop/share/hadoop/common/lib/*,
                        /usr/local/hadoop/share/hadoop/hdfs/*,
                        /usr/local/hadoop/share/hadoop/hdfs/lib/*,
                        /usr/local/hadoop/share/hadoop/mapreduce/*,
                        /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
                        /usr/local/hadoop/share/hadoop/yarn/*,
                        /usr/local/hadoop/share/hadoop/yarn/lib/*
                </value>
        </property>
</configuration>
```
etc/hadoop/workers
```text
hadoop1
hadoop2
hadoop3
```
8. 配置ssh免登录、bash启动命令
```shell
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
echo service ssh start >> ~/.bashrc
```
9. 导出镜像
```shell
docker commit yourcontainerid hadoop_template
```
10. 创建docker网络，以便容器间访问时使用主机名来访问
```shell
docker network create -d bridge hadoop
```
11. 创建集群
```shell
docker run -dit --network hadoop -h hadoop1 hadoop_template /bin/bash
docker run -dit --network hadoop -h hadoop2 hadoop_template /bin/bash
docker run -dit --network hadoop -h hadoop3 hadoop_template /bin/bash
```
12. 在namenode执行hadoop初始化
```shell
cd /usr/local/hadoop
./bin/hdfs namenode -format
```
13. 启动
```shell
./sbin/start-all.sh
```
> The project includes these modules:<br>
> Hadoop Common: The common utilities that support the other Hadoop modules.<br>
> Hadoop Distributed File System (HDFS™): A distributed file system that provides high-throughput access to application data.<br>
> Hadoop YARN: A framework for job scheduling and cluster resource management.<br>
> Hadoop MapReduce: A YARN-based system for parallel processing of large data sets.

配置好后的节点分配

|节点|HDFS命名节点|HDFS数据节点|YARN节点
|:---:|:---:|:---:|:---:
|hadoop1|NameNode SecondaryNameNode|DataNode|ResourceManager NodeManager
|hadoop2| |DataNode|NodeManager
|hadoop3| |DataNode|NodeManager
14. 节点维护
```shell
apt install net-tools #安装netstat
netstat -tunlp #查看端口
jps #查看java进程
./bin/hdfs dfsadmin -safemode leave #退出安全模式

```
15. 应用开发
- 新建目录
```shell
./bin/hdfs dfs -mkdir -p /demo/demoInput
```
- 上传文件
```text
this is a demo file
```
```shell
docker cp demo.txt hadoop1:/root
./bin/hdfs dfs -put ~/demo.txt /demo/demoInput #可以复制多一份增加mapper数量
```
- 编写代码
> The MapReduce framework operates exclusively on <key, value> pairs, that is, the framework views the input to the job as a set of <key, value> pairs and produces a set of <key, value> pairs as the output of the job, conceivably of different types.<br>
> The key and value classes have to be serializable by the framework and hence need to implement the Writable interface. Additionally, the key classes have to implement the WritableComparable interface to facilitate sorting by the framework.<br>
> Input and Output types of a MapReduce job:<br>
> (input) <k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class JobDemo {

    public static class DemoMapper extends Mapper<Object, Text, Text, IntWritable> {

        private final IntWritable one = new IntWritable(1);
        private final Text character = new Text();

        @Override
        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            for (char c : value.toString().toCharArray()) {
                character.set(Character.toString(c));
                context.write(character, one);
            }
        }
    }

    public static class DemoReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

        private final IntWritable result = new IntWritable();

        @Override
        public void reduce(Text key, Iterable<IntWritable> iterable, Context context) throws IOException, InterruptedException {
            int sum = 0;
            for (IntWritable value : iterable) {
                sum += value.get();
            }
            result.set(sum);
            context.write(key, result);
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration configuration = new Configuration();
        Job job = new Job(configuration, "demo");
        job.setCombinerClass(DemoReducer.class);
        job.setJarByClass(JobDemo.class);
        job.setMapperClass(DemoMapper.class);
        job.setReducerClass(DemoReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        job.setNumReduceTasks(2); // 控制reducer数量
        FileInputFormat.addInputPath(job, new Path("/demo/demoInput/"));
        FileOutputFormat.setOutputPath(job, new Path("/demo/demoOutput/"));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```
- maven编译
- 运行
```shell
docker cp ~/IdeaProjects/mapreduce-demo/target/mapreduce-demo-1.0-SNAPSHOT.jar hadoop1:/root
./bin/hadoop jar ~/mapreduce-demo-1.0-SNAPSHOT.jar JobDemo
```
- 获取结果
```shell
./bin/hdfs dfs -ls /demo/demoOutput
./bin/hdfs dfs -get /demo/demoOutput/part-r-00000 ~/out
docker cp hadoop1:/root/out Desktop
```
```text
 	4
a	1
d	1
e	2
f	1
h	1
i	3
l	1
m	1
o	1
s	2
t	1
```
- 删除输出
```shell
./bin/hdfs dfs -rm -r -f /demo/demoOutput
```
- 关闭应用
```shell
./sbin/start-all.sh
```
16. 图形管理工具

由于hadoop自带的web管理工具涉及到重定向的问题，docker不方便对多个container的端口映射到host，所以这里采用代理的方式访问
```shell
docker pull shadowsocks/shadowsocks-libev
docker run -p 8388:8388 -p 8388:8388/udp -d --restart always -h hadoop --network hadoop -e DNS_ADDRS=127.0.0.11 shadowsocks/shadowsocks-libev
```
host通过ss客户端连接8388后即可访问 http://hadoop1:9870/explorer.html#/