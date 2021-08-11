# 问题表象

某天，运维报了一个线上性能问题。某中型私有云池块存储，客户反应性能非常差，上面跑相同的业务，连预期的一半都达不到。

接到问题后，我们比对相同Ceph版本类似硬件下其它云池的性能，发现性能的确差异很大。而且在该集群新建云盘跑fio测试性能也压不上去，延迟非常高，业务高峰期有100ms以上，这种情况下，用户的IOPS其实只有我们理论最大值的约10%到20%。

诡异的是，没有scub或deepscrub，没有数据迁移，没有慢盘，dmesg很干净，所有RAID卡关掉了patrolread并且正确设置了WriteBack模式，iostat没有明显的异常所有盘看起来都很闲，rocksdb没有异常的compaction，没有进程异常退出记录，top看CPU和内存也都很正常。比对不同集群不同时段的监控数据，似乎只有流量变化，完全看不到任何线索。万般无奈之下下掉一块物理盘测试一下裸盘性能，和其它厂家的性能也是类似的，虽然略差但是看起来造不成这么大的问题。

唯一奇怪的测试结果是，如果在晚上客户很少的情况下新建云盘进行fio测试，性能看起来没太大问题，完全是正常的；但是同一块云盘白天有用户流量的时候测试性能就很差，我前一天晚上还回复无异常第二天白天就被打脸，尴尬了。但这个测试也说明不了问题，毕竟压力小的时候IO性能好也是合理的。

# 太长不看

这个私有云池的客户在部署给磁盘分区的时候用的是parted命令，客户mkpart的时候误将“mkpart primary xfs 0% 100%”写作“mkpart primary xfs 0 100%”，导致该私有云池中所有数据盘的起始扇区都不是4k对齐的，所有IO都不是4k对齐的，落到Ceph Bluestore的都是非对齐覆盖写。

BlueStore由于非对齐覆盖写需要调用一次或两次_do_read()同步读，对应head不对齐或tial不对齐，或head和tial都不对齐。如果在_do_read()中需要读取原数据，时间更长。从HDD磁盘上随机读取数据，延迟一般是ms级别，相对于simple write性能有比较大的退化，而且HDD默认只有5个线程写数据，很可能全都卡在等待同步读完成上，没有并发起来。

```shell
// 一个类似的 *错误* 示例
# parted /dev/vdb
GNU Parted 3.2
Using /dev/vdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt                                                      
Warning: The existing disk label on /dev/vdb will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? y                                                                                                                         
(parted) mkpart primary xfs 0 100%   // 应该是0% ！！！                                     
Warning: The resulting partition is not properly aligned for best performance: 34s % 2048s != 0s
Ignore/Cancel? Ignore                                                     
(parted) quit                                                             
 
// 用fdisk看一下，注意"Start"列的值，是34，不能被8整除也就是非4k对齐，并且最后一行有提示没有对齐。
// Start 34 sector应该是GPT的最小长度了。
# fdisk -lu /dev/vdb
Disk /dev/vdb: 100 GiB, xxxx bytes, xxxx sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
Disklabel type: gpt
Disk identifier: XXXXXXXXXX
 
 
Device     Start        End    Sectors   Size Type
/dev/vdb1     34 XXXXXXXX YYYYYYYY 100G Linux filesystem
 
 
Partition 1 does not start on physical sector boundary.
```
如果正确使用parted命令，fdisk显示应该是下面这样，Start列是2048（sector）。
```typescript
# fdisk -lu /dev/vdc
Disk /dev/vdc: 100 GiB, XXXXXXXX bytes, YYYYYY sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
Disklabel type: gpt
Disk identifier: 05129F3D-D2F1-4098-B971-6E4FED148771



Device     Start        End    Sectors   Size Type
/dev/vdc1   2048 XXXXXXXX YYYYYYYY 100G Linux filesystem
```

其它厂商也明确在用户文档里面提到必须4K对齐，比如

