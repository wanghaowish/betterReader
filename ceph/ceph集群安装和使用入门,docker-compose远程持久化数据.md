## ceph

#### 1.环境准备

准备5台虚拟机（centos7.5）

主机名 | ip | role
---|---|---
admin | 172.21.33.235 | ceph-deploy 管理多个节点
node1 | 172.21.33.236 |  minitor 维护着展示集群状态的各种图表
node2 | 172.21.33.237|  osd
node3 | 172.21.33.238 |  osd
client | 172.21.33.239 |  rgw，rbd

#### 2.预检

##### 2.1安装 CEPH 部署工具
```
1.admin 安装 CEPH 部署工具，把 Ceph 仓库添加到 ceph-deploy 管理节点，然后安装 ceph-deploy 。
sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*

2.把软件包源加入软件仓库。用文本编辑器创建一个 YUM (Yellowdog Updater, Modified) 库文件，其路径为 /etc/yum.repos.d/ceph.repo 
sudo vim /etc/yum.repos.d/ceph.repo

把如下内容粘帖进去，用 Ceph 的主稳定版名字替换 {ceph-stable-release} （如: mimic），用你的Linux发行版名字替换 {distro} （如 el6 为 CentOS 6 、 el7 为 CentOS 7 、 rhel6 为 Red Hat 6.5 、 rhel7 为 Red Hat 7 、 fc19 是 Fedora 19 、 fc20 是 Fedora 20 ）。最后保存到 /etc/yum.repos.d/ceph.repo 文件中。
[ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-{ceph-release}/{distro}/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc


3.更新软件库并安装 ceph-deploy 
sudo yum update && sudo yum install ceph-deploy
```

##### 2.2 CEPH 节点安装

