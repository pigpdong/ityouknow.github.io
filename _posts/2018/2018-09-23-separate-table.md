---
layout:     post
title:    如何定制自己的分库分表中间件
no-post-nav: true
category: other
tags: [arch]
excerpt: 分库分表
---

## 前言


  一般来说，影响数据库最大的性能问题有两个，一个是对数据库的操作，一个是数据库中的数据太大，对于前者我们可以借助缓存来减少一部分读操作，针对一些复杂的报表分析和搜索可以交给hadoop和elasticsearch,对于后者，我们就只能分库分表，读写分离。

  互联网行业随着业务的复杂化，大多数应用都会经历数据的垂直分区，一个复杂的流程会按照领域拆分成不同的服务，每个服务中心都拥有自己独立的数据库，拆分后服务共享，业务更清晰，系统也更容易扩展，同时减少了单库数据库连接数的压力，也在一定程度上提高了单表大数据量下索引查询的效率，当然业务隔离，也可以避免一个业务把数据库拖死导致所有业务都死掉，我们将这种按照业务维度，把一个库拆分为多个不同的库的方式叫做垂直拆分。

  垂直拆分也包含针对长表(属性很多)做冷热分离的拆分，例如，在商品系统设计中，一个商品的生产商，供销商，以及特有属性，这些字段变化频率低，查询次数多，叫做冷数据，而商品的份额，关注量等类似的统计信息变化频率较高，叫做活跃数据或者热数据，在MYSQL中，冷数据查询多更新少，适合用MyISAM存储引擎，而热数据更新比较频繁适合用InoDB，这也是垂直拆分的一种.

  当单表数据量随着业务发展继续膨胀，在MYSQL中当数据量达到千万级时，就需要考虑进行水平拆分了，这样数据就分散到不同的表上，单表的索引大小得到控制，可以提升查询性能，当数据库的实例吞吐量达到性能瓶颈后，我们需要水平扩展数据库的实例，让多个数据库实例分担请求，这种根据分片算法，将一个库拆分成多个一样结构的库，将多个表拆分成多个结构相同的表就叫做水平拆分。
         
        
