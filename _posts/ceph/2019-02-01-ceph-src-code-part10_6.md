---
layout: post
title: ceph peering机制再研究(2)
tags:
- ceph
categories: ceph
description: ceph源代码分析
---

本节我们介绍一下PG Recovery过程中的一些重要数据结构。


<!-- more -->

## 1. RecoveryCtx数据结构
{% highlight string %}
class PG : DoutPrefixProvider {
public:    
  struct BufferedRecoveryMessages {
	map<int, map<spg_t, pg_query_t> > query_map;
	map<int, vector<pair<pg_notify_t, pg_interval_map_t> > > info_map;
	map<int, vector<pair<pg_notify_t, pg_interval_map_t> > > notify_list;
  };
	
  struct RecoveryCtx {
	utime_t start_time;
	map<int, map<spg_t, pg_query_t> > *query_map;
	map<int, vector<pair<pg_notify_t, pg_interval_map_t> > > *info_map;
	map<int, vector<pair<pg_notify_t, pg_interval_map_t> > > *notify_list;
	set<PGRef> created_pgs;
	C_Contexts *on_applied;
	C_Contexts *on_safe;
	ObjectStore::Transaction *transaction;
	ThreadPool::TPHandle* handle;
	
	RecoveryCtx(map<int, map<spg_t, pg_query_t> > *query_map,
		map<int,vector<pair<pg_notify_t, pg_interval_map_t> > > *info_map,
		map<int,vector<pair<pg_notify_t, pg_interval_map_t> > > *notify_list,
		C_Contexts *on_applied,
		C_Contexts *on_safe,
		ObjectStore::Transaction *transaction)
	: query_map(query_map), info_map(info_map), 
	notify_list(notify_list),
	on_applied(on_applied),
	on_safe(on_safe),
	transaction(transaction),
	handle(NULL) {}

	RecoveryCtx(BufferedRecoveryMessages &buf, RecoveryCtx &rctx)
	: query_map(&(buf.query_map)),
	info_map(&(buf.info_map)),
	notify_list(&(buf.notify_list)),
	on_applied(rctx.on_applied),
	on_safe(rctx.on_safe),
	transaction(rctx.transaction),
	handle(rctx.handle) {}

	void accept_buffered_messages(BufferedRecoveryMessages &m) {
		assert(query_map);
		assert(info_map);
		assert(notify_list);
		
		for (map<int, map<spg_t, pg_query_t> >::iterator i = m.query_map.begin();i != m.query_map.end();++i) {
			map<spg_t, pg_query_t> &omap = (*query_map)[i->first];
			
			for (map<spg_t, pg_query_t>::iterator j = i->second.begin();j != i->second.end();++j) {
				omap[j->first] = j->second;
			}
		}
		
		for (map<int, vector<pair<pg_notify_t, pg_interval_map_t> > >::iterator i = m.info_map.begin();i != m.info_map.end();++i) {
			vector<pair<pg_notify_t, pg_interval_map_t> > &ovec = (*info_map)[i->first];
			ovec.reserve(ovec.size() + i->second.size());
			ovec.insert(ovec.end(), i->second.begin(), i->second.end());
		}
		
		for (map<int, vector<pair<pg_notify_t, pg_interval_map_t> > >::iterator i = m.notify_list.begin();i != m.notify_list.end();++i) {
			vector<pair<pg_notify_t, pg_interval_map_t> > &ovec = (*notify_list)[i->first];
			ovec.reserve(ovec.size() + i->second.size());
			ovec.insert(ovec.end(), i->second.begin(), i->second.end());
		}
	}
	
  };
  
  
};
{% endhighlight %}
RecoveryCtx作为一次恢复操作的上下文，我们介绍一下其中几个比较重要的字段：

* query_map： 用于缓存PG Query查询信息，后续会将这些缓存信息构造成MOSDPGQuery消息，然后发送到对应的OSD上。query_map的key部分为OSD的序号。

* info_map: 用于缓存pg_notify_t信息，后续会将这些缓存信息构造成MOSDPGInfo查询的消息，然后发送到对应的OSD上。info_map的key部分为OSD的序号。