###### 安装 NTP
```
1.管理节点必须能够通过 SSH 无密码地访问各 Ceph 节点。如果 ceph-deploy 以某个普通用户登录，那么这个用户必须有无密码使用 sudo 的权限。
建议在所有 Ceph 节点上安装 NTP 服务（特别是 Ceph Monitor 节点），以免因时钟漂移导致故障。
sudo yum install ntp ntpdate ntp-doc
安装完成后设置ntp开机启动并启动ntp，如下：
sudo systemctl enable ntpd
sudo systemctl start ntpd


```
具体配置时间服务器同步时间参考[时间服务器同步时间](https://blog.csdn.net/qq_38591756/article/details/85243965)
###### 安装 SSH 服务器
在所有 Ceph 节点上执行如下步骤：

```
在各 Ceph 节点安装 SSH 服务器（如果还没有）：
sudo yum install openssh-server
```
###### 创建部署 CEPH 的用户
ceph-deploy 工具必须以普通用户登录 Ceph 节点，且此用户拥有无密码使用 sudo 的权限，因为它需要在安装软件及配置文件的过程中，不必输入密码。

较新版的 ceph-deploy 支持用 --username 选项提供可无密码使用 sudo 的用户名（包括 root ，虽然不建议这样做）。使用 ceph-deploy --username {username} 命令时，指定的用户必须能够通过无密码 SSH 连接到 Ceph 节点，因为 ceph-deploy 中途不会提示输入密码。

我们建议在集群内的所有 Ceph 节点上给 ceph-deploy 创建一个特定的用户，但不要用 “ceph” 这个名字。全集群统一的用户名可简化操作（非必需），然而你应该避免使用知名用户名，因为黑客们会用它做暴力破解（如 root 、 admin 、 {productname} ）。后续步骤描述了如何创建无 sudo 密码的用户，你要用自己取的名字替换 {username} 。

```
1.在各 Ceph 节点创建新用户
sudo useradd -d /home/{username} -m {username}
sudo passwd {username}

2.确保各 Ceph 节点上新创建的用户都有 sudo 权限
echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
sudo chmod 0440 /etc/sudoers.d/{username}
```
###### 允许无密码 SSH 登录
正因为 ceph-deploy 不支持输入密码，你必须在管理节点上生成 SSH 密钥并把其公钥分发到各 Ceph 节点。 ceph-deploy 会尝试给初始 monitors 生成 SSH 密钥对。


```
1.生成 SSH 密钥对，但不要用 sudo 或 root 用户。提示 “Enter passphrase” 时，直接回车，口令即为空：
ssh-keygen

Generating public/private key pair.
Enter file in which to save the key (/ceph-admin/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /ceph-admin/.ssh/id_rsa.
Your public key has been saved in /ceph-admin/.ssh/id_rsa.pub.

注意：要使用主机名代替ip，先在管理节点修改/etc/hosts 
注意：确保各主机名与 admin node1 node2 node3 client一致
vim /etc/hosts 

如下：
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.21.33.228 admin
172.21.33.229 node1
172.21.33.230 node2
172.21.33.231 node3
172.21.33.231 client


2.把公钥拷贝到各 Ceph 节点，把下列命令中的 {username} 替换成前面创建部署 Ceph 的用户里的用户名。
ssh-copy-id {username}@node1
ssh-copy-id {username}@node2
ssh-copy-id {username}@node3
172.21.33.231 {username}@client




3.（推荐做法）修改 ceph-deploy 管理节点上的 ~/.ssh/config 文件，这样 ceph-deploy 就能用你所建的用户名登录 Ceph 节点了，而无需每次执行 ceph-deploy 都要指定 --username {username} 。这样做同时也简化了 ssh 和 scp 的用法。

vim ~/.ssh/config
如下：
Host node1
   Hostname node1
   User node1
Host node2
   Hostname node2
   User node2
Host node3
   Hostname node3
   User node3
Host client
   Hostname client
   User client
   
user为各节点创建的用户
```
###### 引导时联网

Ceph 的各 OSD 进程通过网络互联并向 Monitors 上报自己的状态。如果网络默认为 off ，那么 Ceph 集群在启动时就不能上线，直到你打开网络。

某些发行版（如 CentOS ）默认关闭网络接口。所以需要确保网卡在系统启动时都能启动，这样 Ceph 守护进程才能通过网络通信。例如，在 Red Hat 和 CentOS 上，需进入 /etc/sysconfig/network-scripts 目录并确保 ifcfg-{iface} 文件中的 ONBOOT 设置成了 yes 。

###### 确保联通性
用 ping 短主机名（ hostname -s ）的方式确认网络联通性。解决掉可能存在的主机名解析问题。

主机名应该解析为网络 IP 地址，而非回环接口 IP 地址（即主机名应该解析成非 127.0.0.1 的IP地址）。如果你的管理节点同时也是一个 Ceph 节点，也要确认它能正确解析自己的主机名和 IP 地址（即非回环 IP 地址）。

###### 开放所需端口
Ceph Monitors 之间默认使用 6789 端口通信， OSD 之间默认用 6800:7300 这个范围内的端口通信。 Ceph OSD 能利用多个网络连接进行与客户端、monitors、其他 OSD 间的复制和心跳的通信。

某些发行版（如 RHEL ）的默认防火墙配置非常严格，你可能需要调整防火墙，允许相应的入站请求，这样客户端才能与 Ceph 节点上的守护进程通信。

```
1.monitor节点
sudo firewall-cmd --zone=public --add-port=6789-7300/tcp --permanent
2.osd节点
sudo firewall-cmd --zone=public --add-port=6800-7300/tcp --permanent
3.配置生效
sudo firewall-cmd --reload
```

###### 关闭 SELinux
在 CentOS 和 RHEL 上， SELinux 默认为 Enforcing 开启状态。为简化安装，我们建议把 SELinux 设置为 Permissive 或者完全禁用，也就是在加固系统配置前先确保集群的安装、配置没问题。用下列命令把 SELinux 设置为 Permissive ：

```
sudo setenforce 0
```

###### 优先级/首选项

```
确保你的包管理器安装了优先级/首选项包且已启用。在 CentOS 上你也许得安装 EPEL 
sudo yum install yum-plugin-priorities
```



#### 3.存储集群
第一次练习时，我们创建一个 Ceph 存储集群，它有一个 Monitor 和两个 OSD 守护进程。一旦集群达到 active + clean 状态，再扩展它：增加第三个 OSD 、增加元数据服务器和两个 Ceph Monitors。为获得最佳体验，先在管理节点上创建一个目录，用于保存 ceph-deploy 生成的配置文件和密钥对。


```
mkdir my-cluster
cd my-cluster
```

ceph-deploy 会把文件输出到当前目录，所以请确保在此目录下执行 ceph-deploy 。
**注意：在某些发行版（如 CentOS ）上，执行 ceph-deploy 命令时，如果你的 Ceph 节点默认设置了 requiretty 那就会遇到报错。可以这样禁用此功能：执行 sudo visudo ，找到 Defaults requiretty 选项，把它改为 Defaults:ceph !requiretty ，这样 ceph-deploy 就能用 ceph 用户登录并使用 sudo 了。**

##### 1.创建集群
如果在某些地方碰到麻烦，想从头再来，可以用下列命令清除配置：

```
ceph-deploy purgedata {ceph-node} [{ceph-node}]
ceph-deploy forgetkeys

如果运行ceph-deploy命令出现以下错:

Traceback (most recent call last):
  File "/usr/bin/ceph-deploy", line 18, in <module>
    from ceph_deploy.cli import main
  File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
    import pkg_resources
ImportError: No module named pkg_resources

原因是缺python-setuptools，安装它即可：
yum install python-setuptools
```
用下列命令可以连 Ceph 安装包一起清除：

```
ceph-deploy purge {ceph-node} [{ceph-node}]
```
如果执行了 purge ，你必须重新安装 Ceph 。

在管理节点上，进入刚创建的放置配置文件的目录，用 ceph-deploy 执行如下步骤。

1. 创建集群。

```
ceph-deploy new {initial-monitor-node(s)}
```
例如：
```
ceph-deploy new node1
```
在当前目录下用 ls 和 cat 检查 ceph-deploy 的输出，应该有一个 Ceph 配置文件、一个 monitor 密钥环和一个日志文件（ceph.conf  ceph.mon.keyring ceph-deploy-ceph.log
）。详情见 ceph-deploy new -h 。

2. 把 Ceph 配置文件里的默认副本数从 3 改成 2 ，这样只有两个 OSD 也可以达到 active + clean 状态。把下面这行加入 [global] 段：

```
vim ceph.conf
osd_pool_default_size = 2
```
3. 如果你有多个网卡，可以把 public network 写入 Ceph 配置文件的 [global] 段下。

```
public network = {ip-address}/{netmask}
```

4. 安装Ceph

```
ceph-deploy install {ceph-node} [{ceph-node} ...]
```
例如：
```
 ceph-deploy install admin node1 node2 node3 client
 如果出现以下报错：
 RuntimeError: NoSectionError: No section: 'ceph'
执行下面命令：
 yum remove ceph-release

测试是否安装完成：分别在admin node1 node2 node3 client 查看版本，当前安装为 13.2.10

ceph --version

ceph-deploy 将在各节点安装 Ceph 。 注：如果你执行过 ceph-deploy purge ，你必须重新执行这一步来安装 Ceph 。
```
5. 配置初始 monitor(s)、并收集所有密钥：

```
ceph-deploy mon create-initial

完成上述操作后，当前目录里应该会出现这些密钥环：
ceph.bootstrap-mds.keyring  
ceph.bootstrap-mgr.keyring  
ceph.bootstrap-osd.keyring  
ceph.bootstrap-rgw.keyring  
ceph.client.admin.keyring  
ceph.mon.keyring

注意：如果此步失败并输出类似于如下信息 “Unable to find /etc/ceph/ceph.client.admin.keyring”，请确认 ceph.conf 中为 monitor 指定的 IP 是 Public IP，而不是 Private IP。
```

6. 添加两个 OSD 。（本文档为12.2.10以后的版本，中文文档提供的版本太旧，添加osd的
ceph-deploy osd  prepare/active 命令已不可用）


```
查看各节点的磁盘分区和大小
sudo fdisk -l

格式化磁盘：
ceph-deploy disk zap node2 /dev/sda1
ceph-deploy disk zap node3 /dev/sda1

添加osd:
ceph-deploy osd create --data /dev/sda1 node2
ceph-deploy osd create --data /dev/sda1 node3

删除：
ceph osd out  osd.0
ceph osd rm   osd.0
ceph osd crush remove osd.0
ceph auth del osd.0

```

7. 用 ceph-deploy 把配置文件和 admin 密钥拷贝到管理节点和 Ceph 节点，这样每次执行 Ceph 命令行时就无需指定 monitor 地址和 ceph.client.admin.keyring 了。

```
ceph-deploy admin {admin-node} {ceph-node}
```
例如：

```
ceph-deploy admin admin node1 node2 node3 client 
```

8.确保你对 ceph.client.admin.keyring 有正确的操作权限。

```
sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

9.检查集群的健康状况。

```
ceph health
```

##### 2.操作集群

要在 Ceph 节点上启动特定类型的所有守护进程，请执行以下操作之一：

```
sudo systemctl start ceph-osd.target
sudo systemctl start ceph-mon.target
sudo systemctl start ceph-mds.target
```

按类型停止所有守护进程

```
sudo systemctl stop ceph-mon\*.service ceph-mon.target
sudo systemctl stop ceph-osd\*.service ceph-osd.target
sudo systemctl stop ceph-mds\*.service ceph-mds.target
```

要在 Ceph 节点上启动特定的守护程序实例，请执行以下操作之一：

```
sudo systemctl start ceph-osd@{id}
sudo systemctl start ceph-mon@{hostname}
sudo systemctl start ceph-mds@{hostname}
```
例如：

```
sudo systemctl start ceph-osd@1
sudo systemctl start ceph-mon@ceph-server
sudo systemctl start ceph-mds@ceph-server
```
要停止 Ceph 节点上的特定守护程序实例，请执行以下操作之一：

```
sudo systemctl stop ceph-osd@{id}
sudo systemctl stop ceph-mon@{hostname}
sudo systemctl stop ceph-mds@{hostname}
```
例如：

```
sudo systemctl stop ceph-osd@1
sudo systemctl stop ceph-mon@ceph-server
sudo systemctl stop ceph-mds@ceph-server
```







##### 3.存入/检出对象数据
要把对象存入 Ceph 存储集群，客户端必须做到：
指定对象名，指定存储池



```
查看现有pool列表
ceph osd lspools
创建pool

通常在创建pool之前，需要覆盖默认的pg_num，官方推荐：

若少于5个OSD， 设置pg_num为128。

5~10个OSD，设置pg_num为512。

10~50个OSD，设置pg_num为4096。

超过50个OSD，可以参考pgcalc计算。

本文的测试环境只有2个OSD，因此选择64个

创建语法:
ceph osd pool create {pool-name} {pg-num} [{pgp-num}] [replicated] \
     [crush-ruleset-name] [expected-num-objects]
ceph osd pool create {pool-name} {pg-num}  {pgp-num}   erasure \
     [erasure-code-profile] [crush-ruleset-name] [expected_num_objects]
     
例如：
ceph osd pool create test-pool 64 64

创建池后必须管理应用，否则查看集群状态会提示 ：application not enabled on 1 pool
命令：ceph osd pool application enable {pool-name} {application-name}
CephFS 使用应用程序名称cephfs，RBD 使用应用程序名称rbd，RGW 使用应用程序名称rgw。
例如：
ceph osd pool application enable test-pool cephfs

- 删除池：
删除池前在 ，在管理节点打开/etc/ceph/ceph.conf，增加以下
mon_allow_pool_delete = true
覆盖其他节点中的配置文件
ceph-deploy --overwrite-conf admin  node1 node2 node3 client 
然后重启：
systemctl restart ceph-mon@node1.service
执行删除池命令 ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]：
ceph osd pool delete test-pool test-pool --yes-i-really-really-mean-it


