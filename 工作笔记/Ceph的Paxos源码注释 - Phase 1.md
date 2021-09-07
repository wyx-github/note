上篇是Leader 选举部分。这篇主要是Ceph的Paxos协议的Phase1(Prepare)，其目的是就PN达成一致。

## 1. 几个要点说明

#### 1.1 Epoch

> 每次选举产生新的leader，也会产生新的epoch。不选举则不会修改epoch。
> 一个leader当选期间，发送的所有消息，都会带有这个epoch。
> 如果由于网络分割等现象，有新的选举发生，则根据epoch就发现leader已经变了。
> 注意，按照paxos论文描述，没有Leader也是可以正常运行的，只是可能降低效率。
> **没有leader则不需要epoch**

#### 1.2 PN (Proposal Number)

> Leader当选后，会首先执行一次phase 1过程，以确定PN。 在其为leader期间，
> 所有的phase 2操作都共用一个PN。所以省略了大量的phase 1操作，这也是
> paxos能够减小网络开销的原因。 “Paxos made simple”文中说：
> “A newly chosen leader executes phase 1 for infinitely many
> instances of the consensus algorithm”。
> **PN是必须的，无论是否有leader，都必须有PN**

#### 1.3 Version

> 可以理解成Paxos 的instance ID，或者raft的logID。

#### 1.4 持久化

> 对比Raft，虽然ceph的复制也可以看成一个个log的追加，
> 但是所有信息都写在k/v中，而不是写log文件, 比如，instanceID为X的log，
> 在k/v存储中，其key是X，value是log内容。
> 其他各种需要持久化的值，都写在k/v存储中。

#### 1.5其他需要持久化的数据结构

除了log以外，每个paxos成员，都维护以下几个需要持久化的变量。
大家可以跟raft的paper做些简单对比。

| 名称              | 含义                                                      | 其他                                                |
| ----------------- | --------------------------------------------------------- | --------------------------------------------------- |
| last_pn           | 上次当选leader后生成的PN                                  | get_new_proposal_number()使用，下次当选后，接着生成 |
| accepted_pn       | 我接受过的PN，可能是别的leader提议的PN                    | peon根据这个值拒绝较小的PN                          |
| first_committed   | 本节点记录的第一个被commit的版本                          | 更早的版本(日志)，本节点没有了                      |
| last_committed    | 本节点记录的最后一次被commit的版本                        | 往后的版本，未被commit，可能有一个                  |
| uncommitted_v     | 本节点记录的未commit的版本，如果有，只能等于last_commit+1 | ceph只允许有一个未commit的版本                      |
| uncommitted_pn    | 未commit的版本对应的PN                                    | 与uncommitted_v，uncommitted_value在一个事务中记录  |
| uncommitted_value | 为commit的版本的内容                                      | 见上面                                              |

注意，上述三个”uncommitted”开头的值，可能压根就不存在，比如正常关机，全部都commit了。

#### 1.6 Phase 1交互过程简介

Phase 1就是 paxos协议的Propose阶段，包括三个步骤，如下表：

| 步骤 | Leader        | Peons              | 备注                                               |
| ---- | ------------- | ------------------ | -------------------------------------------------- |
| 1    | collect() =>  |                    | Leader给quorum中各个peon发送PN以及其他附带信息     |
| 2    |               | <=handle_collect() | Peon同意或者拒绝PN。并中间可能分享已经commit的数据 |
| 3    | handle_last() |                    | Quorum中peon全部同意leader的PN，才算成功           |

## 2. 代码

#### 2.1 初始化

```
void Paxos::init()
{ //几个持久化的变量，加载时即从从kv读出。
  // load paxos variables from stable storage
  //上次产生的PN
  last_pn = get_store()->get(get_name(), "last_pn");
  //上次接受的pn
  accepted_pn = get_store()->get(get_name(), "accepted_pn");
  //最近或最后一个被commit的verion，实际上是paxos 的instance ID。
  last_committed = get_store()->get(get_name(), "last_committed");
  //保存的最早被commit的版本(log)。更早的log可能已经被truncate掉了
  first_committed = get_store()->get(get_name(), "first_committed");

  //paxos的 first_committed，并不是某个monitor的first_committed，各个monitor
  //对应值可能都是不一样的。
  assert(is_consistent());
}
```

