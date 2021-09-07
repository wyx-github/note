# Ceph的Paxos源码注释之 Election



本博客分为几个部分，先从简单的开始。这是第一部分，Paxos 的Leader Election。

### Leader Election基本设计

- 按照rank表示优先级解决冲突问题，为每个monitor预先分配了一个rank
- 只会接受优先级(rank)比自己高、epoch比上次已接受的epoch大的选举请求
- 当选的leader，不一定有最新的数据。所以在phase 1中，会根据已经commit的数据，进行leader和peon之间的同步
- 用奇数的epoch表示选举状态，偶数表示稳定状态
- 一旦选举成功，会形成一个quorum，在该leader当选期间，所有决定，必须quorum中全部成员同意，否则会引起重新选举。这个跟标准paxos协议不同。

### Leader Election主要过程和函数

- Elector::init()
  初始化处理，从kv读取之前持久化的信息
- Elector::shutdown()
  退出处理
- Elector::start()
  推选自己(自荐)
- Elector::defer()
  接受别人选举，延迟推选自己
- Elector::handle_propose()
  处理别人的自荐消息
- Elector::handle_ack()
  处理别人的ack
- Elector::victory()
  宣布自己当选
- Elector::handle_victory()
  处理别人当选消息
- Elector::expire()
  选举超时处理
- Elector::reset_timer()
  设置选举timer
- Elector::cancel_timer()
  取消timer
- Elector::dispatch()
  选举消息的分发函数

**以下分别介绍这些函数的代码：**

```c++
/*Ceph的monitor使用leveldb作为持久化存储，下面的mon->store
就是leveldb操作的封装，由于很多数据共用同一leveldb，所以对
key的空间做了分级，Monitor::MONITOR_NAME可以认为是第一级key*/
void Elector::init()
{
  //选举的epoch，递增分配。每次修改都做了持久化，这是从kv db读取。
  epoch = mon->store->get(Monitor::MONITOR_NAME, "election_epoch"); 
  if (!epoch)//首次使用
    epoch = 1;
}
void Elector::shutdown()
{
  if (expire_event)
    mon->timer.cancel_event(expire_event);
}1234
/*将自己的epoch修改为参数 e，需要持久化到 kv 存储。*/
void Elector::bump_epoch(epoch_t e) 
{
  dout(10) << "bump_epoch " << epoch << " to " << e << dendl;
  assert(epoch <= e);
  epoch = e;
  //使用一个事务，写kv存储
  MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);
  t->put(Monitor::MONITOR_NAME, "election_epoch", epoch);
  mon->store->apply_transaction(t);//持久化收到的epoch。

  //join election会使 monitor 进入 STATE_ELECTING 状态
  mon->join_election();

  // clear up some state
  electing_me = false;  //因为别人的epoch比自己大，放弃选自己
  acked_me.clear(); //这个acked_m，是选自己才有意义，在此清空。
  classic_mons.clear();
}
```

**推选自己为leader的函数(自荐)**

```c++
//在启动后或者leader超时等场合，会发起自荐
void Elector::start()//推选自己
{
  if (!participating) {
    return;
  }

  //清空，表示还没人响应自己
  acked_me.clear();
  classic_mons.clear();
  //从store获取持久化的epoch.
  init();

  if (epoch % 2 == 0) //epoch是偶数表明是稳定态
    bump_epoch(epoch+1);  //odd == election cycle 选举都是用的奇数epoch
  start_stamp = ceph_clock_now(g_ceph_context);
  electing_me = true; //设置为true，前面的bump_epoch可能设置为false了
  //填写map的key是自己的rank，表明自己先同意了自己
  acked_me[mon->rank] = CEPH_FEATURES_ALL; 

  leader_acked = -1;//无效值，表明我没有ack别人

  //给monmap中每个成员发送消息。可以认为monmap成员是预先配置的，且配置了rank
  for (unsigned i=0; i<mon->monmap->size(); ++i) { 
    if ((int)i == mon->rank) continue;
    //消息中带有自己的epoch
    Message *m = new MMonElection(MMonElection::OP_PROPOSE, epoch, mon->monmap);
    mon->messenger->send_message(m, mon->monmap->get_inst(i));
  }

  reset_timer();//设置选举用的timer，参见expire()函数
}
```

**响应别人自荐的函数**

```c++
//defer就是暂时放弃推选自己
void Elector::defer(int who) 
{
  if (electing_me) {//放弃选自己
    acked_me.clear();
    classic_mons.clear();
    electing_me = false;
  }

  //表明我支持了"who"当leader. leader_acked不需要持久化，因为任何一个monitor在 
  //reboot后都会重新发起election。
  leader_acked = who; 
  ack_stamp = ceph_clock_now(g_ceph_context);
  //返回OP_ACK消息,即赞成对方当leader
  MMonElection *m = new MMonElection(MMonElection::OP_ACK, epoch, mon->monmap);
  m->sharing_bl = mon->get_supported_commands_bl();
  mon->messenger->send_message(m, mon->monmap->get_inst(who));
  // set a timer  对方在一定时间内，应该宣布自己当选才对
  reset_timer(1.0);  // give the leader some extra time to declare victory
}
```