- 创建一个对象，用 rados put命令加上对象名、一个有数据的测试文件路径、并指定存储池
echo {Test-data} > testfile.txt
rados put test-object-1 testfile.txt --pool=test-pool

- 为确认 Ceph 存储集群存储了此对象，可执行：
rados -p test-pool ls

- 定位对象：
ceph osd map test-pool test-object-1

Ceph 应该会输出对象的位置，例如：Ceph 应该会输出对象的位置，例如：
osdmap e20 pool 'test-pool' (1) object 'test-object-1' -> pg 1.74dc35e2 (1.62) -> up ([0,1], p0) acting ([0,1], p0)

- 用``rados rm`` 命令可删除此测试对象，例如：
rados rm test-object-1 --pool=test-pool
```


**随着集群的运行，对象位置可能会动态改变。 Ceph 有动态均衡机制，无需手动干预即可完成。**


#### 4.启用控制台

1.查看 ceph 状态
```
ceph-deploy mgr create node1
ceph -s
输出包含：
 mgr: node1(active)
```

2.修改配置文件 

```
vim ceph.conf
添加下面一行：
mgr modules = dashboard

```

3.启用dashbord

```
ceph mgr module enable dashboard
```

4.禁用ssl

```
ceph config set mgr mgr/dashboard/ssl false
```

5.配置服务地址、端口，如果不配置，禁用ssl后的默认端口是8080

```
ceph config set mgr mgr/dashboard/server_addr 172.21.33.236
ceph config set mgr mgr/dashboard/server_port 7000