#### 2.2 Leader发起的collect

```
// PHASE 1:  collect和handle_collect基本能对应paxos的phase 1
//这是leader的当选后执行函数，用于确定新的PN。
//collect过程，相当于完成当选期间所有提议的phase 1。
//在其当选期间，会一直使用这个PN
void Paxos::collect(version_t oldpn)
{
 // we're recoverying, it seems!
  state = STATE_RECOVERING;
  assert(mon->is_leader());

  // reset the number of lasts received
  uncommitted_v = 0; //新当选，初始化
  uncommitted_pn = 0;
  uncommitted_value.clear();
  peer_first_committed.clear();
  peer_last_committed.clear();

  //ceph的实现中，只允许有一个proposal处于pending状态(跟raft相同)。
  //如果新leader当选后发现有pending的提议，那么其instanceID/version
  //只能是last_committed+1
  if (get_store()->exists(get_name(), last_committed+1)) {
    /*pending_v, pending_pn和last_committed+1是一个事务写的。
     所以一起检查 */
    version_t v = get_store()->get(get_name(), "pending_v");  
    version_t pn = get_store()->get(get_name(), "pending_pn"); 
    if (v && pn && v == last_committed + 1){//这个是正常分支
      uncommitted_pn = pn;
    } else {
      dout(10) << "WARNING: no pending_pn on disk, using previous accepted_pn " << accepted_pn  << " and crossing our fingers" << dendl;
      uncommitted_pn = accepted_pn;
    }
    uncommitted_v = last_committed+1;
    //找到uncommitted_v (这个key)对应的value
    get_store()->get(get_name(), last_committed+1, uncommitted_value);
    //uncommitted_v存在，要求uncommitted_value必须存在。    
    assert(uncommitted_value.length()); 
    logger->inc(l_paxos_collect_uncommitted);
  }

  //生成一个新的更大的PN，并自己先accept
  accepted_pn = get_new_proposal_number(MAX(accepted_pn, oldpn));
  accepted_pn_from = last_committed;
  num_last = 1;//1, 表示自己已经投票了

  //给quorum中各个成员发送
  for (set<int>::const_iterator p = mon->get_quorum().begin();
       p != mon->get_quorum().end();
       ++p) {
    //跳过自己，已经算投过了并修改了accepted_pn       
    if (*p == mon->rank) continue; 

    //epoch的用意: 如果网络分割，别人又发起了选举，现任leader不知道，接收方会发现epoch不对    
    MMonPaxos *collect = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_COLLECT,

    collect->last_committed = last_committed; //用来与peer比较的
    collect->first_committed = first_committed;
    //这个操作本身带的PN是刚生成的。
    collect->pn = accepted_pn; 
    mon->messenger->send_message(collect, mon->monmap->get_inst(*p));
  }

  //设置超时处理
  collect_timeout_event = new C_CollectTimeout(this);
  mon->timer.add_event_after(g_conf->mon_accept_timeout, collect_timeout_event);
}
```

#### 2.2 Peon处理collect请求

