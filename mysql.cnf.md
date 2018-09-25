# yixiang mysql
## Mysqld Finetune Parameters
### Parameters
#### innodb_buffer_pool_size

**这个是InnoDB最重要的变量**， 因为5.7的mysql的默认DB引擎是InnoDB， 所以这个就是最重要的mysql的变量。
InnoDB buffer pool is where InnoDB keeps cached data, secondary indexes, dirty data (data that’s been modified, but not yet written to disk) and various internal structures such as Adaptive Hash Index. 这个变量就是用来指定这个变量的size的。

如果为shared server，
1. 简单的办法就是按照目前内存的一个百分比来进行设置，
2. 如果想要准确一些的话， 我们通过如下语句获得如果需要cache整个数据库所需要的InnoDB buffer pool的大小， 然后在根据具体的值去进行配置。
```sql
SELECT engine,
  count(*) as TABLES,
  concat(round(sum(table_rows)/1000000,2),'M') rows,
  concat(round(sum(data_length)/(1024*1024*1024),2),'G') DATA,
  concat(round(sum(index_length)/(1024*1024*1024),2),'G') idx,
  concat(round(sum(data_length+index_length)/(1024*1024*1024),2),'G') total_size,
  round(sum(index_length)/sum(data_length),2) idxfrac
FROM information_schema.TABLES
WHERE table_schema not in ('mysql', 'performance_schema', 'information_schema')
GROUP BY engine
ORDER BY sum(data_length+index_length) DESC LIMIT 10;
```
#### innodb_log_file_size
这里的log file是指InnoDB的redo log file， 也就是transaction log，此变量不可以设置得太小，否则会很影响性能。 另外要谈到的是，innodb_log_buffer_size,  这个变量保存默认值就好了。
#### innodb_flush_method
在linux上就默认使用O_DIRECT就可以了， 这个是避免double-buffering
#### innodb_buffer_pool_instances
首先这个变量并不会改变a single query response time, the difference will only show with highly concurrent workloads. 系统的default是1， 我们这里改成8.
#### innodb_thread_concurrency
这个值在cpu or io达到一个饱和点时，会有作用， 不过还是要根据具体情况来，我觉的这个要饱和了就差不多要升配了。
#### innodb_io_capacity, innodb_io_capacity_max
这个变量只跟write有关， innodb_io_capacity一般却50%~75%的硬盘random IOPS能力， 而innodb_io_capacity_max就取满就可以了，单个普通的1T硬盘的IOPS是75~100， 系统默认是200和2000， 肯定是达不到的， 特别要是设置了innodb_flush_log_at_trx_commit为1的话那就很有可能硬盘io把系统给卡死。
#### query_cache_type, query_cache_size
just turn it off and disable mutex
#### innodb_flush_log_at_trx_commit
sync是保持cache一致， flush表示写回文件系统
1. set 1 表示在每个transaction commit后，instruct InnoDB to flush and sync；
2. set 0 表示 flush to disk, but do NOT sync(所以这里在commit是， 实际上是不发生硬盘的IO的)
3. set 2 表示 do NOT flush to disk and do NOT sync的(同2， 不发生硬盘IO)
如果set为0 or 2， sync is performed once per second。 如果这个时候**断电**， commit是不会被保存的。
如果要保证数据的绝对安全， 就需要set 1了；
这里保持系统默认就可以了。
### my.cnf 
假设服务器为8G内存
```ini
[mysqld]
innodb_buffer_pool_size=2G
innodb_log_file_size=256M
innodb_flush_method=O_DIRECT
innodb_buffer_pool_instances=8
innodb_thread_concurrency=8
query_cache_type=0
query_cache_size=0
```

## Mysql Auto Backup Linux
1. 如果数据库大于20G, 这种备份方法就可能不太合适，可以使用Percona XtraBackup去处理这种情况；
2. 因为在库中可能会有较大的table，大于5G的table，在跨主机备份时会有较大的时间延迟， 系统默认的net_read_timeout和net_write_timeout都比较小， backup和restore会有可能失败的；
3. 5G数据库大小， 机械硬盘，1Gbps内网，备份时长10分钟；
### mysqld的相关配置
```ini
[mysqld]
net_read_timeout=3600
net_write_timeout=3600
```
### backup script
``` bash
#!/bin/bash
# prerequirement mysql-client, mysql-client-core
USER=''
PASSWORD=''
DB_HOST=''
BACKUP_PATH='/tmp/yxmysqlbackup'
MYSQL_CLIENT='mysql'
MYSQL_DUMP='mysqldump --quick --lock-tables=false --single-transaction --max_allowed_packet 1G '
DAYS_TO_KEEP=15

# Create the backing up directory if it doesn't exist.
if [ ! -d $BACKUP_PATH ]; then
    mkdir -p $BACKUP_PATH
    if (( $? != 0 )); then
        echo "$BACKUP_PATH creation failed."
        exit 1
    fi
fi


# Retrieve all databases in the target mysqld instances
databases=`$MYSQL_CLIENT -u $USER -p$PASSWORD -h $DB_HOST -e "SHOW DATABASES;" | tr -d "|" | grep -v Database`
if (( $? != 0 )); then
    echo "retrieve database list failed."
    exit 1
fi


# Back up all databases except the system internal databases.
for db in $databases; do
    if [ $db == 'information_schema' ] || [ $db == 'performance_schema' ] || [ $db == 'mysql' ] || [ $db == 'sys' ]; then
        # echo "Skipping database: $db"
        continue
    fi
    date=`date -I`
    echo `date`
    echo "Backing up database: $db"
    $MYSQL_DUMP -u $USER -p$PASSWORD -h $DB_HOST --databases $db > $BACKUP_PATH/${date}_$db.sql
    if (( $? != 0 )); then
        echo "These's problem during backing up $db, whole backing up procedure will be halt!" 
        echo "all old backups will remain untouched, please check and solve the problem first"
        exit 1
    fi
    echo "finished backing up database: $db"
done


# Delete backups which are older than specified days, if it's zero, nothing will be deleted.
if [ $DAYS_TO_KEEP -gt 0 ]; then
    echo "Deleting the backups older than $DAYS_TO_KEEP days"
    find $BACKUP_PATH/* -mtime +$DAYS_TO_KEEP -exec rm {} \;
    if (( $? != 0 )); then
        echo "there's problem during cleaning up, please check!"
    else
        echo "Clean up finished."
    fi
fi

```

### cron adding script
每天凌晨0:30分运行备份脚本
```bash
# sudo crontab -e
# 30 0 * * * /home/scripts/yxmysqlbackup.sh >> /var/log/mysqlautobackup.log 2>&1
```

## data restore
### restore script
```bash
/usr/libexec/mysql-workbench/mysql --defaults-file="/tmp/tmp0PEw98/extraparams.cnf"  --protocol=tcp --host=localhost --user=root --port=3306 --default-character-set=utf8 --comments  < "/data/download/2e7753ca-a8fc-4b45-8713-0fce421fcead_backup_20180921011107.sql"
```

## reference
1. https://www.speedemy.com/17-key-mysql-config-file-settings-mysql-5-7-proof/
2. https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/
3. https://blog.toadworld.com/2017/10/19/data-flushing-mechanisms-in-innodb
4. https://www.percona.com/blog/2006/07/03/choosing-proper-innodb_log_file_size/
5. https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/
6. https://www.percona.com/blog/2014/01/24/mysql-server-memory-usage-2/
7. https://hostadvice.com/how-to/how-to-tune-and-optimize-performance-of-mysql-5-7-on-ubuntu-18-04/

