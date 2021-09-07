# Ceph的Paxo源码注释 - Phase 2

上篇是Phase 1，即leader当选后确定PN的部分。这篇主要是Phase 2，即正常工作过程中的Propose、accept和commit过程。

Ceph的paxos实现，不算很精妙，近期修改也活跃。但是对于我们理解paxos协议仍然有帮助。

## 1. 几个要点说明

#### 1.1 Phase 2的交互过程

| Leader            | Peons             | 说明                                                         |
| ----------------- | ----------------- | ------------------------------------------------------------ |
| begin()=>         |                   | Leader给quorum中各个成员发送提议，包含PN、version和内容      |
|                   | <= handle_begin() | Peon处理提议，有可能会拒绝                                   |
| handle_accept()   |                   | 只有quorum中所有成员都同意，才算成功                         |
| commit_start()    |                   | handle_accept在收到所有ack后调用，用事务写commit记录，并设置回调函数 |
| commit_finish()=> |                   | 上一步的回调函数，在实际事务完成时执行                       |
|                   | handle_commit()   | Peon根据leader的commit消息同步状态                           |

从begin()到commit_finish()称为一轮，即一次提议的完成过程。

#### 1.2 串行化提议

Ceph的paxos实现，每次只允许一个提议在执行中，即上面提到
的一轮完成才能执行下一轮。在这个过程中，会有多次写盘操作。
这个过程实际上比较慢。对于ceph自身来说，osd等的状态变更，
不频繁，无需极高的paxos性能。 但是如果用于做用于分布式数据
库等系统的日志，这种方式则有不足。

## 2. 代码

#### 2.1 Leader的提议

```
//这是Phase 2的开始，leader正常工作时，所有提议都是从这个函数开始
//bufferlist是内容的打包，paxos层不需要知道具体语义
//leader
void Paxos::begin(bufferlist& v)
{
  assert(mon->is_leader());
  assert(is_updating() || is_updating_previous()); 
  //STATE_UPDATING_PREVIOUS对应刚当选后，发现有uncommited value，并且是
  //下一个版本(last_committed+1)

  // we must already have a majority for this to work.
  assert(mon->get_quorum().size() == 1 ||
     num_last > (unsigned)mon->monmap->size()/2);

  // and no value, yet.
  assert(new_value.length() == 0);

  //清空"已accept的成员"列表
  accepted.clear(); 
  accepted.insert(mon->rank);//表示 自己先accept
  new_value = v;

  if (last_committed == 0) {
    //paxos从未进行过提交。将"first_committed"的初始化，也打包到一个事务中
    // initial base case; set first_committed too
    t->put(get_name(), "first_committed", 1);
    decode_append_transaction(t, new_value);

    bufferlist tx_bl;
    t->encode(tx_bl);
    new_value = tx_bl; 
  }

  // store the proposed value in the store. IF it is accepted, we will then
  // have to decode it into a transaction and apply it.
  MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);

  //下这几个k/v，在同一个事务中写下去。
  //实际上，我们只会建议last_committed+1，因此不会有多个pending的版本。
  t->put(get_name(), last_committed+1, new_value);
  //pending_v就是上面的version
  t->put(get_name(), "pending_v", last_committed + 1);
  //配套的PN，这些都记为pending.
  t->put(get_name(), "pending_pn", accepted_pn);

  logger->inc(l_paxos_begin);
  logger->inc(l_paxos_begin_keys, t->get_keys());
  logger->inc(l_paxos_begin_bytes, t->get_bytes());
  utime_t start = ceph_clock_now(NULL);

  //保存前面put的三个 k/v
  get_store()->apply_transaction(t);

  utime_t end = ceph_clock_now(NULL);
  logger->tinc(l_paxos_begin_latency, end - start);

  assert(g_conf->paxos_kill_at != 3);

  //quorum size是1，这是all in one 配置才有的场景
  if (mon->get_quorum().size() == 1) {
    // we're alone, take it easy
    commit_start();
    return;
  }


  //给quorum中各个成员发提议
  for (set<int>::const_iterator p = mon->get_quorum().begin();
       p != mon->get_quorum().end();
       ++p) {
    if (*p == mon->rank) continue;

    MMonPaxos *begin = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_BEGIN,
                     ceph_clock_now(g_ceph_context));
    //消息包含下面三个值
    begin->values[last_committed+1] = new_value;
    begin->last_committed = last_committed;//这个直接发送过去了。
    begin->pn = accepted_pn;

    mon->messenger->send_message(begin, mon->monmap->get_inst(*p));
  }

  // set timeout event
  accept_timeout_event = new C_AcceptTimeout(this);
  mon->timer.add_event_after(g_conf->mon_accept_timeout, accept_timeout_event);
}1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071727374757677787980818283848586
```