[https://support.huaweicloud.com/evs_faq/evs_faq_0080.html#evs_faq_0080__zh-cn_topic_0000001073763031_zh-cn_topic_0267670624_section83925230328](https://support.huaweicloud.com/evs_faq/evs_faq_0080.html#evs_faq_0080__zh-cn_topic_0000001073763031_zh-cn_topic_0267670624_section83925230328)

[https://help.aliyun.com/document_detail/147897.htm?spm=a2c4g.11186623.2.6.740a2e84Kni0e0#task-2363356](https://help.aliyun.com/document_detail/147897.htm?spm=a2c4g.11186623.2.6.740a2e84Kni0e0#task-2363356)

# 诊断过程

考虑到latency非常大，最后不得不从latency入手检查。运行以下命令监测单个osd的延迟。注意bluestore这部分，该集群有少量IO，但io_l列是0。这一列是simple write aio完成占用的时间，对应BlueStore代码中l_bluestore_state_aio_wait_lat，一般不会全为0，HDD中一般有毫秒级别至少。

另外s_l列值比较大，对应queue_transactions()完成统计的l_bluestore_submit_lat。对于simple write，该过程一般只有内存操作，延迟应该比较小才对。相应的c_l是commit latency，全是0也很奇怪。又仔细看了一遍代码，应该是由于有deferred write导致的，似乎也能说得过去……

```shell
# ceph daemonperf osd.0 
------bluefs------- ---------------bluestore--------------- -------------osd------------- -----prioritycache------ -----prioritycache:data------ 
jlen j    wal  sst |fl_l ks_l kf_l io_l th_l s_l  c_l  r_l |ops  wr   rd   l    rop  rbt |t    m    u    h    c   |p0   p1   p2   p3   p4   p5  |
 15M   0    0    0 |  0    0    0    0    0   11    0    0 |  0    0    0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  1.0M   0 |  0    0    0    0    0   12    0    0 |  8  118k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  312k   0 |  0    0    0    0    0    6    0    0 |  4   64k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  712k   0 |  0    0    0    0    0   15    0    0 |  7  105k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  678k   0 |  0    0    0    0    0    8    0    0 |  9  144k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  435k   0 |  0    0    0    0    0    9    0    0 | 12  110k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  1.0M   0 |  0    0    0    0    0   18    0    0 |  7  118k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  234k   0 |  0    0    0    0    0   19    0    0 | 11  102k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
 15M   0  272k   0 |  0    0    0    0    0   20    0    0 | 27  256k   0    0    0    0 |4.2G 4.2G 1.1G 5.3G 2.8G|  0  1.3G   0    0    0    0 
^C
```
运行 iostat -x检查磁盘IO，发现虽然上图用户基本没有读请求，但是HDD data盘上有读，平均大小为4k。这个就很奇怪，io数量和用户来的IO从监控上看对不上，算一下，平均读IO大小还是4k这种值。
```shell
# iostat -x 1
...

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sdb		0.00	  0.00   53.00   61.00   212.00  5564.00     0.01    0.07    0.73    0.04   0.02   0.37
sdc		0.00      3.00   21.00   76.00    84.00  3624.00    59.09     0.07    0.23    0.96    0.06   0.02   0.65
sdd		0.00      0.00   67.00   81.00   268.00  1350.00     0.01    0.07    0.69    0.03   0.02   0.36



```
运行以下命令检查统计。注意，必须在OSD所在服务器上运行。
```typescript
ceph daemon osd.0 perf reset
sleep 60
ceph daemon osd.0 perf dump
```
检查以下几项。
|**"bluestore": {**<br>**...**<br>**"state_prepare_lat": {**<br>**"avgcount": 4567,**<br>**"sum": 105.78929960,**<br>******"avgtime": 0.0257509  // 25ms，这个值给人感觉是有磁盘IO。如果正常对齐写这个值一般是很小的，毕竟queue_transactions大多是内存操作。什么情况下会在这个阶段有IO呢？**<br>**},**<br>**"state_aio_wait_lat": {**<br>**"avgcount": 4567,**<br>**"sum": 0.0382559234,**<br>**"avgtime": 0.000006262// 对应BlueStore状态机中aio wait，一般应该是5ms级别对应HDD的一次磁盘IO，这里非常小，应该是对应很多deferred write，没有在aio wait状态等待。**<br>**},**<br>**...**<br>**"state_deferred_queued_lat": {**<br>**"avgcount": 4562,**<br>**"sum": 2292.078134233,**<br>**"avgtime": 0.577432341 // 这个就非常大了，500ms，但考虑到commit在此之前，应该不是deferred aio阶段造成的集群性能差。**<br>**},**<br>**"state_deferred_aio_wait_lat": {**<br>**"avgcount": 4572,**<br>**"sum": 3.300695320,**<br>**"avgtime": 0.000718531**<br>**},**<br>**...**<br>**"deferred_write_ops": 3742,**<br>**...**<br>******"write_penalty_read_ops": 8172, // 这个值引起了我的注意。在其它正常集群一般很小。**<br>**...**<br>**"bluestore_txc": 4562,**<br>**...**<br>**"osd": {**<br>**...**<br>**"op_w": 1234,**<br>**"op_w_in_bytes": 49020163,**<br>**"op_w_latency": {**<br>**"avgcount": 1234,**<br>**"sum": 710.40573,**<br>**"avgtime": 0.70823     // 这几个latency都很夸张。平均一共用了700多ms，其中等待38ms，处理140ms+，完全不能接受。**<br>**},**<br>**"op_w_process_latency": {**<br>**"avgcount": 1234,**<br>**"sum": 147.42239729,**<br>******"avgtime": 0.146981449**<br>**},**<br>**"op_w_prepare_latency": {**<br>**"avgcount": 1234,**<br>**"sum": 38.17929124,**<br>**"avgtime": 0.038023857**<br>**},**<br>**...**<br>**"rocksdb": { // rocksdb元数据占用的时间并不大，略**|
|:----|
用其它命令比如“ceph daemon osd.0 dump_historic_ops | grep duration”看，latency也很夸张，几乎到了秒级别，完全不能忍。

在代码中搜了一下write_penalty_read_ops这个值，发现幸好只有一处，就是在_do_write_small中用于统计l_bluestore_write_penalty_read_ops，对应非对齐写。_do_write_small()会调用_do_read()，进行一次同步读操作，从HDD盘上读出数据，并以deferred write方式写到rocksdb里。

如前所述，非对齐覆盖写对于性能影响非常大，其它厂商也不支持在这种情况下保证性能。而且系统中write_penalty_read_ops有8172，而bluestore中的deferred write才有3762次，说明大多数deferred write伴随着读惩罚。

注意，增加线程或者更改deferred write参数是无法解决这个问题的。具体可以检查“ceph daemon osd.N perf dump”中deferred_write_ops，write_penalty_read_ops相对OSD的总IOPS数。如果这两个值中write_penalty_read_ops很大，或者发现明明用户IO读很少，但落盘的读IO很多甚至和写IO成比例，就非常值得怀疑是客户端发来的IO是非对齐覆盖写IO造成的性能差。注意，单deferred_write_ops大并不一定是问题，因为Ceph默认允许高效块也就是HDD OSD上对于小IO进行deferred write，具体参考bluestore_prefer_deferred_size。

有了上面统计值作为依据，我们可以合理怀疑这个系统里面大多数写IO都是非对齐的。可以短时间内打开osd的日志打印级别20，来验证一下。也可以在有问题的时候用下面这个命令检查一下，结果类似

|# ceph daemon osd.0 dump_historic_ops \| grep write<br>            "description": "osd_op(client.749653.0:151048 17.689 17:9178cf30:::rbd_data.b4a68be1376fe.00000000000043ee:head [set-alloc-hint object_size 4194304 write_size 4194304,write 844800~12288] snapc 0=[] ondisk+write+known_if_redirected e11285)",<br><br># python<br>Python 2.7.5 (default, Oct 14 2020, 14:45:30) <br>[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux2<br>Type "help", "copyright", "credits" or "license" for more information.<br>>>> 844800.0 / 4096        // 起始地址不是4k对齐的。<br>206.25<br>|
|:----|
事实上我们发现有些IO不光不4k对齐，有的甚至不是1k对齐的……

更恶劣的是，如果一个中小型Ceph集群中小非对齐覆盖写非常多，整个Ceph集群的性能都会变得非常差，即使新建云盘进行对齐IO测试，也会发现性能很差。

# 背景知识

下面这个链接详细讲了BlueStore状态机的各个状态和IO流程。“对于用户或osd层面的一次IO写请求，到BlueStore这一层，可能是simple write，也可能是deferred write，还有可能既有simple write的场景，也有deferred write的场景。”具体流程可以参考[http://blog.wjin.org/posts/ceph-bluestore.html](http://blog.wjin.org/posts/ceph-bluestore.html)。




