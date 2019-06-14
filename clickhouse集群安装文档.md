# Clickhouse 集群安装文档
Clickhouse 官网支持在 Ubantu 下安装，这里选用的是 Altinity 公司针对 CentOS 更改并适配的 rpm 包。

本次使用系统是 CentOS7.5，并在该系统下，进行 Clickhouse 搭建，节点这里使用 clickhouse79，clickhouse80，clickhouse81（IP或显隐私，略感不适，别名替之）。

# 下面开始介绍具体的安装配置流程

1. 单节点基本安装情况

安装之前，确保 Linux 下有以下工具 unixODBC、curl、epel-release，后面会有涉及，提前安装，命令使用如下即可
```bash 
yum install -y *
```

下载相关的 rpm 包，这里使用 Altinity 给的下载脚本
```bash 
curl -s https://packagecloud.io/install/repositories/altinity/clickhouse/script.rpm.sh | sudo bash
```

确保有以下文件存在，以供安装
```bash
sudo yum list 'clickhouse*'
```
显示如下内容：
```bash
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
Installed Packages
clickhouse-client.x86_64               19.6.2.11-1.el7            @Altinity_clickhouse
clickhouse-common-static.x86_64        19.6.2.11-1.el7            @Altinity_clickhouse
clickhouse-server.x86_64               19.6.2.11-1.el7            @Altinity_clickhouse
clickhouse-server-common.x86_64        19.6.2.11-1.el7            @Altinity_clickhouse
Available Packages
clickhouse-compressor.x86_64           1.1.54336-3.el7            Altinity_clickhouse 
clickhouse-debuginfo.x86_64            19.6.2.11-1.el7            Altinity_clickhouse 
clickhouse-mysql.noarch                0.0.20180319-1             Altinity_clickhouse 
clickhouse-odbc.x86_64                 20180903-1                 Altinity_clickhouse 
clickhouse-test.x86_64                 19.6.2.11-1.el7            Altinity_clickhouse
```

使用 yum 进行安装
```bash
sudo yum install -y clickhouse-server clickhouse-client
```
再重启下服务
```bash
sudo /etc/init.d/clickhouse-server restart
```
这是便可进入客户端（用户名，密码是默认的，不用输入，后面介绍如何添加到配置文件）
```bash
clickhouse-client
```
显示如下：
```bash
ClickHouse client version 19.6.2.1.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 19.6.2 revision 54418.

clickhouse79 :) 
```
以上是在单节点上安装启动 Clickhouse，上面步骤重复在 clickhouse80 和 clickhouse81 上执行，截至当前，三个服务是各自独立的。

2. 下面是针对集群，修改相关配置

配置前，需要将三个节点实现免密登陆，主要将公钥添加到.ssh/authorized_keys，如有需要，自行搜索相关文档，这里不再详细描述。并且，还要配置 zookeeper 服务，这里受资源限制暂配置一个 zk，集群配置的步骤在附录。

以下文件均在 /etc/clickhouse-server 目录下
首先，创建 metrika.xml，并添加以下内容：
```bash
<yandex>
    <macros>
        <shard>1</shard>
        <replica>clickhouse79</replica>
    </macros>
    <zookeeper-servers>
        <node index="1">
            <host>172.28.5.188</host>
            <port>2181</port>
        </node>
    </zookeeper-servers>
    <clickhouse_remote_servers>
            <cluster_3shards_1replicas>
                <shard>
                    <replica>
                      <host>clickhouse79</host>
                      <port>9000</port>
                    </replica>
                </shard>
                <shard>
                    <replica>
                      <host>clickhouse80</host>
                      <port>9000</port>
                    </replica>
                </shard>
                <shard>
                    <replica>
                      <host>clickhouse81</host>
                      <port>9000</port>
                    </replica>
                </shard>
            </cluster_3shards_1replicas>
    </clickhouse_remote_servers>
</yandex>
```
上面，```<macros>```：宏配置，用于分布式建表时做替换，其中 replica 属性每个节点是不一样的；```<zookeeper-servers>```：用作Clickhouse集群同步数据，保证一致性；```<cluster_3shards_1replicas>```：配置数据分片，这里是3个分片的集群，每个分片有一个副本。

