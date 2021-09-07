### 1. ceph集群部署流程

```shell
ceph-deploy new NODE1 NODE2 NODE3 --cluster-network 192.168.44.0/24 --public-network 180.180.0.0/24		#创建配置文件及密钥,会生成三个文件，ceph.conf  ceph-deploy-ceph.log ceph.mon.keyring
ceph-deploy --overwrite-conf mon create-initial		#配置初始 monitor(s)、并收集所有密钥,此时在/etc/ceph生成2个文件：ceph.client.admin.keyring和icfs.conf;在/var/lib/ceph/bootstrap-mds、bootstrap-osd、bootstrap-rgw下生成icfs.keyring，及/var/lib/ceph/mon/ceph-NODE1/下生成keyring等
ceph-deploy disk zap NODE1:/dev/sdc
ceph-deploy osd create NODE1:/dev/sdc	#prepare osd并activate（可将create替换成prepare,activate分两步执行）
ceph-deploy osd prepare NODE1:/dev/sdc
或icfs-disk -v prepare --bluestore /dev/sdc --block.db /dev/sdg1
ceph-deploy osd activate NODE1:/dev/sdc1
ceph-deploy admin NODE1 NODE2 NODE3		# 把配置文件和 admin 密钥拷贝到管理节点和 Ceph 节点，这样你每次执行 Ceph 命令行时就无需指定 monitor 地址和 ceph.client.admin.keyring 了
```

* 将osd手动移除集群

  ```
  icfs osd out osd.0				#先置为out
  systemctl stop icfs-osd@0
  icfs osd crush rm osd.0     #从crushmap中移除
  icfs auth del osd.0			#移除osd认证密钥
  icfs osd rm osd.0			#移除osd
  icfs-deploy disk zap NODE1:/dev/sdb	#擦除分区
  ```

* 将osd手动加入集群

  ```
  icfs osd create [{uuid} [{osd-num}]]	#向Monitor申请osd编号
  mkdir /var/lib/ceph/osd/ceph-{osd-num}
  若承载设备是裸设备，进行格式化：mkfs -t {fstype} /dev/{hdd}
  然后挂载到:mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-num}
  ceph-osd -i {osd-num} --mkfs --mkkey	#初始化上面的目录
  ceph auth add osd.{osd-num} osd ‘allow *’ mon ’allow profile osd’ mgr 'allow profile osd’ -i /var/ lib/ceph/osd/ceph-{osd-num}/keyring	#注册鉴权密钥（ OSD 上电后需要先和 AuthMonitor 完成鉴权）
  ceph osd crush add {osd-num} {weight} [{bucket-type}={bucket-name}.]
  ```

  

#### 1.1 构建crush rule

ceph osd crush rule dump可查看故障域类型。若为host故障域，3副本，osd所在root下需要至少三个host

```shell
icfs osd crush add-bucket mpool_root root   #增加root类型，名为mpool_root
icfs osd crush add-bucket www host          #增加host类型，名为www
icfs osd crush move www root=mpool_root     #将host www移动到root mpool_root下
icfs osd crush add osd.0 73 host=www		#将权重为73的osd.0添加到host www下
icfs osd crush batch add osd.0,osd.1 host=www #将游离态的(不需要加权重)osd.0,osd.1批量添加到host www下
icfs osd crush batch move osd.0,osd.1 73,73 host=www#将游离态的某个host下权重为73的osd.0,osd.1批量添加到host www下
icfs osd crush rm osd.0						#将osd.0从www下移除成为游离态
icfs osd crush rm root_test					#将root类型的root_test移除，若提示busy，需从管软界面先删除该root下的存储池，再执行该命令

```



#### 1.2 以cephFS方式访问

部署集群并创建crushmap后

```shell
ceph-deploy mds create NODE1   #创建mds服务，需要事先mkdir /var/lib/ceph/mds目录
ceph osd pool create test 20 20 3   #创建名为test的存储池，pg个数为20个
ceph osd pool create test_m 20  #创建名为test_m的元数据池
ceph fs new fs_m test_m test	#创建好存储池后，就可以用 fs new 命令创建文件系统了
检查mds管理节点状态
[cephuser@ceph-admin cluster]$ ceph mds stat
e5: 1/1/1 up {0=NODE1=up:active}

查看ceph集群状态
[cephuser@ceph-admin cluster]$ sudo ceph -s
cluster 33bfa421-8a3b-40fa-9f14-791efca9eb96
health HEALTH_OK
monmap e1: 1 mons at {ceph-admin=192.168.10.220:6789/0}
election epoch 3, quorum 0 ceph-admin
fsmap e5: 1/1/1 up {0=ceph-admin=up:active} #多了此行状态
osdmap e19: 3 osds: 3 up, 3 in
flags sortbitwise,require_jewel_osds
pgmap v48: 84 pgs, 3 pools, 2068 bytes data, 20 objects
101 MB used, 45945 MB / 46046 MB avail
84 active+clean
ceph-fuse /mnt/test					#挂载文件系统到test目录
或ceph-fuse -m 172.16.90.95:6789 /mnt/test  #将指定节点挂载到test目录
```

###  部署某个osd为bluestore

* ```
  将该osd移除集群
  sgdisk -n 0:0:+10G /dev/sdg  	#将sdg做为db存放元数据，此时会有一个10G的sdg1
  icfs-disk -v prepare --bluestore /dev/sdc --block.db /dev/sdg1	#将sdc部署成bluestore
  将该osd加入crushmap
  在每个节点上的/etc/icfs/icfs.conf加入参数	
  	[osd.x]
  	osd_objectstore = bluestore
  	osd_max_object_size = 2147483648
  	bluestore = true
  	单节点集群务必要增加：
  	osd_crush_chooseleaf_type = 0
  依次重启所有节点icfs-osd.target或icfs-osd@x
  
  ```

  

