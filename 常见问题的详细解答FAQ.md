# 问题解答 FAQ

## 问题1: ProxySQL中的连接池

    答：与任何其他数据库技术一样，ProxySQL也维护一个连接池。
    连接池是维护的数据库连接缓存，以便在将来需要数据库请求时可以重用连接。
    正如我们所看到的，每个主机的max_connection设置为1000。
    因此，ProxySQL可以为该特定主机打开总共1000个后端连接。

    mysql> select hostgroup_id,hostname,port,status,weight,max_connections from mysql_servers;
    +--------------+------------+------+--------+--------+-----------------+
    | hostgroup_id | hostname   | port | status | weight | max_connections |
    +--------------+------------+------+--------+--------+-----------------+
    | 0            | 172.17.0.1 | 3306 | ONLINE | 100    | 1000            |
    +--------------+------------+------+--------+--------+-----------------+
    3 rows in set (0.00 sec)
    
    下面的变量控制池中打开的空闲连接占主机组中特定服务器的最大连接总数的百分比。
    
    mysql> select * from global_variables where variable_name like 'mysql-free_connections_pct';
    +----------------------------+----------------+
    | variable_name              | variable_value |
    +----------------------------+----------------+
    | mysql-free_connections_pct | 20             |
    +----------------------------+----------------+
    1 row in set (0.01 sec)

    这是连接池的工作流：
    会话需要连接到服务器，它将签入连接池。
    如果该后端的连接池中存在连接，则使用该连接，否则将创建新连接。
    会话释放连接时，会将其发送回主机组管理器。如果主机组管理器确定连接可以安全共享，并且连接池未满，则会将其放置在连接池中。
    如果共享连接不安全（存在会话变量或临时表等），主机组管理器将销毁该连接。
    对于每个后端服务器，主机组管理器将在连接池中保留最多mysql-free_connections_pct*mysql_servers.max_connections/100个连接。
    通过定期ping保持连接打开。
    
    因此，通过上述示例：
    池中的空闲连接数=（20*1000）/ 100=200 空闲连接。

## 问题2: 使用DBNAME 

    答：一些用户提出了一个问题，为什么即使数据库不存在，使用数据库也会在ProxySQL中始终成功。
    GitHub上报告的问题：https://github.com/sysown/proxysql/issues/876
    
    首先要注意的是：ProxySQL在执行"use"命令时不会给出错误，在执行查询时会出现错误。（稍后将提供更多详细信息）
    
    让我们看看MySQL CLI执行USE dbname时会发生什么：
    
        它发送一个COM_INIT_DB命令来更改数据库.
        它发送一个查询来显示表.
    
    [多余的`show tables`不是由ProxySQL执行的。来自客户端（mysql cli）和ProxySQL的请求正在尝试转发。
    若要禁止`show tables`命令，应使用`-A`执行mysql cli以禁用自动刷新]
    
    当客户端执行USE命令时，ProxySQL不会将请求转发到任何后端，它只在内部跟踪哪个是该特定客户端所需的默认模式[mysql_users.default_schema]。
    
    "USE DB"之所以在ProxySQL客户机上没有出现任何错误，是因为该用户的默认主机组上可能没有选定的DB，
    但在同一ProxySQL后面添加的其他主机组上也有选定的DB。
    
    请参见以下示例：
    假设我们有两台服务器，每台服务器都有一个schema，并且都在同一个ProxySQL后面运行：
    employees DB on `172.17.0.1` [Hostgroup 0] 
    digital DB on `172.17.0.3` [Hostgroup 2] 


    1、用户将employees数据库作为默认schema，并使用ProxySQL将所有查询转发到172.17.0.1。
    2、现在用户希望在digital schema上运行查询，但无法在172.17.0.1上执行请求，因为digital schema不存在。
    3、在这种情况下，ProxySQL只会向客户机回复OK for USE命令，并等待客户机发送查询。
    当客户端发送查询时，ProxySQL将通过检查查询规则来决定如何处理该查询，如果查询规则匹配，则会将请求发送到172.17.0.3。

    存在两台主机，它们属于不同的主机组。
    mysql> select hostgroup_id,hostname,port,status,weight,max_connections from mysql_servers;
    +--------------+------------+------+--------+--------+-----------------+
    | hostgroup_id | hostname   | port | status | weight | max_connections |
    +--------------+------------+------+--------+--------+-----------------+
    | 0            | 172.17.0.1 | 3306 | ONLINE | 100    | 1000            |
    | 2            | 172.17.0.3 | 3306 | ONLINE | 100    | 1000            |
    +--------------+------------+------+--------+--------+-----------------+
        
    用户用于通过ProxySQL连接到MySQL服务器。

    所以，每当同一个用户连接到服务器时，它都会到达default_hostgroup 0和default_schema employees
    mysql> select username,password,active,default_hostgroup,default_schema,max_connections from mysql_users;
    +----------+----------+--------+-------------------+----------------+-----------------+
    | username | password | active | default_hostgroup | default_schema | max_connections |
    +----------+----------+--------+-------------------+----------------+-----------------+
    | sysbench | sysbench | 1      | 0                 | employees      | 10000           |
    +----------+----------+--------+-------------------+----------------+-----------------+

    在下面的示例中，我们正在查找数字数据库中存在的表dept，该数据库仅在主机名172.17.0.3上存在
    
    用于转发hostgroup –2（属于digital schema）上的所有查询的查询规则。
    mysql> select rule_id,active,destination_hostgroup, apply,schemaname from mysql_query_rules;
    +---------+--------+-----------------------+-------+------------+
    | rule_id | active | destination_hostgroup | apply | schemaname |
    +---------+--------+-----------------------+-------+------------+
    | 1       | 1      | 0                     | 1     | employees  |
    | 2       | 1      | 2                     | 1     | digital    |
    +---------+--------+-----------------------+-------+------------+

    * 没有任何查询规则：
    查询将访问默认主机172.17.0.1 2上的employees数据库，由于找不到数据库，查询将失败。
    
    root@8a9f96bb26f9:/# mysql -usysbench -psysbench -h127.0.0.1 -P6033 -A -e "select @@server_id;use digital; select * from dept; select @@server_id;"
    +————-————-————-+
     | @@server_id | 
    +————-————-————-+ 
     | 1 | 
    +———————-————-—-+ 
    ERROR 1049 (42000) at line 1: Unknown database 'digital'
    
    * 使用查询规则：它成功工作
    
    1.下面的查询将命中默认host和默认schema，即主机"172.17.0.1"上的employees DB
    2.查询规则获取匹配项，因为找到use digital字符串。
    3.匹配规则后，下一个连续查询将被重新路由到服务器"172.17.0.3"，执行完成。

    root@8a9f96bb26f9:/# mysql -usysbench -psysbench -h127.0.0.1 -P6033 -A -e "select @@server_id;use digital; select * from dept; select @@server_id;"
    +-------------+
    | @@server_id |
    +-------------+
    |           1 |
    +-------------+
    +------+------+
    | id   | name |
    +------+------+
    |  101 | ashw |
    +------+------+
    +-------------+
    | @@server_id |
    +-------------+
    |           3 |
    +-------------+
    root@8a9f96bb26f9:/#

    因此，我们所看到的行为是意料之中的。使用数据库将始终成功，但在选择有效的schema之前，查询将失败。

