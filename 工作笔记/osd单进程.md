## pseudo进程

- run_as_pseudo()
  - g_osd_pseudo->init()
    - 拼接admin_asok_path
    - 启动定时器线程timer.init()
  - global_init_daemonize()将进程转为后台守护进程
    - global_init_prefork()
    - 创建守护进程daemon(1,1)
  - g_osd_pseudo->start()
    - 每隔10s启动osd_start()3次，直到成功
      - do_command(“osd start",&result)
        - asok_client.do_request(“prefix:osd start,id:{id}”，result)
          - asok_connect()创建套接字
          - asok_request()从套接字中写
          - safe_read_exact()从套接字中读
    - reset_timer()
      - 注册5s的定时事件tick()
        - osd_status()
          - do_command(“osd status",&result)
            - asok_client.do_request(“prefix:osd status,id:{id}”，result)
              - asok_connect()创建套接字并设置超时时间
              - asok_request()从套接字中写，即将需求发给别人，以获取结果
              - safe_read_exact()从套接字中读的结果取出来
        - 根据osd返回值判断osd状态
  - 注册信号
  - g_osd_pseudo->final_init() {}#do nothing
  - g_osd_pseudo->wait()
    - 判断状态是否为STOPPED，进行while()循环，等待条件变量触发cond.wat(lock)
  - 其他退出操作

## admin进程

- g_osd_manager = new OSDManager()
  - 若配置文件中设置共享messengers则messengers = new OSDMessengers()，没设置则成员messengers为空
  - 满盘检测disk_mon_thread = new DiskMon()
- run_as_admin()
  - g_osd_manager->init()
    - 启动线程heartbeat_back_timer、heartbeat_front_timer
    - 若存在共享的messengers
      - messengers->init()
        - pick_addresses()
        - OSDMessengers::create()
          - 创建各个Messenger::create()：ms_public、ms_cluster、ms_hbclient、ms_hb_back_server、ms_hb_front_server、ms_objectoer，都是 new virtual_messenger(里面的成员变量messgr会被实例化成simplemessenger或aysnc)
        - 其他关于Messenger的设置(包括IP、端口绑定)
      - 为dispatch添加处理osd心跳
    - 通过heartbeat_back_timer、heartbeat_front_timer里的线程定时发送18秒心跳信息
    - preprocess_args()
    - osd open classes on start
    - 启动慢盘检测线程disk_mon_thread->init()
  - global_init_daemonize()将进程转为后台守护进程
    - global_init_prefork()
    - 创建守护进程daemon(1,1)
  - common_init_finish() 
    - cct->start_service_thread()
      - 启动cephContextServiceThread用来完成重新打开log file和更新performance counter
      - 启动adminSocker线程
  - g_osd_manager->start()
    - messengers->start()
      - 启动上面创建的Messenger线程：例如ms_public.start()...
  - 注册信号
  - g_osd_manager->final_init()注册监听管理命令
    - [接受到osd start id 命令](##接受到osd start id 命令)
  - g_osd_manager->wait()
    - 判断状态是否为STOPPED，进行while()循环，等待条件变量触发cond.wat(lock)
  - 其他退出操作



## 接受到osd start id 命令

- AdminSocket线程通过套接字接受到命令内容后调用OSDManager的call()函数里的start_osd()
  - 调用某个OSDInstance的start()
    - 启动osd_thread线程，进入osd_thread_entry()的_start()
      - init_cct()
      - create_store()
      - 根据配置决定是否创建单独的OSDMessengers
      - 创建MonClient
      - 构造osd类对象
    - 启动成功后_wait(),等待条件变量触发循环判断当前状态
    - stop时_clean_up()

## OSD处理的消息

* MSG_OSD_MARK_ME_DOWN
  * 若是由于慢盘导致的直接exit
  * 否则service.got_stop_ack()
* ICFS_MSG_PING   #与monitor
* ICFS_MSG_OSD_MAP 
