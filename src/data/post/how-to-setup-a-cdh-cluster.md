---
publishDate: 2024-08-06T00:00:00Z
title: how to setup a cdh cluster
image: https://tm-image.tianyancha.com/tm/c5f83ded53e09ae36ee524bf9e795b96.jpg
tags:
  - cdh
  - devops
---
# 一、环境准备

## 1、软件版本选择

| 类目     | 版本                     |
| -------- | ------------------------ |
| 操作系统 | CentOS Linux release 7.9 |

## 2、节点准备（三个节点）

| 名称       | IP             |
| ---------- | -------------- |
| namenode01 | 192.168.125.61 |
| namenode02 | 192.168.125.62 |
| datanode01 | 192.168.125.63 |
| datanode02 | 192.168.125.64 |
| datanode03 | 192.168.125.65 |

## 3、系统配置

```bash
yum -y install git
git clone --depth=1 https://gitee.com/dlcf/studio
cd /root/studio/scripts/linux
rm 03-install-docker.sh
vim 05-upgrade-kernel.sh
输入%s/kernel-ml-devel kernel-ml/kernel-lt.x86_64/g回车，再输入:wq保存。
bash init-all.sh

# 重启以验证配置是否生效
shtudown -r now
```

## 4、环境配置

新建一个sh脚本，把下面内容复制进去并根据需要增加，然后在新加的节点上运行脚本。

```bash
if [ `grep 222.222.222.222 /etc/resolv.conf | wc -l` -lt 1 ]; then
  echo "nameserver 222.222.222.222" >> /etc/resolv.conf
fi
if [ `grep 114.114.114.114 /etc/resolv.conf | wc -l` -lt 1 ]; then
  echo "nameserver 114.114.114.114" >> /etc/resolv.conf
fi
if [ `grep k8s.lb /etc/hosts | wc -l` -lt 1 ]; then
  echo "192.168.125.61 namenode01" >> /etc/hosts
  echo "192.168.125.62 namenode02" >> /etc/hosts
  echo "192.168.125.63 datanode01" >> /etc/hosts
  echo "192.168.125.64 datanode02" >> /etc/hosts
  echo "192.168.125.65 datanode03" >> /etc/hosts  
fi

# 配置hostname
if [ `ip a|grep "192.168.125.61"|wc -l` -gt 0 ]; then
  hostnamectl set-hostname namenode01
fi
if [ `ip a|grep "192.168.125.62"|wc -l` -gt 0 ]; then
  hostnamectl set-hostname namenode02
fi
if [ `ip a|grep "192.168.125.63"|wc -l` -gt 0 ]; then
  hostnamectl set-hostname datanode01
fi
if [ `ip a|grep "192.168.125.64"|wc -l` -gt 0 ]; then
  hostnamectl set-hostname datanode02
fi
if [ `ip a|grep "192.168.125.65"|wc -l` -gt 0 ]; then
  hostnamectl set-hostname datanode03
fi

# 重启以验证配置是否生效
shtudown -r now
```

## 5 、挂载分区

/data目录

## 6、配置免密登陆

```
ssh-keygen -t rsa # 一路回车，生成无密码密钥对。
#各节点分别执行
ssh-keygen -t dsa
ssh-copy-id namenode01
ssh-copy-id namenode02
ssh-copy-id datanode01
ssh-copy-id datanode02
ssh-copy-id datanode03
#测试
ssh namenode01
ssh namenode02
ssh datanode01
ssh datanode02
ssh datanode03
```

## 7、配置 JDK (所有节点)

```css
# 默认安装在/usr/java文件夹下
rpm -ivh oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
# 配置环境变量：编辑/etc/profile 或者 ~/.bash_profile
vim /etc/profile
---
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
export PATH=$PATH:$JAVA_HOME/bin
---
source /etc/profile 
# 测试
java -version
```

## 8、安装第三方依赖及配置系统参数（所有节点）

```
yum install -y chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse fuse-libs redhat-lsb
```

## 9、搭建本地yum源