* notify_list：用于缓存pg_notify_t信息，后续会将这些缓存信息构造成MOSDPGNotify消息，然后发送到对应的OSD上。notify_list的key部分为OSD的序号

* transaction：本RecoveryCtx所关联的事务。在恢复过程中可能涉及到需要将相关信息持久化，就通过此transaction来完成

* handle：ThreadPool::TPHandle的主要作用在于监视线程池中每一个线程的执行时常。每次线程函数执行时，都会设置一个grace超时时间，当线程执行超过该时间，就认为是unhealthy的状态。当执行时间超过suicide_grace时，OSD就会产生断言而导致自杀。这里向transaction对应的ObjectStore传入handle参数，主要是为了处理超时方面的问题
{% highlight string %}
int FileStore::queue_transactions(Sequencer *posr, vector<Transaction>& tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
	...

	if (handle)
		handle->suspend_tp_timeout();
	
	op_queue_reserve_throttle(o);
	journal->reserve_throttle_and_backoff(tbl.length());
	
	if (handle)
		handle->reset_tp_timeout();

	...
}
{% endhighlight %}

### 1.1 RecoveryCtx的使用
在OSD中，通常使用如下函数来创建RecoveryCtx对象：
{% highlight string %}
// ----------------------------------------
// peering and recovery

PG::RecoveryCtx OSD::create_context()
{
	ObjectStore::Transaction *t = new ObjectStore::Transaction;
	C_Contexts *on_applied = new C_Contexts(cct);
	C_Contexts *on_safe = new C_Contexts(cct);

	map<int, map<spg_t,pg_query_t> > *query_map = 
		new map<int, map<spg_t, pg_query_t> >;

	map<int,vector<pair<pg_notify_t, pg_interval_map_t> > > *notify_list =
		new map<int, vector<pair<pg_notify_t, pg_interval_map_t> > >;

	map<int,vector<pair<pg_notify_t, pg_interval_map_t> > > *info_map =
		new map<int,vector<pair<pg_notify_t, pg_interval_map_t> > >;

	PG::RecoveryCtx rctx(query_map, info_map, notify_list,on_applied, on_safe, t);
	return rctx;
}
{% endhighlight %}

之后，调用如下函数将对应map里面的数据发送出去：
{% highlight string %}
void OSD::dispatch_context(PG::RecoveryCtx &ctx, PG *pg, OSDMapRef curmap,
                           ThreadPool::TPHandle *handle)
{
	if (service.get_osdmap()->is_up(whoami) &&is_active()) {
		do_notifies(*ctx.notify_list, curmap);
		do_queries(*ctx.query_map, curmap);
		do_infos(*ctx.info_map, curmap);
	}
	delete ctx.notify_list;
	delete ctx.query_map;
	delete ctx.info_map;

	if ((ctx.on_applied->empty() &&ctx.on_safe->empty() &&
	  ctx.transaction->empty() && ctx.created_pgs.empty()) || !pg) {
		delete ctx.transaction;
		delete ctx.on_applied;
		delete ctx.on_safe;
		assert(ctx.created_pgs.empty());
	} else {
		if (!ctx.created_pgs.empty()) {
			ctx.on_applied->add(new C_OpenPGs(ctx.created_pgs, store));
		}

		int tr = store->queue_transaction(
			pg->osr.get(),
			std::move(*ctx.transaction), ctx.on_applied, ctx.on_safe, NULL, TrackedOpRef(),
			handle);

		delete (ctx.transaction);
		assert(tr == 0);
	}
}
{% endhighlight %}

## 2. NamedState数据结构
{% highlight string %}
class PG : DoutPrefixProvider {
public:    
  struct NamedState {
	const char *state_name;
	utime_t enter_time;
	
	const char *get_state_name() { return state_name; }
	
	NamedState(CephContext *cct_, const char *state_name_)
		: state_name(state_name_),
		enter_time(ceph_clock_now(cct_)) {}
		
	virtual ~NamedState() {}
  };
  
};
{% endhighlight %}
NamedState主要是用于对一种状态进行命名。




## 3. statechart状态机

