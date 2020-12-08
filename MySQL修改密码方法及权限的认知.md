当我们安装好MySQL以后就需要登陆，怎么登陆？我们只列出几个重要参数。

MySQL -h 主机 -P 端口号 -u 用户名 -p

回车后输入密码即可。

如果-h参数没有默认本机，-P(大写的)没有，默认是3306.

####1、修改密码或者忘记了密码

*   如果不是管理员，自己联系管理员操作。

*   如果是管理员，登陆doc进入mysql的bin目录

    *   mysqladmin -u用户名 -p旧密码 password 新密码

        注意-u和用户名、-p和旧密码之间没有空格，password和新密码之间需要空格。

        ~~~msyql
        mysqladmin -utest -p123456 password abc123
        ~~~

    *   直接登陆MySQL

        set password for 用户名@localhost = password('新密码');

        ~~~mysql
        set password for root@localhost = password('abc123');
        ~~~

    *   用update命令直接操作user表

        ~~~mysql
        use mysql;
        update user set password=password('abc123') where user='root';
        ~~~

        注意上面修改密码的方法在5.7以前会报错，因为在MySQL5.7 password字段已从mysql.user表中删除，新的字段名是 authenticalion_string。

        可使用desc user;查看，其中一行如下

        **authenticalion_string text YES NULL**

        ~~~mysql
        update user set authenticalion_string=password('abc123') where user='root';
        ~~~

        这样就可以修改成功。

    *   如果忘记登陆密码，应该怎么操作。

        Linux中：

        打开/etc/my.cnf,在里面增加一行skip-grant-tables保存后退出，并重启MySQL。

        然后直接登录MySQL不需要用户名和密码，重复上面3的操作。

        Windows中：

        打开dos,进入mysql/bin中，输入mysqld --skip-grant-tables 回车。

        再打开一个dos窗口，进入mysql/bin，输入./mysql然后就可以就可以直接进入mysql，重复上面3的操作。

####2、MySQL用户及权限

用户详情的权限

![img](https://f10.baidu.com/it/u=863733630,1961985480&fm=173&app=49&f=JPEG?w=640&h=563&s=1E2A7423199FF1CC0875D0CA0000C0B1&access=215967316)

![img](https://f12.baidu.com/it/u=2211663563,2027763830&fm=173&app=49&f=JPEG?w=640&h=178&s=FFAA3063C53844200879D1C30000C0B1&access=215967316)

####3、权限的主要作用

*   可以限制用户访问哪些库、哪些表。
*   可以限制用户对哪些表执行SELECT、INSERT、DELETE、DELETE等操作。
*   可以限制用户登录的IP
*   可以限制用户自己的权限是否可以授权给别的用户

####4、创建一个用户

~~~mysql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
~~~

这样就创建了一个名为：username，主机为：host(可以是本机，可以是远程主机)，密码为：password 的用户。

host - 本机：localhost，指定主机：对应ip，允许任何机器访问，写成%password - 密码，空字符串的时候，默认不需要密码。

####5、查看用户的权限

~~~mysql
show grants for username@host;
~~~

新建用户只有usage权限，此权限只能连接数据库和查询information_schema。

这个权限是MySQL默认的，无法删除，不管赋予任何权限，都会隐含这个权限。

####6、授权

~~~mysql
GRANT privileges ON databasename.tablename TO 'username'@'host' with grant option
~~~

1.  GRANT - 给用户添加权限，权限会自动叠加，不会覆盖之前赋予的权限，如果你先给用户赋予了一个SELECT权限

    ~~~mysql
    GRANT SELECT ON databasename.tablename TO 'username'@'host';
    ~~~

    后来又给用户赋予了一个INSERT权限

    ~~~mysql
    GRANT INSERT ON databasename.tablename TO 'username'@'host';
    ~~~

    那么该用户就同时拥有了SELECT和INSERT权限。

2.  privileges - 用户的操作权限,如SELECT , INSERT , UPDATE 等。如果要授予所的权限则使用ALL;

    权限这个很重要，赋予权限的时候不要为了图简单，直接给所有的权限，这样用户对数据库就有完全的控制权，如果不小心或者误操作删除一些东西会很麻烦，一般给到用户权限:创建create、增insert、删delete、改update、查select、execute、events、jobs权限就可以了。

3.  ON - 表示这些权限对哪些数据库和表生效。

    指定数据库不指定表，对databasename数据库下面所有表都有SELECT操作权限

    ~~~mysql
    GRANT SELECT ON databasename.*TO 'username'@'host';
    ~~~

    指定数据库又指定表，只能对databasename下面的tablename进行SELECT操作权限

    ~~~mysql
    GRANT SELECT ON databasename.tablename TO 'username'@'host';
    ~~~

    不指定任何数据库和数据表，对所有的数据和数据表都有SELECT操作权限

    ~~~mysql
    GRANT SELECT ON *.*TO 'username'@'host';
    ~~~

4.  TO - 将权限授予哪个用户。

*   这个前面已经说过，这里只想说说host，host表示谁可以访问该机器。一般最好不要设置成%，这样如果别人知道了你的用户名和密码，你的数据库就危险了，最好写成具体的IP。

5.  with grant option 表示允许用户将自己的权限授权给其它用户。这个特性一般不会给用户。实际中，数据库管理员也不可能把这么多权限给普通用户。

    我们还可以把权限作用于表的具体字段上

    ~~~msyql
    grant select(column, column1, column2) on databasename.tablenam to username@host;
    ~~~

    username只能访问以上字段，其它字段没任何操作权限。

    权限有很多我就不一一举例，只列出一些关键字

    create,alter,drop,references(外键),create temporary tables(临时表),index(索引权限),create view/show view(视图)等等

6.  撤销已经赋予的权限

    ~~~mysql
    revoke select, insert, update, delete on test.* from 'username'@'host';
    ~~~

    这样就把username的select, insert, update, delete去掉。

####7、权限一般分配原则

1.  对于线上库

    1.  DBA：有所有权限，超级管理员权限
    2.  应用程序：分配insert、delete、update、select、execute、events、jobs权限。
    3.  测试人员：select某些业务表权限
    4.  开发人员：select某些业务表权限
    5.  原则：所有对线上表的操作，除了应用程序之外，都必须经由DBA来决定是否执行、以及什么时候执行等。

2.  测试库

    1.  DBA：所有权限。
    2.  测试人员：有insert、delete、update、select、execute、jobs权限。
    3.  数据分析人员：只有select查询权限
    4.  开发人员：有select权限。

    原则：DBA有所有权限，而且严格控制表结构的变更，不允许除了dba之外的人对测试环境的库环境进行修改，以免影响测试人员测试。所有对测试库的表结构进行的修改必须由测试人员和DBA一起审核过后才能操作。

3.  开发库
    1.  DBA：所有权限
    2.  测试人员：有库表结构以及数据的所有操作权限。
    3.  开发人员：有库表结构以及数据的所有操作权限。
    4.  数据分析人员：有库表结构以及数据的所有操作权限。