- 我这里选择把yum源配置在namenode01节点上

### 9.1 安装并启动Apache http

- 在namenode01节点完成以下操作**

- 安装Apache http

  ```bash
  yum install -y httpd
  ```

- 启动Apache http

  ```bash
  systemctl start httpd
  ```

- 设置自启动Apache http

  ```bash
  systemctl enable httpd
  ```

### 9.2 上传安装文件

- 在namenode01节点完成以下操作

- 创建安装文件http根目录

  ```bash
  mkdir -p /var/www/html/cm6
  ```

- 上传安装文件到http根目录

- 上传后查看文件

  ```bash
  ll /var/www/html/cm6
  ```

  - 执行结果

    ```bash
    cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
    cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
    cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
    cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
    enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
    ```

### 9.3 创建yum仓库

- 在namenode01节点完成以下操作

```bash
cd /var/www/html/cm6
yum install -y createrepo
createrepo .
```

### 9.4 配置yum仓库文件

- 在所有节点完成以下操作

```bash
vim /etc/yum.repos.d/cloudera-manager.repo
[cloudera-manager]
name=Cloudera Manager 6.3.1
baseurl=http://namenode01/cm6
gpgcheck=0
enabled=1
autorefresh=0
type=rpm-md
```

### 9.5 更新仓库信息，确认本地yum源已被添加

~~~powershell
yum clean all
yum makecache
~~~

# 二、安装 CM 和 CDH



## 1、修改centos内核参数

vim /etc/sysctl.conf，增加以下内容

```
kernel.sysrq = 0
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 6580110
kernel.shmall = 6580110
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.icmp_echo_ignore_all = 0
net.ipv4.icmp_echo_ignore_broadcasts
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 10240 87380 12582912
net.ipv4.tcp_wmem = 10240 87380 12582912
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 40960
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.ip_local_port_range = 1024 65000
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.netfilter.nf_conntrack_max = 6553500
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
net.nf_conntrack_max = 6553500
vm.overcommit_memory = 0
vm.swappiness = 0
fs.file-max = 999999
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_syncookies = 1
net.core.somaxconn = 40960
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_rmem = 10240  87380  12582912
net.ipv4.tcp_wmem = 10240 87380 12582912
net.core.rmem_default = 6291456
net.core.wmem_default = 6291456
net.core.rmem_max = 12582912
net.core.wmem_max = 12582912
net.ipv4.tcp_syncookies = 1
```

vim /etc/security/limits.conf，增加以下内容

```
* soft nproc 65535 
* hard nproc 65535 
* soft nofile 65535 
* hard nofile 65535
```

```
sed -i "s/vm.swappiness = 30/vm.swappiness = 10/g" /usr/lib/tuned/virtual-guest/tuned.conf
```



## 2、安装mysql数据库(namenode01)

```sql
# 本地安装mysql, 按照如下顺序安装
yum -y localinstall mysql-community-common-5.7.29-1.el7.x86_64.rpm
yum -y localinstall mysql-community-libs-5.7.29-1.el7.x86_64.rpm
yum -y localinstall mysql-community-libs-compat-5.7.29-1.el7.x86_64.rpm
yum -y localinstall mysql-community-devel-5.7.29-1.el7.x86_64.rpm
yum -y localinstall mysql-community-client-5.7.29-1.el7.x86_64.rpm
yum -y localinstall mysql-community-server-5.7.29-1.el7.x86_64.rpm
```

启动mysql

```
# 启动
 systemctl start mysqld
# 开机自启动
 systemctl enable mysqld
#查看临时密码
cat /var/log/mysqld.log | grep password
mysql ‐uroot ‐p输入上面的临时密码
## 首次登录设置密码 root
set password=password('Sw++0066')
 quit;
# 再次登陆验证密码是否生效
mysql -uroot -pSw++0066
```

## 3、初始化数据库

```bash
/usr/bin/mysql_secure_installation
```

按照下面提示输入。

```
[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] Y
New password:
Re-enter new password:
[...]
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
[...]
All done! 
```

