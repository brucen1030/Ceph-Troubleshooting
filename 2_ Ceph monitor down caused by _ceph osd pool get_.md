某天，运维反映一个测试环境中的Luminous版本Ceph集群连不上了。

登陆存储服务器，发现该集群的3个monitor都没有起来。考虑到Luminus版本不算太老，Ceph monitor一般没有什么问题，该集群也没什么IO压力，就直接拉起来观察。没想到拉起来之后没过多久又无法正常工作，3个monitor全部挂掉……

检查monitor对应的log，发现monitor异常退出之前都有类似打印：

```plain
-18> 2021-07-05T06:43:29.115+0000 7f3d3410c700 1 -- [v2:10.0.0.1:40236/0,v1:10.0.0.1:40237/0] --> 10.0.0.1:0/2820881513 -- osd_map(54..54 src has 1..54) v4 -- 0x5599b311a8c0 con 0x5599b2e4e000
-17> 2021-07-05T06:43:29.118+0000 7f3d3410c700 1 -- [v2:10.0.0.1:40236/0,v1:10.0.0.1:40237/0] <== client.? 10.0.0.1:0/2820881513 5 ==== mon_command({"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"} v 0) v1 ==== 105+0+0 (secure 0 0 0) 0x5599b2fea000 con 0x5599b2e4e000
-16> 2021-07-05T06:43:29.118+0000 7f3d3410c700 20 mon.a@0(leader) e1 _ms_dispatch existing session 0x5599b2e2c900 for client.?
-15> 2021-07-05T06:43:29.118+0000 7f3d3410c700 20 mon.a@0(leader) e1 entity_name client.admin global_id 4170 (new_ok) caps allow *
-14> 2021-07-05T06:43:29.118+0000 7f3d3410c700 0 mon.a@0(leader) e1 handle_command mon_command({"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"} v 0) v1
-13> 2021-07-05T06:43:29.118+0000 7f3d3410c700 20 is_capable service=osd command=osd pool get read addr 10.0.0.1:0/2820881513 on cap allow *
-12> 2021-07-05T06:43:29.118+0000 7f3d3410c700 20 allow so far , doing grant allow *
-11> 2021-07-05T06:43:29.118+0000 7f3d3410c700 20 allow all
-10> 2021-07-05T06:43:29.118+0000 7f3d3410c700 10 mon.a@0(leader) e1 _allowed_command capable
-9> 2021-07-05T06:43:29.118+0000 7f3d3410c700 0 log_channel(audit) log [DBG] : from='client.? 10.0.0.1:0/2820881513' entity='client.admin' cmd=[{"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"}]: dispatch
-8> 2021-07-05T06:43:29.118+0000 7f3d3410c700 10 log_client _send_to_mon log to self
-7> 2021-07-05T06:43:29.118+0000 7f3d3410c700 10 log_client log_queue is 3 last_log 224 sent 223 num 3 unsent 1 sending 1
-6> 2021-07-05T06:43:29.118+0000 7f3d3410c700 10 log_client will send 2021-07-05T06:43:29.118728+0000 mon.a (mon.0) 224 : audit [DBG] from='client.? 10.0.0.1:0/2820881513' entity='client.admin' cmd=[{"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"}]: dispatch
-5> 2021-07-05T06:43:29.118+0000 7f3d3410c700 1 -- [v2:10.0.0.1:40236/0,v1:10.0.0.1:40237/0] --> [v2:10.0.0.1:40236/0,v1:10.0.0.1:40237/0] -- log(1 entries from seq 224 at 2021-07-05T06:43:29.118728+0000) v1 -- 0x5599b311bc00 con 0x5599b2964000
-4> 2021-07-05T06:43:29.118+0000 7f3d3410c700 10 mon.a@0(leader).paxosservice(osdmap 1..54) dispatch 0x5599b2fea000 mon_command({"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"} v 0) v1 from client.? 10.0.0.1:0/2820881513 con 0x5599b2e4e000
-3> 2021-07-05T06:43:29.118+0000 7f3d3410c700 5 mon.a@0(leader).paxos(paxos active c 1..475) is_readable = 1 - now=2021-07-05T06:43:29.118900+0000 lease_expire=1970-01-01T00:00:00.000000+0000 has v0 lc 475
-2> 2021-07-05T06:43:29.118+0000 7f3d3410c700 10 mon.a@0(leader).osd e54 preprocess_query mon_command({"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"} v 0) v1 from client.? 10.0.0.1:0/2820881513
-1> 2021-07-05T06:43:29.133+0000 7f3d3410c700 -1 ../src/mon/OSDMonitor.cc: In function 'bool OSDMonitor::preprocess_command(MonOpRequestRef)' thread 7f3d3410c700 time 2021-07-05T06:43:29.119200+0000
../src/mon/OSDMonitor.cc: 6196: FAILED ceph_assert(i != ALL_CHOICES.end())
 
ceph version 17.0.0-5658-gc42712dd180 (c42712dd18032d9651e2ca5f6fb9ae5a078378df) quincy (dev)
1: (ceph::__ceph_assert_fail(char const*, char const*, int, char const*)+0x1aa) [0x7f3d4267d9a2]
2: /work/ceph/ceph8/build/lib/libceph-common.so.2(+0x1665c24) [0x7f3d4267dc24]
3: (OSDMonitor::preprocess_command(boost::intrusive_ptr<MonOpRequest>)+0x79e1) [0x5599aa7eb127]
4: (OSDMonitor::preprocess_query(boost::intrusive_ptr<MonOpRequest>)+0x240) [0x5599aa7c2dec]
5: (PaxosService::dispatch(boost::intrusive_ptr<MonOpRequest>)+0x99d) [0x5599aa7a263f]
6: (Monitor::handle_command(boost::intrusive_ptr<MonOpRequest>)+0x28ac) [0x5599aa4ca478]
7: (Monitor::dispatch_op(boost::intrusive_ptr<MonOpRequest>)+0xb67) [0x5599aa4d63cf]
8: (Monitor::_ms_dispatch(Message*)+0xfd0) [0x5599aa4d5502]
9: (Monitor::ms_dispatch(Message*)+0x4d) [0x5599aa51a671]
10: (Dispatcher::ms_dispatch2(boost::intrusive_ptr<Message> const&)+0x5c) [0x5599aa50d076]
11: (Messenger::ms_deliver_dispatch(boost::intrusive_ptr<Message> const&)+0xe9) [0x7f3d42875d9b]
12: (DispatchQueue::entry()+0x61d) [0x7f3d428747e7]
13: (DispatchQueue::DispatchThread::entry()+0x1c) [0x7f3d429f9dbe]
14: (Thread::entry_wrapper()+0x83) [0x7f3d4263dfa7]
15: (Thread::_entry_func(void*)+0x18) [0x7f3d4263df1a]
16: /lib64/libpthread.so.0(+0x814a) [0x7f3d3ed7e14a]
17: clone()
```
看log，应该是“ceph osd pool get PoolName param”这个命令出的问题。先找到用这个API的同事要来了能触发问题的脚本，核心部分非常简单，用API访问，相当于本来想“ceph osd pool get PoolName param”但是写错成“ceph osd pool get PoolName”。注意，用命令行是无法复现的，因为命令行本身已经检查过非法参数直接返回失败，问题出在ceph-monitor。
```python
import json
import rados
 
c = rados.Rados(conffile='/etc/ceph/ceph.conf')
c.connect()
cmd = json.dumps({"prefix": "osd pool get", "pool": "mytestpoolname", "format": "json"})
print (c.mon_command(cmd, b''))
```
找到对应代码，其实Luminus，Nautilus和master分支都是一样的。在处理“ceph osd pool get PoolName param”的时候，如果之后的参数param不支持，monitor应该返回错误码，而不是直接ceph_assert。
fix在下面这个链接，很简单：

[https://github.com/ceph/ceph/pull/42179](https://github.com/ceph/ceph/pull/42179)

