#ambari离线安装以及hadoop环境搭建详细过程
##一、安装环境
六台相同配置的虚拟机
OS:CentOS release 6.5 (Final)(x86_64)
Cores (CPU):8 (8)
Disk:50GB 
Memory:8GB

最好先安装自己的jdk，配置好环境变量。

##二、安装包下载
安装包过大(最大的hdp包3GB)，虚拟机环境外网网络不稳定，在线安装太耗时，只能用制作离线源方式进行安装，so，先下载以下安装包，备用。
http://public-repo-1.hortonworks.com/ambari/centos6/ambari-1.7.0-centos6.tar.gz
http://public-repo-1.hortonworks.com/HDP/centos6/HDP-2.2.0.0-centos6-rpm.tar.gz
http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.20/repos/centos6/HDP-UTILS-1.1.0.20-centos6.tar.gz


##三、配置hostname以及hosts
以便节点间通讯，hadoop集群管理。

```
vi /etc/sysconfig/network ，修改 HOSTNAME=master1

vi /etc/hosts
172.16.1.137 master1
172.16.1.138 master2
172.16.1.139 slave1
172.16.1.140 slave2
172.16.1.149 slave3
172.16.1.136 slave4
```

以master1节点为例，其他同理。

##四、关闭iptables，关闭SELinux
避免节点间通讯问题，关闭iptables&SELinux。

```
service iptables stop
setenforce 0
```

##五、安装ntp服务
节点间需要时间同步，否则会失败

```
yum install ntp
service ntpd start
```

##六、ssh无密码登陆配置
只需设置ambari安装节点（这里选取slave4为安装节点）到其他节点的无密码登陆即可。
在slave4上依次执行：

```
ssh-keygen
chmod 700 ~/.ssh  
chmod 600 ~/.ssh/authorized_keys 
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

至此，本机ssh无密码登陆配置成功，可以ssh slave4尝试下是否成功，接下来将生成的authorized_keys复制到其他节点对应位置：

```
scp ~/.ssh/authorized_keys root@master1:/root/.ssh/authorized_keys
scp ~/.ssh/authorized_keys root@master2:/root/.ssh/authorized_keys
scp ~/.ssh/authorized_keys root@slave1:/root/.ssh/authorized_keys
scp ~/.ssh/authorized_keys root@slave2:/root/.ssh/authorized_keys
scp ~/.ssh/authorized_keys root@slave3:/root/.ssh/authorized_keys
```

至此，slave4对其他节点ssh无密码登陆配置成功，可以ssh下对应节点尝试是否成功。

##七、制作离线安装源
选一台机器，作为离线源服务器，这里同样是slave4（ps.为什么都是slave4，主要是之前其他机子都有点问题，手动苦笑ing）。

```
yum install httpd
service httpd start
```

访问slave4的80端口

启动成功，
接着，创建两个目录：

```
mkdir  /var/www/html/ambari
mkdir  /var/www/html/hdp
```

解压ambari-1.7.0-centos6.tar.gz 和HDP-UTILS-1.1.0.20-centos6.tar.gz 到目录 /var/www/html/ambari 
命令：

```
tar  zxvf  ambari-1.7.0-centos6.tar.gz  -C /var/www/html/ambari
tar  zxvf  HDP-UTILS-1.1.0.20-centos6.tar.gz  -C /var/www/html/ambari
```

进入/var/www/html/ambari 目录，执行命令：

```
createrepo  ./ 
```

ambari本地源制作完成

解压HDP-2.2.0.0-centos6-rpm.tar.gz 到目录 /var/www/html/hdp
命令：`tar  zxvf  HDP-2.2.0.0-centos6-rpm.tar.gz  -C /var/www/html/hdp`
进入/var/www/html/hdp 目录，执行命令：

```
createrepo  ./ 
```

 
hadoop本地源制作完成

进入http://slave4/ambari/

以及http://slave4/hdp/

至此，离线源制作部分成功。

##八、安装ambari
vim  /etc/yum.repos.d/ambari.repo  添加以下内容

```
[ambari-1.7]
name=Ambari 1.7
baseurl=http://slave4/ambari/
gpgcheck=0
enabled=1

[HDP-UTILS-1.1.0.20]
name=Hortonworks Data Platform Utils Version - HDP-UTILS-1.1.0.20
baseurl=http://slave4/ambari/
gpgcheck=0
enabled=1
```

slave4为上一步制作离线源的主机。
执行安装ambari 命令：

```
yum -y install ambari-server
```

开始配置ambari 服务：

```
ambari-server setup
```

最好选择3，jdk提前自己安装，只需要设置JAVA_HOME的路径即可，否则选择1、2的话，需要通过网络下载，速度很慢。
Success后，可以启动了：

```
ambari-server start
```

##九、给ambari配置本地hadoop
进入目录：/var/lib/ambari-server/resources/stacks/HDP/2.2/repos

```
vi repoinfo.xml
<os type="redhat6">
    <repo>
      <baseurl>http://slave4/hdp</baseurl>
      <repoid>HDP-2.2</repoid>
      <reponame>HDP</reponame>
    </repo>
    <repo>
      <baseurl>http://slave4/ambari</baseurl>
      <repoid>HDP-UTILS-1.1.0.20</repoid>
      <reponame>HDP-UTILS</reponame>
    </repo>
  </os>
```

修改redhat6中的配置，node3为ambari安装的节点，根据自己安装的实际节点配置
这样在ambari执行hadoop安装时，会将本地源地址配置到所有主机上，在为所有主机安装ambari-agent时，会将ambari节点下的/etc/yum.repo/ambari.repo文件复制到所有主机源地址。

##十、安装hadoop集群环境以及各组件
浏览器进入http://slave4:8080/
账户：admin 密码：admin
接下来一步步按提示即可，有一些要注意的点：
第三步手动设置下redhat6的源地址为之前做的离线源：
redhat6 (HDP-2.2): 
http://slave4/hdp/HDP/centos6/2.x/GA/2.2.0.0/

redhat6 (HDP-UTILS-1.1.0.20): 
http://slave4/ambari/HDP-UTILS-1.1.0.20/repos/centos6/

第四步INSTALL OPTION
配置hosts里的节点hostname
把slave4下生成的私钥导入

然后第五步会检查是否可以跟各节点通讯，可能会失败，查看详情，但无非是防火墙没禁，ntp服务没装之类的，warm尽量全部解决掉，要不然可能会影响后面的安装。

最后第二步，会开始到个节点安装各组件，此处出现问题可能性最大，有两个很明显的问题
①遇到的关于yum的错误(异常当时忘记记录了orz)，无法连接映像地址等，解决办法：
清空&重建yum缓存

```
yum clean
yum	makecache
```

②('ERROR 2015-02-06 20:09:43,441 NetUtil.py:56 - [Errno 1] _ssl.c:492: error:100AE081:elliptic curve routines:EC_GROUP_new_by_curve_name:unknown group
ERROR 2015-02-06 20:09:43,442 NetUtil.py:58 - SSLError: Failed to connect. Please check openssl library versions.
这是openssl版本导致的错误
解决办法：`yum upgrade openssl`

Bingo
