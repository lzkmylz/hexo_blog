---
title: 'Redis Summary'
date: 2019-10-10 21:23:26
tags: ['redis']
---

Redis是目前运用最为广泛的单机/分布式NoSQl数据库，可以作为RabbitMQ/RocketMQ/Celery的booker，有着相当广泛的应用。

Redis支持的数据结构有5种，分别是String、List、Set、Hash、Zset。同时，Redis没有显式的加锁机制，而是需要通过watch或通过对某一个变量的值的判断来监视是否有一个key-value pair正在被操作。

# STRING类型

必须牢记Redis是一个key-value的数据库，因此，即使value是String类型的，也是有一个对应的key的。对于一个String类型的value，比较常用的操作如下所示：

```
Conn = redis.Redis()
Conn.get('key') # 若key不存在则返回值为None

Conn.incr('key')  # 对key对应的值进行自增操作，如果key不存在，则会新建一个，并返回值为1. 
Conn.decr('key') # 自减操作，与自增操作相似

Conn.append('key', 'hello') # 将传入的值追加到key对应的value的末尾，返回值是value字符串的长度

Conn.substr('key', 3, 7) # 左闭右开的对字符串的截取。
```

# LIST类型

Redis的LIST允许从两端pop或push一个key-value pair。

```
Conn.lpush('key', 'first')
Conn.rpush('key', 'last')

Conn.lpop('key')
Conn.rpop('key')

Conn.lrange('key', 0, 3) # 读取LIST中的一段

Conn.ltrim('key', 0, 2) # 将LIST删减到只剩选中的那一段
```

# SET集合

SET以无序集合的方式存储不相同的数据

```
Conn.sadd('key', 'a', 'b', 'c')

Conn.srem('key', 'c') # remove操作在成功时返回true，否则返回False

Conn.scard('key') # 查看集合包含的所有item的数量
Conn.smembers('key') # 查看集合所有元素
```

# HASH 哈希

HASH用于将一系列key-value数据存储在一个key值中

# ZSET 有序集合

按score排列key的有序集合，score都是整数

# Redis分布式锁

1、通过SETNX进行类似加锁的操作
```
identifier = str(uuid.uuid4())

if conn.setnx('lock', identifier): # 尝试获得锁，如果已经有其他线程获取了锁，则会返回False
  return identifier

pipe = conn.pipeline(True)
try:
  ......operations
  pipe.execute()
  return True
finally:
  release_lock(conn, lockname, identifier)
```

2、通过watch来监视可能进行修改的键值来撤销当前操作
```
pipe = Conn.pipeline()
try:
  pipe.watch('key')
  ..... operations
  pipe.execute()
  return True
except redis.exceptions.WatchError:
  pass
return False
```

# Redis两种持久化

1、通过snappingshot快照持久化
当redis存储数据量较大时会造成较长时间的服务中止，但能保证较快的故障恢复。

2、AOF持久化
进行只追加的持久化记录，有AOF文件大小不停增加的问题，而且故障恢复需要较长时间。