```
//Peon，可以对应Raft的follower
void Paxos::handle_collect(MonOpRequestRef op)
{
  op->mark_paxos_event("handle_collect");

  MMonPaxos *collect = static_cast<MMonPaxos*>(op->get_req());

  assert(mon->is_peon()); // mon epoch filter should catch strays

  // we're recoverying, it seems!
  state = STATE_RECOVERING;

  //我落后的太远，中间相差的已无法通过log补齐，只有bootstrap(自举)了。
  if (collect->first_committed > last_committed+1) {
    dout(5) << __func__
            << " leader's lowest version is too high for our last committed"
            << " (theirs: " << collect->first_committed
            << "; ours: " << last_committed << ") -- bootstrap!" << dendl;
    op->mark_paxos_event("need to bootstrap");
    mon->bootstrap();
    return;
  }

  // reply
  MMonPaxos *last = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_LAST,
                  ceph_clock_now(g_ceph_context));
  //本地保存的两个committed，返回给leader                
  last->last_committed = last_committed;
  last->first_committed = first_committed;

  version_t previous_pn = accepted_pn;//这个是本地记录的以前的accepted_pn

  //这个是标准的paxos PN比较，如果收到的PN大于我之前接受过的PN ,则同意
  if (collect->pn > accepted_pn) {
    accepted_pn = collect->pn;
    accepted_pn_from = collect->pn_from;
    dout(10) << "accepting pn " << accepted_pn << " from " 
         << accepted_pn_from << dendl;

    MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);
    //需要先持久化，然后再回复
    t->put(get_name(), "accepted_pn", accepted_pn);
    dout(30) << __func__ << " transaction dump:\n";
    JSONFormatter f(true);
    t->dump(&f);
    f.flush(*_dout);
    *_dout << dendl;

    logger->inc(l_paxos_collect);
    logger->inc(l_paxos_collect_keys, t->get_keys());
    logger->inc(l_paxos_collect_bytes, t->get_bytes());
    utime_t start = ceph_clock_now(NULL);
    get_store()->apply_transaction(t);

    utime_t end = ceph_clock_now(NULL);
    logger->tinc(l_paxos_collect_latency, end - start);
  } else {//其他情况，不接受
    // don't accept!
    dout(10) << "NOT accepting pn " << collect->pn << " from " << collect->pn_from
         << ", we already accepted " << accepted_pn
         << " from " << accepted_pn_from << dendl;
  }
  //如果collect->pn(对方发过来的pn)小于我的PN，那么这个回复，就是拒绝。
  last->pn = accepted_pn;
  last->pn_from = accepted_pn_from;

  // share whatever committed values we have
  /*已经committed的数据都是可以信任的，如果对方的last_committed比我的小，
    那么我把我知道的已经commit的都分享做同步。share_state时，
    对方的处理函数是store_stat() 。完成后，对方也会修改了last_committed*/
  share_state(last, collect->first_committed, collect->last_committed);

  // do we have an accepted but uncommitted value?
  //  (it'll be at last_committed+1)
  bufferlist bl;
  if (collect->last_committed <= last_committed &&
      get_store()->exists(get_name(), last_committed+1)) {
      //前面提过，last_committed+1这个版本如果存在，那是一个未决的提议，
      //需要告诉leader。
    get_store()->get(get_name(), last_committed+1, bl);
    assert(bl.length() > 0);
    dout(10) << " sharing our accepted but uncommitted value for " 
         << last_committed+1 << " (" << bl.length() << " bytes)" << dendl;
    last->values[last_committed+1] = bl;

    version_t v = get_store()->get(get_name(), "pending_v");
    version_t pn = get_store()->get(get_name(), "pending_pn");

    if (v && pn && v == last_committed + 1) {
      /*如果有pending_pn，那么返回的uncommitted_pn就是 pending_pn，
      否则就在下面直接用previous_pn代替了*/
      last->uncommitted_pn = pn; 
    } else {
      // previously we didn't record which pn a value was accepted
      // under!  use the pn value we just had...  :(
      dout(10) << "WARNING: no pending_pn on disk, using previous accepted_pn " << previous_pn
           << " and crossing our fingers" << dendl;
      last->uncommitted_pn = previous_pn;
    }

    logger->inc(l_paxos_collect_uncommitted);
  }

  //reply可能是拒绝，如果我的pn比leader给的大
  collect->get_connection()->send_message(last);
}
```

#### 2.3 分享已经commit的数据的两个函数

