---
title:  "Mysql Backup XtraBackup"
---

{% include toc icon="cog" title="Mysql Backup XtraBackup" %}


## 备份原理

[参考blog](https://blog.csdn.net/leshami/article/details/41043269)

InnoDB引擎很大程度上与Oracle类似，使用redo，undo机制，因此在热备期间需要考虑对于日志缓冲区在线事物日志及时写出到文件的问题。如果log buffer没有及时写出将被日志的循环写特性覆盖。xtrabackup在启动时会记住log sequence number（LSN），然后一页一页地复制InnoDB的数据。与此同时，监控log buffer中的日志情况，一旦log buffer发生变化，即数据发生了不一致，该过程会立即被捕获并把变化的页面复制到xtrabckup log，直到全部innoDB数据文件复制完成之后，停止监控log buffer及日志复制。
​    xtrabackup在恢复期间对提交的事务前滚，未提交或失败的事务进行回滚，从而保证数据的一致性。因此对于InnoDB表在备份期间不会锁表。由于XtraBackup其内置的InnoDB库打开文件的时候是rw的，所以运行XtraBackup的用户，必须对InnoDB的数据文件具有读写权限。

   ![img](./images/xtrabackup.jpg)

## 工具安装

```bash
Wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.9/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.9-1.el7.x86_64.rpm

yum localinstall percona-xtrabackup-24-2.4.9-1.el7.x86_64.rpm
```

`qpress`解压工具，在备份是使用`--compress`选项时候进行压缩处理；

恢复时需要先使用`qpress`进行解压缩才能恢复。

```bash
wget http://www.quicklz.com/qpress-11-linux-x64.tar
tar -xvf qpress-11-linux-x64.tar
mv qpress /usr/bin/
```



## 备份

备份时可以对数据库制定**最小权限用户**，备份用户密码使用明文存放在脚本中

```sql
CREATE USER bakuser;
GRANT RELOAD, LOCK TABLES,REPLICATION CLIENT,CREATE TABLESPACE,PROCESS,SUPER ON *.* TO 'bakuser'@'localhost' IDENTIFIED BY 'bakpass';
FLUSH PRIVILEGES;
```



### 完全备份

#### 备份脚本示例

**脚本需要修改对应信息**

```bash
#!/bin/bash
#explain:xtrabackup full backup of mysql,author:ZZC,date:20180918
USER='bakuser'
PASSWORD='xxxxxxxx'
MY_CONFIG='/etc/my.cnf'
BACKUP_BASE_DIR='/home/DB_Backup/xtrabackup'  
LOG_DIRE="$BACKUP_BASE_DIR/Logs"

if [[ ! -d $LOG_DIRE ]]
then
	mkdir -p $LOG_DIRE
        touch $LOG_DIRE/backuplog.log
else
	echo "`LogDate`：$LOG_DIRE Directory already exists!!"|tee -a $LOG_DIRE/backuplog.log
fi

LogDate(){
	# 日志函数
	echo `date +"%Y-%m-%d %H:%M:%S"`
}
echo "`LogDate`：Start to backup" |tee -a $LOG_DIRE/backuplog.log
  
innobackupex --defaults-file=$MY_CONFIG --compress --user=$USER --password=$PASSWORD --no-timestamp $BACKUP_BASE_DIR/$(date +%Y%m%d%H%M) >> $LOG_DIRE/xtrabackup-$(date +%Y%m%d%H%M).log 2>&1
if [ $? -eq 0 ];then  
    echo "`LogDate`：Backup is finish! at $(date +%Y%m%d%H%M)" |tee -a $LOG_DIRE/backuplog.log
    echo "`LogDate`：$hostname mysql full backup Success! at $(date +"%y-%m-%d %H:%M:%S")" |tee -a
 $LOG_DIRE/backuplog.log    exit 0  
else  
    echo "`LogDate`：Backup is Fail! at $(date +%Y%m%d%H%M)" |tee -a $LOG_DIRE/backuplog.log
    echo "`LogDate`：$hostname mysql full backup Fail! at $(date +"%y-%m-%d %H:%M:%S")"  |tee -a $
LOG_DIRE/backuplog.log    exit 1
fi  
echo "`LogDate`：Backup Process Done" |tee -a $LOG_DIRE/backuplog.log
```

#### 备份命令

**--no-timestamp：**选项来阻止命令自动创建一个以时间命名的目录

**--compress：**压缩备份

```bash
innobackupex --user=bakuser --password=bakpass --port=3306 --socket=/data/mysql/mysql.sock /data/backup/xtrabackup/full_bak --no-timestamp
```



### 增量备份

每个InnoDB的页面都会包含一个LSN信息，每当相关的数据发生改变，相关的页面的LSN就会自动增长。这正是InnoDB表可以进行增量备份的基础，即innobackupex通过备份上次完全备份之后发生改变的页面来实现

**/data/backup/xtrabackup/full_bak：**指的是完全备份所在的目录

**--incremental-basedir：**指向上一次增量/全备所在的目录

`增量备份仅能应用于InnoDB或XtraDB表，对于MyISAM表而言，执行增量备份时其实进行的是完全备份`

```bash
innobackupex --user=bakuser --password=bakpass --port=3306 --incremental /data/backup/xtrabackup/inc_bak --incremental-basedir=/data/backup/xtrabackup/full_bak --no-timestamp
```



### 远程备份

```shell
# SSH 公钥互信提前配置好
innobackupex --user=bakuser --password=bakpass --stream=tar ./ | gzip | ssh -p 60028 root@172.16.6.x  "cat - > /data/backup/xtrabackup/backup.ta
r.gz"
```

**--stream：**以特殊的 tar 或 xbstream 格式将备份发送到STDOUT，而不是将文件复制到备份目录；

详见 man 手册

**To extract Percona XtraBackup's archive you must use tar with` -i` option:**

    tar -xizf backup.tar.gz
### 日志备份

在数据库中将`binglog`保存到指定位置中；

执行不完全恢复需要使用到`可用备份+binglog`



## 恢复

### 恢复前提

* 整理完全备份

  ```bash
  innobackupex --apply-log --redo-only --user=bakuser --password=bakpass --port=3306 /data/backup/xtrabackup/full_bak
  ```


* 准备增量备份


  * `准备`(prepare)增量备份与整理完全备份有着一些不同，尤其要注意的是：
    1. 需要在每个备份(包括完全和各个增量备份)上，将已经提交的事务进行“重放”。“重放”之后，所有的备份数据将合并到完全备份上。
    2. 基于所有的备份将未提交的事务进行“回滚”。

  此时没有--redo-only，如果有多个增备，`仅最后一个增备无需指定--redo-only `

  ```bash
  innobackupex --apply-log --user=bakuser --password=bakpass --port=3306 /data/backup/xtrabackup/full_bak --incremental-dir=/data/backup/xtrabackup/inc_bak
  ```

  **/data/backup/xtrabackup/full_bak：**全备目录

  **--incremental-dir：**增备目录（如果有多个，需要运行多次）

  **--apply-log：**一般情况下,在备份完成后，数据尚且不能用于恢复操作，因为`备份的数据中可能会包含尚未提交的事务或已经提交但尚未同步至数据文件中的事务`。此时数据 文件仍处理不一致状态。**--apply-log**的作用是通过回滚未提交的事务及同步已经提交的事务至数据文件使数据文件处于一致性状态

* 备份解压缩

  * **方式一**

  压缩备份后的文件在恢复之前需要先解压缩，使用前面安装的 qoress 命令；

  `qpress -d xxxx.qp /Backup_dir/data_dir/`

  实例数量多的情况下可以使用 for 循环进行解压缩；

  ```shell
  for x in `find $BACKUP_DIR -name "*.qp"`; do qpress -d $x $(dirname $x) && rm -rf $x ; done
  ```

  * **方式二**

  `innobackupex`命令恢复并解压缩；

  **--decompress：**在以前使用-compress选项进行的备份中，使用.qp扩展名解压缩所有文件。innobackupex—parallel选项将支持多个文件，同时解密和/或解压缩。为了解压，必须在路径中安装和访问qpress实用程序。

  ```shell
  # 恢复命令
  innobackupex --defaults-file=/etc/my.cnf --decompress --user=bakuser --password=bakpass --port=3306 --copy-back /data/backup/xtrabackup/full_bak
  # 为了清理备份目录用户应该手动删除*.qp文件
  find /data/backup -name "*.qp" | xargs rm
  ```


### 完全恢复

```bash
# 停止mysql实例
systemctl stop mysqld

# 已经将增量备份与全备进行整理并且执行过 --redo-only、--apply-log
innobackupex --defaults-file=/etc/my.cnf --user=bakuser --password=bakpass --port=3306 --copy-back /data/backup/xtrabackup/full_bak

# 将数据目录权限变更为mysql
chown -R mysql.mysql /data/mysql

# 启动mysql实例
systemct start mysql
```

**--copy-back：**用于执行服务器datadir的备份恢复

**--redo-only：**回滚合并**最后一个增量备份不需要指定**



### 不完全恢复

不完全恢复指的是数据库故障时，备份时间点到故障时间点中间的数据恢复；

主要使用binglog进行不完全恢复；

思路：

* 在增量备份完成后：

```sql
insert into test01.mytab01 values(4,'pointrecover');
Query OK, 1 row affected (0.04 sec)

system date;
Fri Sep 21 11:31:43 CST 2018

truncate table test01.mytab01;
Query OK, 0 rows affected (0.09 sec)
```

* 使用全备+增量备份，恢复到备份时间点；

```bash
# 执行恢复全备
innobackupex --defaults-file=/etc/my.cnf --user=bakuser --password=bakpass --port=3306 --copy-back /data/backup/xtrabackup/full_bak

# 修改文件夹权限
chown mysql.mysql -R /data/mysql

# 启动mysql实例
systemctl start mysqld

# 查看恢复数据
mysql> select * from test01.mytab01;
+------+--------+
| id   | name   |
+------+--------+
|    1 | zzzz   |
|    2 | zzzz   |
|    3 | incbak |
+------+--------+
3 rows in set (0.01 sec)

# 获取增量之后的log position
cd /data/backup/xtrabackup/inc_bak/
more xtrabackup_binlog_info
mysql-bin.000001	3638

# 在恢复之前已经记录下故障时间点（如果不知道，可以查看日志或者逐条binlog进行排查）
# binlog 保存在原来数据目录下，测试恢复时可将原来的数据目录备份或者重命名一份
# --start-datetime：从二进制日志中读取指定等于时间戳或者晚于本地计算机的时间
# --stop-datetime：从二进制日志中读取指定小于时间戳或者等于本地计算机的时间 取值和上述一样
# --start-position：从二进制日志中读取指定position 事件位置作为开始。
# --stop-position：从二进制日志中读取指定position 事件位置作为事件截至
mysqlbinlog --no-defaults /data/mysql.bak/mysql-bin.000001 --start-position=3638 --stop-datetime="2018-09-21 11:31:43" |mysql -uroot -p
Enter password: 

# 查看应用日志后恢复的数据
mysql> select * from test01.mytab01;
+------+--------------+
| id   | name         |
+------+--------------+
|    1 | zzzz         |
|    2 | zzzz         |
|    3 | incbak       |
|    4 | pointrecover |
+------+--------------+
4 rows in set (0.00 sec)
```

* 如果需要继续恢复后面的事务，可以找到 `truncate`之后的 position，然后跳过这个 position

```bash
mysqlbinlog /data/mysql.bak/mysql-bin.000001 --start-datetime="2014-12-25 11:53:54"|grep truncate -A5
​```
truncate table tb
/*!*/;
# at 1180
#141225 11:55:35 server id 3606  end_log_pos 1260 CRC32 0x12f55fc5   Query  thread_id=928   exec_time=0   error_code=0
SET TIMESTAMP=1419479735/*!*/;
/*!\C latin1 *//*!*/;
--
create table tb_after_truncate(id int,val varchar(20))
/*!*/;
# at 1392
#141225 13:06:47 server id 3606  end_log_pos 1415 CRC32 0xf956f311      Stop
DELIMITER ;
# End of log file
​```
--我们找出的position为1260，跳过1260之前的继续追加binlog
SHELL> mysqlbinlog /data/mysql.bak/mysql-bin.000001 --start-position=1260|mysql -uroot -p
```



## 单库备份恢复

```bash
# 备份表结构
mysqldump -u root -p -d test01 > test01.sql

# 备份数据库
innobackupex --user=root --password=Pwd@123456 --databases=test01 /data/backup/test01 --no-timestamp

# 模拟数据库故障
mysql> drop database test01;
Query OK, 2 rows affected (0.14 sec)

# 恢复表结构
mysql -u root -p -e 'create database test01;use test01;source ./test01.sql'

# 准备（prepare）备份
innobackupex --apply-log --redo-only --export --user=root --password=Pwd@123456 /data/backup/test01


# 循环恢复每个表
# 释放表空间
# 复制文件到数据目录
# 修改属主
# 导入表空间

# sed 中 “#” 与替换同意，等价为 “sed "s/\`//g"” 前后符号替换为空；

for i in $(grep "CREATE TABLE" dssp_log.sql |awk '{print $3}'|sed "s#\`##g")
do
mysql -u root -pPwd@123456 -e "set foreign_key_checks=0;alter table dssp_log.$i discard tablespace;"
/usr/bin/cp -f dssp_log/dssp_log/$i.cfg dssp_log/dssp_log/$i.ibd dssp_log/dssp_log/$i.frm /data/mysql/dssp_log/
chown -R mysql:mysql /data/mysql/dssp_log/
mysql -u root -pPwd@123456 -e "set foreign_key_checks=0;alter table dssp_log.$i import tablespace;analyze table dssp_log.$i"
done
```





&copy;zhouzunchi@gmail.com