```
6.创建一个用户、密码

```
ceph dashboard set-login-credentials admin 123456
```
7.重启mgr

```
systemctl restart ceph-mgr@node1
```
8.访问

```
ceph mgr services
输出：
{
 "dashboard": "http://172.21.33.229:7000/"
}
```

#### 5.CEPH 文件系统
1.在管理节点上，通过 ceph-deploy 把 Ceph 安装到 client 节点上。

```
ceph-deploy install client
```
2.至少需要一个元数据服务器才能使用 CephFS ，执行下列命令创建元数据服务器：

```
ceph-deploy mds create node1
ceph osd pool create metadata 64 64
ceph osd pool create data 64 64
ceph fs new cephfs metadata data
```
3.确保 Ceph 存储集群在运行，且处于 active + clean 状态。同时，确保至少有一个 Ceph 元数据服务器在运行。

```
ceph -s

输出：
  cluster:
    id:     3b81a00c-1793-42fb-ad83-197d48a4c6db
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum node1
    mgr: node1(active) --
    mds: cephfs-1/1/1 up  {0=node1=up:active} --元数据服务器
    osd: 2 osds: 2 up, 2 in

  data:
    pools:   3 pools, 250 pgs
    objects: 24  objects, 33 KiB
    usage:   731 MiB used, 1.3 GiB / 2.0 GiB avail
    pgs:     250 active+clean

