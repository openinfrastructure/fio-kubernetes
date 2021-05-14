# fio k8s

Forked to have better defaults sourced from the [SRE
Handbook](https://s905060.gitbooks.io/site-reliability-engineer-handbook/content/fio.html).

# Usage

```
# Add the configmap
kubectl apply -f https://raw.githubusercontent.com/openinfrastructure/fio-kubernetes/master/configs.yaml
# Run the jobs for 60 seconds.
kubectl apply -f https://raw.githubusercontent.com/openinfrastructure/fio-kubernetes/master/fio.yaml
```

Collect the results using `kubectl get pods` and `kubectl logs`.

Clean up:

```
kubectl delete configmap/fio-job-config
kubectl delete job/fio
```

# Proxmox 6.4 VE

My use case is to test a base install of the following stack:

 1. 3x Dell R620 servers
   * Each: CPU(s) 32 x Intel(R) Xeon(R) CPU E5-2670 0 @ 2.60GHz (2 Sockets)
   * 256 MB Ram
   * 4x 556GB SAAS drives
 2. Proxmox 6.4 VE
 3. ZFS Raid 10
 4. Debian 10 VM
 5. Kubernetes 1.21.0
 6. 3x controller nodes
 7. 4x worker nodes

With a relatively un-tuned system the system is brought to a halt.  The kube
API starts failing with errors related to etcd storage.  Luckily, deleting the
deployment quickly resolves the situation.

```
Error from server: etcdserver: request timed out
```

```
Error from server: etcdserver: leader changed
```

```
Error from server (InternalError): an error on the server ("") has prevented the request from succeeding (get pods fio-c8xm8)
```

# First Run in k8s (job1)

This run, all 7 of the nodes are Proxmox guest VM's running on zvols in Proxmox
6.4.  I think this what's causing the etcd timeouts according to [this
thread](https://forum.proxmox.com/threads/high-disk-io-overhead-for-clients-on-zfs.80151/)

The virtio scsi driver is used in the guest.

```
[global]
ioengine=libaio
iodepth=128
rw=randrw
bs=8K
direct=1
size=1G
numjobs=12
ramp_time=5
runtime=60
time_based
group_reporting
[job1]
```

```
❯ k get pods -o wide
NAME        READY   STATUS      RESTARTS   AGE     IP              NODE      NOMINATED NODE   READINESS GATES
fio-c8xm8   0/1     Completed   0          5m59s   10.65.91.73     k1-9f87   <none>           <none>
fio-g4vzl   0/1     Completed   0          5m59s   10.65.97.10     k1-8843   <none>           <none>
fio-lchqb   0/1     Completed   0          5m59s   10.65.128.202   k1-9e1c   <none>           <none>
```

```
# k logs fio-c8xm8
job1: (groupid=0, jobs=12): err= 0: pid=10: Fri May 14 04:48:08 2021
  read: IOPS=3099, BW=24.3MiB/s (25.5MB/s)(1469MiB/60441msec)
    slat (usec): min=6, max=3540.7k, avg=1558.36, stdev=20618.60
    clat (msec): min=2, max=8167, avg=249.77, stdev=519.20
     lat (msec): min=2, max=8168, avg=251.36, stdev=521.86
    clat percentiles (msec):
     |  1.00th=[   11],  5.00th=[   16], 10.00th=[   22], 20.00th=[   37],
     | 30.00th=[   56], 40.00th=[   80], 50.00th=[  108], 60.00th=[  146],
     | 70.00th=[  201], 80.00th=[  296], 90.00th=[  550], 95.00th=[  877],
     | 99.00th=[ 2467], 99.50th=[ 3876], 99.90th=[ 6812], 99.95th=[ 7416],
     | 99.99th=[ 7819]
   bw (  KiB/s): min=  298, max=134826, per=100.00%, avg=26346.06, stdev=2160.49, samples=1350
   iops        : min=   37, max=16853, avg=3292.96, stdev=270.05, samples=1350
  write: IOPS=3102, BW=24.3MiB/s (25.5MB/s)(1471MiB/60441msec); 0 zone resets
    slat (usec): min=7, max=2726.3k, avg=1544.94, stdev=20722.35
    clat (msec): min=3, max=8168, avg=263.67, stdev=533.98
     lat (msec): min=3, max=8168, avg=265.27, stdev=536.77
    clat percentiles (msec):
     |  1.00th=[   13],  5.00th=[   20], 10.00th=[   26], 20.00th=[   43],
     | 30.00th=[   62], 40.00th=[   87], 50.00th=[  116], 60.00th=[  155],
     | 70.00th=[  211], 80.00th=[  309], 90.00th=[  575], 95.00th=[  927],
     | 99.00th=[ 2500], 99.50th=[ 3876], 99.90th=[ 7215], 99.95th=[ 7416],
     | 99.99th=[ 7819]
   bw (  KiB/s): min=  259, max=138774, per=100.00%, avg=26411.35, stdev=2180.11, samples=1349
   iops        : min=   32, max=17346, avg=3301.10, stdev=272.51, samples=1349
  lat (msec)   : 4=0.01%, 10=0.73%, 20=6.52%, 50=18.50%, 100=20.62%
  lat (msec)   : 250=29.37%, 500=13.36%, 750=4.65%, 1000=2.33%, 2000=2.83%
  lat (msec)   : >=2000=1.50%
  cpu          : usr=0.54%, sys=0.93%, ctx=220001, majf=0, minf=699
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwts: total=187319,187496,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=24.3MiB/s (25.5MB/s), 24.3MiB/s-24.3MiB/s (25.5MB/s-25.5MB/s), io=1469MiB (1541MB), run=60441-60441msec
  WRITE: bw=24.3MiB/s (25.5MB/s), 24.3MiB/s-24.3MiB/s (25.5MB/s-25.5MB/s), io=1471MiB (1542MB), run=60441-60441msec

Disk stats (read/write):
  sda: ios=188636/188853, merge=6/176, ticks=27586579/28446782, in_queue=57486792, util=100.00%
```

```
# k logs fio-g4vzl
job1: (groupid=0, jobs=12): err= 0: pid=9: Fri May 14 04:46:14 2021
  read: IOPS=11.3k, BW=87.0MiB/s (92.3MB/s)(5284MiB/60051msec)
    slat (usec): min=5, max=251618, avg=423.78, stdev=2730.90
    clat (usec): min=252, max=1086.7k, avg=65384.45, stdev=74248.45
     lat (usec): min=312, max=1101.9k, avg=65812.03, stdev=74737.90
    clat percentiles (msec):
     |  1.00th=[    6],  5.00th=[    8], 10.00th=[    9], 20.00th=[   12],
     | 30.00th=[   18], 40.00th=[   27], 50.00th=[   40], 60.00th=[   56],
     | 70.00th=[   77], 80.00th=[  107], 90.00th=[  157], 95.00th=[  209],
     | 99.00th=[  347], 99.50th=[  418], 99.90th=[  600], 99.95th=[  684],
     | 99.99th=[  793]
   bw (  KiB/s): min= 9408, max=583781, per=98.87%, avg=89094.02, stdev=7590.92, samples=1428
   iops        : min= 1176, max=72965, avg=11135.63, stdev=948.77, samples=1428
  write: IOPS=11.3k, BW=88.0MiB/s (92.3MB/s)(5285MiB/60051msec); 0 zone resets
    slat (usec): min=6, max=194375, avg=428.54, stdev=2746.07
    clat (usec): min=1137, max=1086.7k, avg=70104.71, stdev=76850.28
     lat (usec): min=1159, max=1086.7k, avg=70537.12, stdev=77319.37
    clat percentiles (msec):
     |  1.00th=[    7],  5.00th=[    9], 10.00th=[   11], 20.00th=[   13],
     | 30.00th=[   21], 40.00th=[   31], 50.00th=[   44], 60.00th=[   61],
     | 70.00th=[   84], 80.00th=[  114], 90.00th=[  165], 95.00th=[  218],
     | 99.00th=[  359], 99.50th=[  430], 99.90th=[  609], 99.95th=[  693],
     | 99.99th=[  802]
   bw (  KiB/s): min=10160, max=578179, per=98.89%, avg=89121.73, stdev=7576.27, samples=1428
   iops        : min= 1270, max=72267, avg=11139.18, stdev=946.95, samples=1428
  lat (usec)   : 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.03%, 4=0.29%, 10=12.21%, 20=18.56%, 50=24.59%
  lat (msec)   : 100=21.52%, 250=19.79%, 500=2.86%, 750=0.22%, 1000=0.02%
  lat (msec)   : 2000=0.01%
  cpu          : usr=1.54%, sys=4.13%, ctx=1292901, majf=0, minf=711
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwts: total=675673,675692,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=87.0MiB/s (92.3MB/s), 87.0MiB/s-87.0MiB/s (92.3MB/s-92.3MB/s), io=5284MiB (5541MB), run=60051-60051msec
  WRITE: bw=88.0MiB/s (92.3MB/s), 88.0MiB/s-88.0MiB/s (92.3MB/s-92.3MB/s), io=5285MiB (5542MB), run=60051-60051msec

Disk stats (read/write):
  sda: ios=719725/720400, merge=233/373, ticks=26710138/28593412, in_queue=62634044, util=100.00%
```

```
# logs fio-lchqb
job1: (groupid=0, jobs=12): err= 0: pid=9: Fri May 14 04:48:26 2021
  read: IOPS=5104, BW=39.0MiB/s (41.9MB/s)(2402MiB/60079msec)
    slat (usec): min=6, max=802100, avg=966.58, stdev=8960.66
    clat (msec): min=3, max=2816, avg=145.48, stdev=208.61
     lat (msec): min=3, max=2816, avg=146.45, stdev=209.87
    clat percentiles (msec):
     |  1.00th=[    7],  5.00th=[   13], 10.00th=[   19], 20.00th=[   34],
     | 30.00th=[   51], 40.00th=[   69], 50.00th=[   88], 60.00th=[  110],
     | 70.00th=[  140], 80.00th=[  190], 90.00th=[  309], 95.00th=[  468],
     | 99.00th=[ 1116], 99.50th=[ 1485], 99.90th=[ 2123], 99.95th=[ 2232],
     | 99.99th=[ 2601]
   bw (  KiB/s): min=  591, max=141856, per=100.00%, avg=41109.57, stdev=2567.72, samples=1417
   iops        : min=   73, max=17732, avg=5138.41, stdev=320.96, samples=1417
  write: IOPS=5108, BW=40.0MiB/s (41.0MB/s)(2404MiB/60079msec); 0 zone resets
    slat (usec): min=7, max=762950, avg=983.32, stdev=8862.25
    clat (msec): min=3, max=2949, avg=154.51, stdev=213.48
     lat (msec): min=3, max=3083, avg=155.51, stdev=214.72
    clat percentiles (msec):
     |  1.00th=[   11],  5.00th=[   17], 10.00th=[   23], 20.00th=[   41],
     | 30.00th=[   57], 40.00th=[   77], 50.00th=[   95], 60.00th=[  117],
     | 70.00th=[  148], 80.00th=[  201], 90.00th=[  326], 95.00th=[  489],
     | 99.00th=[ 1150], 99.50th=[ 1502], 99.90th=[ 2123], 99.95th=[ 2265],
     | 99.99th=[ 2702]
   bw (  KiB/s): min=  527, max=141696, per=100.00%, avg=41117.14, stdev=2550.11, samples=1418
   iops        : min=   65, max=17712, avg=5139.34, stdev=318.76, samples=1418
  lat (msec)   : 4=0.01%, 10=2.18%, 20=7.21%, 50=18.45%, 100=26.55%
  lat (msec)   : 250=31.77%, 500=9.41%, 750=2.35%, 1000=1.03%, 2000=1.13%
  lat (msec)   : >=2000=0.15%
  cpu          : usr=0.84%, sys=1.54%, ctx=359679, majf=0, minf=699
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwts: total=306670,306939,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=39.0MiB/s (41.9MB/s), 39.0MiB/s-39.0MiB/s (41.9MB/s-41.9MB/s), io=2402MiB (2518MB), run=60079-60079msec
  WRITE: bw=40.0MiB/s (41.0MB/s), 40.0MiB/s-40.0MiB/s (41.0MB/s-41.0MB/s), io=2404MiB (2521MB), run=60079-60079msec

Disk stats (read/write):
  sda: ios=313580/314089, merge=19/178, ticks=25889231/27114911, in_queue=55047900, util=100.00%
```

Proxmox host job1
===

The same job1, but executed directly on the host using a ZFS dataset:

```
root@r620a:~# zpool status
  pool: rpool
 state: ONLINE
config:

	NAME                                              STATE     READ WRITE CKSUM
	rpool                                             ONLINE       0     0     0
	  mirror-0                                        ONLINE       0     0     0
	    scsi-36c81f660cc719400283053fb1893524f-part3  ONLINE       0     0     0
	    scsi-36c81f660cc7194002830540d19a51e68-part3  ONLINE       0     0     0
	  mirror-1                                        ONLINE       0     0     0
	    scsi-36c81f660cc719400283054211ae0d9c7-part3  ONLINE       0     0     0
	    scsi-36c81f660cc719400283054351c0eaefd-part3  ONLINE       0     0     0
```

```
testjob: (groupid=0, jobs=12): err= 0: pid=29763: Thu May 13 22:02:17 2021
  read: IOPS=2804, BW=22.0MiB/s (23.1MB/s)(1321MiB/60004msec)
    slat (nsec): min=0, max=3466.4k, avg=39298.56, stdev=24129.56
    clat (nsec): min=0, max=480676k, avg=270941882.56, stdev=38148652.68
     lat (nsec): min=0, max=480720k, avg=270981502.50, stdev=38150822.03
    clat percentiles (msec):
     |  1.00th=[  199],  5.00th=[  220], 10.00th=[  230], 20.00th=[  243],
     | 30.00th=[  251], 40.00th=[  259], 50.00th=[  268], 60.00th=[  275],
     | 70.00th=[  288], 80.00th=[  296], 90.00th=[  313], 95.00th=[  334],
     | 99.00th=[  397], 99.50th=[  414], 99.90th=[  443], 99.95th=[  456],
     | 99.99th=[  468]
   bw (  KiB/s): min=    0, max= 2656, per=8.30%, avg=1869.73, stdev=268.23, samples=1438
   iops        : min=    0, max=  332, avg=233.69, stdev=33.54, samples=1438
  write: IOPS=2808, BW=22.0MiB/s (23.1MB/s)(1323MiB/60004msec); 0 zone resets
    slat (nsec): min=0, max=45675k, avg=4224369.44, stdev=568060.37
    clat (nsec): min=0, max=480158k, avg=271049781.62, stdev=38197160.42
     lat (nsec): min=0, max=486997k, avg=275278475.52, stdev=38506306.73
    clat percentiles (msec):
     |  1.00th=[  199],  5.00th=[  220], 10.00th=[  230], 20.00th=[  243],
     | 30.00th=[  251], 40.00th=[  259], 50.00th=[  268], 60.00th=[  275],
     | 70.00th=[  288], 80.00th=[  296], 90.00th=[  313], 95.00th=[  334],
     | 99.00th=[  397], 99.50th=[  414], 99.90th=[  443], 99.95th=[  451],
     | 99.99th=[  464]
   bw (  KiB/s): min=    0, max= 2448, per=8.29%, avg=1872.18, stdev=219.02, samples=1438
   iops        : min=    0, max=  306, avg=234.00, stdev=27.39, samples=1438
  lat (usec)   : 10=0.01%, 20=0.01%
  lat (msec)   : 4=0.01%, 10=0.01%, 20=0.01%, 50=0.06%, 100=0.09%
  lat (msec)   : 250=28.81%, 500=71.46%
  cpu          : usr=0.27%, sys=2.73%, ctx=169053, majf=1, minf=183
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwts: total=168309,168539,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=22.0MiB/s (23.1MB/s), 22.0MiB/s-22.0MiB/s (23.1MB/s-23.1MB/s), io=1321MiB (1385MB), run=60004-60004msec
  WRITE: bw=22.0MiB/s (23.1MB/s), 22.0MiB/s-22.0MiB/s (23.1MB/s-23.1MB/s), io=1323MiB (1387MB), run=60004-60004msec
```

# Notes

Running the same job1 again, `zpool iostat 3` reports good write throughput:

```
              capacity     operations     bandwidth
pool        alloc   free   read  write   read  write
----------  -----  -----  -----  -----  -----  -----
rpool       49.0G  1.04T      0  15.9K      0   617M
rpool       49.0G  1.04T      0  3.42K      0   639M
rpool       49.0G  1.04T      0  17.5K      0   632M
rpool       49.0G  1.04T      0  2.32K      0   607M
rpool       49.0G  1.04T      0  16.6K      0   620M
rpool       49.0G  1.04T      0  9.26K      0   620M
rpool       53.1G  1.03T      0  9.65K      0   612M
rpool       53.1G  1.03T      0  9.19K      0   613M
rpool       53.1G  1.03T      0  8.67K      0   612M
rpool       53.1G  1.03T      0  8.83K      0   601M
rpool       53.1G  1.03T      0  10.1K      0   620M
rpool       53.1G  1.03T      0  18.1K      0   562M
rpool       57.1G  1.03T      0  14.2K      0   462M
rpool       57.1G  1.03T      0  17.2K      0   559M
rpool       57.1G  1.03T      0  4.79K      0   370M
rpool       57.1G  1.03T      0  2.32K      0   224M
rpool       57.1G  1.03T      0  2.25K      0   218M
rpool       57.1G  1.03T      0  5.92K      0   367M
rpool       57.1G  1.03T      0  12.5K      0   426M
rpool       57.1G  1.03T      0  11.9K      0   423M
rpool       57.1G  1.03T      0  1.44K      0   172M
rpool       58.7G  1.03T      0  1.72K      0   172M
rpool       58.7G  1.03T      0  2.58K      0   272M
rpool       58.7G  1.03T      0  2.63K      0   276M
rpool       58.7G  1.03T      0  2.98K      0   306M
rpool       58.7G  1.03T      0  5.51K      0   295M
rpool       58.7G  1.03T      0  9.81K      0   445M
rpool       58.7G  1.03T      0  2.57K      0   273M
rpool       58.7G  1.03T      0  3.13K      0   298M
rpool       58.7G  1.03T      0  4.10K      0   298M
rpool       58.7G  1.03T      0  2.34K      0   228M
rpool       58.7G  1.03T      0  5.12K      0   317M
rpool       58.7G  1.03T      0  3.29K      0   302M
rpool       58.7G  1.03T      0  7.20K      0   319M
rpool       60.2G  1.03T      0  10.1K      0   514M
rpool       60.2G  1.03T      0  4.14K      0   595M
rpool       58.3G  1.03T      0  3.52K      0   449M
rpool       58.3G  1.03T      0     20      0   370K
rpool       53.1G  1.03T      0    127      0  3.46M
rpool       53.1G  1.03T      0     19      0   285K
```

# Perc H310 replacement

Comparison of the random read/write job1 between two configurations on the same node:

 1. Dell stock firmware for Perc H310.
 2. LSI initiator target (IT) mode firmware.

The hypothesis is the 512MB of cache on the dell firmware is causing a large
IOPS backlog, which is too much latency for the k8s controller nodes.  etcd
leader writes specifically.

## Dell Firmware

Run on r620c only with the dell firmware:

```
testjob: (groupid=0, jobs=12): err= 0: pid=10: Fri May 14 16:43:23 2021
  read: IOPS=10.1k, BW=79.1MiB/s (82.0MB/s)(4751MiB/60053msec)
    slat (usec): min=5, max=382179, avg=463.84, stdev=3584.34
    clat (usec): min=735, max=1863.9k, avg=72486.63, stdev=97045.23
     lat (usec): min=780, max=1863.9k, avg=72953.53, stdev=97707.59
    clat percentiles (msec):
     |  1.00th=[    5],  5.00th=[    7], 10.00th=[    9], 20.00th=[   14],
     | 30.00th=[   21], 40.00th=[   33], 50.00th=[   45], 60.00th=[   59],
     | 70.00th=[   78], 80.00th=[  105], 90.00th=[  159], 95.00th=[  232],
     | 99.00th=[  493], 99.50th=[  625], 99.90th=[ 1003], 99.95th=[ 1116],
     | 99.99th=[ 1351]
   bw (  KiB/s): min= 2830, max=504550, per=98.84%, avg=80065.34, stdev=6278.23, samples=1428
   iops        : min=  352, max=63063, avg=10007.24, stdev=784.71, samples=1428
  write: IOPS=10.1k, BW=79.1MiB/s (82.9MB/s)(4750MiB/60053msec); 0 zone resets
    slat (usec): min=6, max=349939, avg=470.04, stdev=3621.64
    clat (usec): min=1291, max=1864.0k, avg=78277.90, stdev=101175.43
     lat (usec): min=1314, max=1910.4k, avg=78751.05, stdev=101814.17
    clat percentiles (msec):
     |  1.00th=[    7],  5.00th=[    9], 10.00th=[   11], 20.00th=[   17],
     | 30.00th=[   25], 40.00th=[   37], 50.00th=[   50], 60.00th=[   64],
     | 70.00th=[   84], 80.00th=[  112], 90.00th=[  169], 95.00th=[  247],
     | 99.00th=[  510], 99.50th=[  651], 99.90th=[ 1036], 99.95th=[ 1183],
     | 99.99th=[ 1418]
   bw (  KiB/s): min= 2831, max=508061, per=98.83%, avg=80046.54, stdev=6311.22, samples=1428
   iops        : min=  353, max=63503, avg=10004.93, stdev=788.84, samples=1428
  lat (usec)   : 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.39%, 10=11.56%, 20=15.01%, 50=25.51%
  lat (msec)   : 100=25.07%, 250=17.97%, 500=3.59%, 750=0.70%, 1000=0.20%
  lat (msec)   : 2000=0.11%
  cpu          : usr=1.40%, sys=3.56%, ctx=1090869, majf=0, minf=713
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued rwts: total=607356,607215,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=128

Run status group 0 (all jobs):
   READ: bw=79.1MiB/s (82.0MB/s), 79.1MiB/s-79.1MiB/s (82.0MB/s-82.0MB/s), io=4751MiB (4982MB), run=60053-60053msec
  WRITE: bw=79.1MiB/s (82.9MB/s), 79.1MiB/s-79.1MiB/s (82.9MB/s-82.9MB/s), io=4750MiB (4981MB), run=60053-60053msec

Disk stats (read/write):
  sda: ios=660161/660256, merge=172/341, ticks=28084610/30045312, in_queue=61318072, util=100.00%
```

# Reference

Forked from [this post](https://medium.com/@joshua_robinson/storage-benchmarking-with-fio-in-kubernetes-14cf29dc5375)
