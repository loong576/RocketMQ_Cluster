## 前言

集群使用的模式是 2m-2s-sync，采用同步复制、异步刷盘方式。

## 环境说明

| ip地址      | 主机名       | 操作系统版本 | RocketMQ版本 |  JDK版本  | maven版本 | 备注            |
| ----------- | ------------ | :----------: | :----------: | :-------: | :-------: | --------------- |
| 172.16.7.91 | nameserver01 |  centos 7.6  |    4.8.0     | 1.8.0_291 |    3.6    | Name Server集群 |
| 172.16.7.92 | nameserver03 |  centos 7.6  |    4.8.0     | 1.8.0_291 |    3.6    | Name Server集群 |
| 172.16.7.93 | master01     |  centos 7.6  |    4.8.0     | 1.8.0_291 |    3.6    | Broker集群      |
| 172.16.7.94 | slave01      |  centos 7.6  |    4.8.0     | 1.8.0_291 |    3.6    | Broker集群      |
| 172.16.7.95 | master02     |  centos 7.6  |    4.8.0     | 1.8.0_291 |    3.6    | Broker集群      |
| 172.16.7.96 | slave02      |  centos 7.6  |    4.8.0     | 1.8.0_291 |    3.6    | Broker集群      |

## 部署概况

![image-20210615154528229](https://i.loli.net/2021/06/15/hmPAruV5g4RnKaG.png)

## 一、准备工作

### 1.设置hostname

```bash
[root@rabbitmq01 network-scripts]# hostnamectl set-hostname nameserver01
[root@rabbitmq01 network-scripts]# hostname
nameserver01
```

6台服务器均设置主机名，详见环境说明。

### 2.设置hosts

```bash
[root@nameserver01 ~]# cat >> /etc/hosts <<EOF
> 172.16.7.91  nameserver01  
> 172.16.7.92  nameserver02
> 172.16.7.93  master01
> 172.16.7.94  slave01
> 172.16.7.95  master02
> 172.16.7.96  slave02
> EOF
```

![image-20210611095837221](https://i.loli.net/2021/06/15/QDExdI5SCbn4mtK.png)

6台服务器都执行以上操作

### 3.关闭防火墙和selinux

详见[Centos7.6操作系统安装及优化全纪录](https://blog.51cto.com/u_3241766/2398136)

## 二、安装jdk

### 1.下载jdk

下载地址：https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html

![image-20210601170259948](https://i.loli.net/2021/06/15/hHyGb5jdatAVvSZ.png)

### 2.安装jdk

新建目录/opt/java并将下载的包jdk-8u291-linux-x64.rpm 上传

```bash
[root@nameserver01 ~]# mkdir /opt/java
[root@nameserver01 ~]# cd /opt/java
[root@nameserver01 java]# rpm -ivh jdk-8u291-linux-x64.rpm 
```

![image-20210611100322574](https://i.loli.net/2021/06/15/MI7XoP1cYvWem6K.png)

6台都执行java安装

## 三、安装maven

6台服务器都执行如下操作

### 1.下载安装包

```bash
[root@nameserver01 ~]# wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
```

也可以到官网下载，下载地址：http://maven.apache.org/download.cgi

### 2.解压并配置环境变量

```bash
[root@nameserver01 ~]# tar -zxvf apache-maven-3.6.3-bin.tar.gz -C /usr/local/
[root@nameserver01 ~]# mv /usr/local/apache-maven-3.6.3/ /usr/local/maven3.6
[root@nameserver01 ~]# sed -i '$a export PATH=$PATH:/usr/local/maven3.6/bin' /etc/profile && source /etc/profile
[root@nameserver01 ~]# mvn -v
```

![image-20210611101107969](https://i.loli.net/2021/06/15/BPpugQlVdnbFe87.png)

### 3.maven优化

#### 3.1配置阿里源

```bash
[root@rabbitmq01 ~]# cd /usr/local/maven3.6/conf/
[root@rabbitmq01 conf]# view settings.xml

```

修改前：

```bash
<url>http://my.repository.com/repo/path</url>
```

修改后：

```bash
<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
```

![image-20210602100342700](https://i.loli.net/2021/06/15/FjA5EJKyhkmclWM.png)

#### 3.2指定资源下载路径

修改前：

```bash
<localRepository>/path/to/local/repo</localRepository>
```

修改后：

```bash
<localRepository>/usr/local/maven3.6/repository</localRepository>
```

![image-20210602100624371](https://i.loli.net/2021/06/15/WBAj2hDXwk3JMbc.png)

#### 3.3指定JDK版本

将默认的1.4替换为实际的1.8

修改前：

![image-20210602100754413](https://i.loli.net/2021/06/15/tydsSqujIJehKgl.png)

修改后：

![image-20210602100902226](https://i.loli.net/2021/06/15/v1qrK2hnjgF7YSQ.png)

## 四、RocketMQ安装

### 1.版本选取

下载地址：https://github.com/apache/rocketmq/tags

![image-20210601162652495](https://i.loli.net/2021/06/15/EdvpirYRDFOjPxU.png)

本次选择的版本为最新的4.8.0

### 2.下载安装包

```bash
[root@nameserver01 ~]# wget https://github.com/apache/rocketmq/archive/refs/tags/rocketmq-all-4.8.0.tar.gz
```

![image-20210611101809418](https://i.loli.net/2021/06/15/p2gbvOCrWjkB7XU.png)

### 3.解压并编译

```bash
[root@nameserver01 ~]# mkdir /rocketmq
[root@nameserver01 ~]# tar -zxvf rocketmq-all-4.8.0.tar.gz -C /rocketmq
[root@nameserver01 ~]# mv /rocketmq/rocketmq-rocketmq-all-4.8.0/* /rocketmq/ && rm -rf /rocketmq/rocketmq-rocketmq-all-4.8.0/
[root@nameserver01 ~]# cd /rocketmq/
[root@nameserver01 rocketmq]# mvn -Prelease-all -DskipTests clean install -U
```

![image-20210611103522494](https://i.loli.net/2021/06/15/vKGNm3l2kod19R6.png)

中间省略了些步骤

![image-20210611103433336](https://i.loli.net/2021/06/15/XPrRNCGvxl8VA7I.png)

### 4.免编译安装

除了使用maven编译安装方式外，还可以直接下载带bin的安装包，免编译安装，步骤3和步骤4执行一个就行。

**本文后面章节基于免编译安装。**

下载地址：[**https://apache.claz.org/rocketmq/4.8.0/rocketmq-all-4.8.0-bin-release.zip**](https://apache.claz.org/rocketmq/4.8.0/rocketmq-all-4.8.0-bin-release.zip)

```bash
[root@nameserver01 ~]# cd /rocketmq/
[root@nameserver01 rocketmq]# wget https://apache.claz.org/rocketmq/4.8.0/rocketmq-all-4.8.0-bin-release.zip
[root@nameserver01 rocketmq]# unzip rocketmq-all-4.8.0-bin-release.zip
[root@nameserver01 rocketmq]# cd rocketmq-all-4.8.0-bin-release && mv * /rocketmq/ && rm -rf /rocketmq/rocketmq-all-4.8.0-bin-release*
[root@nameserver01 rocketmq]# pwd
/rocketmq
[root@nameserver01 rocketmq]# ll
总用量 40
drwxr-xr-x 2 root root   102 12月  9 19:46 benchmark
drwxr-xr-x 3 root root  4096 12月  4 14:26 bin
drwxr-xr-x 6 root root   211 12月  4 14:26 conf
drwxr-xr-x 2 root root  4096 12月  9 19:46 lib
-rw-r--r-- 1 root root 17336 10月 23 2020 LICENSE
-rw-r--r-- 1 root root  1338 12月  4 14:26 NOTICE
-rw-r--r-- 1 root root  5132 12月  4 14:26 README.md
```

rocketmq-all-4.8.0-bin-release.zip带bin版本的介质无需maven编译，解压后即可使用。

### 5.配置环境变量

```bash
[root@nameserver01 ~]# sed -i '$a export ROCKETMQ_HOME=/rocketmq' /etc/profile
[root@nameserver01 ~]# sed -i '$a export PATH=$PATH:$ROCKETMQ_HOME/bin' /etc/profile && source /etc/profile
```

![image-20210611170252130](https://i.loli.net/2021/06/15/Hz9VRAiCXIeqvus.png)

## 五、Name Server集群安装

本节操作在nameserver集群nameserver01和nameserver02都执行。

### 1.调整runserver.sh参数

```bash
[root@nameserver01 ~]# cd /rocketmq/bin/
[root@nameserver01 bin]# view runserver.sh 
```

**调整前：**

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

**调整后：**

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

由于默认启动的最大内存比较大，在虚机测试时需要修改小点保证服务能正常启动。

### 2.启动NameServer

```bash
[root@nameserver01 bin]# nohup sh mqnamesrv &
```

![image-20210611172311030](https://i.loli.net/2021/06/15/XiyPekqGchzxDT7.png)

NameServer启动成功，同样的nameserver02也执行相同启动命令。

## 六、Broker集群安装

本节操作在Broker集群master01、slave01、master02、slave02分别执行。

### 1.调整runbroker.sh参数

```bash
[root@master01 ~]# cd /rocketmq/bin
[root@master01 bin]# view runbroker.sh 
```

**调整前：**

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
```

**调整后：**

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m"
```

由于默认启动的最大内存比较大，在虚机测试时需要修改小点保证服务能正常启动。

### 2.配置broker

由于本文采用的是2主2从同步方式，对应的目录为2m-2s-sync

```bash
[root@master01 2m-2s-sync]# pwd
/rocketmq/conf/2m-2s-sync
[root@master01 2m-2s-sync]# ll
总用量 16
-rw-r--r-- 1 root root 928 10月 23 2020 broker-a.properties
-rw-r--r-- 1 root root 922 10月 23 2020 broker-a-s.properties
-rw-r--r-- 1 root root 928 10月 23 2020 broker-b.properties
-rw-r--r-- 1 root root 922 10月 23 2020 broker-b-s.properties
```

#### 2.1修改master01配置broker-a.properties

```bash
[root@master01 2m-2s-sync]# pwd
/rocketmq/conf/2m-2s-sync
[root@master01 2m-2s-sync]# more broker-a.properties 
namesrvAddr=172.16.7.91:9876;172.16.7.92:9876
brokerClusterName=MyRocketmq
brokerName=broker-a
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=SYNC_MASTER
flushDiskType=ASYNC_FLUSH
```

新增了namesrvAddr，指定对应的ip和端口；修改了brokerClusterName，自定义集群名为MyRocketmq；刷盘方式为默认的ASYNC_MASTER即异步方式。

#### 2.2修改slave01配置broker-a-s.properties

```bash
[root@slave01 2m-2s-sync]# more broker-a-s.properties
namesrvAddr=172.16.7.91:9876;172.16.7.92:9876
brokerClusterName=MyRocketmq
brokerName=broker-a
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
```

#### 2.3修改master02配置broker-b.properties

```bash
[root@master02 2m-2s-sync]# more broker-b.properties
namesrvAddr=172.16.7.91:9876;172.16.7.92:9876
brokerClusterName=MyRocketmq
brokerName=broker-b
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=SYNC_MASTER
flushDiskType=ASYNC_FLUSH
```

#### 2.4修改slave02配置broker-b-s.properties

```bash
[root@slave02 2m-2s-sync]# more broker-b-s.properties
namesrvAddr=172.16.7.91:9876;172.16.7.92:9876
brokerClusterName=MyRocketmq
brokerName=broker-b
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
```

### 3.启动broker

#### 3.1启动master01

```bash
[root@master01 rocketmq]# pwd
/rocketmq
[root@master01 rocketmq]# nohup sh bin/mqbroker -c conf/2m-2s-sync/broker-a.properties &
```

![image-20210611172415497](https://i.loli.net/2021/06/15/8kUyC5BTOmSNzIt.png)

master01节点启动成功

#### 3.2启动slave01

```bash
[root@slave01 rocketmq]# pwd
/rocketmq
[root@slave01 rocketmq]# nohup sh bin/mqbroker -c conf/2m-2s-sync/broker-a-s.properties &
```

![image-20210611180018178](https://i.loli.net/2021/06/15/4zB86uaxmpLYKNy.png)

slave01启动成功

#### 3.3启动master02

```bash
[root@master02 rocketmq]# pwd
/rocketmq
[root@master02 rocketmq]# nohup sh bin/mqbroker -c conf/2m-2s-sync/broker-b.properties &
```

![image-20210611172452728](https://i.loli.net/2021/06/15/S3rvWwZoizpI7Cj.png)

master02启动成功

#### 3.4启动slave02

```bash
[root@slave02 rocketmq]# pwd
/rocketmq
[root@slave02 rocketmq]# nohup sh bin/mqbroker -c conf/2m-2s-sync/broker-b-s.properties &
```

![image-20210611172510609](https://i.loli.net/2021/06/15/QcRYZO3UnzWAeBt.png)

slave02启动成功

## 七、安装管理面板

本节在nameserver01上执行

### 1.介质下载

git下载介质

```bash
[root@nameserver01 ~]# yum -y install git
[root@nameserver01 ~]# git clone https://github.com/apache/rocketmq-externals.git
```

或者下载到本地再上传，下载地址：https://github.com/apache/rocketmq-externals/archive/refs/heads/master.zip

这里选择先下载再上传方式

### 2.解压

```bash
[root@nameserver01 ~]# unzip rocketmq-externals-master--console.zip 
[root@nameserver01 ~]# cd rocketmq-externals-master
```

![image-20210611173616087](https://i.loli.net/2021/06/15/bQYTE1Dm85G7czA.png)

### 3.修改配置文件

```bash
[root@nameserver01 resources]# pwd
/root/rocketmq-externals-master/rocketmq-console/src/main/resources
[root@nameserver01 resources]# view application.properties
```

![image-20210611174000379](https://i.loli.net/2021/06/15/cNTdzxQkER8jibh.png)

自定义应用端口和nameserver服务器

### 4.编译

```bash
[root@nameserver01 rocketmq-console]# pwd
/root/rocketmq-externals-master/rocketmq-console
[root@nameserver01 rocketmq-console]# mvn clean package -Dmaven.test.skip=true
[INFO] Scanning for projects...
```

![image-20210611174810590](https://i.loli.net/2021/06/15/BYpdcFmPLVNGCka.png)

编译过程如图，中间过程省略

![image-20210611174735373](https://i.loli.net/2021/06/15/oIvGDHW7LN4ceyO.png)

编译成功，console目录下会生成target目录

### 5.运行

```bash
[root@nameserver01 rocketmq-console]# pwd
/root/rocketmq-externals-master/rocketmq-console
[root@nameserver01 rocketmq-console]# cd target/
[root@nameserver01 target]# java -jar rocketmq-console-ng-1.0.1.jar
```

![image-20210615143747490](https://i.loli.net/2021/06/15/M54vKf8ydUex3qN.png)

### 6.访问console

访问地址：http://172.16.7.91:57690

[![image-20210615150135262](https://i.loli.net/2021/06/15/qcmt1CSGRMidxAB.png)](http://172.16.7.91:57690/)

![image-20210615150120075](https://i.loli.net/2021/06/15/wbrnJFekZaoMxhf.png)

## 八、集群测试

### 1.生产者

nameserver01上执行生产者测试

```bash
[root@nameserver01 rocketmq]# export NAMESRV_ADDR='172.16.7.91:9876;172.16.7.92:9876'
[root@nameserver01 rocketmq]# sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```

![image-20210615151552972](https://i.loli.net/2021/06/15/kxtfz2W4IQcY6Rm.png)

### 2.消费者

nameserve02上执行消费者测试

```bash
[root@nameserver02 rocketmq]# export NAMESRV_ADDR='172.16.7.91:9876;172.16.7.92:9876'
[root@nameserver02 rocketmq]# sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

![image-20210615151627933](https://i.loli.net/2021/06/15/9u4I1belnZUTMPk.png)

### 3.console查看

![image-20210615151745553](https://i.loli.net/2021/06/15/Lhf9xtcEQUFoAHY.png)

![image-20210615151756641](https://i.loli.net/2021/06/15/9O58skAidc63hvV.png)

![image-20210615151850951](https://i.loli.net/2021/06/15/rv9s5RNGtW3nO4L.png)