## 4、安装 MySQL JDBC 驱动(所有节点)

用于各节点连接数据库。

```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
tar xf mysql-connector-java-5.1.46.tar.gz
mkdir -p /usr/share/java/
cd mysql-connector-java-5.1.46
cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
```

## 5、为 Cloudera 各软件创建数据库

密码的长度是由validate_password_length决定的,但是可以通过以下命令修改

```
set global validate_password_length=4;
```

validate_password_policy决定密码的验证策略,默认等级为MEDIUM(中等),可通过以下命令修改为

LOW(低)

```
set global validate_password_policy=0;

create user 'root'@'%' identified by 'Sw++0066';
create user 'root'@'namenode01' identified by 'Sw++0066';
create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database monitor DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database cmf DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
grant all privileges on *.* to 'root'@'localhost';
grant all privileges on *.* to 'root'@'%';
grant all privileges on *.* to 'root'@'namenode01';
flush privileges;
exit
```

## 6、安装 CM Server 和 Agent

- **namenode0[1-2]**：

```sql
yum -y install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
```

- **datanode0[1-3]**：

```
yum -y install cloudera-manager-daemons cloudera-manager-agent
```

修改agent配置，指向server节点namenode01(所有节点)

```
sed -i "s/server_host=localhost/server_host=namenode01/g" /etc/cloudera-scm-agent/config.ini
```

修改主节点namenode01的server配置

```
vim /etc/cloudera-scm-server/db.properties
# Copyright (c) 2012 Cloudera, Inc. All rights reserved.
#
# This file describes the database connection.
#
# The database type
# Currently 'mysql', 'postgresql' and 'oracle' are valid databases.
com.cloudera.cmf.db.type=mysql
# The database host
# If a non standard port is needed, use 'hostname:port'
com.cloudera.cmf.db.host=namenode01:3306
# The database name
com.cloudera.cmf.db.name=cmf
# The database user
com.cloudera.cmf.db.user=root
# The database user's password
com.cloudera.cmf.db.password=Sw++0066
# The db setup type
# After fresh install it is set to INIT
# and will be changed post config.
# If scm-server uses Embedded DB then it is set to EMBEDDED
# If scm-server uses External DB then it is set to EXTERNAL
com.cloudera.cmf.db.setupType=EXTERNAL
```

启动cm的sever和agent，开始安装cdh集群

```
# 在namenode01上启动server
systemctl start cloudera-scm-server
# 另开一个窗口，查看相关日志。有异常就解决异常
tail ‐200f /var/log/cloudera‐scm‐server/cloudera‐scm‐server.log
# 这个异常可以忽略 ERROR ParcelUpdateService:com.cloudera.parcel.component
ParcelDownloaderImpl: Unable to retrieve remote parcel repository manifes
# 等待日志输出 started jetty server, 则cm-server启动成功

# 在全部节点上启动agent
systemctl start cloudera-scm-agent
 # 当在 namenode01上 netstat ‐antp | grep 7180 有内容时，说明我们可以访问web页面了
# 查看运行状态
systemctl status cloudera-scm-agent
systemctl status cloudera-scm-server
```

## 7、安装CDH

1.上传CDH parcel到Cloudera Manager Server节点的/opt/cloudera/parcel-repo路径下，上传CDH所需parcel

2.为parcel文件生成SHA1校验文件

```
sha1sum CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel | awk '{ print $1 }' > CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha
```

3.更改parcel文件所有者

```
chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/*
```

4.重启Cloudera Manager，令其识别到本地库

```
systemctl restart cloudera-scm-server
```

5.登录Cloudera Manager，初始用户名和密码均为admin。

浏览器打开`http://192.168.125.61:7180`，用户名和密码默认都是`admin`。

![image-20230529091705052](/images/image-20230529091705052.png)

接收许可。

![image-20230530121712410](/images/image-20230530121712410.png)
这里我们选择免费版。

![image-20230530121746876](/images/image-20230530121746876.png)

# 三、集群安装

![image-20230529092249259](/images/image-20230529092249259.png)
指定要添加的节点。