```
/**对方的处理函数是： store_state。share的是二者last_committed之间的各个版本对应的value。
 * @note This is Okay. We share our versions between peer_last_committed and
 *   our last_committed (inclusive), and add their bufferlists to the
 *   message. It will be the peer's job to apply them to its store, as
 *   these bufferlists will contain raw transactions.
 *   This function is called by both the Peon and the Leader. The Peon will
 *   share the state with the Leader during handle_collect(), sharing any
 *   values the leader may be missing (i.e., the leader's last_committed is
 *   lower than the peon's last_committed). The Leader will share the state
 *   with the Peon during handle_last(), if the peon's last_committed is
 *   lower than the leader's last_committed.
 */
void Paxos::share_state(MMonPaxos *m, version_t peer_first_committed,
            version_t peer_last_committed)
{
  assert(peer_last_committed < last_committed);

  dout(10) << "share_state peer has fc " << peer_first_committed 
       << " lc " << peer_last_committed << dendl;
  version_t v = peer_last_committed + 1;

  // include incrementals
  uint64_t bytes = 0;
  for ( ; v <= last_committed; v++) {
    /*注意这里面并没有进行消息传递，只是把两个版本之间的内容给打包
      进了msg，随着msg的其他内容一起发送*/
    if (get_store()->exists(get_name(), v)) {
      get_store()->get(get_name(), v, m->values[v]);
      assert(m->values[v].length());
      dout(10) << " sharing " << v << " ("
           << m->values[v].length() << " bytes)" << dendl;
      bytes += m->values[v].length() + 16;  // paxos_ + 10 digits = 16
    }
  }
  logger->inc(l_paxos_share_state);
  logger->inc(l_paxos_share_state_keys, m->values.size());
  logger->inc(l_paxos_share_state_bytes, bytes);

  m->last_committed = last_committed;
}

/**
 * Store on disk a state that was shared with us
 *
 * Basically, we received a set of version. Or just one. It doesn't matter.
 * What matters is that we have to stash it in the store. So, we will simply
 * write every single bufferlist into their own versions on our side (i.e.,
 * onto paxos-related keys), and then we will decode those same bufferlists
 * we just wrote and apply the transactions they hold. We will also update
 * our first and last committed values to point to the new values, if need
 * be. All all this is done tightly wrapped in a transaction to ensure we
 * enjoy the atomicity guarantees given by our awesome k/v store.
 */
bool Paxos::store_state(MMonPaxos *m)
{
  MonitorDBStore::TransactionRef t(new MonitorDBStore::Transaction);
  map<version_t,bufferlist>::iterator start = m->values.begin();
  bool changed = false;

  // build map of values to store
  // we want to write the range [last_committed, m->last_committed] only. 
  //对方状态比我快太多，没法根据收到的值去catchup
  if (start != m->values.end() &&
      start->first > last_committed + 1) {
    // ignore everything if values start in the future.
    dout(10) << "store_state ignoring all values, they start at " << start->first
         << " > last_committed+1" << dendl;
    start = m->values.end();
  }

  // push forward the start position on the message's values iterator, up until
  // we run out of positions or we find a position matching 'last_committed'.
  while (start != m->values.end() && start->first <= last_committed) {
  //移到我的last_committed开始
    ++start;
  }

  // make sure we get the right interval of values to apply by pushing forward
  // the 'end' iterator until it matches the message's 'last_committed'.
  map<version_t,bufferlist>::iterator end = start;
  while (end != m->values.end() && end->first <= m->last_committed) {
    last_committed = end->first;//内存中先修改
    ++end;
  }

  if (start == end) {
    dout(10) << "store_state nothing to commit" << dendl;
  } else {
    dout(10) << "store_state [" << start->first << ".." 
         << last_committed << "]" << dendl;
    //用一个事务，写入所有变化，包括last_committed和各个version
    t->put(get_name(), "last_committed", last_committed);

    // we should apply the state here -- decode every single bufferlist in the
    // map and append the transactions to 't'.
    map<version_t,bufferlist>::iterator it;
    for (it = start; it != end; ++it) {
      // write the bufferlist as the version's value
      //要store的version和相应value，先推入t
      t->put(get_name(), it->first, it->second);
      // decode the bufferlist and append it to the transaction we will shortly
      // apply.
      decode_append_transaction(t, it->second);
    }

    // discard obsolete uncommitted value?
    if (uncommitted_v && uncommitted_v <= last_committed) {
      dout(10) << " forgetting obsolete uncommitted value " << uncommitted_v
           << " pn " << uncommitted_pn << dendl;
      uncommitted_v = 0;
      uncommitted_pn = 0;
      uncommitted_value.clear();
    }
  }
  if (!t->empty()) {//t非空，说明有值要写

    logger->inc(l_paxos_store_state);
    logger->inc(l_paxos_store_state_bytes, t->get_bytes());
    logger->inc(l_paxos_store_state_keys, t->get_keys());
    utime_t start = ceph_clock_now(NULL);

    /*事务提交，包括last_committed和一些version及values。
      这个函数实际上会等待事务完成。*/
    get_store()->apply_transaction(t); 

    utime_t end = ceph_clock_now(NULL);
    logger->tinc(l_paxos_store_state_latency, end - start);

    //first_committed可能在事务执行过程中trim被修改了(log被trim了)，刷新下
    first_committed = get_store()->get(get_name(), "first_committed");

    _sanity_check_store();
    changed = true;//说明有修改的值
  }

  remove_legacy_versions();//erase掉比first_committed更早的

  return changed;
}
```