```

4.Ceph 存储集群默认启用认证，你应该有个包含密钥的配置文件（但不是密钥环本身）。用下述方法获取某一用户的密钥：

```
cat ceph.client.admin.keyring

输出：

[client.admin]
        key = AQCTy71gwuE0ChAAe3QqoJ5tF0TX9tyczxZzpQ==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"

```

6.把 Ceph FS 挂载为内核驱动。

```
登录到client节点：
ssh client
sudo mkdir /mnt/cephfs
sudo mount -t ceph 172.21.33.236:6789:/ /mnt/cephfs -o name=admin,secret=AQCTy71gwuE0ChAAe3QqoJ5tF0TX9tyczxZzpQ==

sudo df -h /mnt/cephfs/
输出:
172.21.33.229:6789:/  608M     0  608M    0% /mnt/cephfs

```

#### 6.块设备

1.先在admin节点创建rbd专用池，并且初始化，以及关联rbd应用

```
ceph osd pool create rbd-pool 64 64
rbd pool init rbd-pool
ceph osd pool application enable rbd-pool rbd
ceph osd lspools
输出：
11 rbd-pool

```
#### **接下来的操作都在client节点进行**


2.创建块设备:

```
rbd create --size {megabytes} {pool-name}/{image-name}
例如指定池为rbd-pool，名称为rnb-image的块设备：
rbd create --size 256 rbd-pool/rbd-image --image-feature layering
注意：--image-feature layering，是因为centos7 kernel不支持块设备镜像的一些特性，只支持layering
列出池中块设备：
rbd ls rbd-pool

查看块设备的信息：
rbd info rbd-pool/rbd-image

```
3.映射块设备

```
sudo rbd map rbd-pool/rbd-image 
输出：
/dev/rbd0    # 注：rbd0 说明这是映射的第一个块设备

rbd showmapped
输出:
id pool     image     snap device
0  rbd-pool rbd-image -    /dev/rbd0

事实上，创建的块设备映像，就在 /dev/rbd/{pool-name}下：
cd /dev/rbd/rbd-pool
ll
输出:
总用量 0
lrwxrwxrwx. 1 root root 10 6月   8 18:45 rbd-image -> ../../rbd0

```
4.使用块设备 /dev/rbd/{pool-name}/{image-name} 创建文件系统

```
sudo mkfs.xfs /dev/rbd/rbd-pool/rbd-image

```
6.将该文件系统挂载到 /mnt/ceph-block-device 文件夹下

```
sudo mkdir /mnt/ceph-block-device
sudo mount /dev/rbd/rbd-pool/rbd-image /mnt/ceph-block-device
df -h
输出:

文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 476M     0  476M    0% /dev
tmpfs                    488M     0  488M    0% /dev/shm
tmpfs                    488M   14M  474M    3% /run
tmpfs                    488M     0  488M    0% /sys/fs/cgroup
/dev/mapper/centos-root  8.0G  1.9G  6.2G   24% /
/dev/sda1               1014M  160M  855M   16% /boot
tmpfs                     98M     0   98M    0% /run/user/0
172.21.33.229:6789:/     560M     0  560M    0% /mnt/cephfs
/dev/rbd0                240M  2.1M  234M    1% /mnt/ceph-block-devic

cd /mnt/ceph-block-device
sudo dd if=/dev/zero of=testrbd bs=50M count=1 #生成100m文件
可在dashboard看到osd节点的已用增加了50m证明同步成功