其次，在 config.xml 文件中添加引入 metrika.xml 文件的配置参数，注意：这里添加是在```<yandex>```下一级，否则可能无法生效。
```bash
<include_from>/etc/clickhouse-server/metrika.xml</include_from>
```
同时，在该配置文件中添加监听，方便后期程序连接处理数据。
```bash
<listen_host>0.0.0.0</listen_host>
```
最后，在 user.xml 下更改配置
```bash
<users>
<!-- If user name was not specified, 'default' user is used. -->
    <default>
        <password></password>
        <ip>::/0</ip>
        <!-- List of networks with open access.
             To open access from everywhere, specify:
                <ip>::/0</ip>
             To open access only from localhost, specify:
                <ip>::1</ip>
                <ip>127.0.0.1</ip>
             Each element of list has one of the following forms:
             <ip> IP-address or network mask. Examples: 213.180.204.3 or 10.0.0.1/8 or 10.0.0.1/255.255.255.0
                 2a02:6b8::3 or 2a02:6b8::3/64 or 2a02:6b8::3/ffff:ffff:ffff:ffff::.
             <host> Hostname. Example: server01.yandex.ru.
                 To check access, DNS query is performed, and all received addresses compared to peer address.
             <host_regexp> Regular expression for host names. Example, ^server\d\d-\d\d-\d\.yandex\.ru$
                 To check access, DNS PTR query is performed for peer address and then regexp is applied.
                 Then, for result of PTR query, another DNS query is performed and all received addresses compared to peer address.
                 Strongly recommended that regexp is ends with $
             All results of DNS requests are cached till server restart.
        -->
        <networks incl="networks" replace="replace">
            <ip>::/0</ip>
        </networks>
    </default>
</users>
```
上面中主要添加``` <ip> ```及```<networks> ```，同 config.xml 种 ```<listen_host>```，用作程序在其他节点上连接处理数据。当然还有其他的配置，这里简单介绍一些，如```<default>```：这个即为默认用户名，可以更改为其他指定用户名，登陆时便要指定才可登陆；```<password>```：内部添加具体密码，也是用作登陆使用；还有未列出的参数如```<readonly>```配置只读权限等，这里不再过多列举说明。

配置完成后，重启下服务，即可生效。
Clickhouse 后续使用的注意事项稍后更新。

# 附录

1. zookeeper 配置安装

首先，下载 zookeeper 安装包
```bash
wget http://tools.mip.ipinyou.com/static/zookeeper-3.4.14.tar.gz 
 ```
其次，在解压后的问件下，进入 conf 下，将 zoo_sample.cfg 改名为 zoo.cfg，该配置在三个节点相同，并在配置文件中添加如下：
```bash
dataDir=/data/zk/zk-data
dataLogDir=/data/zk/zk-logs
# 三个接点配置，格式为：
# server.服务编号=服务地址、LF通信端口、选举端口
 server.1=salve1:2888:3888
 server.2=slave2:2888:3888
 server.3=slave3:2888:3888
```
并在不同节点下```/data/zk/zk-data```目录下，创建 myid 文件，再分别写入节点标记 1、2、3 ，它们分别对应上面三个服务编号的服务地址。
之后，进入 bin 目录下，执行命令服务```./zkServer.sh status ```查看主从状态。
启动后，执行```./zkCli.sh ```可进入客户端，并查看。

2. Altinity-ClickHouse 要求我们添加 Altinity repo，在上面 script.rpm.sh 脚本已包含下面执行流程，用途主要是安装时，将所需要的依赖添加进来，方便安装时加入所需的文件，类似 Maven 添加依赖，安装完毕后，不必再关注。
首先，需要添加一些工具，方便后续使用
``` sudo yum install -y pygpgme yum-utils coreutils epel-release ```
其次，创建配置文件```/etc/yum.repos.d/Altinity_clickhouse.repo```，针对 CentOS7，具体如下：
```bash
[Altinity_clickhouse]
name=Altinity_clickhouse
baseurl=https://packagecloud.io/Altinity/clickhouse/el/7/$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/Altinity/clickhouse/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300

[Altinity_clickhouse-source]
name=Altinity_clickhouse-source
baseurl=https://packagecloud.io/Altinity/clickhouse/el/7/SRPMS
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/Altinity/clickhouse/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
```
最后，更新 cache
```bash 
sudo yum -q makecache -y --disablerepo='*' --enablerepo='Altinity_clickhouse'
```