![image-20230529115211986](/images/image-20230529115211986.png)
选择存储库，之前我们已经在 CM Server 节点配置好了。

![image-20230529115234989](/images/image-20230529115234989.png)
等待在集群中分配、解压、激活CDH parcel，完成后点击继续，检查主机正确性。

![image-20230529115943125](/images/image-20230529115943125.png)

选择安装的服务

![image-20230529124821194](/images/image-20230529124821194.png)

分配集群角色

![image-20230630172303308](/images/image-20230630172303308.png)

测试数据库连接

![image-20230630172946342](/images/image-20230630172946342.png)

审核各服务参数的默认配置，所有目录均放置在/data下。

![image-20230630173244279](/images/image-20230630173244279.png)

等待安装，完成后点继续。![image-20230630175330831](/images/image-20230630175330831.png)

![image-20230529141236014](/images/image-20230529141236014.png)

# 四、KUDU安装

## 1、上传压缩包到 ~/

```
unzip kudu-1.12-x.zip
tar zxf gradle.tar.gz
```

## 2、准备安装目录和yum安装基础依赖

```
mkdir -p /data/soft/
yum -y install numactl numactl-devel autoconf automake cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain flex gcc gcc-c++ gdb git krb5-server krb5-workstation libtool make openssl-devel patch pkgconfig redhat-lsb-core rsync unzip vim-common which scl-utils git devtoolset-3-toolchain numactl-libs doxygen gem graphviz ruby-devel zlib-devel
```

## 3、安装 memkind

```
git clone https://github.com/memkind/memkind.git
cd /data/soft/
yum remove memkind
unzip memkind-master.zip
cd memkind-master
./build.sh --prefix=/usr
ldconfig
make install
ldconfig
```

## 4、安装kudu

```
cd /data/soft/
unzip kudu-1.12-x.zip
cp src/gradle-wrapper.jar /data/soft/kudu-1.12.x/java/gradle/wrapper
cd kudu-1.12.0
mkdir -p /data/soft/kudu-1.12.x/thirdparty/src/
mkdir -p /data/soft/kudu-1.12.x/thirdparty/src/postgresql-42.2.10
cp kudu-1.12-x/src/postgresql-42.2.10.jar /data/soft/kudu-1.12.x/thirdparty/src/postgresql-42.2.10
cd /data/soft/kudu-1.12.x/
build-support/enable_devtoolset.sh thirdparty/build-if-necessary.sh
mkdir -p build/release
cd build/release
../../build-support/enable_devtoolset.sh \
../../thirdparty/installed/common/bin/cmake \
-DCMAKE_BUILD_TYPE=release ../..
ldconfig
make -j4
！！！注意，这个步骤安装后只是编译完成，没有进行install，可以根据自己的情况进行修改
```

## 5、增加环境变量 或者ln 仅供参考

```
kudu-tserver and kudu-master executables in /usr/local/sbin
Kudu command line tool in /usr/local/bin
Kudu client library in /usr/local/lib64/
Kudu client headers in /usr/local/include/kudu
ln -sv /data/soft/kudu-1.12.x/build/release/bin/kudu-master /usr/sbin/
ln -sv /data/soft/kudu-1.12.x/build/release/bin/kudu-tserver /usr/sbin/
```

## 6、web页面如果不在默认位置，记得加启动参数指定web目录

```
--webserver_doc_root=/data/soft/kudu-1.12.0/www
```

## 7、如果要安装到指定目录，需要直接make install

```
make DESTDIR=/data/soft/kudu-1.12.0/ install
```

## 8、编译doc文档

```
make docs
```

## 9、集群内安装

登录http://192.168.120.201:7180/，点击添加服务

![image-20230531093130963](/images/image-20230531093130963.png)

添加Kudu

![image-20230531093149490](/images/image-20230531093149490.png)自定义角色分配

![image-20230531093224221](/images/image-20230531093224221.png)

审核更改

![image-20230531094041187](/images/image-20230531094041187.png)

