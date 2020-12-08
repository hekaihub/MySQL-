#安装

redis安装很简单，到官网下载最新版Redis，解压，进入目录，make&make install,等待一会就编译好了。

redis命令都在目录src里面(redis-server,redis-cli等)。

###1、redis慢日志配置及查询

​	慢查询的作用：通过慢查询分析，找到有问题的命令进行优化。

​	Redis slowlog是Redis用来记录查询执行时间的日志系统。

​	查询执行时间指的是不包括像客户端响应(talking)、发送回复等IO操作，而单单是执行一个查询命令所耗费的时间。

​	slow log保存在内存里面，读写速度非常快，因此你可以放心地使用它，不必担心因为开启slow log而损害Redis的速度。

1.  慢查询主要配置两个参数

    *   slowlog-max-len 主要用来说明超过多久算是慢查询，默认是10ms，一般设置1ms。
    *   slowlog-log-slower-than 慢查询存储的命令个数，超过这个设置就会把最早的删除，一般设置1000

    慢查询日志由以下四个属性组成

    1.  标识ID - (integer) 3
    2.  发生时间戳 - (integer) 1561291570
    3.  命令耗时 - (integer) 6
    4.  执行命令和参数 - 1) "get" 2) "name"

2.  慢查询的命令

    *   config get slowlog-log-slower-than 获取系统设置慢查询时间

    *   config set slowlog-log-slower-than n 设置慢查询时间

    *   config get slowlog-max-len 获取存储慢查询命令个数

    *   config set slowlog-max-len 设置慢查询命令个数

    *   slowlog get 获取慢查询日志

        ![img](https://f10.baidu.com/it/u=3011776449,2451483614&fm=173&app=49&f=JPEG?w=640&h=581&s=6960A342DBE4B74F5ADDD58F0200E081&access=215967316)

    *   slowlog len 获取日志长度

    *   slowlog get n 获取前n条慢查询日志

    *   slowlog reset 清除慢日志队列

        ![img](https://f10.baidu.com/it/u=3688409049,3927189217&fm=173&app=49&f=JPEG?w=640&h=667&s=6960A342DAA6A64F5441DC8F0200E081&access=215967316)

    因为我设置的是记录所有命令，所以下面两条记录的是slowlog reset和slowlog len命令。

###2、性能优化

1.  优化的一些建议

    1.  尽量使用短的key，对于value有些也可精简，能使用int就int。
    2.  避免使用keys *，因为keys *, 这个命令是阻塞的，即操作执行期间，其它任何命令在你的实例中都无法执行。当redis中key数据量小时到无所谓，数据量大就很糟糕了。所以我们应该避免去使用这个命令。可以去使用SCAN,来代替。
    3.  我们应该尽可能的利用key有效期。过了有效期Redis就会自动为你清除！
    4.  选择回收策略(maxmemory-policy)，当Redis的实例空间被填满了之后，将会尝试回收一部分key。根据你的使用方式，强烈建议使用 volatile-lru（默认） 策略——前提是你对key已经设置了超时。但如果你运行的是一些类似于 cache 的东西，并且没有对 key 设置超时机制，可以考虑使用 allkeys-lru 回收机制，具体讲解查看 。maxmemory-samples 3 是说每次进行淘汰的时候 会随机抽取3个key 从里面淘汰最不经常使用的（默认选项）。

2.  maxmemory-policy 六种方式

    1.  volatile-lru：只对设置了过期时间的key进行LRU（默认值）
    2.  allkeys-lru ： 是从所有key里 删除 不经常使用的key
    3.  volatile-random：随机删除即将过期key
    4.  allkeys-random：随机删除
    5.  volatile-ttl ： 删除即将过期的
    6.  noeviction ： 永不过期，返回错误

3.  使用bit位级别操作和byte字节级别操作来减少不必要的内存使用

    1.  bit位级别操作：GETRANGE, SETRANGE, GETBIT and SETBIT
    2.  byte字节级别操作：GETRANGE and SETRANGE
    3.  尽可能地使用hashes哈希存储
    4.  当业务场景不需要数据持久化时，关闭所有的持久化方式可以获得最佳的性能
    5.  想要一次添加多条数据的时候可以使用管道
    6.  限制redis的内存大小（64位系统不限制内存，32位系统默认最多使用3GB内存）
    7.  数据量不可预估，并且内存也有限的话，尽量限制下redis使用的内存大小，这样可以避免redis使用swap分区或者出现OOM错误。（使用swap分区，性能较低，如果限制了内存，当到达指定内存之后就不能添加数据了，否则会报OOM错误。可以设置maxmemory-policy，内存不足时删除数据）

4.  hash的应用

    **示例**：我们要存储一个用户信息对象数据，包含以下信息：key为用户ID，value为用户对象（姓名，年龄，生日等）如果用普通的key/value结构来存储，主要有以下2种存储方式：

    1.  将用户ID作为查找key,把其他信息封装成一个对象以序列化的方式存储

        缺点：增加了序列化/反序列化的开销，引入复杂适应系统，修改其中一项信息时，需要把整个对象取回，并且修改操作需要对并发进行保护。

    2.  用户信息**对**象有多少成员就存成多少个key-value对，虽然省去了序列化开销和并发问题，但是用户ID为重复存储。

        Redis提供的Hash很好的解决了这个问题，提供了直接存取这个Map成员的接口。Key仍然是用户ID, value是一个Map，这个Map的key是成员的属性名，value是属性值。( 内部实现：Redis Hashd的Value内部有2种不同实现，Hash的成员比较少时Redis为了节省内存会采用类似一维数组的方式来紧凑存储，而不会采用真正的HashMap结构，对应的value redisObject的encoding为zipmap,当成员数量增大时会自动转成真正的HashMap,此时encoding为ht )。

    5.  优化案例

        1.  修改**linux**中**TCP**监听的最大容纳数量

            在高并发环境下你需要一个高backlog值来避免慢客户端连接问题。注意Linux内核默默地将这个值减小到/proc/sys/net/core/somaxconn的值，所以需要确认增大somaxconn和tcp_max_syn_backlog两个值来达到想要的效果。

            echo 511 > /proc/sys/net/core/somaxconn

            **注意**：这个参数并不是限制redis的最大链接数。如果想限制redis的最大连接数需要修改maxclients，默认最大连接数为10000

        2.  修改linux内核内存分配策略

            redis在备份数据的时候，会fork出一个子进程，理论上child进程所占用的内存和parent是一样的，比如parent占用的内存为8G，这个时候也要同样分配8G的内存给child,如果内存无法负担，往往会造成redis服务器的down机或者IO负载过高，效率下降。所以内存分配策略应该设置为 1（表示内核允许分配所有的物理内存，而不管当前的内存状态如何）。

            内存分配策略有三种

            可选值：0、1、2。

            0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。

            1， 不管需要多少内存，都允许申请。

            2， 只允许分配物理内存和交换内存的大小(交换内存一般是物理内存的一半)。

            3、关闭Transparent Huge Pages(THP)

            THP会造成内存锁影响redis性能，建议关闭。

### 3、集群基础知识

​	redis-cluster把所有的物理节点映射到[0-16383]slot上,cluster 负责维护node<->slot<->value

​	Redis 集群中内置了 16384 个哈希槽，当需要在 Redis 集群中放置一个 key-value 时，redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点

![img](https://f11.baidu.com/it/u=3638883844,2162293081&fm=173&app=49&f=JPEG?w=640&h=385&s=99A35F30077373245E7DC9CD020030B3&access=215967316)

Key:a

计算a的hash值，例如值为100，100这个槽在server1上，所以a应该放到server1.

Key:hello

Hash值：10032，此槽在server2上。Hell可以应该存在server2.

###4、集群容错投票

![img](https://f12.baidu.com/it/u=345214706,912909655&fm=173&app=49&f=JPEG?w=493&h=362&s=0933EC128B7251AF066504DA0200C0B2&access=215967316)

1.  领着投票过程是集群中所有master参与,如果半数以上master节点与master节点通信超过(cluster-node-timeout),认为当前master节点挂掉.
2.  什么时候整个集群不可用(cluster_state:fail)?
    *   如果集群任意master挂掉,且当前master没有slave.集群进入fail状态,也可以理解成集群的slot映射[0-16383]不完成时进入fail状态. ps : redis-3.0.0.rc1加入cluster-require-full-coverage参数,默认关闭,打开集群兼容部分失败.
    *   如果集群超过半数以上master挂掉，无论是否有slave集群进入fail状态.

 Ps：当集群不可用时,所有对集群的操作做都不可用，收到((error) CLUSTERDOWN The cluster is down)错误。

###5、集群搭建

​	Redis安装已经讲过，我们直接进入正题，在电脑上创建一个文件夹cluster，里面分别创建7000，7001，7002，7003，7004，7005，把Redis里面的redis.conf分别复制到相应的文件夹中，打开7000里面Redis.conf修改

​	port port 端口号，对应的文件夹，好记

​	bind ip //默认ip为127.0.0.1 需要改为其他节点机器可访问的ip 否则创建集群时无法访问对应的端口，无法创建集群

~~~redis
daemonize yes   //redis后台运行
pidfile /var/run/redis_7000.pid   //pidfile文件对应7000
cluster-enabled yes   //开启集群 把注释#去掉
cluster-config-file nodes_7000.conf //集群的配置 配置文件首次启动自动生成7000
cluster-node-timeout 15000   //请求超时 默认15秒，可自行设置
appendonly yes   //aof日志开启 有需要就开启，它会每次写操作都记录一条日志
dump7000.db   //不同数据库
~~~

其它配置文件都是一样的。配置完成后，我们启动6个Redis服务。

![img](https://f12.baidu.com/it/u=727967666,2396919670&fm=173&app=49&f=JPEG?w=640&h=107&s=AC65C34A6FA1B3685E694C0E020080C2&access=215967316)

都正常运行。

然后启动集群，原来老版本中集群用redis-trib.rb这个工具，这个工具还要安装ruby，有点麻烦，新版本直接默认用redis-cli启动集群

sudo src/redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1

--cluster-replicas 1表示希望为集群中的每个主节点创建一个从节点(一主一从)。

--cluster-replicas 2表示希望为集群中的每个主节点创建两个从节点(一主二从)。

![img](https://f10.baidu.com/it/u=814743489,1916450054&fm=173&app=49&f=JPEG?w=640&h=448&s=80D413CAFBAC9F6C5CD5F58D0200E081&access=215967316)

出现上面提示表示创建成功。

有时候出现错误，如：

![img](https://f11.baidu.com/it/u=2142971023,698533687&fm=173&app=49&f=JPEG?w=640&h=67&access=215967316)

我们需要把配置文件里面的cluster-config-file生成的配置文件删除，把数据库删除，kill掉所有的Redis服务，重新启动即可。

查看所有nodes节点

![img](https://f11.baidu.com/it/u=154053289,757547103&fm=173&app=49&f=JPEG?w=640&h=213&s=C8E4D14A3FE89B6C54519C8F02003081&access=215967316)

###6、测试

​	我们打开src/redis-cli -p 7000 -c

​	\#结尾的-c一定要写！不然会出现move失败的错误！

![img](https://f12.baidu.com/it/u=943776746,759889642&fm=173&app=49&f=JPEG?w=640&h=92&access=215967316)

​	我们写入，提示写入了5798这个槽。

​	我在打开一个src/redis-cli -p 7003 -c ，获取这个key

![img](https://f11.baidu.com/it/u=32724883,4225076624&fm=173&app=49&f=JPEG?w=640&h=60&access=215967316)

​	上面显示从节点7001获取了这个值。

###7、配置主从

​	配置主从我们可以同一台机器启动多个Redis，也可以自己准备n台电脑(原理都是一样)。

​	我们主要讲一台电脑上配置多个Redis。

​	进入redis目录，将redis.conf复制2-3个

![img](https://f11.baidu.com/it/u=301412587,1501466923&fm=173&app=49&f=JPEG?w=640&h=128&s=4170A3664FE99B7A14E5659C0200D082&access=215967316)

​	我这里复制两份redis6380.conf\redis6381，我做一主两从测试。

​	然后打开redis.conf编辑。

###8、主库设置

​	bind选项可以绑定ip，也可以不绑定，绑定ip比较安全。默认bind 127.0.0.1，也可以注释，所有ip都可以链接。

​	port 6379 不要和其它两个配置重复.

​	daemonize yes/no 是否后台运行。

​	protected-mode yes/no 这个是保护模式，安全配置，这个配置防止外网链接。自己做测试的时候免得麻烦直接赋值no,生成环境改成yes，与bind和requestpass一起使用，比较安全。

​	requirepass 设置密码，登陆该redis需要此密码。

**从库也差不多就多几个配置**

​	slave-read-only 这个配置redis只读属性。(看自己)

​	slaveof 127.0.0.1 6379 这个就是绑定主库，说明自己是从库。(其实主从配置就是这一步)

​	masterauth 主库密码，同步时候需要主库密码，不然连不上。

​	到此配置就完成了。如果自己测试的话，所有的密码都可以不设置。

​	我们启动Redis。redis-server redis.conf/redis-server redis6380.conf/redis-server redis6381.conf

![img](https://f12.baidu.com/it/u=2076901811,3041513151&fm=173&app=49&f=JPEG?w=640&h=54&access=215967316)

​	测试一下，首先在主库写入set name zhangsan

![img](https://f10.baidu.com/it/u=2723074452,428396710&fm=173&app=49&f=JPEG?w=640&h=187&s=6970A3428BA0BB700259CC830200E081&access=215967316)

​	然后去从库获取，get name

![img](https://f11.baidu.com/it/u=4069118908,1942462810&fm=173&app=49&f=JPEG?w=640&h=175&s=697083428BACBB7058F5F18B02003081&access=215967316)

![img](https://f11.baidu.com/it/u=1647054409,1697204257&fm=173&app=49&f=JPEG?w=640&h=193&s=497083428BACBB7058F5D58B02003081&access=215967316)

​	从截图看到，主从配置成功并且数据同步

​	如果主库宕掉，怎么办？手动改库？Redis提供哨兵模式，可以监控每个Redis服务器运行状况。下面说说哨兵。

###9、哨兵

​	哨兵顾名思义，放哨的呀！

​	Sentinel(哨兵)进程是用于监控redis集群中Master主服务器工作的状态，在Master主服务器发生故障的时候，可以实现Master和Slave服务器的切换，保证系统的高可用。

​	每个哨兵(Sentinel)进程会向其它哨兵(Sentinel)、Master、Slave定时发送消息，以确认对方是否”活”着，如果发现对方在指定配置时间(可配置的)内未得到回应，则暂时认为对方已掉线，也就是所谓的”主观认为宕机” ，英文名称：Subjective Down，简称SDOWN。有主观宕机，肯定就有客观宕机。当“哨兵群”中的多数Sentinel进程在对Master主服务器做出 SDOWN 的判断，并且通过 SENTINEL is-master-down-by-addr 命令互相交流之后，得出的Master Server下线判断，这种方式就是“客观宕机”，英文名称是：Objectively Down， 简称 ODOWN。通过一定的vote算法，从剩下的slave从服务器节点中，选一台提升为Master服务器节点，然后自动修改相关配置，并开启故障转移（failover）。

​	上面是百度资料。下面就来配置，上面主从已经配置好了，下面就直接配置哨兵，一般哨兵配置奇数个，防止同一个哨兵多次投票。我们演示一个哨兵模式(多个哨兵模式原理一样)，哨兵配置文件一模一样，除了端口。

​	port 26379 一般都是前面加2

​	sentinel monitor mymaster ip port num 设置那个ip是主库，几个哨兵投票决定是否更换主库。

​	sentinel auth-pass mymaster pass 这个是重点，如果Redis配置密码，这个一定要写，不然链接不上，而且位置一定要在ip下面

​	sentinel down-after-milliseconds mymaster 5000 哨兵每隔5秒检查一次服务器

​	sentinel failover-timeout mymaster 60000 故障转移的超时时间

​	sentinel parallel-syncs mymaster n 选项指定了在执行故障转移时，最多可以有多少个slave同时对新的master进行异步复制（并发数量），这个数字越小，完成故障转移所需的时间就越长（数字越小，同时能进行复制的slave越少）。

​	启动哨兵

![img](https://f10.baidu.com/it/u=4168986822,491932010&fm=173&app=49&f=JPEG?w=640&h=367&s=4951A3465BA487681CC9BC0E02003081&access=215967316)

​	哨兵启动，监听了主库和从库。

​	我们在看看哨兵里面

![img](https://f11.baidu.com/it/u=3769616818,2567848108&fm=173&app=49&f=JPEG?w=640&h=133&s=69608346EFE09B7014EDD48D0200E081&access=215967316)

​	上图可以到6379是主库，运行正常，还有两个从库。

​	我们现在kill掉主库，看哨兵运行模式

![img](https://f10.baidu.com/it/u=1144014558,250697739&fm=173&app=49&f=JPEG?w=640&h=316&s=4960BB42D3B5826912E5F4890300E081&access=215967316)

​	看到哨兵，把6381选为主库。进入哨兵客户端

![img](https://f12.baidu.com/it/u=3195537536,457801155&fm=173&app=49&f=JPEG?w=640&h=124&s=4970A346EFE0BB705665548D0200E081&access=215967316)

​	现在主库切换为了6381

​	到此主从和哨兵配置完成。多个哨兵模式配置和一个哨兵模式是一样的。