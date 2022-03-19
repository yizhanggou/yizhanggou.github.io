---
title: "Gluster vs Ceph"
author: 一张狗
lastmod: 2019-07-06 08:15:08
date: 2018-08-15 14:30:15
tags: []
---



# 测试报告：

首先祭出2014年网上的测试报告：

[Donvito_2014_J._Phys._Conf._Ser._513_042014](http://yizhanggou.top/wp-content/uploads/2018/08/Donvito_2014_J._Phys.3A_Conf._Ser._513_042014.pdf)

其中最重要的对比图：

![387ea4d231056b470d14de62de2167b1](http://yizhanggou.top/imgs/2019/07/387ea4d231056b470d14de62de2167b1.png)
显示glusterFS的读写速度是最优的。


# 自测结果:

cephfs和glustefs都是部署的三节点；


## cephfs:
```
[work@localhost io-test]$ fio -filename=/data/cephfs/io-test/sdb1 -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=10G -numjobs=30 -runtime=100 -group_reporting -name=mytest1
mytest1: (g=0): rw=randrw, bs=(R) 16.0KiB-16.0KiB, (W) 16.0KiB-16.0KiB, (T) 16.0KiB-16.0KiB, ioengine=psync, iodepth=1
...
fio-3.1
Starting 30 threads
mytest1: Laying out IO file (1 file / 10240MiB)
Jobs: 30 (f=30): [m(30)][100.0%][r=12.7MiB/s,w=5392KiB/s][r=811,w=337 IOPS][eta 00m:00s]
mytest1: (groupid=0, jobs=30): err= 0: pid=33932: Wed Aug 15 17:01:47 2018
read: IOPS=333, BW=5333KiB/s (5461kB/s)(521MiB/100117msec)
#完成延迟（completion latency）：表示提交给kernel后到IO做完之间的时间，不包括submission latency，这是评估延迟性能最好指标
clat (usec): min=245, max=5155.0k, avg=10128.49, stdev=94849.15
lat (usec): min=246, max=5155.0k, avg=10128.77, stdev=94849.16
#完成延迟百分数
clat percentiles (usec):
| 1.00th=[ 326], 5.00th=[ 375], 10.00th=[ 412],
| 20.00th=[ 498], 30.00th=[ 603], 40.00th=[ 725],
| 50.00th=[ 873], 60.00th=[ 1037], 70.00th=[ 1221],
| 80.00th=[ 1467], 90.00th=[ 1844], 95.00th=[ 2376],
| 99.00th=[ 200279], 99.50th=[ 484443], 99.90th=[1501561],
| 99.95th=[1635779], 99.99th=[3305112]
#带宽
bw ( KiB/s): min= 31, max= 2400, per=4.48%, avg=239.02, stdev=230.10, samples=4468
iops : min= 1, max= 150, avg=14.93, stdev=14.38, samples=4468
write: IOPS=144, BW=2318KiB/s (2373kB/s)(227MiB/100117msec)
clat (msec): min=11, max=5354, avg=183.73, stdev=267.18
lat (msec): min=11, max=5354, avg=183.73, stdev=267.18
clat percentiles (msec):
| 1.00th=[ 16], 5.00th=[ 19], 10.00th=[ 59], 20.00th=[ 83],
| 30.00th=[ 94], 40.00th=[ 110], 50.00th=[ 131], 60.00th=[ 153],
| 70.00th=[ 178], 80.00th=[ 209], 90.00th=[ 288], 95.00th=[ 409],
| 99.00th=[ 1351], 99.50th=[ 1888], 99.90th=[ 3507], 99.95th=[ 3742],
| 99.99th=[ 4799]
bw ( KiB/s): min= 31, max= 608, per=4.26%, avg=98.74, stdev=76.36, samples=4690
iops : min= 1, max= 38, avg= 6.16, stdev= 4.78, samples=4690
#下面四行，这是一组组数据，表示延迟，只是单位不同
lat (usec) : 250=0.01%, 500=14.07%, 750=14.96%, 1000=11.31%
lat (msec) : 2=24.00%, 4=2.70%, 10=0.22%, 20=2.62%, 50=0.92%
lat (msec) : 100=8.00%, 250=16.66%, 500=3.01%, 750=0.44%, 1000=0.17%
lat (msec) : 2000=0.77%, >=2000=0.15%
#用户/系统CPU占用率，进程上下文切换(context switch)次数，主要和次要(major and minor)页面错误数量(page faults)。（若使用直接IO，page faults数量应该极少）
cpu : usr=0.01%, sys=0.04%, ctx=48741, majf=4, minf=31
#Fio有一个iodepth设置，用来控制同一时刻发送给OS多少个IO。这完全是纯应用层面的行为，和盘的IO queue不是一回事。这里iodepth设成1，所以IO depth在全部时间都是1
IO depths : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
#submit和complete代表同一时间段内fio发送上去和已完成的IO数量。对于产生这个输出的垃圾回收测试用例来说，iodepth是默认值1，所以100%的IO在同一时刻发送1次，放在1-4栏位里。通常来说，只有iodepth大于1才需要关注这一部分数据
submit : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
complete : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
issued rwt: total=33373,14503,0, short=0,0,0, dropped=0,0,0
latency : target=0, window=0, percentile=100.00%, depth=1
Run status group 0 (all jobs):
#io方式，io：总的IO量, bw：带宽KB/s, iops：每秒钟的IO数, runt：总运行时间, lat (msec)：延迟(毫秒), msec毫秒, usec 微秒
READ: bw=5333KiB/s (5461kB/s), 5333KiB/s-5333KiB/s (5461kB/s-5461kB/s), io=521MiB (547MB), run=100117-100117msec
WRITE: bw=2318KiB/s (2373kB/s), 2318KiB/s-2318KiB/s (2373kB/s-2373kB/s), io=227MiB (238MB), run=100117-100117msec
```


## glusterfs:
```
[root@localhost data]# fio -filename=/home/gluster/data/sdb1 -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=10G -numjobs=30 -runtime=100 -group_reporting -name=mytest1
mytest1: (g=0): rw=randrw, bs=(R) 16.0KiB-16.0KiB, (W) 16.0KiB-16.0KiB, (T) 16.0KiB-16.0KiB, ioengine=psync, iodepth=1
...
fio-3.1
Starting 30 threads
mytest1: Laying out IO file (1 file / 10240MiB)
fio: native_fallocate call failed: Operation not supported

Jobs: 30 (f=30): [m(30)][100.0%][r=7152KiB/s,w=2608KiB/s][r=447,w=163 IOPS][eta 00m:00s]
mytest1: (groupid=0, jobs=30): err= 0: pid=22014: Wed Aug 15 18:51:14 2018
read: IOPS=408, BW=6529KiB/s (6685kB/s)(638MiB/100080msec)
clat (usec): min=639, max=1284.7k, avg=71869.26, stdev=83535.79
lat (usec): min=639, max=1284.7k, avg=71869.37, stdev=83535.79
clat percentiles (msec):
| 1.00th=[ 5], 5.00th=[ 8], 10.00th=[ 10], 20.00th=[ 17],
| 30.00th=[ 24], 40.00th=[ 33], 50.00th=[ 45], 60.00th=[ 59],
| 70.00th=[ 80], 80.00th=[ 110], 90.00th=[ 167], 95.00th=[ 230],
| 99.00th=[ 401], 99.50th=[ 493], 99.90th=[ 751], 99.95th=[ 877],
| 99.99th=[ 986]
bw ( KiB/s): min= 32, max= 608, per=3.37%, avg=220.13, stdev=93.20, samples=5931
iops : min= 2, max= 38, avg=13.76, stdev= 5.83, samples=5931
write: IOPS=176, BW=2828KiB/s (2896kB/s)(276MiB/100080msec)
clat (usec): min=184, max=849929, avg=3721.72, stdev=35582.29
lat (usec): min=185, max=849929, avg=3722.16, stdev=35582.29
clat percentiles (usec):
| 1.00th=[ 265], 5.00th=[ 347], 10.00th=[ 775], 20.00th=[ 832],
| 30.00th=[ 865], 40.00th=[ 914], 50.00th=[ 1004], 60.00th=[ 1237],
| 70.00th=[ 1467], 80.00th=[ 1762], 90.00th=[ 2868], 95.00th=[ 4293],
| 99.00th=[ 10028], 99.50th=[ 22938], 99.90th=[683672], 99.95th=[809501],
| 99.99th=[843056]
bw ( KiB/s): min= 32, max= 448, per=3.89%, avg=109.92, stdev=70.56, samples=5148
iops : min= 2, max= 28, avg= 6.87, stdev= 4.41, samples=5148
lat (usec) : 250=0.16%, 500=1.70%, 750=0.51%, 1000=12.68%
lat (msec) : 2=9.94%, 4=3.70%, 10=8.31%, 20=10.82%, 50=20.07%
lat (msec) : 100=16.18%, 250=12.98%, 500=2.57%, 750=0.29%, 1000=0.09%
lat (msec) : 2000=0.01%
cpu : usr=0.01%, sys=0.04%, ctx=73176, majf=0, minf=3
IO depths : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
submit : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
complete : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
issued rwt: total=40837,17689,0, short=0,0,0, dropped=0,0,0
latency : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
READ: bw=6529KiB/s (6685kB/s), 6529KiB/s-6529KiB/s (6685kB/s-6685kB/s), io=638MiB (669MB), run=100080-100080msec
WRITE: bw=2828KiB/s (2896kB/s), 2828KiB/s-2828KiB/s (2896kB/s-2896kB/s), io=276MiB (290MB), run=100080-100080msec

Disk stats (read/write):
dm-2: ios=40798/19432, merge=0/0, ticks=2851714/420383, in_queue=3275352, util=100.00%, aggrios=40843/19341, aggrmerge=0/201, aggrticks=2859858/404609, aggrin_queue=3264438, aggrutil=100.00%
sda: ios=40843/19341, merge=0/201, ticks=2859858/404609, in_queue=3264438, util=100.00%
```


## 测试结果显示glusterfs略胜一筹

附上fio的使用方法：https://www.cnblogs.com/shiyiwen/p/5407896.html

filename=/dev/sdb1 #对整个磁盘或分区测试，也可以用于对裸设备进行测试（会损坏数据的，除非你知道你的目的，否在请不要使用）  
 directory=/root/ss #对本地磁盘的某个目录进行测试（ filename | directory 二者选一）

direct=1 #测试过程绕过机器自带的buffer。使测试结果更真实。（布尔型）  
 rw=read #测试顺序读的I/O，下面是可选的参数  
 （ write 顺序写 | read 顺序读 | rw,readwrite 顺序混合读写 | randwrite 随机写 | randread 随机读 | randrw 随机混合读写）

ioengine=libaio #定义使用哪种IO，此为libaio，（默认是sync）  
 userspace_reap #配合libaio，提高异步io的收割速度（只能配合libaio引擎使用）

iodepth=16 #设置IO队列的深度，表示在这个文件上同一时刻运行16个I/O，默认为1。如果ioengine采用异步方式，该参数表示一批提交保持的io单元数。  
 iodepth_batch=8 #当队列里面的IO数量达到8值的时候，就调用io_submit批次提交请求，然后开始调用io_getevents开始收割已经完成的IO  
 iodepth_batch_complete=8 #每次收割多少呢？由于收割的时候，超时时间设置为0，所以有多少已完成就算多少，最多可以收割iodepth_batch_complete值个  
 iodepth_low=8 #随着收割，IO队列里面的IO数就少了，那么需要补充新的IO，当IO数目降到iodepth_low值的时候，就重新填充，保证OS可以看到至少iodepth_low数目的io在电梯口排队着  
 thread #fio使用线程而不是进程  
 bs=4k #单次io的块文件大小为4k  
 bsrange=512-2048 #同上，提定数据块的大小范围（单位：字节）  
 size=5g #本次的测试文件大小为5g，以每次4k的io进行测试（可以基于时间，也可以基于容量测试）  
 runtime=120 #测试时间为120秒，如果不定义时间，则一直将5g文件分4k每次写完为止。。  
 numjobs=4 #本次的测试线程为4  
 group_reporting #关于显示结果的，汇总每个（线程/进程）的信息

max-jobs=10 #最大允许的作业数线程数  
 rwmixwrite=30 #在混合读写的模式下，写占30%  
 bssplit=4k/30:8k/40:16k/30 #随机读4k文件占30%、8k占40%、16k占30%  
 rwmixread=70 #读占70%  
 name=ceshi #指定job的名字，在命令行中表示新启动一个job  
 invalidate=1 #开始io之前就失效buffer-cache（布尔型）  
 randrepeat=0 #设置产生的随机数是不可重复的  
 ioscheduler=psync #将设备文件切换为这里指定的IO调度器（看场合使用）

lockmem=1g #只使用1g内存进行测试。  
 zero_buffers #用0初始化系统buffer。  
 nrfiles=8 #每个进程生成文件的数量。

参考http://noops.me/?p=1098

http://www.10tiao.com/html/683/201708/2650716269/1.html