命令信息详情

![image-20230531093857241](/images/image-20230531093857241.png)

完成，访问http://192.168.120.201:8051/可以看到如下页面，表示安装成功。

![image-20230531094507037](/images/image-20230531094507037.png)

## 10、安装好后的配置

impala启用kudu

![image-20230531094704711](/images/image-20230531094704711.png)

hive启用kudu

![image-20230531095008035](/images/image-20230531095008035.png)

安装YARN MapReduce框架JAR

![image-20230531132734072](/images/image-20230531132734072.png)

http://192.168.120.202:888/ hue登录账号和密码均为hue

# 五、TEZ安装

## 1、下载TEZ

官网：Apache Tez – Welcome to Apache TEZ®

采用Apache下载路径寻找：Welcome to The Apache Software Foundation!

选择编译版或者源码版下载：Index of /tez/0.9.2

直接下载：wget https://dlcdn.apache.org/tez/0.9.2/apache-tez-0.9.2-src.tar.gz

## 2、下载Maven

wget https://archive.apache.org/dist/maven/maven-3/3.8.5/binaries/apache-maven-3.8.5-bin.tar.gz

## 3、下载安装protobuf

注：源码编译过程中hadoop中需要使用protobuf，需要下载protobuf-2.5.0

wget https://github.com/protocolbuffers/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz

## 4、安装maven

解压

[root@node02 ~]# cd /data1/software/tez/
[root@node02 tez]# tar -zxvf apache-maven-3.8.5-bin.tar.gz -C /data1/program/
配置/etc/profile

[root@node02 tez]# vim /etc/profile
#在最后一行输入以下内容
#maven
export MAVEN_HOME=/data1/program/apache-maven-3.8.5
export PATH=$PATH:$MAVEN_HOME/bin
使配置生效并校验

[root@node02 tez]# source /etc/profile
[root@node02 tez]# mvn --version
Apache Maven 3.8.5 (3599d3414f046de2324203b78ddcf9b5e4388aa0)
Maven home: /data1/program/apache-maven-3.8.5
Java version: 1.8.0_181, vendor: Oracle Corporation, runtime: /usr/java/jdk1.8.0_181-cloudera/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "5.4.243-1.el7.elrepo.x86_64", arch: "amd64", family: "unix"
##备注，若是maven有新版本，安装后不退出当前窗口还是会显示旧版本信息。退出后重新登录后显示正常。

## 5、安装protobuf

解压

[root@node02 tez]# tar -zxvf protobuf-2.5.0.tar.gz -C /data1/program/

配置并生效

[root@node02 tez]# vim /etc/profile
#在最后一行输入以下内容
#protobuf-2.5.0
export POF_HOME=/data1/program/protobuf-2.5.0
export PATH=$PATH:$POF_HOME/bin
[root@node02 tez]# source /etc/profile
编译安装

[root@node02 tez]# cd /data1/program/protobuf-2.5.0/
[root@node02 protobuf-2.5.0]# ./configure
[root@node02 protobuf-2.5.0]# make && make install
##注：如果在编译的过程中报错缺少c或者c++，使用yum方式安装后重试即可
验证

[root@node02 protobuf-2.5.0]# protoc --version
libprotoc 2.5.0
注：当configure校验不通过的时候,缺少哪些包就安装,一般需要安装gcc

## 6、tez编译0.9.2版本

解压

[root@node02 protobuf-2.5.0]# cd /data1/software/tez/
[root@node02 tez]# tar -zxvf apache-tez-0.9.2-src.tar.gz

修改配置文件pom.xml

##进行第二步配置文件命令
[root@node02 tez]# cd /data1/software/tez/apache-tez-0.9.2-src
[root@node02 apache-tez-0.9.2-src]# vim pom.xml
<!--以下是修改不走 -->
<!--第一处:这里才是修改，后续2,3,4都是增加-->
<hadoop.version>3.0.0-cdh6.3.2</hadoop.version>
<!--第二处：repositories标签内最后增加-->
<repository>
  <id>cloudera</id>
  <url><https://repository.cloudera.com/artifactory/cloudera-repos/></url>
  <name>Cloudera Repositories</name>
  <snapshots>
    <enabled>false</enabled>
  </snapshots>