![](https://pigpdong.github.io/assets/images/2018/split/his.png)

  数据拆也有很多缺点，数据分散，数据库的Join操作变得更加复杂，分片后数据的事务一致性很难保证，同时数据的扩容和维护难度增加，拆分规则也可能导致某个业务需要同时查询所有的表然后进行聚合，如果需要排序和函数计算则更加复杂，所以不到万不得已可以先不必拆分。

  根据分库分表方案中实施切片逻辑的层次不同，我们将分库分表的实现方案分成以下3种
   
### 在应用层直接分片
  这种方式将分片规则直接放在应用层，虽然侵入了业务，开发人员不仅既需要实现业务逻辑也需要实现分库分表的配置的开发，但是实现起来简单，适合快速上线，通过编码方式也更容易实现跨表遍历的情况，后期故障定位也更容易定位，大多数公司都会在业务早期采用此种方式过渡，后期分表需求增多，则会寻求中间件来解决，以下代码为铜板街早期订单表在DAO层将分片信息以参数形式传到mybatis的mapper文件中的实现方案。
          
```
@Override
public OrderDO findByPrimaryKey(String orderNo) {
    
    Assert.hasLength(orderNo, "订单号不能为空");
    
    Map<String, Object> map = new HashMap<String, Object>(3);
    map.put("tableSuffix", orderRouter.routeTableByOrderNo(orderNo));
    map.put("dbSuffix", orderRouter.routeDbByOrderNo(orderNo));
    map.put("orderNo", orderNo);
    
    Object obj = getSqlSession().selectOne("NEW_ORDER.FIND_BY_PRIMARYKEY", map);
    if (obj != null && obj instanceof OrderDO) {
        return (OrderDO) obj;
    }
    return null;
}
	
```

### 在ORM层框架内分片
  这种方式通过扩展第三方ORM框架，将分片规则和路由机制嵌入到ORM框架中，如hibernate和mybatis，也可以基于spring jdbctemplate来实现，目前实现较少。

### 客户端定制JDBC协议
  这种方式比较常见,对业务侵入低，通过定制JDBC协议，针对业务逻辑层提供与JDBC一致的接口，让开发人员不必要关心分库分表的具体实现，分库分表在JDBC内部搞定，对业务层透明。目前流行的ShardingJDBC，TDDL便采用了这种方案。这种方案需要开发人员熟悉JDBC协议，研发成本较低，适合大多数中型企业

### 代理分片
  此种分片方式，是在应用层和数据库层增加一个代理，把分片的路由规则配置在代理层，代理层提供与JDBC兼容的接口给应用层，开发人员不用关心分片逻辑实现，只需要在代理层配置即可。增加代理，需要解决代理的单点问题增加硬件成本，同时所有的数据库请求增加了一层网络传输影响性能，当然维护也需要更资深的专家，目前采用这种方式的框架有cobar和mycat.


![](https://pigpdong.github.io/assets/images/2018/split/type.png)


##  切片算法
### 选取分片字段
  分片后，如果查询的标准是根据分片的字段，则根据切片算法，可以路由到对应的表进行查询，如果查询条件中不包含分片的字段，则需要将所有的表都扫描一遍然后在进行合并，所以我们在设计分片的时候我们一般会选择一个字段作为分片的依据，后续的分片算法会基于该字段的值进行，例如根据创建时间字段取对应的年份，每年一张表，取电话号码里面的最后一位进行分表等，这个分片的字段我们一般会根据查询频率来选择，例如在互金行业，用户的持仓数据，我们一般选择用户id进行分表，而用户的交易订单也会选择用户id进行分表，但是如果我们要查询某个供应商下在某段时间内的所有订单就需要遍历所有的表，所以有时候我们可能会需要根据多个字段同时进行分片，数据进行冗余存储。

### 分片算法
  分片规则必须保证路由到每张物理表的数据量大致相同，不然上线后某一张表的数据膨胀的特别快，二其他表数据相对很少，这样就失去了分表的意义，后期数据迁移也有很高的复杂度，通过分片字段定位到对应的数据库和物理表（我们将分表后在数据库上物理存储的表名叫物理表，如trade_order_01,trade_order_02,将未进行切分前的表名称作逻辑表如trade_order），有哪些算法呢？根据字段属性不同，可以有以下分类
     
- 按日期 如年份，季度，月进行分表，这种维度分表需要注意在边缘点的垮表查询，例如如果是根据创建时间按月进行分片，则查询最近3天的数据可能需要遍历两张表，这种业务比较常见，但是放中间件层处理起来就比较复杂，还是需要应用层特殊处理。
- 哈希 ，这种是目前比较常用的算法，但是这里谨慎推荐，因为他再后期扩容的时候是件很头痛的事情，例如根据用户ID对64取模，得到一个0到63的数字，这里最多可以切分64张表{0，1，2，3，4... 63}，前期可能用不到这么多，我们可以借助一致性哈希的算法，每4个连续的数字分成放到一张表里。例如 0，1，2，3 分到00这张表，4，5，6，7分到04这张表，用算法表示
floor(userID % 64 / 4) * 4 假设 floor为取整的效果

![](https://pigpdong.github.io/assets/images/2018/split/hash.png)

- 按照一致性哈希算法，当需要进行扩容一倍时需要迁移一半的数据量，虽然不至于迁移所有的数据，如果没有工具也是需要很大的工作量。下图中根据分表字段对16取余后分到4张表中，后面如果要扩容则需要迁移一半的数据。

![](https://pigpdong.github.io/assets/images/2018/split/move.png)

- 截取 这种算法将字段中某一段位置的数据截取出来，例如取电话号码里面的尾数，这种方式实现起来简单，但在上线前一定要预测最终的数据分布是否会平均。比如地域，姓氏等。


### 特别注意点
  分库分表算法需要保证不相干,上线前一定要用线上数据做预测，例如分库算法用 用户id%64 分64个库 分表算法也用 用户id%64 分64张表，总计 64 * 64 张表，最终数据都将落在
以下 64张表中 00库00表，01库01表... 63库63表， 其他 64 * 63张表则没有数据。这里可以推荐一个算法，分库用 用户ID/64 % 64  分表用 用户ID%64 测试1亿笔用户id发现分布均匀，在分库分表前需要规划好业务增长量，以预备多大的空间，计算分表后可以支持按某种数据增长速度可以维持多久。


## 如何实现客户端分片

  客户端需要定制JDBC协议，在拿到待执行的sql后，解析sql,根据查询条件判断是否存在分片字段，如果存在,再根据分片算法获取到对应的数据库实例和物理表名，重写sql,然后找到对应的数据库datasource并获取物理连接，执行sql,将结果集进行合并筛选后返回，如果没有分片字段，则需要查询所有的表，注意，即使存在分片字段，但是分片字段在一个范围内，可能也需要查询多个表，针对select以外的sql如果没有传分片字段建议直接抛出异常。

![](https://pigpdong.github.io/assets/images/2018/split/flow.png)

### JDBC协议
  我们先回顾下一个完整的通过JDBC执行一条查询sql的流程，其实druid也是在jdbc上做增强来做监控的，所以我们也可以适当参考druid的实现
               
```
@Test
public void testQ() throws SQLException,NamingException{
    Context context = new InitialContext();
    DataSource dataSource = (DataSource)context.lookup("java:comp/env/jdbc/myDataSource");
    Connection connection = dataSource.getConnection();
    PreparedStatement preparedStatement = connection.prepareStatement("select * from busi_order where id = ?");
    preparedStatement.setString(1,"1");
    ResultSet resultSet = preparedStatement.executeQuery();
    while(resultSet.next()){
       String orderNo =  resultSet.getString("order_no");
       System.out.println(orderNo);
    }

    preparedStatement.close();
    connection.close();
}

```


- datasource 需要提供根据分片结果获取对应的数据源的datasource,返回的connection应该是定制后的connection,因为在执行sql前还无法知道是哪个库哪个表，所以只能返回一个逻辑意义上德connection
- connection 定制的connection，需要实现获取statement，执行sql,关闭，设置auto commit等方法，在执行sql和获取statement的时候应该 进行路由找到物理表后 在执行操作，由于该connection是逻辑意义上的，针对关闭，设置auto commit等需要将关联的多个物理connection一起设置
- statement 定制化的statement，由于和connection都提供了执行sql的方法，所以我们可以将执行sql都交给一个执行器执行，connection和statement中都通过这个执行器执行sql,在执行器重，解析sql，获取物理连接，结果集处理等操作
- resultset resultset是一个迭代器，遍历的时候数据源由数据库提供，但我们在某些有排序和limit的查询中，可能迭代器直接在内存中遍历数据

### SQL解析
  sql解析一般借助druid框架里面的SQLStatementParser类，解析好的数据都在SQLStatement 中，当然有条件的可以自己研究SQL解析，不过可能工作量有点大。
       
- 解析出sql类型，目前生成环境主要还是4中sql类型： SELECT DELETE UPDATE INSERT ，目前是直接解析sql 是否以上面4个单词开头即可，不区分大小写
- insert类型需要区分，是否是批量插入，解析出insert插入的列的字段名称和对应的值，如果插入的列中不包含分片字段，将无法定位到具体插入到哪个物理表，此时应该抛出异常
- delete和update 都需要解析where后的条件，根据查询条件里的字段，尝试路由到指定的物理表，注意此时可能会出现where条件里面 分片字段可能是一个范围，或者分片字段存在多个限制
- select和其他类型不同的是，返回结果是一个list,而其他三种sql直接返回状态和影响行数即可，同时select可能出现关联查询，以及针对查询结果进行筛选的操作，例如where条件中除了普通的判断表达式，还可能存在 limit，order by ,group by ,having等，select的结果中也可能包含聚合统计等信息，例如sum,count,max,min,avg等，这些都需要解析出来方便后续结果集的处理，后续重新生成sql主要是替换逻辑表名为物理表名，并获取对应的数据库物理连接
- 针对avg这种操作，如果涉及查询多个物理表的，可能需要改写sql去查询 sum和count的数据 或者avg和count的数据，改写需要注意可能原sql里面已经包含了count，sum等操作了


### 分片路由算法
  分片算法，主要通过一个表达式，从分片字段对应的值获取到分片结果，可以提供简单地EL表达式，就可以实现从值中截取某一段作为分表数据，也可以提供通用的一致性哈希算法的实现，应用方只需要在xml或者注解中声明分去配置即可，以下为一致性哈希在铜板街的实现
              
```
/**
 * 最大真实节点数
 */
private int max;

/**
 * 真实节点的数量
 */
private int current;

private int[] bucket;

private Set suffixSet;

public void init()  {
    bucket = new int[max];
    suffixSet = new TreeSet();

    int length = max / current;
    int lengthIndex = 0;

    int suffix = 0;

    for (int i = 0; i < max; i++) {
        bucket[i] = suffix;
        lengthIndex ++;
        suffixSet.add(suffix);
        if (lengthIndex == length){
            lengthIndex = 0;
            suffix = i + 1;
        }
    }
}

public VirtualModFunction(int max, int current){
    this.current = current;
    this.max = max;
    this.init();
}


@Override
public Integer execute(String columnValue, Map<String, Object> extension) {
    return bucket[((Long) (Long.valueOf(columnValue) % max)).intValue()];
} 

```



  这里也可以顺带做一下读写分离，配置一些读操作哪个实例，写操作哪个实例，并且做到负载均衡，对应用层透明。
       
### 结果集合并
  如果需要在多个物理表上执行查询，则需要对结果集进行合并处理，此处需要注意返回是一个迭代器
1.统计类   针对sum count ,max,min 只需要将每个结果集的返回结果在做一个max和min，count和sum直接相加即可，针对avg需要通过上面改写的sql获取sum和count 然后相除计算平均值

2.排序类  大部分的排序都伴随着limit限制查询条数，例如返回结果需要查询最近的2000条记录，并且根据创建时间倒序排序，根据路由结果需要查询所有的物理表，假设是4张表，如果此时4张表的数据没有时间上的排序关系，则需要每张表都查询2000条记录，并且按照创建时间倒序排列，现在要做的就是从4个已经排序好的链表，每个链表最多2000条数据，重新排序，选择2000条时间最近的，我们可以通过插入排序的算法，每次分别从每个链表中取出时间最大的一个，在新的结果集里找到位置并插入，直到结果集中存在2000条记录，但是这里可能存在一个问题，如果某一个链表的数据普遍比其他链表数据偏大，这样每个链表取500条数据肯定排序不准确，所以我们还需要保证当前所有链表中剩下的数据的最大值比新结果集中的数据小。  而实际上业务层的需求可能并不是仅仅取出2000条数据，而是要遍历所有的数据，这种要遍历所有数据集的情况，建议在业务层控制一张表一张表的遍历，如果每次都要去每张表中查询在排序严重影响效率，如果在应用层控制，我们在后面在聊。

3.聚合类  group by 应用层需要尽量避免这种操作，这些需求最好能交给搜索引擎和数据分析平台进行，但是作为一个中间件，对于group by 这种我们经常需要统计数据的类型还是应该尽量支持的，目前的做法是 和统计类处理类似，针对各个子集进行合并处理。
         
     
## 优化

以上流程基本可以实现一个简易版本的数据库分库分表中间件，为了让我们的中间件更方便开发者使用，为日常工作提供更多地遍历性，我们还可以从以下几点做优化

### 如何和spring集成
  针对哪些表需要进行分片，分片算法等，这些需要定制化的配置，我们可以在程序里面手工编码，但是这样业务层又需要耦合了分表的逻辑，我们可以借助spring的配置文件，直接将xml里地内容映射成对应的bean实例。
1.我们首先要设计好对应的配置文件的格式，有哪些节点，每个节点包含哪些属性，然后设计自己命名空间，和对应的XSD格式校验文件，XSD文件放在META-INF下，

2.编写 NamespaceHandlerSupport 类，注册每个节点元素对应的解析器
                       
```
public class BaymaxNamespaceHandler extends NamespaceHandlerSupport {

	//com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
	@Override
	public void init() {
		registerBeanDefinitionParser("table", new BaymaxBeanDefinitionParser(TableConfig.class, false));
		registerBeanDefinitionParser("context", new BaymaxBeanDefinitionParser(BaymaxSpringContext.class, false));
		registerBeanDefinitionParser("process", new BaymaxBeanDefinitionParser(ColumnProcess.class, false));
	}

}

```

3.在META-INF文件中增加配置文件 spring.handlers 中配置 spring遇到某个namespace下的节点后 通过哪个解析器解析，最终返回配置实例
                       
```
http\://baymax.tongbanjie.com/schema/baymax-3.0=com.tongbanjie.baymax.spring.BaymaxNamespaceHandler
```

4.在META-INF文件中增加配置文件 spring.schema  中配置 spring遇到某个namespace下的节点后 通过哪个XSD文件进行校验
                       
```
http\://baymax.tongbanjie.com/schema/baymax-3.0.xsd=META-INF/baymax-3.0.xsd
```

5.可以借助ListableBeanFactory的getBeansOfType(Class<T> clazz) 获取配置的结果
   
当然也可以通过自定义注解进行申明，这种方式我们可以借助BeanPostProcessor的时候判断类上是否包含指定的注解，但是这种方式比较重，还必须所加注解的类是在spring容器管理中，也可以借助 ClassPathScanningCandidateComponentProvider   和AnnotationTypeFilter 实现，也可以直接通过 classloader扫描指定的包路径。

### 如何支持分布式事物

  由于框架本身通过定制JDBC协议实现，虽然最终执行sql的是通过原生JDBC，但是对上层应用透明，同时也对上层基于JDBC实现的事物透明，spring的事物管理器可以直接使用，但是spring的声明式事物是通过aop在切面做增强，事物开始先获取connection并设置setAutocommit为fasle,事物结束调用connection进行commit或者rollback，通过threadlocal保存事物上下文和使用的connection,来保证事物内多个sql共用一个connection, 但是如果我们在解析sql后发现要执行多条sql语句，我们通过线程池并发执行，然后等所有的结果返回后进行合并，（这里先不考虑，多个sql可能需要在不同的数据库实例上执行)，虽然通过线程池将导致threadlocal失效，但是我们在threadlocal维护的是我们自己定制的connection，并不是原生的JDBC里的connection,而且这里并发执行并不会让事物处理器没办法判断是否所有的线程都已经结束，然后进行commit或者rollback,因为这里的线程池是在我们定制的connection执行sql过程中运用的，肯定会等到所有线程处理结束后并且合并数据集才会返回。所以在本地事物层面，通过定制化JDBC可以做到透明.

  如果我们进行了分库，同一个表可能在多个数据库实例里，这种如果要对不同实例里的表进行更新，那么将无法在使用本地事物，这里我们不在讨论分布式事物的实现，由于二阶段提交的各种缺点，目前很少有公司会基于二阶段做分布式事物，所以我们的中间价也可以根据自己的具体业务考虑是否要实现XA，目前铜板街大部分分布式事物需求是通过基于TCC的事物补偿做的，这种方式对业务幂等要求较高，同时要基于业务层实现回滚逻辑。


### 提供一个通用发号器
  为什么要提供一个发号器，我们在单表的时候，可能会用到数据库的自增ID，但当分成多表后，每个表都进行单独的ID自增，这样一个逻辑表内的ID就会出现重复。
  我们可以提供一个基于逻辑表自增的主键ID获取方式，如果没有分库，只分表，可以在数据库中增加一个表维护每张逻辑表对应的自增ID，每次需要获取ID的时候都先查询这个标当前的ID然后加一返回，然后在写入数据库，为了并发获取的情况，我们可以采用乐观锁，类似于CAS，update的时候传人以前的ID，如果被人修改过则重新获取，当然我们也可以一次性获取一批ID例如一次获取100个，等这100个用完了在重新获取，为了避免这100个还没用完，程序正常或非正常退出，在获取这100个值的时候就将数据库通过CAS更新为已经获取了100个值之和的值
  不推荐用UUID，无序，太长占内存影响索引效果，不携带任何业务含义
  借助ZOOKEEPER 的zone的版本号来做序列号，
  借助REDIS的INCR命令，进行自增，每台redis设置不同的初始值，但是设置相同的歩长。
```
A：1,6,11,16,21
B：2,7,12,17,22
C：3,8,13,18,23
D：4,9,14,19,24
E：5,10,15,20,25
```


  snowflake算法：其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个 ID）

  铜板街目前所使用的订单号规则：
- 15位时间戳，4位自增序列，2位区分订单类型，7位机器ID，2位分库后缀，2位分表后缀 共32位
- 7位机器ID 通过IP来获取
- 15位时间戳精确到毫秒，4位自增序列，意味着单JVM1毫秒可以生成9999个订单
![](https://pigpdong.github.io/assets/images/2018/split/order.png)

- 最后4位可以方便的根据订单号定位到物理表，这里需要注意分库分表如果是根据一致性哈希算法，这个地方最好存最大值，
  例如 用户id % 64 取余 最多可以分64张表，而目前可能用不到这么多，每相邻4个数字分配到一张表，共16张表，既 userID % 64 / 4 * 4 ,而这个地方存储 userID % 64 即可，不必存最终分表的结果，这种方式方便后续做扩容，可能分表的结果变更了，但是订单号却无法进行变更
          
```	
@Override
public String routeDbByUserId(String userId) {
    Assert.hasLength(userId, "用户ID不能为空");

    Integer userIdInteger = null;
    try {
        userIdInteger = Integer.parseInt(userId);
    } catch (Exception ex) {
        logger.error("解析用户ID为整数失败" + userId, ex);
        throw new RuntimeException("解析用户ID为整数失败");
    }
    
    //根据路由规则确定,具体在哪个库哪个表 例如根据分库公式最终结果在0到63之间  如果要分两个库 mod为32 分1个库mod为64 分16个库 mod为4
    //规律为 64 = mod * （最终的分库数或分表数）
    int mod = orderSplitConfig.getDbSegment();
    
    Integer dbSuffixInt = userIdInteger / 64 % 64 / mod * mod ;

    return StringUtils.leftPad(String.valueOf(dbSuffixInt),  2, '0');
}
	

@Override
public String routeTableByUserId(String userId) {

    Assert.hasLength(userId, "用户ID不能为空");

    Integer userIdInteger = null;
    try {
        userIdInteger = Integer.parseInt(userId);
    } catch (Exception ex) {
        logger.error("解析用户ID为整数失败" + userId, ex);
        throw new RuntimeException("解析用户ID为整数失败");
    }

    //根据路由规则确定,具体在哪个库哪个表 例如根据分表公式最终结果在0到63之间  如果要分两个库 mod为32 分1个库mod为64 分16个库 mod为4
    //规律为 64 = mod * （最终的分库数或分表数）
    int mod = orderSplitConfig.getTableSegment();
    
    Integer tableSuffixInt = userIdInteger % 64 / mod * mod;
    
    return StringUtils.leftPad( String.valueOf(tableSuffixInt),  2, '0');
}	
	
```
      

### 如何实现跨表遍历
  如果业务需求是遍历所有满足条件的数据，而不是只是为了取某种条件下前面一批数据，这种建议在应用层实现，一张表一张表的遍历，每次查询结果返回下一次查询的起始位置和物理表名，查询的时候建议根据 大于或小于某一个ID进行分页，不要limit500,500这种，以下为铜板街的实现方式
          
```
public List<T> select(String tableName, SelectorParam selectorParam, E realQueryParam) {

    List<T> list = new ArrayList<T>();

    // 定位到某张表
    String suffix = partitionManager.getCurrentSuffix(tableName, selectorParam.getLocationNo());

    int originalSize = selectorParam.getLimit();

    while (true) {

        List<T> ts = this.queryByParam(realQueryParam, selectorParam, suffix);

        if (!CollectionUtils.isEmpty(ts)) {
            list.addAll(ts);
        }

        if (list.size() == originalSize) {
            break;
        }

        suffix = partitionManager.getNextSuffix(tableName, suffix);

        if (StringUtils.isEmpty(suffix)) {
            break;
        }

        // 查询下一张表 不需要定位单号 而且也只需要查剩下的size即可
        selectorParam.setLimit(originalSize - list.size());
        selectorParam.setLocationNo(null);
    }

    return list;
}
	
```


### 提供一个扩容工具和管理控制台做配置可视化和监控

1.监控可以借助druid,也可以在定制的jdbc层自己做埋点，将数据以报表的形式进行展示，也可以针对特定的监控指标进行配置，例如执行次数，执行时间大于某个指定时间

2.管理控制台，由于目前配置是在应用层，当然也可以把配置独立出来放在独立的服务器上，由于分片配置基本上无法在线修改，每次修改可能都伴随着数据迁移，所以基本上只能做展示，但是分表后我们在测试环境执行sql去进行逻辑查询的时候，传统的sql工具无法帮忙做到自动路由，这样我们每次查询可能都需要手工计算下分片结果，或者要连续写好几个sql之后在聚合，通过这个管理控制台我们就可以直接根据逻辑表名写sql，这样我们在测试环境或者在线上核对数据的时候，就提高了效率

3.扩容工具，笨办法只能先从老表查询在insert到新表，等到新表数据完全同步完后，在切换到新的切片规则，所以我们设计分片算法的时候，需要考虑到后面扩容，例如一致性哈希就需要迁移一半的数据（扩容一倍的话） 数据迁移如果出现故障，那将是个灾难，所以我们如果要在 不停机的情况下完成扩容，可能需要通过配置文件按以下流程来
        
- 准解阶段.将截至到某一刻的历史表数据同步到新表  例如截至2017年10月1日之前的历史数据，这些历史数据最好不会在被修改
- 阶段一.访问老表，写入老表
- 阶段二.访问老表，写入老表同时写入新表 （插入和修改）
- 阶段三.将10月1日到首次写入新表之间的数据同步到新表 需要保证此时被迁移的数据全部都是终态
- 阶段四.访问新表，写入老表和新表
- 阶段五.访问新表，写入新表

以上流程适用于，订单这种历史数据在达到终态后将不会在被修改，如果历史数据也可能被修改，则可能需要停机，或者通过canel进行数据同步