```
##### 7.RGW对象网关，对外提供访问接口
###### 1.在client节点安装对象网关，这里暂时将client作为部署对象网关的节点

```
ceph-deploy install --rgw client
```


ceph-common 包是它的一个依赖性，所以 ceph-deploy 也将安装这个包。 ceph 的命令行工具就会为管理员准备好。为了让你的 Ceph 对象网关节点成为管理节点，可以在管理节点的工作目录下执行以下命令:

```
ceph-deploy admin client
```
在你的管理节点的工作目录下，使用命令在 Ceph 对象网关节点上新建一个 Ceph对象网关实例。举例如下:
```

ceph-deploy rgw create client
```

在网关服务成功运行后，你可以使用未经授权的请求来访问端口 7480 ，就像这样:
```

http://172.21.33.232:7480/

```


```
如果网关实例工作正常，你接收到的返回信息大概如下所示:

<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
       <Owner>
       <ID>anonymous</ID>
       <DisplayName></DisplayName>
   </Owner>
   <Buckets>
   </Buckets>
</ListAllMyBucketsResult>


```

###### 2.创建用户，在api调用接口时需要用到


```
为 S3 访问创建 RADOSGW 用户，它与亚马逊 S3 API 的基本数据访问模型兼容。
sudo radosgw-admin user create --uid="testuser" --display-name="First User" --access=full
```


命令的输出跟下面的类似:

```
    "user_id": "testuser",
    "display_name": "First User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "testuser",
            "access_key": "BH4XYONLXM4U6NH6W81E",
            "secret_key": "ndWJ8Xz4KzW7fbHR0ATOP0DBkZuXpCWqzUUMGCUG"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []

}
```

新建一个 SWIFT 用户 ，它与 Swift API 的基本访问模型兼容。

```
sudo radosgw-admin subuser create --uid=testuser --subuser=testuser:swift --access=full