</repository>
<!--第三处:pluginRepositories标签内最后增加-->
<pluginRepository>
  <id>cloudera</id>
  <name>Cloudera Repositories</name>
  <url><https://repository.cloudera.com/artifactory/cloudera-repos/></url>
</pluginRepository>
<!--第四处:dependencyManagement标签里的dependencies标签内最后一行。不过这里有个1.9版本，我暂时不理，有兴趣可以验证下需不需要加这个。成功可以留言告知下-->
<dependency>
  <groupId>com.sun.jersey</groupId>
  <artifactId>jersey-client</artifactId>
  <version>1.19</version>
</dependency>
<!--第五处:-->
<!--<module>tez-ext-service-tests</module>
<module>tez-ui</module>-->
<!--注:将这两个注释掉,如果有需要可以不用注释，赶时间的建议最好注释，要不maven编译很耗时。-->
maven编译

[root@node02 apache-tez-0.9.2-src]# mvn clean package -Dmaven.javadoc.skip=true -Dmaven.test.skip=true
##注:这样会跳过test编译,很快就编译完成
##经过一段时间等待，所有输出选项均为SUCCESS则表示为编译通过

##Maven编译后的压缩包在项目的tez-dist/target目录下，没有编译成功是没有target这个目录的
##查看是否成功
[root@node02 apache-tez-0.9.2-src]# cd /data1/software/tez/apache-tez-0.9.2-src/tez-dist/target
[root@node02 apache-tez-0.9.2-src]# ll
drwxr-xr-x 2 root root     6 6月   1 10:46 archive-tmp
drwxr-xr-x 2 root root   183 6月   1 10:46 tez-0.9.2
drwxr-xr-x 2 root root   159 6月   1 10:46 tez-0.9.2-minimal
-rw-r--r-- 1 root root 18642 6月   1 10:46 tez-0.9.2-minimal.tar.gz
-rw-r--r-- 1 root root 20024 6月   1 10:46 tez-0.9.2.tar.gz

## 7、配置

上传到hdfs路径

[root@node02 target]# su hdfs
bash-4.2$ hdfs dfs -mkdir /tez
bash-4.2$ hdfs dfs -put tez-0.9.2.tar.gz /tez
配置客户端（每台集群机子都需要配置，切记，弄好了也可以scp过去）

在CDH里的lib目录生成tez目录

[root@node02 target]# mkdir -p /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/conf
生成tez_site.xml文件

[root@node02 target]# vim /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/conf/tez-site.xml
##一定要加${fs.defaultFS}，要不就找不到，这个就是第一步上传到hdfs的压缩包路径

```
<configuration>
 <property>
  <name>tez.lib.uris</name>
  <value>${fs.defaultFS}/tez/tez-0.9.2.tar.gz</value>
 </property>
 <property>
  <name>tez.use.cluster.hadoop-libs</name>
  <value>false</value>
 </property>
 <property>
  <name>tez.am.launch.env</name>
  <value>LD_LIBRARY_PATH=${PARCELS_ROOT}/CDH/lib/hadoop/lib/native</value>
 </property>
 <property>
  <name>tez.task.launch.env</name>
  <value>LD_LIBRARY_PATH=${PARCELS_ROOT}/CDH/lib/hadoop/lib/native</value>
 </property>
</configuration>
```


拷贝jar包和lib目录

```
[root@node02 target]# cd tez-0.9.2-minimal
[root@node02 tez-0.9.2-minimal]# cp ./*.jar /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/
[root@node02 tez-0.9.2-minimal]# cp -r ./lib /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/
```


避免kryo的错误java.lang.ClassNotFoundException: com.esotericsoftware.kryo.Serializer

