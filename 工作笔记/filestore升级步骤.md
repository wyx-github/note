1. 执行OSD缩容前需要注意集群状态，集群状态必须是OK的。
2. 一个节点的OSD更换为bluestore后需要等待重构结束，然后再执行下一个节点。
3. 文件场景8节点6+2纠删，每节点1块sata SSD 作为元数据OSD使用，1块nvme SSD 作为PGcache使用，35块HDD 盘作为数据OSD使用。
4. 需要在禁用PGcache后对nvme SSD 格式化，将nvme SSD空间平均分，分出35个分区作为bluestore db分区备用。
5. 由于元数据的OSD也需要升级到bluestore,所以sata SSD 也需要格式化。由于sata SSD 为元数据OSD独享，不用分db分区。

### 部署要求

1. 部署3.7.16.18版本，文件场景，数据池选择纠删模式

2. 两块ssd，一块作为元数据分离使用，一块作为PGcache使用。

3. 搭建好集群后预埋10~20%的数据，然后禁用PGcache，等待PGcache数据下刷。（禁用PGcache找一下臧林劼）

4. 在线升级到UV2.

1~4为更换store前的准备工作。

5. 从新跑上vdbench

6. 设置noout 

7. 先升级非mon节点，比如08节点，停掉08节点的所有OSD。

8.  将08节点的OSD移除（crush rm等）
9. 将08节点元数据盘和PGcache盘格式化（如果mon分区在元数据盘或PGcache盘上，不能格式化整盘），icfs-deploy disk zap
10. 将PGcache盘的空间均分，划分db分区（分区个数非元数据分离的OSD个数，该节点）。由于sata SSD 为元数据OSD独享，不用分db分区。(使用for i in {1..24};do sgdisk -n 0:0:+55G /dev/nvme0n1;done分区后执行partprobe /dev/{盘符})
11. 将元数据盘的OSD创建出来:icfs-disk -v prepare --bluestore /dev/sdf   
12. 08节点的创建出来的元数据OSD加入元数据池对应的root（mpool_root）下(weight值为原来的weight值)
13. 将08节点的HDD格式化。（也可以放到第9步）
14. 将08节点的非元数据分离的OSD创建出来。icfs-disk -v prepare --bluestore /dev/sdf --block.db /dev/ nvme0n1p1 （指定的db分区为第10步分出来的）（35个HDD的OSD）
15. 将08节点的非元数据分离OSD加入到数据池的root（default）