#### 2.4 Leader处理Peon的回复

```c++
/*Leader收到回复后的处理。
  在 Ceph的election过程中，用预设的rank作为优先级。
  当选的leader不一定持有最新的数据，因此collection过程中，
  Leader需要更新下自己的数据。这些更新，都是根据已经"commit"的数据。*/
void Paxos::handle_last(MonOpRequestRef op)
{
  op->mark_paxos_event("handle_last");
  MMonPaxos *last = static_cast<MMonPaxos*>(op->get_req());
  bool need_refresh = false;

  //from是对方的编号
  int from = last->get_source().num();

  dout(10) << "handle_last " << *last << dendl;

  if (!mon->is_leader()) {
    dout(10) << "not leader, dropping" << dendl;
    return;
  }

  // note peer's first_ and last_committed, in case we learn a new
  // commit and need to push it to them.

  //本次返回的结果，插入map。
  peer_first_committed[from] = last->first_committed;
  peer_last_committed[from] = last->last_committed;

  //跟peer相比，自己落后很多，以至于别人也没有保留当时的各个版本的raw transaction信息。
  //只有直接走bootstrap流程，做完全同步。
  if (last->first_committed > last_committed + 1) {
    dout(5) << __func__
            << " mon." << from
        << " lowest version is too high for our last committed"
            << " (theirs: " << last->first_committed
            << "; ours: " << last_committed << ") -- bootstrap!" << dendl;
    op->mark_paxos_event("need to bootstrap");
    mon->bootstrap();
    return;
  }

  assert(g_conf->paxos_kill_at != 1);

  /*对应handle_collect 内部的share_state，对方可能给我共享了
    一部分更新的已commit数据(leader的状态比较旧)*/
  need_refresh = store_state(last);

  assert(g_conf->paxos_kill_at != 2);

  //store_state()会改变leader的last_committed和first_committed。
  //然后就可能发现某个peon也需要被更新
  for (map<int,version_t>::iterator p = peer_last_committed.begin();
       p != peer_last_committed.end();
       ++p) {
    if (p->second + 1 < first_committed && first_committed > 1) {
    //对方版本太旧，没法同步了。
      dout(5) << __func__
          << " peon " << p->first
          << " last_committed (" << p->second
          << ") is too low for our first_committed (" << first_committed
          << ") -- bootstrap!" << dendl;
      op->mark_paxos_event("need to bootstrap");
      mon->bootstrap();
      return;
    }

    //对方比我旧，但是还在可同步范围。
    if (p->second < last_committed) {
      // share committed values
      dout(10) << " sending commit to mon." << p->first << dendl;
      MMonPaxos *commit = new MMonPaxos(mon->get_epoch(),
      ceph_clock_now(g_ceph_context));
      //构造一条commit消息，给peon分享已经commit的数据
      share_state(commit, peer_first_committed[p->first], p->second);
      mon->messenger->send_message(commit, mon->monmap->get_inst(p->first));
    }
  }

  //Peon接受过的PN比Leader生成的PN大，按照paxos协议，提高PN，重试！
  if (last->pn > accepted_pn) {
    // no, try again.
    dout(10) << " they had a higher pn than us, picking a new one." << dendl; 

    // cancel timeout event
    mon->timer.cancel_event(collect_timeout_event);
    collect_timeout_event = 0;
    //注意，这次用新的PN继续collect。但不是重新选举。
    collect(last->pn);
  } else if (last->pn == accepted_pn) {//对方接受了
    // yes, they accepted our pn.  great.
    num_last++;

    // did this person send back an accepted but uncommitted value?
    if (last->uncommitted_pn) {
    //last_commited对应的肯定在此前达成quorum一致的，
    //而uncommitted则是认为没有形成quorum一致的，需要处理。
    //保证有大的uncommitted_pn，才符合paxos只直接受更大PN原则
      if (last->uncommitted_pn >= uncommitted_pn && 
      last->last_committed >= last_committed &&
      last->last_committed + 1 >= uncommitted_v) {
      //这个比较，是因为Leader会收到多个peon的uncommitted_v，取大的。
    uncommitted_v = last->last_committed+1; 
    // uncommitted_v 会一直朝大的变化
    uncommitted_pn = last->uncommitted_pn;
    uncommitted_value = last->values[uncommitted_v];
    dout(10) << "we learned an uncommitted value for " << uncommitted_v
         << " pn " << uncommitted_pn
         << " " << uncommitted_value.length() << " bytes"
         << dendl;
      } else {
    dout(10) << "ignoring uncommitted value for " << (last->last_committed+1)
         << " pn " << last->uncommitted_pn
         << " " << last->values[last->last_committed+1].length() << " bytes"
         << dendl;
      }
    }

    // is that everyone?
    if (num_last == mon->get_quorum().size()) {
      //这里要求quorum成员全体都响应
      // cancel timeout event
      mon->timer.cancel_event(collect_timeout_event);
      collect_timeout_event = 0;
      peer_first_committed.clear();
      peer_last_committed.clear();

      // almost...

      // did we learn an old value?
      if (uncommitted_v == last_committed+1 &&  //只允许差1
      //消息中携带的，上面刚刚赋了值
    dout(10) << "that's everyone.  begin on old learned value" << dendl;
    //选举结束，但是发现选举前，有未commit的value。
    //之前的value不一定形成了多数派，所以要重新走一次accept过程。
    state = STATE_UPDATING_PREVIOUS;

    //这个value可能只形成了少数派，不能直接commit。
    //而是用原来的PN最大的value，使用新的PN，重新走一次phase 2。
    begin(uncommitted_value);
      } else {//这个分支，实际上存在少数派宕机重启的不确定性问题 
    // active!
    dout(10) << "that's everyone.  active!" << dendl;
    extend_lease();

    need_refresh = false;
    if (do_refresh()) {
      finish_round();
    }
      }
    }
  } else {
    // no, this is an old message, discard
    dout(10) << "old pn, ignoring" << dendl;
  }

  if (need_refresh)
    (void)do_refresh();
}
```

#### 2.5 超时处理函数

```
/*collect的超时处理：直接调用bootstrap。同步monitor信息，并重新选举leader*/
void Paxos::collect_timeout()
{
  dout(1) << "collect timeout, calling fresh election" << dendl;
  collect_timeout_event = 0;
  logger->inc(l_paxos_collect_timeout);
  assert(mon->is_leader());
  mon->bootstrap();
}
```