## 基于neo4j的最短路径算法的实现、对比与优化

| 专业     | 班级            | 姓名   | 学号       | 分工比例 |
| :------- | :-------------- | :----- | ---------- | -------- |
| 电子信息 | 大数据工程202班 | 龙文赫 | 2020214403 |          |
| 电子信息 | 大数据工程202班 | 孙宏博 | 2020       |          |
| 电子信息 | 大数据工程202班 | 闻道恒 | 2020       |          |
| 电子信息 | 大数据工程202班 | 丁诗婷 | 2020       |          |

#### 一、 背景知识

1. Neo4J 的三种部署模式

   - stand-alone instance（单机模式）
   - HA cluster（高可用模式）
   - causal cluster（因果集群模式，简称为集群模式）

   <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/5c0fb0d2ab833f765abb2c9918616dc2&showdoc=.jpg" style="zoom: 67%;" />

   图中间是由3个CORE节点构成的核心读写集群，使用raft协议进行数据一致性保障，会自动选择一个节点作为LEADER提供读写服务，其余节点作为FOLLOWER节点复制LEADER推送的raft日志；如果LEADER节点由于网络隔离或crash而无法正常服务，FOLLOWER节点会进行重新选择一个新的LEADER节点。为了提高集群稳定性，减少不必要的选主操作，选主会分为2个阶段：预选和正式选举。如果在预选时没有得到足够多的节点回复，就不会发起正式选举（与MongoDB类似）。
   外围一圈为只读集群，节点角色均为READ_REPLICA，提供查询、报表和分析等只读服务。只读副本是通过事务日志传送从Core Servers异步复制的。 
   只读副本将定期（通常在ms范围内）轮询核心服务器以查找自上次轮询以来已处理的任何新事务，并且核心服务器会将这些事务发送到只读副本。 可以从相对较少的Core Server中馈送许多只读副本数据，从而使查询工作量大为增加，从而扩大规模。
   但是，与核心服务器不同，只读副本不参与有关群集拓扑的决策。 只读副本通常应以相对较大的数量运行，并视为一次性使用。 丢失只读副本不会影响群集的可用性，除了丢失其图表查询吞吐量的一部分。 它不会影响群集的容错能力。


2. causal cluster（因果集群模式，简称为集群模式）
   - Core Servers核心服务器
     使用Raft协议复制所有事务来保护数据
     如果Core Server集群遭受足够的故障而无法处理写入，会变为只读状态以保持安全。
   - Replica Servers
     提供查询、报表和分析等只读服务
     通过事务日志传送从Core Servers异步复制的。
   - 因果一致性
     Raft 协议 + 异步同步复制协议 保证了一致性
   - “写后读”一致性问题
     通过因果一致性模式 （即用户应该能够读取到其之前写入数据库的数据）解决了一般“多读少写”的数据库的问题。
     图数据库通常是一类“多读少写”的数据库。即使在写操作期间，也必须读取和遍历图数据。这就是在 Neo4J 集群中通常只读节点要多于核心节点的原因。
     要解决这一“写后读”一致性问题，Webber 介绍了 Neo4J 中提供的一种因果一致性模式，称为“书签”。书签模式的第一阶段包含一次写操作，写操作完成后将返回相应的事务标识为书签。第二阶段是一次读操作，客户端把书签将其作为参数提供给后续事务。 使用该书签，集群可以确保只有处理了该客户已添加书签的事务的服务器才能运行其下一个事务。 这提供了因果链，从客户的角度确保了正确的写后读的顺序。

<img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/6ae7127b6cb9f8f30a11a5a711f04708&showdoc=.jpg" style="zoom: 67%;" />

3. Bolt 驱动
   - Bolt： 
     单机部署模式，
     直连集群下的某个特定节点
   - Bolt+routing：
     连接到整个集群上
     通过自动切换到新的节点来提高业务的服务可用性，
     确保集群负载均衡

<img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/7a87e70e17853677a5bd611547d2222a&showdoc=.jpg" style="zoom:50%;" />

4. Neo4j分片
    Neo4j分片包含由Fabric数据库管理的所有结构图（实例或数据库）。 Fabric数据库实际上是一个虚拟数据库，该数据库无法存储数据，但充当其余图形的入口点。 我们可以将其视为处理请求和连接信息的代理服务器。 它有助于分配负载并将请求发送到适当的端点。
    - 分拆成单独的数据库或实例是有意义的。其中许多与业务环境和数据大小有关。我们可以看看这里列出的几个。
      - 出于法律或隐私原因，您需要将一些数据存储在隔离的数据库中以保护或限制访问权限（即GDPR法律）。
    
      - 数据已经隔离到日常业务的单独实例中，但是您可能需要将它们编织到一个统一的图（即知识图）中。
    
      - 为了最小化各个区域中查询的延迟，可以将图形的相关段存储在靠近查询请求源的云区域中（即，欧洲托管的应用程序用于查询存储在欧洲云区域中的数据）。
    
      - 归档数据可能按日期（例如，年份）分开，但是临时报告或其他需求可能需要在这些图表上进行查询。
    
      - 图的大小变得足够大（数百亿个节点），因此有必要将数据划分为较小的图，以在较小尺寸的硬件上运行并由必要的各方访问。

<img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/6ea8da8e73e304ee207b925641612cc8&showdoc=.jpg" style="zoom: 67%;" />


#### 二、 环境搭建

1. 单节点搭建，在对应服务器上执行如下命令
```docker
docker run -d \
—publish=10474:7474 —publish=10687:7687 \
—env NEO4J_dbms_connector_bolt_advertisedaddress=localhost:10687 \
—env NEO4J_dbms_connector_http_advertisedaddress=localhost:10474 \
—volume=$HOME/neo4j/data:/data \
—name neo4j-standalone \
—env=NEO4J_ACCEPT_LICENSE_AGREEMENT=yes \
—env NEO4J_dbms_mode=SINGLE \
neo4j:4.2-enterprise
```
web访问节点信息如下