#### 2.2 Peon处理提议

```
//Phase 2, Peon收到提议后的处理
void Paxos::handle_begin(MonOpRequestRef op)
{
  op->mark_paxos_event("handle_begin");
  MMonPaxos *begin = static_cast<MMonPaxos*>(op->get_req());

  //比较PN，确定是否应该accept。这个是标准paxos协议做法
  if (begin->pn < accepted_pn) {
    /*可能已经有人发起了新的选举，新的leader做了collect修改了PN。
      比如之前的leader网络故障，然后恢复了，它仍然按照旧状态在运行*/
    dout(10) << " we accepted a higher pn " << accepted_pn << ", ignoring" << dendl;
    op->mark_paxos_event("have higher pn, ignore");
    return;
  }
  //实际上保证leader还是那个leader，即朝代没改。
  assert(begin->pn == accepted_pn);
  assert(begin->last_committed == last_committed);//因为leader没有变化，
  //请求一个一个处理，所以总是维持一致的last_committed，双方知道下一个版本是啥

  assert(g_conf->paxos_kill_at != 4);

  logger->inc(l_paxos_begin);

  // set state.
  state = STATE_UPDATING; //不同状态下能做不同的事情。
  lease_expire = utime_t();  // cancel lease

  // yes.
  version_t v = last_committed+1; //设置version，这时不会有uncommitted吧？ 应该是同步完成了。
  dout(10) << "accepting value for " << v << " pn " << accepted_pn << dendl;
  // store the accepted value onto our store. We will have to decode it and
  // apply its transaction once we receive permission to commit.
  MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);
  //以下几个k/v，在一个事务中写入 
  t->put(get_name(), v, begin->values[v]);
  // note which pn this pending value is for.
  t->put(get_name(), "pending_v", v);//下面会apply，但是这些都是pending的
  t->put(get_name(), "pending_pn", accepted_pn);//不用写pending_version，
  //因为有last_committed已经持久化了，这个含义明确。

  logger->inc(l_paxos_begin_bytes, t->get_bytes());
  utime_t start = ceph_clock_now(NULL);

  get_store()->apply_transaction(t);

  utime_t end = ceph_clock_now(NULL);
  logger->tinc(l_paxos_begin_latency, end - start);

  assert(g_conf->paxos_kill_at != 5);

  // reply
  MMonPaxos *accept = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_ACCEPT,
                    ceph_clock_now(g_ceph_context));
  accept->pn = accepted_pn;
  accept->last_committed = last_committed;
  begin->get_connection()->send_message(accept);
}
```

#### 2.3 Leader收到ack后处理