## 问题3: ProxySQL 监控

    答：ProxySQL在执行故障检测时非常独特，它可以检测到服务器停机，这不是因为其监控系统无法检查后端的状态，而是因为查询失败。
    
    这意味着，即使禁用了监视器，当节点试图向其发送查询时，ProxySQL也能够检测到节点是否停机。
    
    ProxySQL的核心是这样工作的：（这意味着没有启用监视器）

    1.最初，所有节点都处于联机状态。
    2.向节点发送流量时会产生错误，如果在1秒内产生超过mysql-shun_on_failures错误，则在mysql-shun_recovery_time_seconds内会避开该节点。
    3.mysql-shun_recovery_time_sec后，ProxySQL将使节点重新联机，并尝试向其发送流量
    4.如果第2点中描述的情况再次发生，则再次回避该节点。
    
    关于第3点，需要注意的一点是，ProxySQL没有一个计时器，在mysql-shun_recovery_time_seconds之后使节点恢复在线。
    只有在特定主机组的连接池中存在活动时，该节点才会在mysql-shun_recovery_time_sec秒后恢复联机。

    以上所有功能均在未启用监视器的情况下工作！
    
        监控模块扩展了ProxySQL核心的功能。
        
    例如，如果存在网络问题，ProxySQL的核心将无法理解数据库服务器是否因为仍在运行查询或存在网络问题而没有响应。
    监视器将能够检测到这一点，并通知ProxySQL的核心后端似乎无法访问。

    如前所述，这并不完全正确。
    如果您向ProxySQL发送更多流量，并且ProxySQL尝试打开到该后端的新连接，那么ProxySQL的核心仍然可以检测到存在问题。
    backend.connect()将失败，ProxySQL将避开该节点。不过，当前连接不会受到影响。    
        
## 问题5: ProxySQL HA高可用

    答：无论ProxySQL的可用性有多高，由于未知的bug，崩溃总是有可能的。
    
    因此，ProxySQL能够在发生故障时在不到一秒钟内通过angel进程自动重启。
    
    下面是一些如何在我们的体系结构中集成ProxySQL的示例。

        * 一个单独的ProxySQL实例
        * 多个ProxySQL主机
        * 多个ProxySQL主机+LB
        * 每个应用服务器一个ProxySQL实例
        * Silos approach 筒仓法
        * Keepalived+VIP+ProxySQL
        * Keepalived+VIP+LB
        * Keepalived+VIP+LVS
        * 多层+Keepalived+VIP+LVS
        * 多层
        * Silos approach筒仓+多层

