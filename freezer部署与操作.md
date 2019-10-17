# freezer部署
### 进入容器
#docker exec -it freezer_api bash
### 配置环境变量
#vi /etc/freezer-openrc.sh

export OS_PROJECT_DOMAIN_NAME=default <br>
export OS_USER_DOMAIN_NAME=default <br>
export OS_PROJECT_NAME=admin <br>
export OS_TENANT_NAME=admin <br>
export OS_USERNAME=admin <br>
export OS_PASSWORD=hdu629 <br>
export OS_AUTH_URL=http://10.89.127.124:5000/v3 <br>
export OS_INTERFACE=internal <br>
export OS_IDENTITY_API_VERSION=3 <br>

### 加载变量
#source /etc/freezer-openrc.sh
### 启动freezer-scheduler服务
#freezer-scheduler start

# 组件介绍

### Freezer Web UI：
freezer web界面，与Freezer API进行交互，可以创建job等操作；
### Freezer API：
提供RESTful API，通过这些API维护备份相关的元数据，如client、job、action、backup和session等，<br>
元数据保存于Elasticsearch数据库中；与Freezer Scheduler交互并存储和提供相关的元数据。

* client：运行freezer agent/freezer scheduler的主机。

* action：freezer执行的一次备份、恢复或者删除等动作。

* backup：备份的相关信息。

* job：在client上执行的一个或者多个 actions，包含了scheduling信息。

* session：共享同样的scheduling时间的一组jobs，用于跨节点同步备份。

### Freezer Scheduler：
* 以守护进程的形式存在，运行在需要备份的节点上，调度Freezer-agent定期执行特定任务，并通过Freezer API client接口更新job状态及元数据信息。Freezer Scheduler每60s定时获取未完成状态的job信息，若有新的job产生，则启动该job并保存该job信息。遍历保存的job信息，若job是调度的，调度时间到则调用Freezer-agent执行任务。若job非调度的，则在当前时间延迟2s后调用Freezer-agent执行任务。任务执行结束或调度结束则删除本地
保存的该job信息。freezer-scheduler服务要手动启动：<br>
#freezer-scheduler start
* 服务启动后就会注册client。可以通过 freezer client-list查看。

### Freezer Agent：

负责执行 备份，恢复，删除等任务，可以单独执行也可以接受Freezer Scheduler的调度。支持的job如下：

### BackupJob：

数据备份，主要用于数据的备份任务，对应的action为backup。目前Freezer支持的数据备份如下：

* FS：基于文件的数据备份

* Mysql: 对Mysql数据库进行有效的备份

* MongoDB: 对MongoDB数据库进行有效的备份

* SQL Server: 对Microsoft sql server数据库进行有效的备份

* OpenStack Nova instance(nova): 对OpenStack 云主机进行备份，支持单机和租户内所有虚机的备份，备份时需要指定engine为nova。

* OpenStack Cinder volume(Cinder or Cindernative): 对OpenStack 云硬盘进行备份，目前Freezer 对云硬盘的备份有三种方式：

* 1） Freezer自主的一套对云硬盘的备份机制(Cinder)，需指定mode="cinder"，cindere_vol_id=volume_id，将卷创建快照，快照创建卷然后生成镜像，保存数据。

* 2） Cinder Backup 的方式进行备份(Cindernative)，需指定mode="cindernative"，cindernative_vol_id=volume_id，用cinder-backup服务进行备份，备份在swift里。

* 3） Cinderbrick方式：默认tar方式压缩，mode=’fs’，需要指定cinderbrick_vol_id=volume_id，将卷创建快照，快照创建卷然后挂载在本地目录，之后类似文件备份。

* 备份数据可以支持的存储包括： 对象存储Swift（默认值），本地挂载存储Local，备份数据到远程的SSH以及亚马逊的云存储s3。

* 备份数据默认的engine为tar，支持的engine为rsync，nova，osbrick。

* 默认的压缩方式为gzip,支持的压缩方式有 bzip2、xz。加密方式采用openssl的 AES-256-CFB方式。

参数限制：

* 1）mode == 'fs'：<br>

需指定path_to_backup，命令行对应--path-to-backup；<br>
no_incremental 与 max_level，always_level不兼容，即no_incremental为true，则后面两个参数不能指定有效值。

* 2）mode == 'nova'：

虚机备份不支持增量备份，即no_incremental必须为true；<br>
需指定nova_inst_id 或 project_id，前者是备份单个虚机，后者是备份租户内所有虚机；

* 3）mode == 'cinder'：

需指定cinder_vol_id；

* 4）mode == "cindernative"：

需指定cindernative_vol_id；

### RestoreJob

数据恢复，支持指定时间点的恢复，对应的action为restore。

