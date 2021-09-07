## AdminSocket

* main()
  * global_init()里创建cct，调IcfsContext()构造函数
    * new AdminSocket()
    * new AdminSocketHook
    * 注册命令即将命令做为key，上面的hook作为value放入map<string,AdminSocketHook> m_hooks;
  * common_init_finish()
    * 调IcfsContext::start_service_thread()
      * AdminSocket::init()
        * 创建读写的pipe
        * bind_and_listen()
        * create()创建adminSocket线程
          * entry()
            * pool()监听socket_fd及shutdown_fd
            * do_accept()
              * 收到消息时execute_command()
                * 调用上面及其他地方注册的hook的call()
                  * hook子类call()对命令进行处理