```
//Phase 2, Leader 收到Peon的ack后的处理
// leader
void Paxos::handle_accept(MonOpRequestRef op)
{
  op->mark_paxos_event("handle_accept");
  MMonPaxos *accept = static_cast<MMonPaxos*>(op->get_req());
  dout(10) << "handle_accept " << *accept << dendl;
  int from = accept->get_source().num();

  if (accept->pn != accepted_pn) {//更高的PN，可能别人已经当选leader
    // we accepted a higher pn, from some other leader 
    dout(10) << " we accepted a higher pn " << accepted_pn << ", ignoring" << dendl;
    op->mark_paxos_event("have higher pn, ignore");
    return;
  }
  if (last_committed > 0 &&
      accept->last_committed < last_committed-1) {//旧的响应，比如网络延迟太大引起。抛弃。
    dout(10) << " this is from an old round, ignoring" << dendl;
    op->mark_paxos_event("old round, ignore");
    return;
  }
  assert(accept->last_committed == last_committed ||   // not committed
     accept->last_committed == last_committed-1);  // committed   
     // 只允许差1

  assert(is_updating() || is_updating_previous());
  assert(accepted.count(from) == 0); //确认同一个peon不重复发送
  accepted.insert(from);
  dout(10) << " now " << accepted << " have accepted" << dendl;

  assert(g_conf->paxos_kill_at != 6);

  // only commit (and expose committed state) when we get *all* quorum
  // members to accept.  otherwise, they may still be sharing the now
  // stale state.
  // FIXME: we can improve this with an additional lease revocation message
  // that doesn't block for the persist.  
  if (accepted == mon->get_quorum()) {
    //所有的属于quorum的都响应才行,否则会走到timeout分支
    // yay, commit!
    op->mark_paxos_event("commit_start");
    commit_start(); //这个函数，最终引起调用commit_finish，修改last_committed。
  }
}

void Paxos::accept_timeout()
{
  dout(1) << "accept timeout, calling fresh election" << dendl;
  accept_timeout_event = 0;
  assert(mon->is_leader());
  assert(is_updating() || is_updating_previous() || is_writing() ||
     is_writing_previous());
  logger->inc(l_paxos_accept_timeout);
  mon->bootstrap();//注意，是直接自举，即触发重新核对qurom，并选leader。Ceph实现的特殊之处。
}

struct C_Committed : public Context {
  Paxos *paxos;
  C_Committed(Paxos *p) : paxos(p) {}
  void finish(int r) {
    assert(r >= 0);
    //这个类，在构造函数中获得锁，析构函数中释放锁
    Mutex::Locker l(paxos->mon->lock);
    //下面这一段执行，是受mon->lock保护的
    paxos->commit_finish();
  }
};

//commit_start只有leader会调用，所以commit_finish 也就只有leader用了
void Paxos::commit_start()
{
  dout(10) << __func__ << " " << (last_committed+1) << dendl;

  assert(g_conf->paxos_kill_at != 7);

  MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);

  // commit locally
  t->put(get_name(), "last_committed", last_committed + 1);
  //这个修改，下面是queue_trans，不是apply

  // decode the value and apply its transaction to the store.
  // this value can now be read from last_committed.
  decode_append_transaction(t, new_value);

  dout(30) << __func__ << " transaction dump:\n";
  JSONFormatter f(true);
  t->dump(&f);
  f.flush(*_dout);
  *_dout << dendl;

  logger->inc(l_paxos_commit);
  logger->inc(l_paxos_commit_keys, t->get_keys());
  logger->inc(l_paxos_commit_bytes, t->get_bytes());
  commit_start_stamp = ceph_clock_now(NULL);
//C_Committed在finish执行时，会获取锁(在finish函数)，但是当前上下文应该持有了锁，
//什么时候放锁的?  否则C_Committed:finish() 没法拿到锁。
  get_store()->queue_transaction(t, new C_Committed(this));
  // 钩子函数，txn结束时callback, C_Committed的finish? trans
  //在MonitorDBStore.h的finish()中有注释，实际上是在做了txn apply后，调用callback
  if (is_updating_previous())
    state = STATE_WRITING_PREVIOUS;
  else if (is_updating())
    state = STATE_WRITING;//设置这两个状态后，依赖于异步的commit完成回调，去做清除，
    //本context实际上很快结束，不会修改此状态。参见paxos::commit_finish()
  else
    assert(0);

  if (mon->get_quorum().size() > 1) {
    // cancel timeout event
    mon->timer.cancel_event(accept_timeout_event);
    accept_timeout_event = 0;
  }
}
//Leader的函数
void Paxos::commit_finish()
{
  dout(20) << __func__ << " " << (last_committed+1) << dendl;
  utime_t end = ceph_clock_now(NULL);
  logger->tinc(l_paxos_commit_latency, end - commit_start_stamp);

  assert(g_conf->paxos_kill_at != 8);

  // cancel lease - it was for the old value.
  //  (this would only happen if message layer lost the 'begin', but
  //   leader still got a majority and committed with out us.)
  lease_expire = utime_t();  // cancel lease

  //这里才修改last_committed
  last_committed++;//这里才修改last_committed
  last_commit_time = ceph_clock_now(NULL);

  // refresh first_committed; this txn may have trimmed.  //说了可能trim log。流程还么仔细看
  first_committed = get_store()->get(get_name(), "first_committed");

  _sanity_check_store();

  // tell everyone
  for (set<int>::const_iterator p = mon->get_quorum().begin();
       p != mon->get_quorum().end();
       ++p) {
    if (*p == mon->rank) continue;

    dout(10) << " sending commit to mon." << *p << dendl;
    MMonPaxos *commit = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_COMMIT,
                      ceph_clock_now(g_ceph_context));
    //leader 在commit完成后，通知peon
    commit->values[last_committed] = new_value;
    commit->pn = accepted_pn;
    commit->last_committed = last_committed;

    mon->messenger->send_message(commit, mon->monmap->get_inst(*p));
    //本地的last_committed等信息已经修改了，通知其他的
  }

  assert(g_conf->paxos_kill_at != 9);

  // get ready for a new round.
  new_value.clear();

  remove_legacy_versions();

  // WRITING -> REFRESH
  // among other things, this lets do_refresh() -> mon->bootstrap() know
  // it doesn't need to flush the store queue
  assert(is_writing() || is_writing_previous());
  state = STATE_REFRESH;
  //do_refresh()，是让服务层刷新知道该propose了。
  if (do_refresh()) {//do_refresh的注释中(.h)，在它返回false时，abort，应该是异常情况吧。
    commit_proposal();//这个应该是调用上次注册的complete函数
    if (mon->get_quorum().size() > 1) {//如果只有我自己，单个monitor，那就无需lease了。
      extend_lease();
    }
    //唤醒等待者
    finish_contexts(g_ceph_context, waiting_for_commit); 
    assert(g_conf->paxos_kill_at != 10);

    finish_round();//修改状态，本轮结束。让pending的proposal可以继续执行。
    //修改了last_committed和status。但是这个同步写完再做下一个的方式，比较慢。
  }
}
```