<img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/55f7dd871aac1daca03161055684eeda&showdoc=.jpg" style="zoom: 67%;" />

2. 集群搭建，在对应服务器上分别执行如下命令

```
docker run —name=neo4j-core —detach \
—network=host \
—publish=7474:7474 —publish=7687:7687 \
—publish=5000:5000 —publish=6000:6000 —publish=7000:7000 \
—hostname=10.103.12.65 \
—env NEO4J_dbms_mode=CORE \
—env NEO4J_causalclustering_expectedcoreclustersize=3 \
—env NEO4J_causalclustering_initialdiscoverymembers=10.103.12.66:5000,10.103.12.67:5000,10.103.12.65:5000 \
—env NEO4J_causalclustering_discoveryadvertisedaddress=10.103.12.65:5000 \
—env NEO4J_causalclustering_transactionadvertisedaddress=10.103.12.65:6000 \
—env NEO4J_causalclustering_raftadvertisedaddress=10.103.12.65:7000 \
—env NEO4J_dbms_connectors_defaultadvertisedaddress=10.103.12.65 \
—env NEO4J_ACCEPT_LICENSE_AGREEMENT=yes \
—env NEO4J_dbms_connector_bolt_advertisedaddress=10.103.12.65:7687 \
—env NEO4J_dbms_connector_http_advertisedaddress=10.103.12.65:7474 \
—volume=/opt/neo4j_cluster/data:/data \
—volume=/opt/neo4j_cluster/logs:/logs \
—volume=/opt/neo4j_cluster/import:/import \
—volume=/opt/neo4j_cluster/plugins:/plugins \
neo4j:4.2-enterprise

docker run —name=neo4j-core —detach \
—network=host \
—publish=7474:7474 —publish=7687:7687 \
—publish=5000:5000 —publish=6000:6000 —publish=7000:7000 \
—hostname=10.103.12.66 \
—env NEO4J_dbms_mode=CORE \
—env NEO4J_causalclustering_expectedcoreclustersize=3 \
—env NEO4J_causalclustering_initialdiscoverymembers=10.103.12.66:5000,10.103.12.67:5000,10.103.12.65:5000 \
—env NEO4J_causalclustering_discoveryadvertisedaddress=10.103.12.66:5000 \
—env NEO4J_causalclustering_transactionadvertisedaddress=10.103.12.66:6000 \
—env NEO4J_causalclustering_raftadvertisedaddress=10.103.12.66:7000 \
—env NEO4J_dbms_connectors_defaultadvertisedaddress=10.103.12.66 \
—env NEO4J_ACCEPT_LICENSE_AGREEMENT=yes \
—env NEO4J_dbms_connector_bolt_advertisedaddress=10.103.12.66:7687 \
—env NEO4J_dbms_connector_http_advertisedaddress=10.103.12.66:7474 \
—volume=/opt/neo4j_cluster/data:/data \
—volume=/opt/neo4j_cluster/logs:/logs \
—volume=/opt/neo4j_cluster/import:/import \
—volume=/opt/neo4j_cluster/plugins:/plugins \
neo4j:4.2-enterprise

docker run —name=neo4j-core —detach \
—network=host \
—publish=7474:7474 —publish=7687:7687 \
—publish=5000:5000 —publish=6000:6000 —publish=7000:7000 \
—hostname=10.103.12.67 \
—env NEO4J_dbms_mode=CORE \
—env NEO4J_causalclustering_expectedcoreclustersize=3 \
—env NEO4J_causalclustering_initialdiscoverymembers=10.103.12.66:5000,10.103.12.67:5000,10.103.12.65:5000 \
—env NEO4J_causalclustering_discoveryadvertisedaddress=10.103.12.67:5000 \
—env NEO4J_causalclustering_transactionadvertisedaddress=10.103.12.67:6000 \
—env NEO4J_causalclustering_raftadvertisedaddress=10.103.12.67:7000 \
—env NEO4J_dbms_connectors_defaultadvertisedaddress=10.103.12.67 \
—env NEO4J_ACCEPT_LICENSE_AGREEMENT=yes \
—env NEO4J_dbms_connector_bolt_advertisedaddress=10.103.12.67:7687 \
—env NEO4J_dbms_connector_http_advertisedaddress=10.103.12.67:7474 \
—volume=/opt/neo4j_cluster/data:/data \
—volume=/opt/neo4j_cluster/logs:/logs \
—volume=/opt/neo4j_cluster/import:/import \
—volume=/opt/neo4j_cluster/plugins:/plugins \
neo4j:4.2-enterprise
```
web访问集群信息如下

<img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/e4af0138382c23e5072048c71bf66a43&showdoc=.jpg" style="zoom:80%;" />

3. 在每个节点上修改配置文件，并导入对应的依赖包，这样才可以执行neo4j自带的最短路径算法包，如下所示。
```
dbms.security.procedures.unrestricted=gds.,jwt.security.
dbms.security.procedures.whitelist=gds.*
```

4. 更详细的安装内容参考《neo4j环境搭建》。

#### 三、 实验内容

