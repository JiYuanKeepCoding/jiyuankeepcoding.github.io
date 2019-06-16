---
layout:     post
title:      探秘apache-common-dbcp和apache-common-pool
date:       2019-06-16
author:     JY
header-img: img/cat0_dns.jpeg
catalog: true
tags:
    - JAVA
    - 数据库
    - JDBC
---

#### 什么是apache-common-dbcp?
当你为你的Java应用配置数据库连接的时候需要指定dataSource。在javax.sql.DataSource中定义了dataSource的接口。这个接口在jdbc driver里面会有对应的实现。dataSource主要的作用是:
1.配置目标地址和端口，用户名，密码，数据库实例
2.对外暴露一个getConnection接口。查db的时候会先拿到connection再进行查询

试想你的web application有上万人甚至更多同时访问的时候，如果你想去数据库里面的user表来校验信息。你肯定不希望创建出来上万个db connection，这样会压垮你的db。因此就产生了我们对复用数据库连接的需求了。所以当上层应用在调用dataSource.getConnection的时候我们希望不一定要真的创建一个数据库连接，而是看看有没有现成的连接可以拿来用。当我们调用connection.close的时候我们也不想真的close掉数据库连接，最好把它放在一个池子里面供后续使用。这就是数据库连接池。
从代码的角度来说，如果你想实现一个数据库连接池。你需要:
1. 实现dataSource接口的getConnection方法。从连接池里面借出来一个connection
2. 从你的getConnection返回的connection必须要实现Connection接口并且在close的时候把连接还给连接池


#### 9012年了为啥还要写这种存在已久的东西
目前我在维护一个java webapp已经两年时间了,由于数据量比较小mysql db这块几乎从来没有出过问题。我也并不关心spring-data-jpa下面的一切包括transactionManager和数据库连接池的配置。知道我们的测试环境db query突然变得特别慢（有时甚至不能正常返回）。我所观察的现象是所有的db query要么就都很快，要么就都很慢并且不会自动恢复。在重启了集群里面所有的java进程之后db query的速度又恢复了正常。在不了解数据库连接池的工作机制的情况下，于是我开始yy所有的可能性：
1. 是不是连接池满了然后新的query来的时候等也等不到可用的连接
2. 是不是拿到的连接其实是不可用的，直接执行query的时候由于没有设置超时导致查询特别慢


#### 开始读代码
从项目代码里面分析出我们用的连接池是apache-common-dbcp（我之前一直以为是c3p0来着。。。）
先找到BasicDatasource.getConnection方法
```
public Connection getConnection() throws SQLException {
    return this.createDataSource().getConnection();
}
```
createDataSource()里面做了很多事:
1. 如果已经有数据库连接池了直接返回
2. 如果没有的话用给定的配置创建一个新的数据库连接池(new GenericObjectPool)并且初始化里面的数据库连接，后面会讲具体的配置
3. 把PoolableConnectionFactory配置到数据库连接池里面。

GenericObjectPool是apache-common-pool提供的，不管是db connection还是tcp connection或者其他的对象你都可以把它放在里面管理。它会要求你提供一个PoolableObjectFactory
```
public interface PoolableObjectFactory {
  /**
   * Creates an instance that can be returned by the pool.
   * @return an instance that can be returned by the pool.
   */
  Object makeObject() throws Exception;

  /**
   * Destroys an instance no longer needed by the pool.
   * @param obj the instance to be destroyed
   */
  void destroyObject(Object obj) throws Exception;

  /**
   * Ensures that the instance is safe to be returned by the pool.
   * Returns <tt>false</tt> if this object should be destroyed.
   * @param obj the instance to be validated
   * @return <tt>false</tt> if this <i>obj</i> is not valid and should
   *         be dropped from the pool, <tt>true</tt> otherwise.
   */
  boolean validateObject(Object obj);

  /**
   * Reinitialize an instance to be returned by the pool.
   * @param obj the instance to be activated
   */
  void activateObject(Object obj) throws Exception;

  /**
   * Uninitialize an instance to be returned to the pool.
   * @param obj the instance to be passivated
   */
  void passivateObject(Object obj) throws Exception;
}
```
你需要告诉common-pool如何真的创建一个object，在这里就是真的创建，销毁一个数据库连接，当然这块果断委托给jdbcDriver去做啦。
还有就是要告诉common-pool如何校验当前的object是不是可用的，对于一个数据库连接object我们的做法通常是select 1看看有木有返回结果。有就说明连接可用。后面会讲这个方法在什么时候被调用。
activateObject和passiveObject就是当object在pool里面的时候它是idle状态，在被borrow出去的时候需要activate一下，等borrow完了还给pool的时候会调用一下passivateObject让他继续睡觉。

后面的getConnection就很简单了，就是去刚才创建的GenericObjectPool里面borrow一个连接出来。
读到这里我们应该就知道common-dbcp就是一个中间层，它实现了dataSource接口对外暴露接口，真正创建，销毁连接的时候它会委托给jdbc driver，数据库连接管理的逻辑统统交给common-pool来做。

接下来就看看common-pool是怎么管理我的数据库连接的。这里还是不贴代码了，感兴趣的可以直接读一下GenericObjectPool.borrowObject
提前声明，后面说的所有的"对象"这个词，在我的场景代表数据库连接。
1. 从pool里面拿出第一个对象。这里是用linklist.removeFirst()来实现的。由此可以看出我们只能按顺序拿里面的对象，并且这个对象一旦借出去了,pool里面就不存了只能等上层应用啥时候把它还回来
2. 如果借出去的对象已经超出了配置项maxActive所规定的，那么就等，等待时长在配置项waitTime中规定。等完了之后开始重试，如果还是不够的话继续等，直到超时maxWait规定的时间就会抛出Timeout waiting for idle object
3. 如果#1里面返回null的话调用makeObject创建一个新的
4. 如果#1里面返回不为空的话，而且又配置了testOnBorrow=true的话就会调用validateObject看看对象是不是能用，如果能用的话就把它激活


#### common-pool中对象驱逐
想一下这个场景，如果你是用cname来访问的数据库。然后cname最终指向的ip被更改了，你的数据库连接池还在一直用老的tcp connection。那这个老的connection什么时候会被刷成新的访问到最新的ip呢？
common-pool自带的evict功能让用户可以通过配置来刷新pool里面的对象。不像复杂的缓存eviction算法，common-pool里面的eviction很简单，就是一个interval每隔固定的时间(配置项timeBetweenEvictionRunsMillis)就会扫一下pool里面所有的idle对象，如果闲置的时间太长或者是validateObject失败就会把这个idle的object从pool里面剔除。所以其实要解决刚才的问题不是简单的配置就可以，validateObject的实现中要能够分辨出老的ip是不合法的这样才能利用common-pool做自动的connection refresh


#### 所以我是怎么处理我的webapp所碰到的问题的
默认的配置里面已经有testOnBorrow所以连接池给到上层的连接一定是可用的。但是为了应对不稳定的网络环境我姑且加上了eviction（之前没有）让它每30s来校验一下idle的连接是否仍然可用。
在阅读源码的过程中发现common-pool考虑的真的已经很完备了，我当前的配置来看也不应该导致刚才的问题。所以我严重怀疑是上层JpaTransactionManager向pool借了连接之后没有还给pool。结果确实发现JpaTransactionManager的没有默认的超时时间，果断先设成15秒。这样配是为了在运行时环境恢复正常的时候，我的应用也能立马恢复正常。不然我就只能重启我的应用来重刷连接池了。