**设置timer的工具函数**

```c++
void Elector::reset_timer(double plus)
{
  // set the timer
  cancel_timer();
  expire_event = new C_ElectionExpire(this);
  mon->timer.add_event_after(g_conf->mon_lease + plus,
                 expire_event);
}
```

**取消timer的工具函数**

```
void Elector::cancel_timer()
{
  if (expire_event) {
    mon->timer.cancel_event(expire_event);
    expire_event = 0;
  }
}
```

**超时处理**

```
void Elector::expire()
{
  // 如果是自荐，只要超过半数同意，就认为成功
  if (electing_me &&
      acked_me.size() > (unsigned)(mon->monmap->size() / 2)) {
    //注意，expire判断的是 > monmap->size()/2，而handle_ack里面是等待全部ack。
    // i win
    victory();
  } else {//没有推选自己
    // whoever i deferred to didn't declare victory quickly enough.
    if (mon->has_ever_joined)
      start();//之前我加入过quorum，直接重新发动选举。因为monmap中会包含我。
    else
      mon->bootstrap();//否则，走bootstrap
  }
}
```

**自己当选leader**

```
void Elector::victory()
{
  leader_acked = -1;
  electing_me = false;

  uint64_t features = CEPH_FEATURES_ALL;
  set<int> quorum;
  for (map<int, uint64_t>::iterator p = acked_me.begin(); p != acked_me.end();
       ++p) {//如果是从expire()调用的victory()，则不是monmap的记录的所有node， 
       //但是肯定是超过半数。
    quorum.insert(p->first);//ack过我的，全部进入quorum。
    features &= p->second;
  }

  // decide what command set we're supporting
  bool use_classic_commands = !classic_mons.empty();
  // keep a copy to share with the monitor; we clear classic_mons in bump_epoch
  set<int> copy_classic_mons = classic_mons;

  cancel_timer();

  assert(epoch % 2 == 1);  // 选举期间用的奇数epoch
  bump_epoch(epoch+1);     // 选举完成，epoch变成偶数

  // decide my supported commands for peons to advertise
  const bufferlist *cmds_bl = NULL;
  const MonCommand *cmds;
  int cmdsize;
  if (use_classic_commands) {
    mon->get_classic_monitor_commands(&cmds, &cmdsize);
    cmds_bl = &mon->get_classic_commands_bl();
  } else {
    mon->get_locally_supported_monitor_commands(&cmds, &cmdsize);
    cmds_bl = &mon->get_supported_commands_bl();
  }

  //通知大家自己当选
  for (set<int>::iterator p = quorum.begin();
       p != quorum.end();
       ++p) {
    if (*p == mon->rank) continue;
    MMonElection *m = new MMonElection(MMonElection::OP_VICTORY, epoch, mon->monmap);
    m->quorum = quorum;
    m->quorum_features = features;
    m->sharing_bl = *cmds_bl;
    mon->messenger->send_message(m, mon->monmap->get_inst(*p));
  }

  //调用monitor的函数，它会发起paxos的propose 
  mon->win_election(epoch, quorum, features, cmds, cmdsize, &copy_classic_mons);
}
```

**处理别人的自荐消息，根据情况决定是否支持，或者决定该推荐自己**