1. 首先本地计算机上实现该功能。

   - 下载最新版本的neo4j即neo4j-desktop-offline-1.3.10-setup.exe，安装过程中注意以管理员权限选择for all users，否则会出现创建database时permission denied的错误。同样的，最新版本的neo4j需要依赖jdk11的版本才能正常创建数据库，jdk版本如下信息

     ```
     PS D:\桌面> java -version
     java version "11.0.9" 2020-11-20 LTS
     Java(TM) SE Runtime Environment 18.9 (build 11.0.9+7-LTS)
     Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.9+7-LTS, mixed mode)
     PS D:\桌面>
     ```

     此时可以正常创建database，如下图所示

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/e1073d89dda5239e08ed3254c9d06bd6&showdoc=.jpg" style="zoom:33%;" />

   - 启动刚刚创建的ShenZen ITS Database数据库，执行导入命令时会报错，信息如下

     ```cypher
     ERROR:Neo.ClientError.Statement.ExternalResourceFailed
     Couldn't load the external resource at: file:/C:/Users/longw/AppData/Local/Neo4j/Relate/Data/dbmss/dbms-285b49d0-d234-4f3e-bd94-bb882220607d/import/Shenzhen_Edgelist.csv
     ```

     需要将深圳市交通数据Shenzhen_Edgelist.csv上传到该目录下，此时再次执行如下命令

     ```cypher
     :auto USING PERIODIC COMMIT 2000
     LOAD CSV WITH HEADERS FROM 'file:///Shenzhen_Edgelist.csv' AS row
     WITH row.START_NODE AS START_NODE, toFloat(row.XCoord) AS XCoord,  toFloat(row.YCoord) AS YCoord
     MERGE (n:Node {NodeId: START_NODE, XCoord: XCoord, YCoord: YCoord})
     RETURN count(n);
     ```

     上述命令中，需要注意的地方有3点

     1）必须使用:auto USING PERIODIC COMMIT 2000来指定每次批量导入数据的数量，否则会出现内存溢出的错误，即因为现有的数据量非常大为100972条，所以个人电脑无法一次性导入这么多数据

     2）文件路径需要上传到指定位置，本文中对应的数据库文件存放位置为

     ```
     C:/Users/longw/AppData/Local/Neo4j/Relate/Data/dbmss/dbms-285b49d0-d234-4f3e-bd94-bb882220607d/import/
     ```

     3） 使用MERGE (...）命令而非create命令，这样在执行新增操作时，neo4j会先去查询库中是否已经存在相同的数据节点，如果有则更新（因为更新的数据相同，相当于保持不变），如果没有则新增

     由于数据量较大，命令执行的时间会根据个人电脑的性能而变化，本机大约等待30min，最终读取的数据100972条，但实际写入的数据37004条，如下图所示，可以用excel表格验证没有问题，该步骤最终完成节点数据的导入。

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/bf380445d5e80b767c8656841f452a46&showdoc=.jpg" style="zoom:33%;" />

   - 接下来导入各节点之间的关系，即道路信息数据，执行如下命令

     ```cypher
     :auto USING PERIODIC COMMIT 2000
     LOAD CSV WITH HEADERS FROM 'file:///Shenzhen_Edgelist.csv' AS row
     MATCH (n1:Node {NodeId: row.START_NODE})
     MATCH (n2:Node {NodeId: row.END_NODE})
     MERGE (n1)-[:FROM_TO {EDGE: row.EDGE,LENGTH: row.LENGTH}]->(n2)
     RETURN *;
     ```

     上述命令除了同样需要注意，由于数据量大的原因需要分批次导入，同时还需要MERGE(...)命令应该如上文所述，而不是如下

     ```cypher
     MERGE (n1)-[e:FROM_TO ]->(n2) set e={EDGE: row.EDGE,LENGTH: row.LENGTH}
     ```

     因为这会导致两个节点之间最多建立两个关系，但事实上，数据中存在两个节点有4个或更多个关系，即从某一地方出发去相邻的另一个地方，直接相通的路径不止一条，这种创建方式与数据节点的创建有所不同。

     经过一个多小时以后，最终以图形显示，效果如下

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/838d579f520283613825f638d80c953a&showdoc=.jpg" style="zoom:33%;" />

   - 使用Dijkstra最短路径算法实现计算2022节点到80240节点的最短路径

     修改示例代码

     其次需要下载依赖包，如下界面所示，否则一直报错

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/d7e9fe602863813569cd6a5b9b6d6fec&showdoc=.jpg" style="zoom: 33%;" />

     上述自动导入网速太慢，最终手动导入，如下步骤所示

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/8f2ee67b30eba25c3fee30693df7b89a&showdoc=.jpg" style="zoom: 50%;" />

     此时执行修改后的示例代码

     ```cypher
     MATCH (start:Node {NodeId: '2022'}), (end:Node {NodeId: '80240'})
     CALL gds.alpha.shortestPath.stream({
       nodeProjection: 'Node',
       relationshipProjection: {
         FROM_TO: {
           type: 'FROM_TO',
           properties: 'LENGTH',
           orientation: 'UNDIRECTED'
         }
       },
       startNode: start,
       endNode: end,
       relationshipWeightProperty: 'LENGTH'
     })
     YIELD nodeId
     RETURN gds.util.asNode(nodeId).name AS name
     ```

     获得如下错误信息，上述步骤导入的边属性均为string，需要重新导入边的关系

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/c6104b685450468203c9a585dc9d9e0e&showdoc=.jpg" style="zoom:50%;" />

     清空之前的边关系

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/121ccaae4834435ed76b52c4ccb44c0d&showdoc=.jpg" style="zoom: 50%;" />

     修改导入代码

     ```cypher
     :auto USING PERIODIC COMMIT 2000
     LOAD CSV WITH HEADERS FROM 'file:///Shenzhen_Edgelist.csv' AS row
     WITH row.START_NODE AS START_NODE, row.END_NODE AS END_NODE, row.EDGE AS EDGE, toFloat(row.LENGTH) AS unitCost
     MATCH (n1:Node {NodeId: START_NODE})
     MATCH (n2:Node {NodeId: END_NODE})
     MERGE (n1)-[:ROAD {roadId: EDGE,cost: unitCost}]->(n2)
     RETURN *
     // Limit 10 该语句表示只导入10条
     ```

     再次执行修改后的Dijkstra代码，边的方向应该是orientation: 'NATURAL'，而不是UNDIRECTED

     ```cypher
     MATCH (start:Node {NodeId: '2022'}), (end:Node {NodeId: '80240'})
     CALL gds.alpha.shortestPath.stream({
       nodeProjection: 'Node',
       relationshipProjection: {
         ROAD: {
           type: 'ROAD',
           properties: 'cost',
           orientation: 'NATURAL'
         }
       },
       startNode: start,
       endNode: end,
       relationshipWeightProperty: 'cost'
     })
     YIELD nodeId, cost
     RETURN gds.util.asNode(nodeId).NodeId AS NodeId,cost
     ```

     获得如下结果

     ```
     ╒════════╤══════════════════╕
     │"NodeId"│"cost"            │
     ╞════════╪══════════════════╡
     │"2022"  │0.0               │
     ├────────┼──────────────────┤
     │"2040"  │36.0834583745602  │
     ├────────┼──────────────────┤
     │"2147"  │105.43471581137291│
     ├────────┼──────────────────┤
     │...		...
     ├────────┼──────────────────┤
     │"80228" │48162.782547770425│
     ├────────┼──────────────────┤
     │"80240" │48175.556878898155│
     └────────┴──────────────────┘
     ```

     上述结果太长，为简化输出结果，执行如下代码

     ```cypher
     MATCH (start:Node {NodeId: '2022'}), (end:Node {NodeId: '80240'})
     CALL gds.alpha.shortestPath.stream({
       nodeProjection: 'Node',
       relationshipProjection: {
         ROAD: {
           type: 'ROAD',
           properties: 'cost',
           orientation: 'NATURAL'
         }
       },
       startNode: start,
       endNode: end,
       relationshipWeightProperty: 'cost',
       writeProperty: 'sssp'
     })
     YIELD nodeId, cost
     RETURN count(nodeId) as PathNodeNumber,Max(cost) as totalcost
     ```

     获得结果

     ```
     ╒════════════════╤══════════════════╕
     │"PathNodeNumber"│"totalCost"       │
     ╞════════════════╪══════════════════╡
     │205             │48175.556878898155│
     └────────────────┴──────────────────┘
     ```

     **即表明使用Dijkstra最短路径算法实现计算2022节点到80240节点的最短路径，总共需要经历205个节点，路程总长48175.556878898155。**

   - 使用单源最短路径算法（Single Source Shortest Path）实现计算2022节点到80240节点的最短路径

     代码如下

     ```cypher
     MATCH (n:Node {NodeId: '2022'})
     CALL gds.alpha.shortestPath.deltaStepping.stream({
       nodeProjection: 'Node',
       relationshipProjection: {
         ROAD: {
           type: 'ROAD',
           properties: 'cost'
         }
       },
       startNode: n,
       relationshipWeightProperty: 'cost',
       delta: 3.0
     })
     YIELD nodeId, distance
     RETURN collect(gds.util.asNode(nodeId).NodeId)[29920..29930] AS NodeId,collect(distance)[29920..29930] as distance
     ```

     获得结果如下

     ![](http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/707d6e6ee8f90179c7a70dbd990f5629&showdoc=.jpg)

     **可以看出来单源最短路径算法获取2022节点到80240节点的最短路径，结果和Dijkstra一致。**

   - 使用A*算法实现计算2022节点到80240节点的最短路径，代码如下

     ```cypher
     MATCH (start:Node {NodeId: '2022'}), (end:Node {NodeId: '80240'})
     CALL gds.alpha.shortestPath.astar.stream({
       nodeProjection: {
         Node: {
           properties: ['XCoord', 'YCoord']
         }
       },
       relationshipProjection: {
         ROAD: {
           type: 'ROAD',
           orientation: 'NATURAL',
           properties: 'cost'
         }
       },
       startNode: start,
       endNode: end,
       relationshipWeightProperty: 'cost',
       propertyKeyLat: 'XCoord',
       propertyKeyLon: 'YCoord'
     })
     YIELD nodeId, cost
     RETURN gds.util.asNode(nodeId).NodeId AS NodeId, cost
     ```

     首次执行发现如下错误

     ![](http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/df6744ead344ed7a488fd0265749da63&showdoc=.jpg)

     证明是导入节点数据时，属性值默认为String类型，需要修改其为数字类型例如float，执行如下命令转换

     ```
     MATCH (n:Node)
     SET n.XCoord = toFloat(n.XCoord)
     return count(n)
     MATCH (n:Node)
     SET n.YCoord = toFloat(n.YCoord)
     return count(n)
     ```

     再次执行程序得到结果如下

     ```
     ╒════════╤═══════╕
     │"NodeId"│"cost" │
     ╞════════╪═══════╡
     │"2022"  │0.0    │
     ├────────┼───────┤
     │"2003"  │72.0   │
     ├────────┼───────┤
     │"1923"  │317.0  │
     ├────────┼───────┤
     │"1921"  │352.0  │
     ├────────┼───────┤
     │"1941"  │360.0  │
     ├────────┼───────┤
     │"1929"  │372.0  │
     ├────────┼───────┤
     │"1837"  │539.0  │
     ├────────┼───────┤
     |........|.......|
     ├────────┼───────┤
     │"78622" │53244.0│
     ├────────┼───────┤
     │"78652" │53254.0│
     ├────────┼───────┤
     │"79340" │53672.0│
     ├────────┼───────┤
     │"79916" │53911.0│
     ├────────┼───────┤
     │"80154" │54059.0│
     ├────────┼───────┤
     │"80228" │54128.0│
     ├────────┼───────┤
     │"80240" │54140.0│
     └────────┴───────┘
     
     ```

     修改返回结果

     ```
     RETURN count(nodeId) as PathNodeNumber,Max(cost) as totalcost
     ```

     获得总的结果

     ```
     ╒════════════════╤═══════════╕
     │"PathNodeNumber"│"totalcost"│
     ╞════════════════╪═══════════╡
     │252             │54140.0    │
     └────────────────┴───────────┘
     ```

     **即表明使用A*算法实现计算2022节点到80240节点的最短路径，总共需要经历252个节点，路程总长54140.0，这显然比Dijkstra算法的最短路径要大。**

   - 使用Yen’s K-shortest paths算法实现计算2022节点到80240节点的最短路径，代码如下

     ```cypher
     MATCH (start:Node {NodeId: '2022'}), (end:Node {NodeId: '80240'})
     CALL gds.alpha.kShortestPaths.stream({
       nodeProjection: 'Node',
       relationshipProjection: {
         ROAD: {
           type: 'ROAD',
           properties: 'cost'
         }
       },
       startNode: start,
       endNode: end,
       k: 3,
       relationshipWeightProperty: 'cost'
     })
     YIELD index, nodeIds, costs
     RETURN [node IN gds.util.asNodes(nodeIds) | node.NodeId] AS places,
            costs,
            reduce(acc = 0.0, cost IN costs | acc + cost) AS totalCost
     ```

     返回结果如下

     ```
     ╒══════════════════════════════╤══════════════════════════════╤═══════════╕
     │"places"                      │"costs"                       │"totalCost"│
     ╞══════════════════════════════╪══════════════════════════════╪═══════════╡
     │["2022","2040","2147","2244","│[36.0,69.0,72.0,43.0,113.0,181│48175.556878898155
     │2289","2381","2493","2599","26│.0,191.0,117.0,82.0,170.0,191.│           │
     │44","2676","2780","2817","3030│0,402.0,123.0,12.0,56.0,16.0,1│           │
     │","3114","3120","3172","3188",│2.0,108.0,199.0,21.0,436.0,227│           │
     │"3200","3282","3450","3496","4│.0,265.0,219.0,16.0,292.0,24.0│           │
     │184","4498","4938","5470","557│,154.0,121.0,132.0,333.0,10.0,│           │
     │6","5556","5659","6064","6078"│36.0,119.0,166.0,150.0,154.0,9│           │
     │,"6094","7243","7275","7429","│2.0,476.0,519.0,361.0,12.0,436│           │
     │7993","9002","8999","9603","10│.0,226.0,256.0,104.0,137.0,31.│           │
     │039","12113","14093","14695","│0,48.0,17.0,551.0,66.0,15.0,30│           │
     │14690","14663","14644","14777"│5.0,85.0,41.0,98.0,25.0,60.0,3│           │
     │,"15121","15865","16051","1637│51.0,169.0,89.0,16.0,20.0,347.│           │
     │3","16316","19978","20398","94│0,103.0,43.0,55.0,360.0,125.0,│           │
     │042","20569","20570","20656","│65.0,354.0,161.0,257.0,115.0,5│           │
     │20529","20757","21341","23579"│4.0,494.0,207.0,62.0,151.0,151│           │
     │,"24775","25391","25397","2557│.0,67.0,166.0,619.0,314.0,1346│           │
     │5","27634","28310","28590","28│.0,1384.0,225.0,1311.0,22.0,12│           │
     │990","30994","31108","31278","│4.0,55.0,197.0,129.0,156.0,108│           │
     │31283","31059","30496","30491"│.0,15.0,246.0,174.0,340.0,1199│           │
     │,"30492","31208","31596","3168│.0,187.0,175.0,392.0,93.0,91.0│           │
     │0","31866","32034","32200","32│,222.0,918.0,95.0,56.0,54.0,15│           │
     │506","31103","29894","31206","│8.0,64.0,97.0,352.0,418.0,56.0│           │
     │33034","33256","34792","34806"│,120.0,82.0,653.0,38.0,136.0,7│           │
     │,"34880","34936","35162","3529│3.0,473.0,10.0,515.0,206.0,506│           │
     │0","35598","35584","35620","35│.0,301.0,744.0,234.0,22.0,11.0│           │
     │968","36294","36169","37942","│,86.0,309.0,30.0,692.0,130.0,1│           │
     │38214","38276","38524","38568"│76.0,29.0,11.0,26.0,431.0,43.0│           │
     │,"38561","38778","39955","3961│,169.0,73.0,237.0,263.0,299.0,│           │
     │9","39622","39708","40375","40│548.0,21.0,427.0,227.0,230.0,1│           │
     │614","40508","39888","39588","│25.0,218.0,102.0,41.0,60.0,81.│           │
     │39709","39883","40003","41701"│0,159.0,13.0,278.0,112.0,677.0│           │
     │,"41681","42169","42425","4368│,268.0,164.0,434.0,270.0,246.0│           │
     │3","43676","44888","45420","46│,468.0,723.0,283.0,480.0,69.0,│           │
     │310","46912","48634","49154","│168.0,265.0,122.0,209.0,138.0,│           │
     │49220","49258","49130","49803"│130.0,153.0,192.0,451.0,529.0,│           │
     │,"49891","51199","51335","5154│175.0,53.0,60.0,526.0,982.0,87│           │
     │9","51569","51581","51621","52│3.0,8.0,342.0,96.0,790.0,15.0,│           │
     │225","52305","52669","52966","│854.0,152.0,95.0,112.0,120.0,6│           │
     │53257","53799","54405","55237"│6.0,17.0,348.0,16.0,465.0,37.0│           │
     │,"55257","55397","55403","5544│,16.0,258.0,521.0,239.0,148.0,│           │
     │9","55497","96955","96969","96│69.0,12.0]                    │           │
     │971","96977","55840","55839","│                              │           │
     │55857","56303","56485","57659"│                              │           │
     │,"58045","58277","58655","5874│                              │           │
     │5","58827","58987","59575","59│                              │           │
     │823","60327","60373","60439","│                              │           │
     │60553","60617","60817","61041"│                              │           │
     │,"61279","61589","61843","6259│                              │           │
     │3","63397","63601","63661","63│                              │           │
     │759","64491","66291","67923","│                              │           │
     │67957","68763","69019","71433"│                              │           │
     │,"71471","73611","74033","7423│                              │           │
     │1","74561","74939","75159","75│                              │           │
     │217","76199","76233","77401","│                              │           │
     │77490","77528","78116","79340"│                              │           │
     │,"79916","80154","80228","8024│                              │           │
     │0"]                           │                              │           │
     ├──────────────────────────────┼──────────────────────────────┼───────────┤
     │["2022","2040","2147","2244","│[36.0,69.0,72.0,43.0,113.0,181│48175.755767037466
     │2289","2381","2493","2599","26│.0,191.0,117.0,82.0,170.0,191.│           │
     │44","2676","2780","2817","3030│0,402.0,123.0,12.0,56.0,16.0,1│           │
     │","3114","3120","3172","3188",│2.0,108.0,199.0,21.0,436.0,227│           │
     │"3200","3282","3450","3496","4│.0,265.0,219.0,16.0,292.0,24.0│           │
     │184","4498","4938","5470","557│,154.0,121.0,132.0,333.0,10.0,│           │
     │6","5556","5659","6064","6078"│36.0,119.0,166.0,150.0,154.0,9│           │
     │,"6094","7243","7275","7429","│2.0,476.0,519.0,361.0,12.0,436│           │
     │7993","9002","8999","9603","10│.0,226.0,256.0,104.0,137.0,31.│           │
     │039","12113","14093","14695","│0,48.0,17.0,551.0,66.0,15.0,30│           │
     │14690","14663","14644","14777"│5.0,85.0,41.0,98.0,25.0,60.0,3│           │
     │,"15121","15865","16051","1637│51.0,169.0,89.0,20.0,16.0,347.│           │
     │3","16316","19978","20398","94│0,103.0,43.0,55.0,360.0,125.0,│           │
     │042","20569","20570","20656","│65.0,354.0,161.0,257.0,115.0,5│           │
     │20529","20757","21341","23579"│4.0,494.0,207.0,62.0,151.0,151│           │
     │,"24775","25391","25573","2557│.0,67.0,166.0,619.0,314.0,1346│           │
     │5","27634","28310","28590","28│.0,1384.0,225.0,1311.0,22.0,12│           │
     │990","30994","31108","31278","│4.0,55.0,197.0,129.0,156.0,108│           │
     │31283","31059","30496","30491"│.0,15.0,246.0,174.0,340.0,1199│           │
     │,"30492","31208","31596","3168│.0,187.0,175.0,392.0,93.0,91.0│           │
     │0","31866","32034","32200","32│,222.0,918.0,95.0,56.0,54.0,15│           │
     │506","31103","29894","31206","│8.0,64.0,97.0,352.0,418.0,56.0│           │
     │33034","33256","34792","34806"│,120.0,82.0,653.0,38.0,136.0,7│           │
     │,"34880","34936","35162","3529│3.0,473.0,10.0,515.0,206.0,506│           │
     │0","35598","35584","35620","35│.0,301.0,744.0,234.0,22.0,11.0│           │
     │968","36294","36169","37942","│,86.0,309.0,30.0,692.0,130.0,1│           │
     │38214","38276","38524","38568"│76.0,29.0,11.0,26.0,431.0,43.0│           │
     │,"38561","38778","39955","3961│,169.0,73.0,237.0,263.0,299.0,│           │
     │9","39622","39708","40375","40│548.0,21.0,427.0,227.0,230.0,1│           │
     │614","40508","39888","39588","│25.0,218.0,102.0,41.0,60.0,81.│           │
     │39709","39883","40003","41701"│0,159.0,13.0,278.0,112.0,677.0│           │
     │,"41681","42169","42425","4368│,268.0,164.0,434.0,270.0,246.0│           │
     │3","43676","44888","45420","46│,468.0,723.0,283.0,480.0,69.0,│           │
     │310","46912","48634","49154","│168.0,265.0,122.0,209.0,138.0,│           │
     │49220","49258","49130","49803"│130.0,153.0,192.0,451.0,529.0,│           │
     │,"49891","51199","51335","5154│175.0,53.0,60.0,526.0,982.0,87│           │
     │9","51569","51581","51621","52│3.0,8.0,342.0,96.0,790.0,15.0,│           │
     │225","52305","52669","52966","│854.0,152.0,95.0,112.0,120.0,6│           │
     │53257","53799","54405","55237"│6.0,17.0,348.0,16.0,465.0,37.0│           │
     │,"55257","55397","55403","5544│,16.0,258.0,521.0,239.0,148.0,│           │
     │9","55497","96955","96969","96│69.0,12.0]                    │           │
     │971","96977","55840","55839","│                              │           │
     │55857","56303","56485","57659"│                              │           │
     │,"58045","58277","58655","5874│                              │           │
     │5","58827","58987","59575","59│                              │           │
     │823","60327","60373","60439","│                              │           │
     │60553","60617","60817","61041"│                              │           │
     │,"61279","61589","61843","6259│                              │           │
     │3","63397","63601","63661","63│                              │           │
     │759","64491","66291","67923","│                              │           │
     │67957","68763","69019","71433"│                              │           │
     │,"71471","73611","74033","7423│                              │           │
     │1","74561","74939","75159","75│                              │           │
     │217","76199","76233","77401","│                              │           │
     │77490","77528","78116","79340"│                              │           │
     │,"79916","80154","80228","8024│                              │           │
     │0"]                           │                              │           │
     ├──────────────────────────────┼──────────────────────────────┼───────────┤
     │["2022","2040","2147","2244","│[36.0,69.0,72.0,43.0,113.0,181│48176.10356362162
     │2289","2381","2493","2599","26│.0,191.0,117.0,82.0,170.0,191.│           │
     │44","2676","2780","2817","3030│0,402.0,123.0,12.0,56.0,16.0,1│           │
     │","3114","3120","3172","3188",│2.0,108.0,199.0,21.0,436.0,227│           │
     │"3200","3282","3450","3496","4│.0,265.0,219.0,16.0,292.0,24.0│           │
     │184","4498","4938","5470","557│,154.0,121.0,132.0,333.0,10.0,│           │
     │6","5556","5659","6064","6078"│36.0,119.0,166.0,150.0,154.0,9│           │
     │,"6094","7243","7275","7429","│2.0,476.0,519.0,361.0,12.0,436│           │
     │7993","9002","8999","9603","10│.0,226.0,256.0,104.0,137.0,31.│           │
     │039","12113","14093","14695","│0,48.0,17.0,551.0,66.0,15.0,30│           │
     │14690","14663","14644","14777"│5.0,85.0,41.0,98.0,25.0,60.0,3│           │
     │,"15121","15865","16051","1637│51.0,169.0,89.0,16.0,20.0,347.│           │
     │3","16316","19978","20398","94│0,103.0,43.0,55.0,360.0,125.0,│           │
     │042","20569","20570","20656","│65.0,354.0,161.0,257.0,115.0,5│           │
     │20529","20757","21341","23579"│4.0,494.0,207.0,62.0,151.0,151│           │
     │,"24775","25391","25397","2557│.0,67.0,166.0,619.0,314.0,1346│           │
     │5","27634","28310","28590","28│.0,1384.0,225.0,1311.0,22.0,12│           │
     │990","30994","31108","31278","│4.0,55.0,197.0,129.0,156.0,108│           │
     │31283","31059","30496","30491"│.0,15.0,246.0,174.0,340.0,1199│           │
     │,"30492","31208","31596","3168│.0,187.0,175.0,392.0,93.0,91.0│           │
     │0","31866","32034","32200","32│,222.0,918.0,95.0,56.0,54.0,15│           │
     │506","31103","29894","31206","│8.0,64.0,97.0,352.0,418.0,56.0│           │
     │33034","33256","34792","34806"│,120.0,82.0,653.0,38.0,136.0,7│           │
     │,"34880","34936","35162","3529│3.0,473.0,10.0,515.0,206.0,506│           │
     │0","35598","35584","35620","35│.0,301.0,744.0,234.0,22.0,11.0│           │
     │968","36294","36169","37942","│,86.0,309.0,30.0,692.0,130.0,1│           │
     │38214","38276","38524","38568"│76.0,29.0,11.0,26.0,431.0,43.0│           │
     │,"38561","38778","39955","3961│,169.0,73.0,237.0,263.0,299.0,│           │
     │9","39622","39708","40375","40│548.0,21.0,427.0,227.0,199.0,6│           │
     │614","40508","39888","39588","│7.0,95.0,156.0,57.0,102.0,41.0│           │
     │39709","39883","40003","41701"│,60.0,81.0,159.0,13.0,278.0,11│           │
     │,"41681","42169","42425","4368│2.0,677.0,268.0,164.0,434.0,27│           │
     │3","43676","44888","45420","46│0.0,246.0,468.0,723.0,283.0,48│           │
     │310","46912","48634","49154","│0.0,69.0,168.0,265.0,122.0,209│           │
     │49220","49258","49130","49803"│.0,138.0,130.0,153.0,192.0,451│           │
     │,"49891","51199","51335","5154│.0,529.0,175.0,53.0,60.0,526.0│           │
     │9","51569","51581","51621","52│,982.0,873.0,8.0,342.0,96.0,79│           │
     │225","52305","52669","52966","│0.0,15.0,854.0,152.0,95.0,112.│           │
     │53257","53799","54405","55237"│0,120.0,66.0,17.0,348.0,16.0,4│           │
     │,"55257","55397","55403","5541│65.0,37.0,16.0,258.0,521.0,239│           │
     │9","55472","55471","96943","96│.0,148.0,69.0,12.0]           │           │
     │955","96969","96971","96977","│                              │           │
     │55840","55839","55857","56303"│                              │           │
     │,"56485","57659","58045","5827│                              │           │
     │7","58655","58745","58827","58│                              │           │
     │987","59575","59823","60327","│                              │           │
     │60373","60439","60553","60617"│                              │           │
     │,"60817","61041","61279","6158│                              │           │
     │9","61843","62593","63397","63│                              │           │
     │601","63661","63759","64491","│                              │           │
     │66291","67923","67957","68763"│                              │           │
     │,"69019","71433","71471","7361│                              │           │
     │1","74033","74231","74561","74│                              │           │
     │939","75159","75217","76199","│                              │           │
     │76233","77401","77490","77528"│                              │           │
     ```

     修改返回参数如下：

     ```
     YIELD path
     RETURN path
     LIMIT 1
     ```

     获得最终的最短路径图形如图所示

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/de12c333d9974f76b5dd6a97d95ee9be&showdoc=.jpg" style="zoom:33%;" />

     **即表明使用Yen’s K-shortest paths算法实现计算2022节点到80240节点的最短的三个路径，路程总长分别是48175.556878898155、48175.755767037466、48176.10356362162。最短路径与Dijkstra算法结果相符。**

   - 最终数据集的图形展现如下

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/9770ad60d45c761745763a70960d590b&showdoc=.jpg" style="zoom:33%;" />

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/7ecb555a296fbdcd63fc8731963af93b&showdoc=.jpg" style="zoom: 33%;" />

2. 服务器单机版实现上述功能。

   - 搭建环境，导入数据以后，显示结果（300个节点，723条边）如下所示

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/196d24eddb7a64e5fe29a641a3ebbbe8&showdoc=.jpg" style="zoom: 67%;" />

   - 四种算法测试结果如下

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/f24363daa097b632c896eea24d08b632&showdoc=.jpg" style="zoom:50%;" />

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/de8038e06a6ab064c1191849e496645a&showdoc=.jpg" style="zoom: 50%;" />

   - 其中K最短路径的图形化显示

     <img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/31218d5faa193b3845d798602021d4f4&showdoc=.jpg" style="zoom:67%;" />

3. 同样的，集群版实现上述功能，步骤一致，不再赘述。算法结果如下

<img src="http://219.223.175.37:4999/server/index.php?s=/api/attachment/visitFile/sign/092cdc45ac0a962dea76255ce0e8e5b7&showdoc=.jpg" style="zoom: 50%;" />

4. 总结上述过程，测试在不同环境下，数据的导入时间以及算法时间，并整理成表格和曲线，完成单机版和集群版的性能对比。

   - Standalone Neo4j 导入节点时间表格、曲线

     | lines    | nodes           | properties | time       |
     | :------- | :-------------- | :--------- | ---------- |
     |9999|3705|11115|11873|
     |20000|7272|21816|48362|
     |30000|10847|32541|176988|
     |40000|14352|43056|388154|
     |50000|17968|53904|563326|
     |60000|21601|64803|728089|
     |70000|25341|76023|931429|
     |80000|28887|86661|1156114|
     |90000|32368|97104|1410975|
     |100972|37004|111012|1733342|
     
     <img src="C:\Users\longw\AppData\Roaming\Typora\typora-user-images\image-20210103222454072.png" alt="image-20210103222454072" style="zoom:80%;" />
     
   - Standalone Neo4j 导入边关系时间表格、曲线

     | lines  | relationships | properties | time    |
     | :----- | :------------ | :--------- | ------- |
     | 9999   | 9696          | 19392      | 294559  |
     | 20000  | 19524         | 39048      | 622037  |
     | 30000  | 29428         | 58856      | 1088091 |
     | 40000  | 39316         | 78632      | 1589692 |
     | 50000  | 49252         | 98504      | 2173840 |
     | 60000  | 59096         | 118192     | 2801079 |
     | 70000  | 69070         | 138140     | 3469185 |
     | 80000  | 78997         | 157994     | 4209615 |
     | 90000  | 89010         | 178020     | 5013289 |
     | 100972 | 100972        | 201944     | 6070004 |

     <img src="C:\Users\longw\AppData\Roaming\Typora\typora-user-images\image-20210103222614676.png" alt="image-20210103222614676" style="zoom:80%;" />

   - Cluster Neo4j 导入节点时间表格、曲线

     | lines  | nodes | properties | time    |
     | :----- | :---- | :--------- | ------- |
     | 9999   | 3705  | 11115      | 14611   |
     | 40000  | 14352 | 43056      | 230659  |
     | 70000  | 25341 | 76023      | 716571  |
     | 100972 | 37004 | 111012     | 1305770 |

     <img src="C:\Users\longw\AppData\Roaming\Typora\typora-user-images\image-20210103222530176.png" alt="image-20210103222530176" style="zoom:80%;" />

   - Cluster Neo4j 导入边关系时间表格、曲线

     | lines  | relationships | properties | time    |
     | :----- | :------------ | :--------- | ------- |
     | 9999   | 9696          | 19392      | 271218  |
     | 40000  | 39316         | 78632      | 1477213 |
     | 70000  | 69070         | 138140     | 3237564 |
     | 100972 | 100972        | 201944     | 5705875 |

     <img src="C:\Users\longw\AppData\Roaming\Typora\typora-user-images\image-20210103220236597.png" alt="image-20210103220236597" style="zoom:80%;" />

   - Standalone Neo4j 算法时间表格、曲线

     | relationships | nodes  | Dijkstra | A*   | K-shortest |
     | :------------ | :----- | :------- | ---- | ---------- |
     | 9696          | 3705   | 20.8     | 28.3 | 20.5       |
     | 19524         | 7272   | 33       | 49.2 | 31.2       |
     | 29428         | 10847  | 41       | 63   | 33.2       |
     | 39316         | 14352  | 47       | 65.4 | 35.8       |
     | 49252         | 17968  | 54       | 72.4 | 42         |
     | 59096         | 21601  | 58       | 75.5 | 46.2       |
     | 69070         | 25341  | 63.5     | 86.4 | 53.2       |
     | 78997         | 28887  | 69.8     | 83.8 | 54.6       |
     | 89,010        | 32,368 | 70       | 93   | 69.4       |
     | 100,972       | 37,004 | 72.6     | 96.8 | 74.6       |

     <img src="C:\Users\longw\AppData\Roaming\Typora\typora-user-images\image-20210103220246433.png" alt="image-20210103220246433" style="zoom: 67%;" />

   - Cluster Neo4j 算法时间表格、曲线

     | relationships | nodes  | Dijkstra | A*    | K-shortest |
     | :------------ | :----- | :------- | ----- | ---------- |
     | 9696          | 3705   | 23.2     | 26.5  | 22.2       |
     | 39316         | 14352  | 38       | 52.6  | 37         |
     | 69070         | 25341  | 57.4     | 70.8  | 49.6       |
     | 100,972       | 37,004 | 74.8     | 122.5 | 84.6       |

     <img src="C:\Users\longw\AppData\Roaming\Typora\typora-user-images\image-20210103220309330.png" alt="image-20210103220309330" style="zoom:67%;" />

#### 四、 实验结果

1. 以上采用的四种方法：Dijkstra([Shortest Path](https://neo4j.com/docs/graph-data-science/current/alpha-algorithms/shortest-path/))、[Single Source Shortest Path](https://neo4j.com/docs/graph-data-science/current/alpha-algorithms/single-source-shortest-path/)、[A*](https://neo4j.com/docs/graph-data-science/current/alpha-algorithms/a_star/)、[Yen’s K-Shortest Paths](https://neo4j.com/docs/graph-data-science/current/alpha-algorithms/yen-s-k-shortest-path/)

   其优缺点对比如下，

   Dijkstra：优点是运行速度快，缺点是一次只能算出两个点之间的一条最短路径；

   Single Source Shortest Path：能算出源点到其它点的最短路径，运行速度也较快；

   A*：在搜索路径时的速度比Dijkstra 算法的搜索速度更快，算法所用时间较少，但算法找到的路径，可能不是最好的；

   Yen’s K-Shortest Paths：能够求得多条最短路径，但所求的 K 条最短路径相似度较高且算法用时相对较长。

2. 对于最短路径算法的改进方向的讨论：

   - 对数据结构进行改进。例如可采用二叉堆、桶结构等数据结构存储算法在搜索过程中产生的数据；
   - 减小搜索范围。例如与 Dijkstra 相比，A *算法以起始点和终止点连线的方向为搜索方向有效地减小了搜索范围，搜索过程中所需遍历的节点数目也减少了；
   - 对道路网络进行优化。例如对道路网络采取道路等级分层的处理方式；
   - 对排序方法进行改进。如用二叉堆来优化存储 A 星算法中 OPEN 表中的数值，就可以保证OPEN 表中的元素是按照估价值由小到大顺序排列的，首个元素就是最小值，从而不需要采用像“快速排序”“冒泡排序”之类的算法将元素从小到大进行排序。