#### 2.4 Peon处理commit消息

```
//Peon的函数，leader通知哪些已经commit，这些是可以信任的
void Paxos::handle_commit(MonOpRequestRef op)
{
  op->mark_paxos_event("handle_commit");
  MMonPaxos *commit = static_cast<MMonPaxos*>(op->get_req());
  dout(10) << "handle_commit on " << commit->last_committed << dendl;

  logger->inc(l_paxos_commit);

  if (!mon->is_peon()) {
    dout(10) << "not a peon, dropping" << dendl;
    assert(0);
    return;
  }

  op->mark_paxos_event("store_state");
  store_state(commit);

  if (do_refresh()) {//让service层刷新状态
    finish_contexts(g_ceph_context, waiting_for_commit);//Peon端，没有等待被propose的。
  }
}
```

#### 2.5 让上层服务刷新状态的工具函数

```
/*一个paxos过程结束后，需要让上层的各个service(monitor)刷新状态。
因为paxos这层本身不知道语义，只是确定执行顺序而已。一个paxos决议可能
包含了几个上层service的内容。
*/
bool Paxos::do_refresh()
{
  bool need_bootstrap = false;

  utime_t start = ceph_clock_now(NULL);

  // make sure we have the latest state loaded up
  mon->refresh_from_paxos(&need_bootstrap);

  utime_t end = ceph_clock_now(NULL);
  logger->inc(l_paxos_refresh);
  logger->tinc(l_paxos_refresh_latency, end - start);

  if (need_bootstrap) {//需要bootstrap才返回false，正常都是成功
    dout(10) << " doing requested bootstrap" << dendl;
    mon->bootstrap();
    return false;
  }

  return true;
}

//唤醒上层等待当前提议完成的上下文
void Paxos::commit_proposal()
{
  dout(10) << __func__ << dendl;
  assert(mon->is_leader());
  assert(is_refresh());

  list<Context*> ls;
  ls.swap(committing_finishers);
  //从pending_finishers ==swap==>  committing_finishers  ==swap==>  ls
  finish_contexts(g_ceph_context, ls); 
  //做callback，paxosservice调用 queue_pending_finisher()注册的钩子
}
```

#### 2.6 完成本轮，开始下一轮

```
//已经完成上一轮提议过程，可以开始下一个
void Paxos::finish_round()
{
  dout(10) << __func__ << dendl;
  assert(mon->is_leader());

  // ok, now go active!
  state = STATE_ACTIVE;//不是active是不会去propose的。

  dout(20) << __func__ << " waiting_for_acting" << dendl;
  finish_contexts(g_ceph_context, waiting_for_active);
  dout(20) << __func__ << " waiting_for_readable" << dendl;
  finish_contexts(g_ceph_context, waiting_for_readable);
  dout(20) << __func__ << " waiting_for_writeable" << dendl;
  finish_contexts(g_ceph_context, waiting_for_writeable);

  dout(10) << __func__ << " done w/ waiters, state " << state << dendl;

  if (should_trim()) {
    trim();
  }

  if (is_active() && pending_proposal) {
    propose_pending();
  }
}
```

#### 2.7 其他

