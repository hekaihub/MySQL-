####1、MySQL事务

1.  事物定义

    事物是一个最小的不可再分的工作单元，通常一个事务对应一个完整的数据库操作。一个完整的事物需要批量的DML(insert、update、delete)语句共同联合完成。只有DML语句才有事物相关操作。

2.  事物的四大特征(ACID)

    1.  a 、原子性 (atomicity)

        一个事物必须被视为一个不可分割的最小工作单元，要么全部成功，要么全部失败。

    2.  c 、一致性 (consistency)

        数据库总是从一个状态到另一个状态，之前状态改变必须和之后状态一致。

    3.  i 、隔离性 (isolation)

        事务在提交前对其它事务是不可见的，各自互不干扰。

    4.  d 、持久性 (durability)

        事务一旦提交就不能更改。

3.  事物操作

    1.  开启事务：Start Transaction
    2.  事务结束：End Transaction
    3.  提交事务：Commit Transaction
    4.  回滚事务：Rollback Transaction

    在事物进行过程中，任何DML语句是不会更改数据库，只是将DML操作在内存中完成记录。只有在事物成功的结束的时候，才会修改数据库中的数据。

4.  事物操作事例

    *   成功操作事物例子

        ~~~mysql
        start transaction;  #手动开启事务
        insert into table(id, name) values(1, '张三'); #将数据插入数据库
        update table set name = '李四' where id=2; #更改数据库数据
        commit; #commit之后即可改变数据库数据
        ~~~

    *   失败操作事物例子

        ~~~mysql
        start transaction; #手动开启事务
        insert into table(id, name) values(1, '张三'); #将数据插入数据库
        update table set name = '李四' where id=2;     #更改数据库数据
        rollback;  #rollback之后不管前面DML语句是否执行成功，数据库里面数据都不会改变
        ~~~

    *   php执行事物事例

        ~~~php
        $mysqli->autocommit(FALSE); //关闭自动提交功能
        $sql = "update table set name = '李四' WHERE id = 2";
        $res1 = $mysqli->query($sql);
        $res1_rows = $msyqli->affected_rows;
        $sql2 = "insert into table(id, name) values(1, '张三')";
        $res2 = $mysqli->query($sql2);
        $res2_rows = $mysqli->affected_rows;
        if($res1 && $res1_rows>0 && $res2 && $res2_rows>0){
        	$msyqli->commit(); //提交事务
        	echo '执行成功';
        	$mysqli->autocommit(TRUE); //开启自动提交功能
        else{
        	$mysqli->rollback(); //事务回滚
        	echo '执行失败;
        }
        ~~~

####2、事物四大特性之一(隔离级别-隔离性)

*   隔离性有4个级别

    1.  READ UNCOMMITTED (未提交读)
        *   这个级别，事务没有提交数据，其它事务也可以读取这个事务的数据，这个级别会导致脏读。
    2.  READ COMMITTED（已提交读）
        *   大多数数据库默认隔离级别，只有该事务提交后，其它事务才可以读取这个提交数据，可能导致两次查询的数据不一样。
    3.  REPEATABLE READ （可重复读）
        *   该级别保证同一个事务多次读取同样的结果记录是一致的，解决了脏读，可能导致幻读(读取某个范围内的数据时候，第一次读取后其它事务又插入新数据，第二次读取就变成其它数据)，MySQL默认的隔离级别。
    4.  SERIALIZABLE(可串行化)
        *   这个是最高隔离级别，它通过强制事务串行化执行，避免前面的幻读的问题。这个级别会给每行数据枷锁，这样导致大量的超时和挣锁问题，严重影响性能。

    ![img](https://f12.baidu.com/it/u=3597099122,1299316862&fm=173&app=49&f=PNG?w=412&h=201&s=1AAA742387305D8A1AD594DA030080B1&access=215967316)

####3、死锁

*   多个事务在同时执行，在同一资源上相互占用，并相互请求对方占用的锁资源，导致恶性循环，形成死锁。

    ~~~mysql
    start transaction;
    update table set key=1 where id=3;
    update table set key=2 where id=4;
    commit;
    start transaction;
    update table set key=1 where id=4;
    update table set key=2 where id=3;
    commit;
    ~~~

    两个事务都是执行了第一条语句，同事锁定了这两行数据，这样就导致死锁。

    大事务执行的时候要注意这样的问题，很容造成死锁，消耗资料，造成系统瘫痪。

####4、为什么需要读写分离？

*   一台数据库满足不了访问需要。数据的访问基本都是2-8原则。

####5、主从数据库

*   从数据库不执行写操作，主数据库主要执行写的操作，插入主数据库的数据都要同步到从数据库。

    1.  主数据库将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件）
    2.  slave将master的binary log events拷贝到它的中继日志(relay log)
    3.  slave重做中继日志中的事件，将更改应用到自己的数据上。

    用户在访问的时候把读的请求同步到从数据库，把写的请求分发的主数据库。

####6、主从服务优势

1.  实现服务器负载均衡

    通过服务器复制功能，可以在主服务器和从服务器之间实现负载均衡。即可以通过在主服务器和从服务器之间切分处理客户查询的负荷，从而得到更好的客户相应时间。

2.  通过复制实现数据的异地备份

    可以定期的将数据从主服务器上复制到从服务器上，这无疑是先了数据的异地备份。在传统的备份体制下，是将数据备份在本地。此时备份作业与数据库服务器运行在同一台设备上，当备份作业运行时就会影响到服务器的正常运行。有时候会明显降低服务器的性能。同时，将备份数据存放在本地，也不是很安全。如硬盘因为电压等原因被损坏或者服务器被攻击，此时由于备份文件仍然存放在硬盘上，数据库管理员无法使用备份文件来恢复数据。这显然会给企业带来比较大的损失。

3.  提高数据库系统的可用性

    数据库复制功能实现了主服务器与从服务器之间数据的同步，增加了数据库系统的可用性。当主服务器出现问题时，数据库管理员可以马上让从服务器作为主服务器，用来数据的更新与查询服务。然后回过头来再仔细的检查主服务器的问题。

####7、主从服务器的配置

1.  准备工作

    准备两天电脑或者虚拟机或者同一电脑上运行两个MySQL；下载MySQL，这个自己下载，从库的版本大于等于主库的数据库版本；安装MySQL初始化表，并在后台启动MySQL；修改root密码。

2.  修改主服务器master

    ~~~mysql
    #vi /etc/my.cnf
    [mysqld]
    log-bin=mysql-bin   //[必须]启用二进制日志
    server-id=111       //[必须]服务器唯一ID，默认是1，一般取IP最后一段
    ~~~

3.  修改从服务器slave

    ~~~mysql
    #vi /etc/my.cnf
    [mysqld]
    log-bin=mysql-bin //[不是必须]启用二进制日志
    server-id=222     //[必须]服务器唯一ID，默认是1，一般取IP最后一段
    ~~~

4.  重启两台服务器的mysql

    ~~~msyql
    /etc/init.d/mysql restart
    ~~~

5.  在主服务器上建立帐户并授权slave

    \#/usr/local/mysql/bin/mysql -u root -p

    ~~~mysql
    GRANT REPLICATION SLAVE ON *.* to 'testslave'@'%' identified by '123456';
    ~~~

    一般不用root帐号，%表示所有客户端都可以连到数据库，只要帐号，密码正确；此处可用具体客户端IP代替%，如xxx.xxx.xxx.xxx，加强安全。

6.  登录主服务器的mysql，查询master的状态

    show master status;

    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
    | ---------------- | -------- | ------------ | ---------------- |
    | mysql-bin.000001 | 308      |              |                  |

    1 row in set (0.00 sec)

    注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化

7.  配置从服务器Slave

    ~~~mysql
    change master to master_host='xxx.xxx.xxx.111',master_user='testslave',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=308;
    ~~~

    注意不要断开，308数字前后无单引号。

    ~~~mysql
    Mysql>start slave;  //启动从服务器复制功能
    ~~~

8.  检查从服务器复制功能状态

    ~~~mysql
    mysql> show slave status\G
    *************************** 1. row ***************************
    Slave_IO_State: Waiting for master to send event
    Master_Host: xxx.xxx.xxx.111  //主服务器地址
    Master_User: testslave        //授权帐户名，尽量避免使用root
    Master_Port: 3306             //数据库端口，部分版本没有此行
    Connect_Retry: 60
    Master_Log_File: mysql-bin.000001
    Read_Master_Log_Pos: 600     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
    Relay_Log_File: ddte-relay-bin.000001
    Relay_Log_Pos: 251
    Relay_Master_Log_File: mysql-bin.000001
    Slave_IO_Running: Yes        //此状态必须YES
    Slave_SQL_Running: Yes       //此状态必须YES
    ~~~

    注：Slave_IO_Running及Slave_SQL_Running进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。

    到这里，主从服务器配置完成。

9.  主从服务器测试

    主服务器Mysql，建立数据库，并在这个库中建表插入一条数据

    ~~~mysql
    mysql> create database test_db;
    Query OK, 1 row affected (0.00 sec)
    mysql> use test_db;
    Database changed
    mysql> create table test_tb(id int(3),name char(10));
    Query OK, 0 rows affected (0.00 sec)
    mysql> insert into test_tb values(1,'name');
    Query OK, 1 row affected (0.00 sec)
    mysql> show databases;
    +--------------------+
    | Database |
    +--------------------+
    | information_schema |
    | test_db |
    | mysql |
    | test |
    +--------------------+
    4 rows in set (0.00 sec)
    ~~~

    从服务器Mysql查询

    ~~~mysql
    mysql> show databases;
    +--------------------+
    | Database |
    +--------------------+
    | information_schema |
    | test_db | //I'M here，大家看到了吧
    | mysql |
    | test |
    
    +--------------------+
    4 rows in set (0.00 sec)
    mysql> use test_db
    Database changed
    mysql> select * from test_tb; //查看主服务器上新增的数据是否同步过来
    +------+------+
    | id | name |
    +------+------+
    | 1 | name |
    +------+------+
    1 row in set (0.00 sec)
    ~~~

到这里都是成功，我在操作过程遇到很多问题，1-6操作基本都不会有什么问题，到7问题就来了，因为中途我动过主数据库，导致二进制文件位置改变，一直连接不成功，所以这里一旦主库配置好了，没什么问题就不要动了，就算动了从库中相应的位置也要改变。

到了第8步的时候，如果是同一台机器应该没问题，如果是主机和虚拟机要保证主机和虚拟机可以相互访问，可以ping通，不然Slave_IO_Running，Slave_SQL_Running这两个状态是connecting，这是不成功的。两天不同的电脑也要保证相互之间可以ping通。这个配置不难，关键是要细心。

**服务运行起来了，服务可能有时候会挂掉，我们怎么监测呢？**

​	编写一shell脚本，用nagios监控slave的两个yes（Slave_IO及Slave_SQL进程），如发现只有一个或零个yes，就表明主从有问题了，发短信警报或者切换主从。