Ceph在处理PG的状态转换时，使用了boost库提供的statechart状态机。因此，这里先简单介绍一下statechart状态机的基本概念和涉及的相关知识，以便更好地理解Peering过程PG的状态转换流程。

### 3.1 状态
在statechart里，状态的定义有两种方式：

* 没有子状态情况下的状态定义
{% highlight string %}
//boost
template< class MostDerived,
          class Context,
          class InnerInitial = mpl::list<>,
          history_mode historyMode = has_no_history >
class state : public simple_state<
  MostDerived, Context, InnerInitial, historyMode >
{
};

//ceph
struct Reset : boost::statechart::state< Reset, RecoveryMachine >, NamedState {
};
{% endhighlight %}
定义一个状态需要继承boost::statechart::simple_state或者boost::statechart::state类。上面Reset状态继承了boost::statechart::state类。该类的模板参数中，第一个参数为状态机自己的名字Reset，第二个参数为所属状态机的名字，表明Reset是状态机RecoveryMachine的一个状态。

* 有子状态情况下的状态定义
{%	highlight string %}
struct Start;

struct Started : boost::statechart::state< Started, RecoveryMachine, Start >, NamedState {
};
{% endhighlight %}
状态```Started```也是状态机RecoveryMachine的一个状态，模板参数中多了一个参数```Start```，它是状态```Started```的默认初始子状态，其定义如下：
{% highlight string %}
struct Start : boost::statechart::state< Start, Started >, NamedState {
};
{% endhighlight %}
上面定义的```状态Start```是状态Started的子状态。第一个模板参数是自己的名字，第二个模板参数是该子状态所属父状态的名字。

综上所述，一个状态，要么属于一个状态机，要么属于一个状态，成为该状态的子状态。其定义的模板参数是自己，第二个模板参数是拥有者，第三个模板参数是它的起始子状态。

### 3.2 事件
状态能够接收并处理事件。事件可以改变状态，促使状态发生转移。在boost库的statechart状态机中定义事件的方式如下所示：
{% highlight string %}
struct QueryState : boost::statechart::event< QueryState >{
}; 
{% endhighlight %}
QueryState为一个事件，需要继承boost::statechart::event类，模板参数为自己的名字。

### 3.3 状态响应事件
在一个状态内部，需要定义状态机处于当前状态时，可以接受的事件以及如何处理这些事件的方法：
{% highlight string %}
{% endhighlight %}



### 3.1 statechart示例
{% highlight string %}
// Example program
#include <iostream>
#include <string>
#include <boost/statechart/custom_reaction.hpp>
#include <boost/statechart/event.hpp>
#include <boost/statechart/simple_state.hpp>
#include <boost/statechart/state.hpp>
#include <boost/statechart/state_machine.hpp>
#include <boost/statechart/transition.hpp>
#include <boost/statechart/event_base.hpp>



#define TrivialEvent(T) struct T : boost::statechart::event< T > { \
    T() : boost::statechart::event< T >() {}			   \
    void print(std::ostream *out) const {			   \
      *out << #T;						   \
    }								   \
  };
TrivialEvent(Initialize)
TrivialEvent(Load)
TrivialEvent(NullEvt)
TrivialEvent(GoClean)

struct MInfoRec : boost::statechart::event< MInfoRec > {
    std::string name; 
    MInfoRec(std::string name): name(name){
    }
    
    void print(){
        std::cout<<"MInfoRec: "<<name<<"\n";
    }
};

struct MLogRec : boost::statechart::event< MLogRec > {
    std::string name; 
    MLogRec(std::string name): name(name){
    }
    
    void print(){
        std::cout<<"MLogRec: "<<name<<"\n";
    }
};

struct MNotifyRec : boost::statechart::event< MNotifyRec > {
    std::string name; 
    MNotifyRec(std::string name): name(name){
        
    }
    
    void print(){
        std::cout<<"MNotifyRec: "<<name<<"\n";
    }
};



struct Initial; 

struct RecoveryMachine : boost::statechart::state_machine< RecoveryMachine, Initial > {}; 

struct Reset;

struct Crashed : boost::statechart::state< Crashed, RecoveryMachine > {
    explicit Crashed(my_context ctx) : my_base(ctx)
    {
        std::cout << "Hello, Crashed!\n";
    }
};
    
