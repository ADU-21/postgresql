# Install postgres

CentOS上：

```
rpm -ivh http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-1.noarch.rpm
yum install postgresql94-server postgresql94-contrib
```
如果遇到data文件为空的情况:

```
/usr/pgsql-9.4/bin/postgresql94-setup initdb
```
最重要的两个配置文件:

**pg_hba.conf**涉及到登录权限，网络权限
**postgres.conf**涉及到其他应用层规则
 
 
# Rename database
如果你想重命名一个数据库，但是有session正在使用这个数据库，你可以这样：

```
SELECT
    pg_terminate_backend(pid)
FROM
    pg_stat_activity
WHERE
    -- don't kill my own connection!
    pid <> pg_backend_pid()
    -- don't kill the connections to other databases
    AND datname = 'DATABASE_NAME'
    ;
```

# Master-Slave

首先你要找到所有的配置文件在哪个目录，通常在```/var/lib/pgsql```这个目录下

###主是10.12.12.10这台机器

首先需要配置一个账号进行主从同步。

修改```pg_hba.conf```，增加replica用户，进行同步。

```
host    replication     replica     10.12.12.12/32                 md5
```
这样，就设置了replica这个用户可以从10.12.12.12 对应的网段进行流复制请求。

给postgres设置密码，登录和备份权限。

```
postgres# CREATE ROLE replica login replication encrypted password 'replica'
```
修改```postgresql.conf```，注意设置下下面几个地方：

```
wal_level = hot_standby  # 这个是设置主为wal的主机
max_wal_senders = 32 # 这个设置了可以最多有几个流复制连接，差不多有几个从，就设置几个
wal_keep_segments = 256 ＃ 设置流复制保留的最多的xlog数目
wal_sender_timeout = 60s ＃ 设置流复制主机发送数据的超时时间

max_connections = 100 # 这个设置要注意下，从库的max_connections必须要大于主库的
```
重启主

```
systemctl restart postgresql
```
postgres的从配置
### 从是10.12.12.12这台机器

创建的目录为 ```/var/lib/pgsql/data```

```
pg_basebackup -F p --progress -D /data/pgsql/data2 -h 10.12.12.10 -p 5432 -U replica --password replica
```

这里使用了pg_basebackup这个命令，```/var/lib/pgsql/data```这个目录是空的

成功之后，就可以看到这个目录中现有的文件都是一样的了。

进入到```/var/lib/pgsql/data```目录，复制```recovery.conf```，这个文件可以从pg的安装目录的share文件夹中获取，比如

```
cp /usr/local/postgres94/share/recovery.conf.sample /data/pgsql/data2/recovery.conf
```
修改recovery.conf，只要修改几个地方就行了

```
standby_mode = on  # 这个说明这台机器为从库
primary_conninfo = 'host=10.12.12.10 port=5432 user=replica password=replica'  # 这个说明这台机器对应主库的信息

recovery_target_timeline = 'latest' # 这个说明这个流复制同步到最新的数据
postgresql.conf中也有几个地方要进行修改

max_connections = 1000 ＃ 一般查多于写的应用从库的最大连接数要比较大

hot_standby = on  ＃ 说明这台机器不仅仅是用于数据归档，也用于数据查询
max_standby_streaming_delay = 30s # 数据流备份的最大延迟时间
wal_receiver_status_interval = 1s  # 多久向主报告一次从的状态，当然从每次数据复制都会向主报告状态，这里只是设置最长的间隔时间
hot_standby_feedback = on # 如果有错误的数据复制，是否向主进行反馈
```
### 好了，现在启动从库


## 这个时候，你可以去verify你的数据库是否配置好主从了
确认主库和从库都配置好了
查看进程，主库所在的机器中会看到sender进程

```
8467 postgres  20   0  255m 2396 1492 S  0.0  0.1   0:00.66 postgres: wal sender process replica 
从库所在的机器中会看到receiver进程

8466 postgres  20   0  298m 1968 1096 S  0.0  0.1   0:06.88 postgres: wal receiver process   streaming 3/CF118C18
```
### 查看复制状态
主库中执行：

```
postgres=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 8467       # sender的进程
usesysid         | 44673      # 复制的用户id
usename          | replica    # 复制的用户用户名
application_name | walreceiver  
client_addr      | 10.12.12.12 # 复制的客户端地址
client_hostname  |
client_port      | 55804  # 复制的客户端端口
backend_start    | 2015-05-12 07:31:16.972157+08  # 这个主从搭建的时间
backend_xmin     |
state            | streaming  # 同步状态 startup: 连接中、catchup: 同步中、streaming: 同步
sent_location    | 3/CF123560 # Master传送WAL的位置
write_location   | 3/CF123560 # Slave接收WAL的位置
flush_location   | 3/CF123560 # Slave同步到磁盘的WAL位置
replay_location  | 3/CF123560 # Slave同步到数据库的WAL位置
sync_priority    | 0  #同步Replication的优先度
                      0: 异步、1～?: 同步(数字越小优先度越高)
sync_state       | async  # 有三个值，async: 异步、sync: 同步、potential: 虽然现在是异步模式，但是有可能升级到同步模式
```


# 创建一个用户给应用

在数据库中执行：
```
create user appadmin with password "password";
create schema appadmin;
grant all on database mme_commerce to appadmin;
grant all on table in schema public to appadmin;
grant all on table in schema appadmin to appamin;
```

配置pg_hda.conf，
具体可参照：
[http://mysql.taobao.org/monthly/2016/05/03/](http://mysql.taobao.org/monthly/2016/05/03/)