```
输出如下：

```
 "user_id": "testuser",
    "display_name": "First User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [
        {
            "id": "testuser:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "testuser",
            "access_key": "BH4XYONLXM4U6NH6W81E",
            "secret_key": "ndWJ8Xz4KzW7fbHR0ATOP0DBkZuXpCWqzUUMGCUG"
        }
    ],
    "swift_keys": [
        {
            "user": "testuser:swift",
            "secret_key": "8T688bcdkob9j92RtR7EcPD5dd8DFdTOtqxoi6AT"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []

```

**注意：用swift进行api访问时，使用到的用户和secret_key是 "swift_keys"中的内容**

##### 8.java通过s3和swift访问对象网关

maven引入依赖
```
		<!-- https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk -->
		<dependency>
			<groupId>com.amazonaws</groupId>
			<artifactId>aws-java-sdk</artifactId>
			<version>1.12.1</version>
		</dependency>
		<!-- https://mvnrepository.com/artifact/org.javaswift/file-cli -->
		<dependency>
			<groupId>org.javaswift</groupId>
			<artifactId>file-cli</artifactId>
			<version>1.4.0</version>
		</dependency>

		<!-- https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-s3 -->
		<dependency>
			<groupId>com.amazonaws</groupId>
			<artifactId>aws-java-sdk-s3</artifactId>
			<version>1.12.1</version>
		</dependency>
```
s3示例：

```
package com.example.demo.api;

import java.io.*;
import java.util.List;

import com.amazonaws.ClientConfiguration;
import com.amazonaws.Protocol;
import com.amazonaws.SdkClientException;
import com.amazonaws.auth.AWSCredentials;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.model.*;
import com.amazonaws.util.StringUtils;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3Client;


/**
 * @Author: clownc
 * @Date: 2021-06-09 17:27
 */
public class S3Test {

    private static final String AWS_ACCESS_KEY = "U3GE8W17P0UM2Z5MC60X"; // 【你的 access_key】
    private static final String AWS_SECRET_KEY = "ugVwP8s3lJHs8lX5cI2PhyfvFVabvR3Nvh8wQucf"; // 【你的 aws_secret_key】
    private static final String bucketName = "tets-bucket"; // 【你 bucket 的名字】 # 首先需要保证 s3 上已经存在该存储桶
    //    private static final String AWS_REGION = "";
    private static final String ENDPOINT = "http://172.21.33.239:7480";


    private static AmazonS3 conn;

    //静态块：初始化S3的连接对象s3Client！ 需要3个参数：AWS_ACCESS_KEY，AWS_SECRET_KEY，AWS_REGION
    static {
        AWSCredentials credentials = new BasicAWSCredentials(AWS_ACCESS_KEY, AWS_SECRET_KEY);

        ClientConfiguration clientConfig = new ClientConfiguration();
        clientConfig.setProtocol(Protocol.HTTP);

        conn = new AmazonS3Client(credentials, clientConfig);
        conn.setEndpoint(ENDPOINT);
        List<Bucket> buckets = conn.listBuckets();
        if (buckets.size()==0){
            Bucket bucket = conn.createBucket(bucketName);
        }
    }

    public static void main(String[] args) {
        String inputPath = "F:\\test123\\test.txt";
        String outputPath = "F:\\test123\\test_out.txt";

        uploadToS3(new File(inputPath), "testKey");
        downloadFromS3(bucketName,"testKey",outputPath);
    }

    private static void downloadFromS3( String bucketName, String key, String outputPath) {
        S3Object object = conn.getObject(new GetObjectRequest(bucketName, key));
        if(object!=null){
            System.out.println("Content-Type: " + object.getObjectMetadata().getContentType());
            InputStream input = null;
            FileOutputStream fileOutputStream = null;
            byte[] data = null;
            try {
                //获取文件流
                input=object.getObjectContent();
                data = new byte[input.available()];
                int len = 0;
                fileOutputStream = new FileOutputStream(outputPath);
                while ((len = input.read(data)) != -1) {
                    fileOutputStream.write(data, 0, len);
                }
                System.out.println("下载文件成功");
            } catch (IOException e) {
                e.printStackTrace();
            }finally{
                if(fileOutputStream!=null){
                    try {
                        fileOutputStream.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
                if(input!=null){
                    try {
                        input.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }

    private static void uploadToS3(File fileName, String key) {
        try {
            PutObjectRequest request = new PutObjectRequest(bucketName, key, fileName);
            PutObjectResult putObjectResult = conn.putObject(request);
            System.out.println("上传文件成功！！！");
        } catch (SdkClientException e) {
            e.printStackTrace();
        }
    }
}


```

sift示例：

```
package com.example.demo.api;

import org.javaswift.joss.client.factory.AccountConfig;
import org.javaswift.joss.client.factory.AccountFactory;
import org.javaswift.joss.client.factory.AuthenticationMethod;
import org.javaswift.joss.model.Account;
import org.javaswift.joss.model.Container;
import org.javaswift.joss.model.StoredObject;

import java.io.File;
import java.io.IOException;
import java.util.*;

/**
 * @Author: clownc
 * @Date: 2021-06-09 18:26
 */
public class SwiftTest {
    private static Account account;
    private static String username = "testuser:swift";
    private static String password = "8T688bcdkob9j92RtR7EcPD5dd8DFdTOtqxoi6AT";
    private static String authUrl = "http://172.21.33.232:7480/auth/1.0";
    private static String containName="test-container";
    static {
        AccountConfig config = new AccountConfig();
        config.setUsername(username);
        config.setPassword(password);
        config.setAuthUrl(authUrl);
        config.setAuthenticationMethod(AuthenticationMethod.BASIC);
        account = new AccountFactory(config).createAccount();
        Collection<Container> list = account.list();
        if (list.size()==0) {
            Container container = account.getContainer(containName);
            container.create();//先创建存储对象的容器
        }
    }

    public static void main(String[] args) {
        String inputPath = "F:\\test123\\test.txt";
        String outputPath = "F:\\test123\\test_out_swift.txt";
        String objectName="test-object";
        boolean uploadResult = createObject(containName, objectName, inputPath);
        if (uploadResult) System.out.println("上传成功!");
    }

    /*
     * 该方法用于创建存储时的对象
     * @param containName
     * @param objectName
     * @param filePath
     * @return
     */
    public static boolean createObject(String containName, String objectName, String filePath) {
        Container container = account.getContainer(containName);
        StoredObject object = container.getObject(objectName);
        object.uploadObject(new File(filePath));
        return object.exists();
    }

    /*
     * 该方法用于获取Object到本地文件系统
     * @param containerName
     * @param objectName
     * @param outpath
     */
    public static void retrieveObject(String containerName, String objectName, String outpath) {
        Container container = account.getContainer(containerName);
        StoredObject object = container.getObject(objectName);
        object.downloadObject(new File(outpath));
    }

    /*
     * 列举当前用户已创建的容器名
     *
     * @return
     */
    public static List listContainer() {

        List list = new ArrayList<Container>();
        Collection<Container> containers = account.list();

        for (Container container : containers) {
            list.add(container.getName());
            System.out.println(container.getName());
        }
        return list;

    }
}


```


##### docker-compose通过nfs远程链接存储集群

1.在ceph集群中创建了块设备或者文件系统的机器安装nfs服务端，并且通过nfs对外开放文件系统目录

```
yum install nfs-utils rpcbind -y
mkdir -p /data/nfs
echo "/data/nfs *(rw,no_root_squash,sync)">>/etc/exports
exportfs -r
systemctl start rpcbind nfs-server
systemctl enable rpcbind nfs-server
showmount -e localhost
输出:
Export list for localhost:
/data/nfs *

```
对外开放nfs，关闭防火墙或者开放一些nfs特定端口。

```
1.关闭防火墙：
systemctl stop firewalld

如果开启了iptables的话，也是一样的，iptables的策略里也默认不会为nfs服务开启需要的端口

systemctl stop iptables

2.开放特定端口，在容器启动的机器执行:

yum install nfs-utils rpcbind -y

rpcinfo -p ip（开放远程文件系统的主机ip）
输出：
   100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  44740  status
    100024    1   tcp  34610  status
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  33582  nlockmgr
    100021    3   udp  33582  nlockmgr
    100021    4   udp  33582  nlockmgr
    100021    1   tcp  43764  nlockmgr
    100021    3   tcp  43764  nlockmgr
    100021    4   tcp  43764  nlockmgr

然后在开放文件系统的主机中开放这些端口即可。

开放端口后执行：
showmount -e ip（开放远程文件系统的主机ip）
输出：
Export list for 192.168.5.125:
/data/nfs *


```
2.编写docker-compos.yml

```

version: '3.0'
services:
    mysql:
        network_mode: "host"
        environment:
            MYSQL_ROOT_PASSWORD: "123456"
            MYSQL_USER: 'test'
            MYSQL_PASS: '123456'
        image: "docker.io/mysql:5.7"
        container_name: mysqltest
        restart: always
        volumes:
            - "db:/var/lib/mysql"
            - "./conf/my.cnf:/etc/my.cnf" #本地mysql配置
            - "./init:/docker-entrypoint-initdb.d/" #本地初始化配置文件

volumes:
    db:
        driver_opts:
                type: "nfs"
                o: "addr=192.168.5.125,nolock,soft,rw"
                device: ":/data/nfs/mysql/db" #远程nfs下挂载目录，持久化mysql数据


```

```
启动 ：
docker-compose up -d
```

其中init目录中vim init.sql文件：

```
CREATE USER 'test'@'%' IDENTIFIED BY '123456';
GRANT All privileges ON *.* TO 'test'@'%';
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';

```
conf目录中vim my.cnf：

```

[mysqld]
user=mysql
default-storage-engine=INNODB
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

```

产生mysql测试数据：
```
create database test;
use database test;
create table user
(
    id int auto_increment primary key,
    username varchar(64) unique not null,
    email varchar(120) unique not null,
    password_hash varchar(128) not null,
    avatar varchar(128) not null
);
insert into user values(1, "TOM","test12345@qq.com","passwd","avaterpath");
insert into user values(2, "Lily","12345test@qq.com","passwd","avaterpath");
```


##### 通过docker-volume-cephfs插件直接挂载docker-compose的volums，达到ceph远程持久化volums数据

1.安装插件(前提ceph集群已经有文件系统)

```
docker plugin install n0r1skcom/docker-volume-cephfs
docker plugin ls
输出：
ID             NAME                                    DESCRIPTION                ENABLED
72f423d6a92c   n0r1skcom/docker-volume-cephfs:latest   cephFS plugin for Docker   true

```
2.编写docker-compose.yml


```
version: '3.0'
services:
    mysql:
        network_mode: "host"
        environment:
            MYSQL_ROOT_PASSWORD: "123456"
            MYSQL_USER: 'test'
            MYSQL_PASS: '123456'
        image: "docker.io/mysql:5.7"
        container_name: mysqltest
        restart: always
        volumes:
            - "db:/var/lib/mysql"
            - "./conf/my.cnf:/etc/my.cnf" #本地mysql配置
            - "./init:/docker-entrypoint-initdb.d/" #本地初始化配置文件

volumes:
  db:
    driver: "n0r1skcom/docker-volume-cephfs"
    driver_opts:
      name: "admin" #管理节点名称
      path: "/"
      secret: "AQCVW8lgTSpNNhAAq3NwHswaDCohWdma2GmG+w==" #管理节点ceph.client.admin.keyring文件的key
      monitors: "192.168.5.125:6789" #ceph集群mon节点的ip:6789

```


```
docker-compose up -d
docker ps | grep mysql
输出：
94fac4cb1c91   mysql:5.7   "docker-entrypoint.s…"   24 hours ago   Up 24 hours             mysqltest


docker exec -it 94fac4cb1c91  /bin/bash

```





































