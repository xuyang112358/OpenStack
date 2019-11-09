# ceph本地增量备份虚拟机

进入ceph_mon容器：<br>
docker exec -it ceph_mon bash<br>
创建一个用于存放增量文件的文件夹<br>
mkdir /test_diff<br>
## 一、做一次全量备份
（1）实例做一次快照<br>
rbd snap create vms/<instance_id>_disk@time1<br>

（2）导出差异数据，这个差异数据相当于全量备份<br>
rbd export-diff vms/<instance_id>_disk@time1 /test_diff/time1_diff_file<br>

## 二、再做一次增量备份<br>
（1）再对变化后的实例做一次快照<br>
rbd snap create vms/<instance_id>_disk@time2<br>

（2）导出time1到time2之间这段时间该磁盘的差异数据，相当于备份了增量<br>
rbd export-diff vms/<instance_id>_disk@time2 --from-snap time1 /test_diff/time1_time2<br>

## 三、备份恢复<br>
（1）如果该磁盘还存在，则直接用rbd snap rollback回滚就可以了，比如要回滚到time1这个时间点：<br>
rbd snap rollback vms/<instance_id>_disk@time1 <br>

<br>
（2）导入卷法：<br>
	<1>创建一个卷（大小跟根磁盘一样大小，这里以20G为例子）<br>
  openstack volume create --size 20 restore_disk
	注：删除块设备：rbd rm volumes/{image-name}<br>
	<2>导入差异数据，注意这里的导入顺序，先恢复到time1，再恢复到time2<br>
	rbd import-diff /test_diff/time1_diff_file volumes/volume_<instance_id><br>
	rbd import-diff /test_diff/time1_time2 volumes/volume_<instance_id><br>
	这时这块块设备就恢复回time2的状态了<br>
  <3>上传镜像<br>
  openstack image create --volume <volume_name> --container-format bare --disk-format qcow2 <image_name><br>
  <4>创建虚拟机<br>
（3）导入虚拟机法：<br>
	<1>创建一个虚拟机（跟原虚拟机同样配置）<br>
	<2>导入差异数据到新建虚拟机的根磁盘，注意这里的导入顺序，先恢复到time1，再恢复到time2<br>
	rbd import-diff /test_diff/time1_diff_file volumes/volume_<instance_id><br>
	rbd import-diff /test_diff/time1_time2 volumes/volume_<instance_id><br>
	这时再打开虚拟机就恢复完成了<br>
  