struct Initial : boost::statechart::state< Initial, RecoveryMachine > {
    
    typedef boost::mpl::list < 
    boost::statechart::transition< Initialize, Reset >,
    boost::statechart::custom_reaction< Load >,
    boost::statechart::custom_reaction< NullEvt >,
    boost::statechart::transition< boost::statechart::event_base, Crashed >
    > reactions;
    
    explicit Initial(my_context ctx) : my_base(ctx)
    {
        std::cout << "Hello, Initial!\n";
    } 
    
    boost::statechart::result react(const Load& l){
        return transit< Reset >();
    }
    
     boost::statechart::result react(const MNotifyRec& notify){
         std::cout<<"Initial::react::MLogRec!\n";
         
         return discard_event();
     }
    //  boost::statechart::result react(const MInfoRec& i){
    //      std::cout<<"Initial::react::MNotifiyRec!\n";
         
    //      return discard_event();
    //  }
    //  boost::statechart::result react(const MLogRec& log){
    //      std::cout<<"Initial::react::MLogRec!\n";
         
    //      return discard_event();
    //  }
    
    boost::statechart::result react(const boost::statechart::event_base&) {
        std::cout << "Initial event_base processed!\n";
	    return discard_event();
    }
    
    void exit() { 
        std::cout << "Bye, Initial!\n";
    } 
};

struct Reset : boost::statechart::state< Reset, RecoveryMachine > {
    explicit Reset(my_context ctx) : my_base(ctx)
    {
        std::cout << "Hello, Reset!\n";
    } 
    
    void exit() { 
        std::cout << "Bye, Reset!\n";
    }
};

int main(int argc, char *argv[])
{
  RecoveryMachine machine;
  
  machine.initiate();
  
  //machine.process_event(NullEvt());
  
  //machine.process_event(GoClean());
  
  machine.process_event(MNotifyRec("notify record"));
  
  
  
  return 0x0;
}
{% endhighlight %}

Recovery过程中的所有事件

* QueryState

* MInfoRec

* MLogRec

* MNotifyRec

* MQuery

* AdvMap

* ActMap

* Activate

* RequestBackfillPrio

*  TrivialEvent事件
{% highlight string %}
class PG : DoutPrefixProvider {
public:    
#define TrivialEvent(T) struct T : boost::statechart::event< T > { \
    T() : boost::statechart::event< T >() {}			   \
    void print(std::ostream *out) const {			   \
      *out << #T;						   \
    }								   \
  };
  TrivialEvent(Initialize)
  TrivialEvent(Load)
  TrivialEvent(GotInfo)
  TrivialEvent(NeedUpThru)
  TrivialEvent(CheckRepops)
  TrivialEvent(NullEvt)
  TrivialEvent(FlushedEvt)
  TrivialEvent(Backfilled)
  TrivialEvent(LocalBackfillReserved)
  TrivialEvent(RemoteBackfillReserved)
  TrivialEvent(RemoteReservationRejected)
  TrivialEvent(RequestBackfill)
  TrivialEvent(RequestRecovery)
  TrivialEvent(RecoveryDone)
  TrivialEvent(BackfillTooFull)

  TrivialEvent(AllReplicasRecovered)
  TrivialEvent(DoRecovery)
  TrivialEvent(LocalRecoveryReserved)
  TrivialEvent(RemoteRecoveryReserved)
  TrivialEvent(AllRemotesReserved)
  TrivialEvent(AllBackfillsReserved)
  TrivialEvent(Recovering)
  TrivialEvent(GoClean)

  TrivialEvent(AllReplicasActivated)

  TrivialEvent(IntervalFlush)
  
};
{% endhighlight %}




<br />
<br />

**[参看]**

1. [ceph osd heartbeat 分析](https://blog.csdn.net/ygtlovezf/article/details/72330822)

2. [boost官网](https://www.boost.org/doc/)

3. [在线编译器](https://blog.csdn.net/weixin_39846364/article/details/112328477)

4. [boost在线编译器](http://cpp.sh/)
<br />
<br />
<br />