```
[root@node02 tez-0.9.2-minimal]# cd /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/hive/auxlib
[root@node02 auxlib]# mv hive-exec-2.1.1-cdh6.3.2-core.jar hive-exec-2.1.1-cdh6.3.2-core.jar.bak
[root@node02 auxlib]# mv hive-exec-core.jar hive-exec-core.jar.bak
[root@node02 auxlib]# cp /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/kryo-2.22.jar /opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/lib/
```

tez/lib中包含slf4j的jar包，会打印较多日志，可以在客户端中去掉slf4j-api-1.7.10.jar、slf4j-log4j12-1.7.10.jar这两个jar包，减少日志打印，

配置HADOOP_CLASSPATH路径（这个在cdh上面配置就好），路径为hive-配置-gateway，hive-env.sh 的 Gateway 客户端环境高级配置代码段（安全阀），内容如下：

```
HADOOP_CLASSPATH=/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/conf:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/*:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/tez/lib/*
```


然后保存并部署客户端配置，这样配置的环境变量才会生效。

![image-20230601114530352](/images/image-20230601114530352.png)

（此步无要求可以忽略）JDBC客户端无法连接，需要在Hive配置hive-site.xml 的 Hive 服务高级配置代码段

#插入一下代码
tez.lib.uris
${fs.defaultFS}/tez/tez-0.9.2.tar.gz

tez.use.cluster.hadoop-libs
false

若是还报异常，需要吧之前整理的tez包放进/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/hive/lib/。以下是我放的包
-rw-r--r-- 1 root root   58160 May 11 17:07 commons-codec-1.4.jar
-rw-r--r-- 1 root root   41123 May 11 17:07 commons-cli-1.2.jar
-rw-r--r-- 1 root root  535731 May 11 17:07 async-http-client-1.8.16.jar
-rw-r--r-- 1 root root  588337 May 11 17:07 commons-collections-3.2.2.jar
-rw-r--r-- 1 root root  751238 May 11 17:07 commons-collections4-4.1.jar
-rw-r--r-- 1 root root 1599627 May 11 17:07 commons-math3-3.1.1.jar
-rw-r--r-- 1 root root 1648200 May 11 17:07 guava-11.0.2.jar
-rw-r--r-- 1 root root  770504 May 11 17:07 hadoop-mapreduce-client-common-3.0.0-cdh6.3.2.jar
-rw-r--r-- 1 root root 1644597 May 11 17:07 hadoop-mapreduce-client-core-3.0.0-cdh6.3.2.jar
-rw-r--r-- 1 root root   81743 May 11 17:07 jettison-1.3.4.jar
-rw-r--r-- 1 root root  147952 May 11 17:07 jersey-json-1.9.jar
-rw-r--r-- 1 root root    5450 May 11 17:07 hadoop-shim-2.7-0.9.2.jar
-rw-r--r-- 1 root root    8803 May 11 17:07 hadoop-shim-0.9.2.jar
-rw-r--r-- 1 root root  458642 May 11 17:07 jetty-util-9.3.24.v20180605.jar
-rw-r--r-- 1 root root  520813 May 11 17:07 jetty-server-9.3.24.v20180605.jar
-rw-r--r-- 1 root root    8866 May 11 17:07 slf4j-log4j12-1.7.10.jar
-rw-r--r-- 1 root root   32119 May 11 17:07 slf4j-api-1.7.10.jar
-rw-r--r-- 1 root root  105112 May 11 17:07 servlet-api-2.5.jar
-rw-r--r-- 1 root root  206280 May 11 17:07 RoaringBitmap-0.5.21.jar
-rw-r--r-- 1 root root    5362 May 11 17:07 tez-build-tools-0.9.2.jar
-rw-r--r-- 1 root root 1032647 May 11 17:07 tez-api-0.9.2.jar
-rw-r--r-- 1 root root   85864 May 11 17:07 tez-common-0.9.2.jar
-rw-r--r-- 1 root root   56795 May 11 17:07 tez-examples-0.9.2.jar
-rw-r--r-- 1 root root 1412824 May 11 17:07 tez-dag-0.9.2.jar
-rw-r--r-- 1 root root   76042 May 11 17:07 tez-protobuf-history-plugin-0.9.2.jar
-rw-r--r-- 1 root root  296849 May 11 17:07 tez-mapreduce-0.9.2.jar
-rw-r--r-- 1 root root   78934 May 11 17:07 tez-job-analyzer-0.9.2.jar
-rw-r--r-- 1 root root   15265 May 11 17:07 tez-javadoc-tools-0.9.2.jar
-rw-r--r-- 1 root root   79097 May 11 17:07 tez-history-parser-0.9.2.jar
-rw-r--r-- 1 root root  198892 May 11 17:07 tez-runtime-internals-0.9.2.jar
-rw-r--r-- 1 root root    7741 May 11 17:07 tez-yarn-timeline-history-with-acls-0.9.2.jar
-rw-r--r-- 1 root root   28162 May 11 17:07 tez-yarn-timeline-history-0.9.2.jar
-rw-r--r-- 1 root root  158966 May 11 17:07 tez-tests-0.9.2.jar
-rw-r--r-- 1 root root  768820 May 11 17:07 tez-runtime-library-0.9.2.jar
附上JDBC代码示例：