```
void Elector::handle_propose(MonOpRequestRef op)
{
  MMonElection *m = static_cast<MMonElection*>(op->get_req());
  dout(5) << "handle_propose from " << m->get_source() << dendl;
  int from = m->get_source().num();//获取对方的rank

  assert(m->epoch % 2 == 1); // election
  uint64_t required_features = mon->get_required_features();

  if ((required_features ^ m->get_connection()->get_features()) &
      required_features) {//要求对方的feature，覆盖required_features
    nak_old_peer(op);
    return;
  } else if (m->epoch > epoch) {//对方epoch比我大，放弃选自己，追随它。
    bump_epoch(m->epoch);
  } else if (m->epoch < epoch) {//对方epoch太小，否决
    // got an "old" propose, 
    //发送消息的peer可能是刚刚加进来的，以前不在quorum里面。所以epoch比较小
    if (epoch % 2 == 0 &&    // in a non-election cycle
    mon->quorum.count(from) == 0) {  // from someone outside the quorum
      // a mon just started up, call a new election so they can rejoin!   
      //为什么我要start_election? 因为其epoch太旧，不可能当选。
      mon->start_election(); 
    } else {//认为收到了旧消息，忽略
      dout(5) << " ignoring old propose" << dendl;
      return;
    }
  }
  //我比发送方的rank高。如果我没有响应过其他比我rank高的，就推选自己
  if (mon->rank < from) {
    // i would win over them.
    if (leader_acked >= 0) {        // we already acked someone
      assert(leader_acked < from);  // and they still win, of course 否则不可能ack它
    } else {//如果没有acked过，比我优先级低的在推选自己，那么我应该选自己才对。
      // wait, i should win!
      if (!electing_me) {
    mon->start_election();
      }
    }
  } else {//发送方rank比我高
    // they would win over me  
    //之前我没有赞成谁，或者之前那个优先级没现在这个高，则赞成现在这个
    if (leader_acked < 0 ||      // haven't acked anyone yet, or
    leader_acked > from ||   // they would win over who you did ack, or
    leader_acked == from) {  // this is the guy we're already deferring to
      defer(from);//这个函数内部会发送 OP_ACK,支持发送方
    } else {//我之前响应过别人，坚持之前的选择
      // ignore them!
      dout(5) << "no, we already acked " << leader_acked << dendl;
    }
  }
}
```

**收到 ack之后的处理**

```
//收到别人响应后的处理
void Elector::handle_ack(MonOpRequestRef op)
{
  op->mark_event("elector:handle_ack");
  MMonElection *m = static_cast<MMonElection*>(op->get_req());
  dout(5) << "handle_ack from " << m->get_source() << dendl;
  int from = m->get_source().num();

  assert(m->epoch % 2 == 1); // election状态，必须是奇数
  //下面dout解释了出现这个现象的原因，即我在自己重启，out了。
  if (m->epoch > epoch) {
    dout(5) << "woah, that's a newer epoch, i must have rebooted.
      bumping and re-starting!" << dendl;
    bump_epoch(m->epoch);//必须用新的epoch才能引起有效选举，否则被忽略了
    start();
    return;
  }
  assert(m->epoch == epoch);
  uint64_t required_features = mon->get_required_features();
  if ((required_features ^ m->get_connection()->get_features()) &
      required_features) {
    dout(5) << " ignoring ack from mon" << from
        << " without required features" << dendl;
    return;
  }
  //如果正在推选自己
  if (electing_me) {
    // thanks
    //acked_me是个map
    acked_me[from] = m->get_connection()->get_features(); 
    if (!m->sharing_bl.length())
      classic_mons.insert(from);
    dout(5) << " so far i have " << acked_me << dendl;

    //所有人都赞成我
    if (acked_me.size() == mon->monmap->size()) {
       // if yes, shortcut to election finish
      victory();
    }
  } else {//以前我曾经推选过自己，但是现在我已经投别人了
    // ignore, i'm deferring already.
    assert(leader_acked >= 0);
  }
}
//收到别人宣告选举胜利的消息后的处理
void Elector::handle_victory(MonOpRequestRef op)
{
  op->mark_event("elector:handle_victory");
  MMonElection *m = static_cast<MMonElection*>(op->get_req());
  int from = m->get_source().num();

  assert(from < mon->rank);
  assert(m->epoch % 2 == 0);

  leader_acked = -1;
  //之前我一定选举了它，所以epoch必须match，否则有问题。
  // i should have seen this election if i'm getting the victory.
  if (m->epoch != epoch + 1) { 
  //在victory()中，已经加1，所以是偶数，且比peon看到的大1
    dout(5) << "woah, that's a funny epoch, i must have rebooted. 
    bumping and re-starting!" << dendl;
    bump_epoch(m->epoch);
    start();
    return;
  }

  bump_epoch(m->epoch);//我也变成偶数epoch

  // they win
  mon->lose_election(epoch, m->quorum, from, m->quorum_features);

  // cancel my timer
  cancel_timer();//选举timer没用了

  // stash leader's commands
  if (m->sharing_bl.length()) {
    MonCommand *new_cmds;
    int cmdsize;
    bufferlist::iterator bi = m->sharing_bl.begin();
    MonCommand::decode_array(&new_cmds, &cmdsize, bi);
    mon->set_leader_supported_commands(new_cmds, cmdsize);
  } else { // they are a legacy monitor; use known legacy command set
    const MonCommand *new_cmds;
    int cmdsize;
    mon->get_classic_monitor_commands(&new_cmds, &cmdsize);
    mon->set_leader_supported_commands(new_cmds, cmdsize);
  }
}
```

//nak是个非常见场景。表明对方支持的 feature不足，可能是旧版本。

