### Finisher类

该类下有个vector<Context*> finisher_queue,以及FinisherThread : public Thread 

```
finisher 类是ceph中定义的专门检查操作是否结束的一个类。
我们首先看这个类的start函数
void Finisher::start()
{
  ldout(cct, 10) << __func__ << dendl;
  finisher_thread.create(thread_name.c_str());
}
调用finisher_thread的create函数。其中finisher_thread 是Finisher 中实现的一个子类
class Finisher {
  struct FinisherThread : public Thread {
    Finisher *fin;    
    explicit FinisherThread(Finisher *f) : fin(f) {}
    void* entry() override { return fin->finisher_thread_entry(); }
  } finisher_thread;
  
  }
这里可以看出FinisherThread 是thread的子类，与此同时FinisherThread 并没有实现create函数，因此肯定是调用
thread的creat函数
其中thread类的create 函数实现如下：
void Thread::create(const char *name, size_t stacksize)
{
#从这里可以看出thread的name不能超过16个字符
  assert(strlen(name) < 16);
  thread_name = name;
#调用try_create来设置thread的stack的size，从这里可以知道ceph中thread的stack的size都是固定的
  int ret = try_create(stacksize);
#正常情况下ret的返回值是零
  if (ret != 0) {
    char buf[256];
    snprintf(buf, sizeof(buf), "Thread::try_create(): pthread_create "
	     "failed with error %d", ret);
    dout_emergency(buf);
    assert(ret == 0);
  }
}
我们继续看看try_create的实现
int Thread::try_create(size_t stacksize)
{
  pthread_attr_t *thread_attr = NULL;
  pthread_attr_t thread_attr_loc;
#设置thread的stack的size
  stacksize &= CEPH_PAGE_MASK;  // must be multiple of page
  if (stacksize) {
    thread_attr = &thread_attr_loc;
    pthread_attr_init(thread_attr);
    pthread_attr_setstacksize(thread_attr, stacksize);
  }
#从这里知道原来ceph中的thread就是pthread的一个包装，注意这里thread的回调函数是_entry_func
  r = pthread_create(&thread_id, thread_attr, _entry_func, (void*)this);
}
继续看_entry_func
void *Thread::_entry_func(void *arg) {
  void *r = ((Thread*)arg)->entry_wrapper();
  return r;
}
继续调用entry_wrapper
void *Thread::entry_wrapper()
{
  return entry();
}
entry_wrapper 中又调用entry，很明显需要thread的子类来实现entry，那我们看看thread的子类实现的entry
void* entry() override { return fin->finisher_thread_entry(); }
很明显这里继续调用finisher类的finisher_thread_entry
void *Finisher::finisher_thread_entry()
{
  utime_t start;
  uint64_t count = 0;
  while (!finisher_stop) {
    /// Every time we are woken up, we process the queue until it is empty.
	#从这里知道所有等待接受的进程都会调用finisher类的queue函数把自己添加到finisher_queue 中
    while (!finisher_queue.empty()) {
      // To reduce lock contention, we swap out the queue to process.
      // This way other threads can submit new contexts to complete while we are working.
      vector<Context*> ls;
      list<pair<Context*,int> > ls_rval;
      ls.swap(finisher_queue);
      ls_rval.swap(finisher_queue_rval);
      finisher_running = true;
      finisher_lock.Unlock();
      ldout(cct, 10) << "finisher_thread doing " << ls << dendl;
      // Now actually process the contexts.
	  #这边会遍历finisher_queue。分别调用其context的complete函数
      for (vector<Context*>::iterator p = ls.begin();
	   p != ls.end();
	   ++p) {
#调用context的complete函数又分为是否需要形参，*p 不为null，说明不需要形参
	if (*p) {
	  (*p)->complete(0);
	} else {
	  assert(!ls_rval.empty());
	  Context *c = ls_rval.front().first;
	  c->complete(ls_rval.front().second);
	  ls_rval.pop_front();
	}
      }
      ldout(cct, 10) << "finisher_thread done with " << ls << dendl;
      ls.clear();
   }
}
所以这里总结一下，看起来要结束操作的类会把自己添加到finisher类的queue中，然后finisher类分别调用要结束操作类context的complete函数，complete又调用子类的finisher(r),finisher()又调contex子类的_finisher(r).
```

