在测试中发现一个很奇怪的问题，在性能比较好的全闪Ceph块存储集群上，新建一个云盘，从另一个虚拟机中去做fio 4k随机写测试，如下：

|fio -direct=1 -iodepth=128 -rw=randwrite -ioengine=libaio -bs=4k -numjobs=1 -size=1G -runtime=1000 -group_reporting -filename=/dev/vdb -name=Rand_Write_Testing -time_based|
|:----|
公有云中测试普通全闪和高效云盘的脚本都是类似的，比如

[https://help.aliyun.com/document_detail/147897.htm?spm=a2c4g.11186623.2.6.740a2e84Kni0e0#task-2363356](https://help.aliyun.com/document_detail/147897.htm?spm=a2c4g.11186623.2.6.740a2e84Kni0e0#task-2363356)

[https://support.huaweicloud.com/evs_faq/evs_faq_0019.html](https://support.huaweicloud.com/evs_faq/evs_faq_0019.html)

一般说来，调整depth和numjobs有可能得到更高的IOPS（当然也是更高的平均延迟）。但这次无论我怎么调整这个两个值，如果是在虚拟机中测试云盘，都无法达到预期，感觉是哪里有瓶颈。相反，如果找一台物理服务器，如下直接通过ioengine=rbd访问，可以取得很高的IOPS。

|fio -direct=1 -iodepth=128 -rw=randwrite -ioengine=rbd -bs=4k -size=1G -numjobs=1 -runtime=1000 -pool=rbd -rbdname=disk1 -clientname=admin -group_reporting -name=Rand_Write_Testing -time_based|
|:----|
开始我以为是使用的虚拟机或者该虚拟机的宿主机太差，遂想方设法找了一台同网段内配置更好的服务器，重做了一个虚拟机。结果发现完全没有区别。所以怀疑是虚拟机内部IO机制哪里配置的不对。之后又反复改过虚拟机cpu数和内存，完全没有用。在比如红帽官网等网站上找的IO方面的优化和配置也没有效果。直接后来发现了这个网页，虽然写的很简洁但的确有效：

[http://kvmonz.blogspot.com/p/knowledge-disk-performance-hints-tips.html](http://kvmonz.blogspot.com/p/knowledge-disk-performance-hints-tips.html)

结论是，在虚拟机中单卷性能实际受限于qemu的性能，无论fio增加多少numjobs，结果都无法有效提高单个云盘4k写IOPS。测试和实际使用高性能 Ceph集群时，必须解决qeum的性能瓶颈，使用独立的io thead，才能达到预期IOPS性能。

该问题与qemu实现有关。默认qemu使用单个IO线程，在高IOPS场景下就有可能达到瓶颈。

一个参考虚拟机配置：

|**# virsh dumpxml YOUR_VM_NAME**<br>**<domain type='kvm' id='N'>**<br>**<name>YOUR_VM_NAME</name>**<br>**<uuid>XXX</uuid>**<br>**<memory unit='KiB'>67108864</memory>**<br>**<currentMemory unit='KiB'>67108864</currentMemory>**<br>**<vcpu placement='static'>16</vcpu>**<br>**<iothreads>2</iothreads> ### 注意，如果后面写iothread='2'，那这句必须有，否则报错“error: Failed to start domain test_io**<br>**error: unsupported configuration: Disk iothread '2' not defined in iothreadid”**<br>**...**<br>**<disk type='network' device='disk'>**<br>**<driver name='qemu' type='raw' cache='none' iothread='2'/> ###  io='native', 或‘threads’都可以，实现略有不同。**<br>**<source protocol='rbd' name='rbd/disk1'>**<br>**<host name='10.0.0.1' port='6789'/>**<br>**<auth username='admin'>**<br>**<secret type='ceph' uuid='XXXXXXX'/>**<br>**</auth>**<br>**</source>**<br>**...**|
|:----|





