## 1.Monitor创建pg

* monitor收到 icfs osd pool create 命令
* OSDMonitor进入prepare_update==>prepare_command==>prepare_command_impl==>解析osd pool create
  * 将pgnum pgpnum 等参数加入pending_inc；
* propose_pending
  * 将pending_inc的数据encode_pending并触发trigger_prppse进行议案，落入mon盘
* mon议案完成后调用各个paxos_service的update_from_paxos即OSDMonitor::update_from_paxos
  * 调用PGMonitor::check_osd_map
    * 调用register_new_pgs
    * 如果有新pg则调用regitster_pg
      * 将创建pg的数据放出pending_inc中等待被议案
      * 调用pg_to_up_acting_osds用crush将pg映射到osd
    * 调用propose_pending进行议案
      * commit落盘之后调用Monitor::refresh_from_paxos
        *  paxos_service的refresh==>PGMonitor::update_from_paxos
          * apply_pgmap_delta（）更新pgmap数据
        * paxos_service的post_refresh
          * send_pg_creates()向osd发送创建pg消息MSG_OSD_PG_CREATE

## 2. OSD收到创建pg消息

* handle_pg_create
  * 主osd上handle_pg_peering_evt（）	处理Peering状态机事件的入口。该函数会查找相应的PG，如果该PG不存在，就创建该PG。该PG的状态机进入RecoveryMachine/Stray状态
    * 检查osd是否有这个pg，没有则进行创建
      * 根据创建的池子类型进行pg的创建和初始化
      * PG::handle_create()	#给新创建PG状态机投递事件，PG的状态发生相应的改变
        * 向状态机投递2个状态Initialize和ActMap
      * write_if_dirty     #将创建pg等操作事务落盘
      * queue_peering_event    #将事件入队，pg放入peering的线程工作队列
  * maybe_update_heartbeat_peers();//更新该PG相关的OSD的心跳列表