package com.test.controller;

import java.sql.*;
import java.util.Properties;

/**

* @Author xxxx

* @Date 2022-04-27 14:46

* @Version 1.0
  */
  public class TEZTestTwo {
  public static void main(String[] args) throws SQLException {
  String driverName = "org.apache.hive.jdbc.HiveDriver";
  try {
  Class.forName(driverName);
  } catch (ClassNotFoundException e) {
  e.printStackTrace();
  System.exit(1);
  }
  //*hiverserver 版本jdbc url格式*//*
  //      Connection con = DriverManager.getConnection("jdbc:hive://ip:10000/default", "", "");
  // *hiverserver2 版本jdbc url格式*//*
  //        Connection con = DriverManager.getConnection("jdbc:hive2://ip:10000/default", "root", "");
  Properties properties = new Properties();
  properties.setProperty("user","root");
  properties.setProperty("password","");

      //这段和以下参数设置重叠，这两种都可以

  /*properties.setProperty("hiveconf:hive.execution.engine","tez");
  properties.setProperty("hiveconf:tez.job.name","lytez");
  //设置队列
  prop.setProperty("hiveconf:tez.queue.name", "root.default");
  //这个不一定要设置，不行的话再设置
  properties.setProperty("hiveconf:hive.tez.container.size","1024");**/
  Connection con = DriverManager.getConnection("jdbc:hive2://ip:10000/default", properties);
  Statement stmt = con.createStatement();
  //参数设置测试
  boolean resHivePropertyTest = stmt.execute("set hive.execution.engine=tez");
  //这个是yarn显示名称，这个需要修改源代码打包成：，替换：/opt/cloudera/parcels/CDH/jars/目录的hive-exec-2.1.1-cdh6.3.2.jar
  boolean resHivePropertyTestName = stmt.execute("set tez.job.name=test");
  //这个不一定要设置，不行的话再设置
  boolean resHivePropertyTestContainer = stmt.execute("set hive.tez.container.size=1024");
  //cdh设置动态资源池
  boolean setHivePropertyQUEUE = stmt.execute("set tez.queue.name=root.default");
  System.out.println("resHivePropertyTestContainer="+resHivePropertyTestContainer);
  System.out.println("resHivePropertyTest="+resHivePropertyTest);
  String tableName = "test";
  ResultSet res = stmt.executeQuery("select count(1) AS a from " + tableName);
  while (res.next()) {
  System.out.println(res.getString("a"));
  }
  /*tableName="area_type";
  res = stmt.executeQuery("select count(1) AS a from " + tableName);
  while (res.next()) {
  System.out.println(res.getString("a"));
  }*/
  }
  }

## 8、启动

hive> set hive.execution.engine=tez;
hive> set hive.tez.container.size=1024;

第一次查询的时候要加载jar包，所以比较慢。耐心点就好。示例：

hive> select count(1) from cwqejg_test2 ;