* 参数限制：

* restore_abs_path，nova_inst_id，cinder_vol_id，cindernative_vol_id，cinderbrick_vol_id，project_id，nova_inst_id需要指定其中一个，不同的资源对应不同的参数；

* 需指定container或者使用默认值freezer_backups；

* no_incremental 与 max_level，always_level不兼容，即no_incremental为true，则后面两个参数不能指定有效值。

* AdminJob

* 管理job，对备份数据的管理，删除旧的备份任务，支持指定时间段的删除，对应的action为admin，--remove-from-date 时间要大于备份的时间。

* 需要指定参数remove_from_date 或 remove_older_than。

### ExecJob

* 执行命令或脚本，不能被调度。对应的action为exec，需要指定command参数。

### InfoJob

v获取存储介质中的容器名，大小，对象数等信息，对应的action为info。

# 简单使用

### 备份指定文件目录：
#freezer-agent --path-to-backup /opt/test/ --storage local --container /opt/backup --backup-name backup-name2
* 说明：
* 默认的action为backup，可以通过--action 指定，backup为备份，restore为恢复，admin为删除
### Mysql备份在本地
#freezer-agent --action backup --mode mysql --path-to-backup /var/lib/mysql/ --mysql-conf /etc/mysql/freezer.cnf --container /home/backuptest --storage local --backup-name mysqlbackup
* 说明：
* 需要提供一个conf文件，配置如下：否则会备份失败
>> [default]<br>
>> host = localhost<br>
>> user = root //访问数据库的用户<br>
>> password = stack //访问数据库的密码<br>
### Mysql备份在swift(存在问题)
#freezer-agent --action backup --mode mysql --path-to-backup /var/lib/mysql/ --mysql-conf /etc/mysql/freezer.cnf --container freezer_backup_test --storage swift --backup-name mysqlbackup
### 备份虚机
#freezer-agent --mode nova --no-incremental true --nova-inst-id ID --backup-name nova_backup --engine nova
* 说明：
* 默认的action为backup，可以通过--action 指定，backup为备份，restore为恢复，admin为删除
* 虚机备份需要指定engine为nova，否则默认为tar会报错：Critical Error: 'NoneType' object has no attribute 'strip'
* --no-incremental true指定非增量备份
* 没有指定容器，默认为freezer_backups，可以通过--container指定，会自动创建指定的容器

* 恢复备份<br>
#freezer-agent --action restore --mode nova --nova-inst-id ID --backup-name nova_backup --engine nova<br>
* 删除备份<br>
#freezer-agent --action admin --nova-inst-id ID --backup-name nova_backup --engine nova --remove-from-date 2019-10-19T00:29:27<br>
* 说明：
* --remove-from-date 删除该时间之前的备份
### 查看freezer用户
#freezer client-list
### 备份卷
#freezer-agent --mode cindernative --cindernative-vol-id 545f2b02-8ef5-4393-a3bb-6ed142acbcc5 --backup-name cinder_backup
* 说明：
* --cindernative-vol-id 卷id
* --mode cindernativ 表示用cinder-backup服务进行卷的备份，备份在swift里，默认容器freezer-backups
### 命令行创建调度任务
#freezer job-create --file /opt/json/test.json --client 46c449c804af497d9c22c86c5142f608_localhost.localdomain
>> 显示：Job f7d910b2aa324e428763f5130aa6788a created

json文件： <br>
{ <br>
	"job-action":[ <br>
      { <br>
        "freezer_action": { <br>
          "action": "backup", <br>
          "mode": "nova", <br>
          "nova_inst_id": "ba830e56-6ed7-4001-a455-ae3da165b3b2", <br>
          "backup-name": "nova_backup_by_json", <br>
          "container": "freezer_backups", <br>
          "engine_name": "nova", <br>
          "no_incremental": true <br>
        }, <br>
        "exit_status": "success", <br>
        "max_retries": 2, <br>
        "max_retries_interval": 60, <br>
        "mandatory": true <br>
      } <br>
    ], <br>
 <br>
    "job_schedule": { <br>
        "schedule_start_date": "", <br>
        "schedule_start_date": "", <br>
        "schedule_interval": "30 minutes" <br>
    }, <br>
    "description": "scheduled test" <br>
} <br>
* 说明：
* "schedule_start_date": "2017-08-24T01:00:00", 调度开始时间
* "schedule_end_date": "2017-08-24T01:55:00", 调度结束时间
* "schedule_interval": "30 minutes" 调度间隔时间

* 创建job的同时会创建action，通过 freezer action-list查看；备份结束会有创建backup，可以查看freezer backup-list。
* freezer命令行创建非调度任务
* 命令行：同上
* Json文件：job_schedule为空即可。
