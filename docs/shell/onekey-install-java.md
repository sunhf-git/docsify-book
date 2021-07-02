# java安装脚本 <!-- {docsify-ignore-all} -->
运行环境 centos 7+
```shell script
#!/usr/bin/env bash
JAVA_RES="http://nas-cn.synology.me:1288/public/linux/jdk-8u261-linux-x64.tar.gz";
JAVA_HOME="/apps/java8"
if ! test -d $JAVA_HOME; then
    echo "start install java..."
    yum install -y wget
    wget -q $JAVA_RES && tar -zxf jdk-8u261-linux-x64.tar.gz && rm -rf jdk-8u261-linux-x64.tar.gz
    mkdir -p $JAVA_HOME && rm -rf $JAVA_HOME && mv jdk* $JAVA_HOME
    echo "
    export JAVA_HOME=$JAVA_HOME
    export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
    export PATH=\$PATH:\$JAVA_HOME/bin
    export JAVA_HOME CLASSPATH PATH
    " >> /etc/profile
    source /etc/profile
    java -version
    echo "java install is done!"
fi
source /etc/profile
java -version
```