供参考：幻灯片(https://www.slideshare.net/Severalnines/webinar-slides-high-availability-in-proxysql?from_action=save)
       网络研讨会(https://www.youtube.com/watch?v=kzvazKEv4iY)

## 问题6: ProxySQL 镜像

    答:
    当我们需要工具时，为了在不同或新的平台上测试我们的产品查询，我们可以使用mirroning。
    此功能的工作原理类似于blackhole引擎，它总是在mirror_hostgroup中配置的服务器上执行查询，并记录到自己的binlog中。
    如何配置镜像：详细信息(https://github.com/sysown/proxysql/blob/v1.4.3/doc/mirroring.md)
    
    下面是一些用例，您可以在不安装新插件的情况下利用此功能。
    1.要测试不同MySQL分支（MySQL、Percona、MariaDB）上的SQL查询性能，请执行以下操作：
    2.在较新的MySQL版本上尝试生产查询[验证SQL语法]：
    3.在启用ProxySQL之前，先测试在ProxySQL中配置mysql_query_rules。

    如何验证MySQL查询：示例（https://blog.pythian.com/using-proxysql-validate-mysql-updates/）
    重要注意事项：ProxySQL镜像应仅用于基准测试、测试、升级验证和任何其他不需要正确性的活动。
    整个过程对于服务器来说不是免费的，它增加了box上的CPU利用率。因此，如果我们配置了更多的ProxySQL，我们可以只在一个或两个代理上配置镜像，以避免在每个框上使用额外的cpu。
    注意：ProxySQL镜像不支持预处理语句。如果您正在运行sysbench测试，请不要忘记添加--db-ps-mode=disable
    基准测试结果：https://www.percona.com/blog/2017/05/25/proxysql-and-mirroring-what-about-it/

## 问题10: 停止/关闭ProxySQL

    答复:
    一旦过程接收到SIGTERM信号，就会发生一些不同的情况：
    该过程可能会立即停止
    清理资源后，进程可能会在短暂延迟后停止
    进程可能会无限期地运行
    一旦收到SIGTERM，应用程序就可以确定它想要做什么。
    虽然大多数应用程序会清理它们的资源并停止，但有些可能不会。
    
    查看ProxySQL对Kill的作用
    killall – 按名称终止进程
    
    killall向运行任何指定命令的所有进程发送信号。如果未指定信号名称，则发送SIGTERM。
    要终止所有ProxySQL进程（子进程和父进程），请输入：killall proxysql

## 问题11：配置默认主机组

    回答：在下面的示例中，用户没有为主机组0配置任何主机
    
    mysql> select hostgroup_id,hostname,port,status,weight from mysql_servers;
    +--------------+------------+------+--------+--------+
    | hostgroup_id | hostname   | port | status | weight |
    +--------------+------------+------+--------+--------+
    | 2            | 172.17.0.2 | 3306 | ONLINE | 100    |
    | 3            | 172.17.0.3 | 3306 | ONLINE | 100    |
    | 1            | 172.17.0.1 | 3306 | ONLINE | 100    |
    +--------------+------------+------+--------+--------+
    
     mysql> select * from mysql_replication_hostgroups;
    +------------------+------------------+---------+
    | writer_hostgroup | reader_hostgroup | comment |
    +------------------+------------------+---------+
    | 1                | 2                | stag1   |
    +------------------+------------------+---------+
    
    mysql> select rule_id, match_digest,destination_hostgroup,apply from mysql_query_rules;
    +---------+---------------------+-----------------------+-------+
    | rule_id | match_digest        | destination_hostgroup | apply |
    +---------+---------------------+-----------------------+-------+
    | 11      | ^SELECT.*           | 2                     | 0     |
    | 12      | ^SELECT.*FOR UPDATE | 1                     | 1     |
    +---------+---------------------+-----------------------+-------+

    但仍然存在用户面临的错误：

        10000毫秒后到达hostgroup 0时达到最大连接超时
        
    这是因为用户忘记更改mysql_users中的default_hostgroup。
    
    mysql> select username,password,active,default_hostgroup,default_schema,max_connections, max_connections from mysql_users;
    +----------+----------+--------+-------------------+----------------+-----------------+-----------------+
    | username | password | active | default_hostgroup | default_schema | max_connections | max_connections |
    +----------+----------+--------+-------------------+----------------+-----------------+-----------------+
    | sysbench | sysbench | 1      | 0                 | NULL           | 10000           | 10000           |
    +----------+----------+--------+-------------------+----------------+-----------------+-----------------+
    1 row in set (0.00 sec)
    
    