### 2.命令简介

```shell
ceph-deploy new [initial-monitor-node(s)]
开始部署一个集群，生成配置文件、keyring、一个日志文件。

ceph-deploy install [HOST] [HOST…]
在远程主机上安装ceph相关的软件包, –release可以指定版本，默认是firefly。

ceph-deploy mon create-initial
部署初始monitor成员，即配置文件中mon initial members中的monitors。部署直到他们形成表决团，然后搜集keys，并且在这个过程中报告monitor的状态。

ceph-deploy mon create [HOST] [HOST…]
显示的部署monitor，如果create后面不跟参数，则默认是mon initial members里的主机。

ceph-deploy mon add [HOST]
将一个monitor加入到集群之中。

ceph-deploy mon destroy [HOST]
在主机上完全的移除monitor，它会停止了ceph-mon服务，并且检查是否真的停止了，创建一个归档文件夹mon-remove在/var/lib/ceph目录下。

ceph-deploy gatherkeys [HOST] [HOST…]
获取提供新节点的验证keys。这些keys会在新的MON/OSD/MD加入的时候使用。

ceph-deploy disk list [HOST]
列举出远程主机上的磁盘。实际上调用ceph-disk命令来实现功能。

ceph-deploy disk prepare [HOST:[DISK]]
为OSD准备一个目录、磁盘，它会创建一个GPT分区，用ceph的uuid标记这个分区，创建文件系统，标记该文件系统可以被ceph使用。

ceph-deploy disk activate [HOST:[DISK]]
激活准备好的OSD分区。它会mount该分区到一个临时的位置，申请OSD ID，重新mount到正确的位置/var/lib/ceph/osd/ceph-{osd id}, 并且会启动ceph-osd。

ceph-deploy disk zap [HOST:[DISK]]
擦除对应磁盘的分区表和内容。实际上它是调用sgdisk –zap-all来销毁GPT和MBR, 所以磁盘可以被重新分区。

ceph-deploy osd prepare HOST:DISK[:JOURNAL] [HOST:DISK[:JOURNAL]…]
为osd准备一个目录、磁盘。它会检查是否超过MAX PIDs,读取bootstrap-osd的key或者写一个（如果没有找到的话），然后它会使用ceph-disk的prepare命令来准备磁盘、日志，并且把OSD部署到指定的主机上。

ceph-deploy osd active HOST:DISK[:JOURNAL] [HOST:DISK[:JOURNAL]…]
激活上一步的OSD。实际上它会调用ceph-disk的active命令，这个时候OSD会up and in。

ceph-deploy osd create HOST:DISK[:JOURNAL] [HOST:DISK[:JOURNAL]…]
上两个命令的综合。

ceph-deploy osd list HOST:DISK[:JOURNAL] [HOST:DISK[:JOURNAL]…]
列举磁盘分区。

ceph-deploy admin [HOST] [HOST…]
将client.admin的key push到远程主机。将ceph-admin节点下的client.admin keyring push到远程主机/etc/ceph/下面。

ceph-deploy push [HOST] [HOST…]
将ceph-admin下的ceph.conf配置文件push到目标主机下的/etc/ceph/目录。 ceph-deploy pull [HOST]是相反的过程。

ceph-deploy uninstall [HOST] [HOST…]
从远处主机上卸载ceph软件包。有些包是不会删除的，像librbd1, librados2。

ceph-deploy purge [HOST] [HOST…]
类似上一条命令，增加了删除data。

ceph-deploy purgedata [HOST] [HOST…]
删除/var/lib/ceph目录下的数据，它同样也会删除/etc/ceph下的内容。

ceph-deploy forgetkeys
删除本地目录下的所有验证keyring, 包括client.admin, monitor, bootstrap系列。

ceph-deploy pkg –install/–remove [PKGs] [HOST] [HOST…]
在远程主机上安装或者卸载软件包。[PKGs]是逗号分隔的软件包名列表。
```



### 删除文件系统

```shell
systemctl stop ceph-mds.target    #先停mds
ceph fs rm fs_m --yes-i-really-mean-it  #移除fs

```

### 备份

```
#!/bin/bash



ceph-deploy new tala002 tala003 tala004 tala005 tala006 --cluster-network 100.2.151.0/24 --public-network 100.2.151.0/24



echo "mon_allow_pool_size_one = true" >> ./ceph.conf

echo "mon_allow_pool_delete = true" >> ./ceph.conf

echo "mon_max_pg_per_osd = 1000" >> ./ceph.conf

echo "debug_rocksdb = 0" >> ./ceph.conf



ceph-deploy --overwrite-conf mon create-initial

ceph-deploy admin tala002 tala003 tala004 tala005 tala006



\#exit 0

sleep 5



for i in {sdb,sdc,sdd};do ceph-deploy disk zap tala006 /dev/$i; ceph-deploy osd create tala006 --data /dev/$i;done

for i in {sda,sdc,sdd,sde};do ceph-deploy disk zap tala004 /dev/$i; ceph-deploy osd create tala004 --data /dev/$i;done



ceph-deploy mgr create tala002 tala003 tala004 tala005 tala006

ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

#oceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' 

ceph-deploy admin tala002 tala003 tala004 tala005 tala006



#ceph osd pool ls detail

#ceph osd pool create pool1 4096 4096 replicated

#ceph osd pool set pool1 pg_autoscale_mode off

#ceph osd pool set pool1 size 2

#

#ceph osd pool set pool1 target_size_ratio 1

#ceph osd pool set pool1 pg_num 4096
```

