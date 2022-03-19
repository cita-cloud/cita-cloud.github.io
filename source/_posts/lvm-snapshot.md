---
title: LVM 快照与恢复（备份与恢复的快速方法）
date: 2022-03-19 21:34:24
categories: 云计算
---

问题描述
----

在进行某些验证性操作时，我们需要创建测试数据，并在验证操作的过程中修改数据。但是如果验证操作失败，那么我们又需要重新创建测试数据。为了避免重新创建数据，我们常见的做法是备份测试数据，以在验证失败时能够从备份数据中快速进行恢复。

还有种场景是服务升级的时候：为了能够在升级失败时回滚，需要对服务数据进行备份，否则数据被破坏之后，服务回滚后也无法运行。但是由于服务数据较多，导致备份周期长，服务停机时间长。而且在升级过程中，并非所有的数据都需要备份，因为并非所有的数据都会被破坏。

该笔记将记录：在 LVM 中，使用 Snapshot 快照的方法（对数据进行快速的备份与恢复），以及常见问题的解决办法。

解决方案
----

当创建快照后，如果不小心删除任何文件，也不必担心，因为快照具有我们已删除的原始文件。

注意事项：
1）快照不能用于持久的备份策略 —— 备份是某些数据文件的主副本，而快照是块级别，所以不能使用快照作为备份选项；
2）不要更改快照卷，保持原样，而快照用于快速恢复。

### 环境概述
```
pvcreate /dev/sdb                                                               # 10G
vgcreate vgdt /dev/sdb
lvcreate -n source --size 3G vgdt

mkfs.ext4 /dev/vgdt/source
mount /dev/vgdt/source /mnt/
echo 123456 > /mnt/foo.txt
md5sum /mnt/foo.txt                                                             # f447b20a7fcbf53a5d5be013ea0b15af
```

### 第一步、创建快照
```
# lvcreate --size  1G --snapshot --name backup4source /dev/vgdt/source
  Logical volume "backup4source" created.

# lvextend --size +1G /dev/vgdt/backup4source                                   # 再额外增加 1G 空间
  Size of logical volume vgdt/backup4source changed from 1.00 GiB (256 extents) to 2.00 GiB (512 extents).
  Logical volume vgdt/backup4source successfully resized.

# lvs /dev/vgdt/backup4source                                                   # 查看快照信息
  LV            VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  backup4source vgdt swi-a-s--- 2.00g      source 0.01 

# lvdisplay /dev/vgdt/backup4source 
  --- Logical volume ---
  LV Path                /dev/vgdt/backup4source
  ...
  LV snapshot status     active destination for source                          # 该快照所属的 LV
  ...
```

### 第二步、数据修改
```
# lvs
  LV            VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv     ubuntu-vg -wi-ao---- <9.00g                                                    
  backup4source vgdt      swi-a-s---  2.00g      source 0.01                                   
  source        vgdt      owi-aos---  3.00g

# dd if=/dev/zero of=/mnt/foo.txt bs=1M count=1024 conv=fdatasync 
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 9.26228 s, 116 MB/s

# lvs
  LV            VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv     ubuntu-vg -wi-ao---- <9.00g                                                    
  backup4source vgdt      swi-a-s---  2.00g      source 50.21                   # 原始数据被写入快照分区，以占用 50.21%
  source        vgdt      owi-aos---  3.00g 
```

关于 Snapshot 大小：
1）如果数据变更的总量超过 Snapshot 大小，则会产生 Input/output error 错误，进而导致 Snapshot 不可用（解释扩容也无法恢复）；
2）如果要避免该问题，可以创建相同大小的 Snapshot，或者自动扩容 Snapshot 分区（这里不再展开详细说明）；

### 第三步、恢复快照
```
# umount /mnt

# lvconvert --merge /dev/vgdt/backup4source
  Merging of volume vgdt/backup4source started.
  vgdt/source: Merged: 50.29%
...
  vgdt/source: Merged: 100.00%

# mount /dev/vgdt/source /mnt/

# md5sum /mnt/foo.txt
f447b20a7fcbf53a5d5be013ea0b15af  /mnt/foo.txt
```

补充说明：
1）当 merge 后，Snapshot 会被自动删除；

常用操作
----

### 删除快照

如果没有必要保留快照，则可以删除：
```
# lvremove /dev/vgdt/backup4source
```

### 自动扩容
该特性是为了让 Snapshot 自动扩容，而不需要分配足够的空间，且当空间不足时不需要人工介入：
```
# vim /etc/lvm/lvm.conf
...
snapshot_autoextend_threshold = 70                                              # 当用量超过 70% 时，
snapshot_autoextend_percent = 20                                                # 自动扩容 20%
...
```

参考文献
----

[How to Take 'Snapshot of Logical Volume and Restore' in LVM - Part III](https://www.tecmint.com/take-snapshot-of-logical-volume-and-restore-in-lvm/%20)
[How-to guide: LVM snapshot - Kernel Talks](https://kerneltalks.com/disk-management/how-to-guide-lvm-snapshot/%20)