```c++
/*
 * return a globally unique, monotonically increasing proposal number
 */
version_t Paxos::get_new_proposal_number(version_t gt)
{
  if (last_pn < gt) 
    last_pn = gt;
  //每个monitor有自己的rank，把rank作为自己产生的PN的低位数，则各自不同。 
  //比如，rank=5的，产生的rank只可能是105, 205, 305等，即n*100 +5
  // update. make it unique among all monitors.
  last_pn /= 100; //由于gt可能是别人发过来的，是不同的rank，如果直接 
  //把last_pn +=100，last_pn所带的rank就是别人的，不是自己应该产生的合法pn。
  last_pn++;
  last_pn *= 100;
  last_pn += (version_t)mon->rank;//如果之前last_pn = 306，而我的rank是 5,  
  //则得到了新的last_pn是405

  // write
  MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);
  t->put(get_name(), "last_pn", last_pn);//持久化到kv

  dout(30) << __func__ << " transaction dump:\n";
  JSONFormatter f(true);
  t->dump(&f);
  f.flush(*_dout);
  *_dout << dendl;

  logger->inc(l_paxos_new_pn);
  utime_t start = ceph_clock_now(NULL);

  get_store()->apply_transaction(t);

  utime_t end = ceph_clock_now(NULL);
  logger->tinc(l_paxos_new_pn_latency, end - start);

  dout(10) << "get_new_proposal_number = " << last_pn << dendl;
  return last_pn;
}


void Paxos::cancel_events()
{
  if (collect_timeout_event) {
    mon->timer.cancel_event(collect_timeout_event);
    collect_timeout_event = 0;
  }
  if (accept_timeout_event) {
    mon->timer.cancel_event(accept_timeout_event);
    accept_timeout_event = 0;
  }
  if (lease_renew_event) {
    mon->timer.cancel_event(lease_renew_event);
    lease_renew_event = 0;
  }
  if (lease_ack_timeout_event) {
    mon->timer.cancel_event(lease_ack_timeout_event);
    lease_ack_timeout_event = 0;
  }  
  if (lease_timeout_event) {
    mon->timer.cancel_event(lease_timeout_event);
    lease_timeout_event = 0;
  }
}

void Paxos::shutdown()
{
  dout(10) << __func__ << " cancel all contexts" << dendl;

  // discard pending transaction
  pending_proposal.reset();

  finish_contexts(g_ceph_context, waiting_for_writeable, -ECANCELED);
  finish_contexts(g_ceph_context, waiting_for_commit, -ECANCELED);
  finish_contexts(g_ceph_context, waiting_for_readable, -ECANCELED);
  finish_contexts(g_ceph_context, waiting_for_active, -ECANCELED);
  finish_contexts(g_ceph_context, pending_finishers, -ECANCELED);
  finish_contexts(g_ceph_context, committing_finishers, -ECANCELED);
  if (logger)
    g_ceph_context->get_perfcounters_collection()->remove(logger);
  delete logger;
}

void Paxos::leader_init()
{
  cancel_events();
  new_value.clear();

  // discard pending transaction
  pending_proposal.reset();//当选leader之前的都废弃掉

  finish_contexts(g_ceph_context, pending_finishers, -EAGAIN);
  finish_contexts(g_ceph_context, committing_finishers, -EAGAIN);

  logger->inc(l_paxos_start_leader);

  if (mon->get_quorum().size() == 1) {
    state = STATE_ACTIVE;
    return;
  }

  state = STATE_RECOVERING;
  lease_expire = utime_t();
  dout(10) << "leader_init -- starting paxos recovery" << dendl;
  collect(0);
}

void Paxos::peon_init()
{
  cancel_events();
  new_value.clear();

  state = STATE_RECOVERING;
  lease_expire = utime_t();
  dout(10) << "peon_init -- i am a peon" << dendl;

  // start a timer, in case the leader never manages to issue a lease
  reset_lease_timeout();

  // discard pending transaction
  pending_proposal.reset();

  // no chance to write now!
  finish_contexts(g_ceph_context, waiting_for_writeable, -EAGAIN);
  finish_contexts(g_ceph_context, waiting_for_commit, -EAGAIN);
  finish_contexts(g_ceph_context, pending_finishers, -EAGAIN);
  finish_contexts(g_ceph_context, committing_finishers, -EAGAIN);

  logger->inc(l_paxos_start_peon);
}

void Paxos::restart()
{
  dout(10) << "restart -- canceling timeouts" << dendl;
  cancel_events();
  new_value.clear();

  if (is_writing() || is_writing_previous()) {
    dout(10) << __func__ << " flushing" << dendl;
    mon->lock.Unlock();
    mon->store->flush();
    mon->lock.Lock();
    dout(10) << __func__ << " flushed" << dendl;
  }
  state = STATE_RECOVERING;

  // discard pending transaction
  pending_proposal.reset();

  finish_contexts(g_ceph_context, committing_finishers, -EAGAIN);
  finish_contexts(g_ceph_context, pending_finishers, -EAGAIN);
  finish_contexts(g_ceph_context, waiting_for_commit, -EAGAIN);
  finish_contexts(g_ceph_context, waiting_for_active, -EAGAIN);

  logger->inc(l_paxos_restart);
}1
void Paxos::dispatch(MonOpRequestRef op)
{
  assert(op->is_type_paxos());
  op->mark_paxos_event("dispatch");
  PaxosServiceMessage *m = static_cast<PaxosServiceMessage*>(op->get_req());
  // election in progress?
  if (!mon->is_leader() && !mon->is_peon()) {
    dout(5) << "election in progress, dropping " << *m << dendl;
    return;    
  }

  // check sanity
  assert(mon->is_leader() || 
     (mon->is_peon() && m->get_source().num() == mon->get_leader())); 
     //应该是指peon只接受leader的消息，但是不能接受其他peon的消息

  switch (m->get_type()) {

  case MSG_MON_PAXOS:
    {
      MMonPaxos *pm = reinterpret_cast<MMonPaxos*>(m);

      // NOTE: these ops are defined in messages/MMonPaxos.h
      switch (pm->op) {
    // learner
      case MMonPaxos::OP_COLLECT:
    handle_collect(op);
    break;
      case MMonPaxos::OP_LAST:
    handle_last(op);
    break;
      case MMonPaxos::OP_BEGIN:
    handle_begin(op);
    break;
      case MMonPaxos::OP_ACCEPT:
    handle_accept(op);
    break;      
      case MMonPaxos::OP_COMMIT:
    handle_commit(op);
    break;
      case MMonPaxos::OP_LEASE:
    handle_lease(op);
    break;
      case MMonPaxos::OP_LEASE_ACK:
    handle_lease_ack(op);
    break;
      default:
    assert(0);
      }
    }
    break;

  default:
    assert(0);
  }
}
// -- WRITE --

bool Paxos::is_writeable()
{
  return
    mon->is_leader() &&
    is_active() &&
    is_lease_valid();
}

void Paxos::propose_pending()
{
  assert(is_active());
  assert(pending_proposal);

  cancel_events();

  bufferlist bl;
  pending_proposal->encode(bl);

  dout(10) << __func__ << " " << (last_committed + 1)
       << " " << bl.length() << " bytes" << dendl;
  dout(30) << __func__ << " transaction dump:\n";
  JSONFormatter f(true);
  pending_proposal->dump(&f);
  f.flush(*_dout);
  *_dout << dendl;

  pending_proposal.reset();//让pending_proposal不再ref里面的Transaction类型对象,
  //参见http://en.cppreference.com/w/cpp/memory/shared_ptr/reset

  committing_finishers.swap(pending_finishers); 
  //list::swap()的含义： Exchanges the contents of two lists， 
  // http://www.cplusplus.com/reference/list/list/swap-free/
  state = STATE_UPDATING;
  begin(bl);
}

void Paxos::queue_pending_finisher(Context *onfinished)
{
  dout(5) << __func__ << " " << onfinished << dendl;
  assert(onfinished);
  pending_finishers.push_back(onfinished);
}
//注意，上层通过这个函数获取txn，然后再往里面添加内容。
//按照这么理解，一次propose，可能有多个操作被打包
MonitorDBStore::TransactionRef Paxos::get_pending_transaction()  
//pending transaction，不是proposal
{
  assert(mon->is_leader());
  if (!pending_proposal) {
    pending_proposal.reset(new MonitorDBStore::Transaction);
    assert(pending_finishers.empty());
  }
  return pending_proposal;
}

bool Paxos::trigger_propose()
{
  if (is_active()) {
    dout(10) << __func__ << " active, proposing now" << dendl;
    propose_pending();
    return true;
  } else {
    dout(10) << __func__ << " not active, will propose later" << dendl;
    return false;
  }
}

bool Paxos::is_consistent()
{
  return (first_committed <= last_committed);
}
```