```
void Elector::nak_old_peer(MonOpRequestRef op)
{
  op->mark_event("elector:nak_old_peer");
  MMonElection *m = static_cast<MMonElection*>(op->get_req());
  uint64_t supported_features = m->get_connection()->get_features();

  if (supported_features & CEPH_FEATURE_OSDMAP_ENC) {
    uint64_t required_features = mon->get_required_features();
    dout(10) << "sending nak to peer " << m->get_source()
         << " that only supports " << supported_features
         << " of the required " << required_features << dendl;

    MMonElection *reply = new MMonElection(MMonElection::OP_NAK, m->epoch,
                       mon->monmap);
    reply->quorum_features = required_features;
    mon->features.encode(reply->sharing_bl);
    m->get_connection()->send_message(reply);
  }
}1
//注意这个函数直接exit了，因为自己支持的feature不足，没法协同工作
void Elector::handle_nak(MonOpRequestRef op)
{
  op->mark_event("elector:handle_nak");
  MMonElection *m = static_cast<MMonElection*>(op->get_req());
  dout(1) << "handle_nak from " << m->get_source()
      << " quorum_features " << m->quorum_features << dendl;

  CompatSet other;
  bufferlist::iterator bi = m->sharing_bl.begin();
  other.decode(bi);
  CompatSet diff = Monitor::get_supported_features().unsupported(other);

  derr << "Shutting down because I do not support required monitor features: { "
       << diff << " }" << dendl;

  exit(0);
  // the end!
}
```

**消息的dispatch，里面有些关于monmap的处理。**

```
/*monmap，是集群当前配置的所有monitor的集合。
 *monmap在bootstrp过程中会在montior间同步，这里没仔细讨论。
 *monmap中的各个monitor，只有参与选举投票的，才会进入quorum。*/
void Elector::dispatch(MonOpRequestRef op)
{
  op->mark_event("elector:dispatch");
  assert(op->is_type_election());

  switch (op->get_req()->get_type()) {

  case MSG_MON_ELECTION://elector只收election这个类别的消息
    {
      if (!participating) {
        return;
      }
      if (op->get_req()->get_source().num() >= mon->monmap->size()) {
    dout(5) << " ignoring bogus election message with bad mon rank "
        << op->get_req()->get_source() << dendl;
    return;
      }

      MMonElection *em = static_cast<MMonElection*>(op->get_req());

      // assume an old message encoding would have matched
      if (em->fsid != mon->monmap->fsid) {
    dout(0) << " ignoring election msg fsid "
        << em->fsid << " != " << mon->monmap->fsid << dendl;
    return;
      }
    //选举是根据monmap干活的。monmap在bootstrap阶段大家已经同步了。
      if (!mon->monmap->contains(em->get_source_addr())) {
    dout(1) << "discarding election message: " << em->get_source_addr()
        << " not in my monmap " << *mon->monmap << dendl;
    return;
      }

      MonMap *peermap = new MonMap;
      peermap->decode(em->monmap_bl);
      //比较二者的monmap的epoch，即二者看到的monitor配置应该相同。
      if (peermap->epoch > mon->monmap->epoch) {
    dout(0) << em->get_source_inst() << " has newer monmap epoch " << peermap->epoch
        << " > my epoch " << mon->monmap->epoch
        << ", taking it"
        << dendl;
    mon->monmap->decode(em->monmap_bl);
        MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);
        //更新了自己的monmap，并且写盘。实际上信任了对方的monmap。
        t->put("monmap", mon->monmap->epoch, em->monmap_bl);
        t->put("monmap", "last_committed", mon->monmap->epoch);
        mon->store->apply_transaction(t);
    //mon->monmon()->paxos->stash_latest(mon->monmap->epoch, em->monmap_bl);
    cancel_timer();
    mon->bootstrap();//重新做一次自举。自举后会重新选举
    delete peermap;
    return;
      }
      if (peermap->epoch < mon->monmap->epoch) { 
      //这种情况下，会用我的map去同步对方的。
    dout(0) << em->get_source_inst() << " has older monmap epoch " << peermap->epoch
        << " < my epoch " << mon->monmap->epoch
        << dendl;
      }
      delete peermap;
      switch (em->op) {
      case MMonElection::OP_PROPOSE:
    handle_propose(op);
    return;
      }

      if (em->epoch < epoch) {//为什么又比较这个epoch?
    dout(5) << "old epoch, dropping" << dendl;
    break;
      }

      switch (em->op) {
      case MMonElection::OP_ACK:
    handle_ack(op);
    return;
      case MMonElection::OP_VICTORY:
    handle_victory(op);
    return;
      case MMonElection::OP_NAK:
    handle_nak(op);
    return;
      default:
    assert(0);
      }
    }
    break;

  default:
    assert(0);
  }
}

//admin command处理，触发选举
void Elector::start_participating()
{
  if (!participating) {
    participating = true;
    call_election();
  }
}
```