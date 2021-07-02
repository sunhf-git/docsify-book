# 前言
搭建单机简易开发环境快速搭建，目的是绕过hive默认配置与hadoop默认配置带来的各种坑。

# 环境要求
2核4G虚拟机（最低） 4核8G虚拟机（推荐）。centos8操作系统。
- [hadoop 3.3.0 点击下载](http://nas-cn.synology.me:1288/public/hive/hadoop-3.3.0.tar.gz)
- [hbase 2.3.5 点击下载](http://nas-cn.synology.me:1288/public/hive/hbase-2.3.5-bin.tar.gz)
- [hive 3.1.2 点击下载](http://nas-cn.synology.me:1288/public/hive/apache-hive-3.1.2-bin.tar.gz)
- [java 8 点击下载一键安装脚本](http://nas-cn.synology.me:1288/public/hive/java.sh)
- [mysql 8.0 点击下载JDBC驱动包](http://nas-cn.synology.me:1288/public/hive/mysql-connector-java-8.0.25.jar)
- [guava 27 点击下载](http://nas-cn.synology.me:1288/public/hive/guava-27.0-jre.jar)

# 基本流程
- 安装MySQL
- 安装java
- 配置hadoop环境变量，启动中的hdfs和yarn
- 配置hbase环境变量，无需操作其他
- 配置hive环境变量，安装hive
- 注意每次配置完环境变量需要执行 source /etc/profile 或者重启服务器

# 安装MySQL
1.安装完mysql后需要创建一个hive的管理员用户（我这里用的叫hive，也可以是别的） 
```
dnf install @mysql
systemctl enable --now mysqld
```

# 安装java
1. 运行一键脚本即可
```
wget http://nas-cn.synology.me:1288/public/hive/java.sh && sh java.sh && source /etc/profile
```
# 安装hadoop
1. 下载并解压hadoop安装包
```
wget http://nas-cn.synology.me:1288/public/hive/hadoop-3.3.0.tar.gz && tar -zxvf hadoop-3.3.0.tar.gz && mv hadoop-3.3.0 hadoop330
```
2. 设置环境变量
```
vi /etc/profile
```
添加如下内容
```
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export HDFS_DATANODE_SECURE_USER=yarn
export YARN_NODEMANAGER_USER=root
export HADOOP_HOME=/root/hadoop330
export PATH=$HADOOP_HOME/bin:$PATH
```
3. 编辑hadoop-env文件调大JVM初始内存（不调在insert时会OOM）
```
$HADOOP_HOME/etc/hadoop/hadoop-env.sh
```
修改为以下内容
```
 export JAVA_HOME=/apps/java8
 export HADOOP_HOME=/root/hadoop330
 export HADOOP_HEAPSIZE_MAX=2048
```
4. 修改core-site.xml configuration标签内添加以下内容
```
<property>
    <name>fs.default.name</name>
    <value>hdfs://localhost:9000</value>
 </property>
<property>
    <name>hadoop.proxyuser.root.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.root.groups</name>
    <value>*</value>
</property>
```
5. 配置免密登录
```
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
6. 启动hadoop检查是否出现报错
```
$HADOOP_HOME/sbin/start-all.sh && jps
```
# 安装hbase
1. 下载并解压hbase， 配置环境变量
```
wget http://nas-cn.synology.me:1288/public/hive/hbase-2.3.5-bin.tar.gz && tar -zxvf hbase-2.3.5-bin.tar.gz && mv hbase-2.3.5-bin hbase235
vi /etc/profile
```
添加如下内容
```
export HBASE_HOME=/root/hbase235
export PATH=$HBASE_HOME/bin:$PATH
```
# 安装Hive
1. 下载并解压hive， 配置环境变量, 下载jdbc驱动，修复guava版本冲突错误
```
wget http://nas-cn.synology.me:1288/public/hive/apache-hive-3.1.2-bin.tar.gz && tar -zxvf apache-hive-3.1.2-bin.tar.gz && mv apache-hive-3.1.2-bin hive312 && rm -rf hive312/lib/guava* && wget http://nas-cn.synology.me:1288/public/hive/guava-27.0-jre.jar && wget http://nas-cn.synology.me:1288/public/hive/mysql-connector-java-8.0.25.jar
vi /etc/profile
```
环境变量追加内容
```
export HIVE_HOME=/root/hive312
export PATH=$HIVE_HOME/bin:$PATH
```
2. 拷贝hive-default.xml.template 为 hive-site.xml 并修改配置
-  注意：3215行的文本错误，把description标签删了就ok了
```
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.cj.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
   <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://127.0.0.1:3306/hive_metadb?createDatabaseIfNotExist=true</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
    </description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
    <description>Username to use against metastore database</description>
  </property>
    <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>123456</value>
    <description>password to use against metastore database</description>
  </property>
    <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/tmp/${user.name}</value>
    <description>Local scratch space for Hive jobs</description>
  </property>
  <property>
    <name>hive.querylog.location</name>
    <value>/tmp/${user.name}</value>
    <description>Location of Hive run time structured log file</description>
  </property>
  <property>
    <name>hive.server2.logging.operation.log.location</name>
    <value>/tmp/${user.name}/operation_logs</value>
    <description>Top level directory where operation logs are stored if logging functionality is enabled</description>
  </property>
  <property>
    <name>hive.exec.mode.local.auto</name>
    <value>true</value>
    <description>Let Hive determine whether to run in local mode automatically</description>
  </property>
```

3. 运行hive测试配置,运行成功后ctrl+c退出
```
hive
```
4. 运行hive,运行后就可以照着官网测试了，没有密码验证，任意用户名登录即可。
```
$HIVE_HOME/bin/hiveserver2
```

# 相关资料
- [领域模型设计](https://www.infoq.cn/article/ddd-evolving-architecture/)
- [技术选型的注意事项](https://www.infoq.cn/article/points-for-attention-with-technology-choice)