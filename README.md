### Redis 数据类型

Redis 常用的数据类型：strings（字符串）、Lists（列表）、Hashes（哈希）、Sets（集合）、Sorted sets（有序集合） 等。

官方文档对于数据类型说明 https://redis.io/topics/data-types-intro

#### Redis Strings

Redis String 字符串类型，最简单的数据类型。

```
> set mykey somevalue
OK
> get mykey
"somevalue"
```

#### Redis Lists

Redis Lists 存储的字符串类型的元素，是按插入顺序排序的列表。

```
> rpush mylist A
(integer) 1
> rpush mylist B
(integer) 2
> lpush mylist first
(integer) 3
> lrange mylist 0 -1
1) "first"
2) "A"
3) "B"
> lpop mylist
"first"
> rpop mylist
"B"
> lrange mylist 0 -1
1) "A"
```

#### Redis Hashes

Redis Hashes 是字符串类型的键值对。

```
> hset myhash name liuqi
(integer) 0
> hget myhash name
"liuqi"
> hmset myhash age 27 website liuqitech.com
OK
> hgetall myhash
1) "name"
2) "liuqi"
3) "age"
4) "27"
5) "website"
6) "liuqitech.com"
```

#### Redis Sets

Redis Sets 是字符串类型的无序集合，不能有重复的元素。

```
> sadd myset 1 2 3
(integer) 3
> smembers myset
1) "1"
2) "2"
3) "3"
> sismember myset 1
(integer) 1
> sismember myset 10
(integer) 0
```

#### Redis Sorted sets

Redis Sorted sets 与 Redis Sets 不同的是，每一个元素都会关联一个浮点数类型的分数。

```
> zadd hackers 1940 "Alan Kay"
(integer) 1
> zadd hackers 1957 "Sophie Wilson"
(integer) 1
> zadd hackers 1953 "Richard Stallman"
(integer) 1
> zadd hackers 1949 "Anita Borg"
> zrange hackers 0 -1
1) "Alan Kay"
2) "Anita Borg"
3) "Richard Stallman"
4) "Sophie Wilson"
```



### Redis 部署方式

单机、主从、哨兵、集群

#### 单机

单机方式没什么好说的，使用默认的配置文件启动即可。

```
./redis-server redis.conf
```

#### 主从复制 （replication）

配置主从复制方式非常简单，只需要在 slave 的配置文件中添加如下配置：

```
slaveof 192.168.1.1 6379
```

其中 192.168.1.1 6379 为 master 的IP和端口

官方文档 https://redis.io/topics/replication

#### 哨兵（Sentinel）

哨兵是在主从复制的基础上进行的增强方案。原主从复制的方式中，若master宕机，无法进行主从切，所以会引发一些故障。哨兵可以监控多个，master-slave集群，若发现其中的master宕机时，会把该master下的slave转换为master，同时原master下的slave也会slaveof为新的master。

哨兵启动的方式有以下两种，sentinel的默认端口为26379。

```
redis-sentinel /path/to/sentinel.conf
```

```
redis-server /path/to/sentinel.conf --sentinel
```

我们需要配置监听的master，slave无需手动配置。

```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```

以上为监听两个master的例子。sentinel monitor 语句参数的含义如下：

```
sentinel monitor <master-group-name> <ip> <port> <quorum>
```

其中quorum的意义为，当sentinel为集群时，若quorum为2，此时其中监听的一个master发生了宕机，当有2个sentinel认为它为不可用状态的时候才会真正判定该master已经为不可用状态。

官方文档 https://redis.io/topics/sentinel

#### 集群 （cluster）

按照文档做个简单的搭建，复制6份redis到文件夹（如 7000 7001 7002 7003 7004 7005），7000到7005的redis.conf分别按以下模板进行配置

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

分别启动这6个reids实例，然后redis-cli创建集群（5以上版本）

```
./redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1
```

官方文档 https://redis.io/topics/cluster-tutorial



### Spring Boot 配置

我demo中使用的 Spring Boot 版本为 ```2.1.6.RELEASE```

添加依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

查看依赖可知现在版本的使用的默认的reids客户端为 ```Lettuce```

通过查看`LettuceConnectionConfiguration` 可发现，它可以为我们初始化一个 `RedisTemplate<Object, Object>` 类型的 redisTemplate，和一个 `RedisTemplate<String, String>` 类型的 stringRedisTemplate。

```
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```

后面的例子中为了方便测试，直接注入```stringRedisTemplate``` 来使用，当然你也可以自定义自己需要类型的 RedisTemplate。针对不同的部署方式，修改application.yml 配置文件如下：

#### 单机

```
spring:
redis:
  host: 127.0.0.1
  port: 6379
```

#### 哨兵

```
spring:
redis:
  sentinel:
	master: mymaster
	nodes: 127.0.0.1:26379, 127.0.0.1:26380
```

#### 集群

```
spring:
redis:
  cluster:
	nodes: 127.0.0.1:7000, 127.0.0.1:7001, 127.0.0.1:7002, 127.0.0.1:7003, 127.0.0.1:7004, 127.0.0.1:7005
```

  



