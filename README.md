# demo-guava-cache 缓存使用讲解

## 一、简介
guava cache是google guava中的一个内存缓存模块,用于将数据缓存到JVM内存中.实际项目开发中经常将一些比较公共或者常用的数据缓存起来方便快速访问.

内存缓存最常见的就是基于HashMap实现的缓存,为了解决并发问题也可能也会用到ConcurrentHashMap等并发集合,但是内存缓存需要考虑很多问题,包括并发问题、缓存过期机制、缓存移除机制、缓存命中统计率等.

guava cache已经考虑到这些问题,可以上手即用.通过CacheBuilder创建缓存、然后设置缓存的相关参数、设置缓存的加载方法等.本例子主要讲解guava cache的基本用法,详细的说明已在代码中说明.

## 二、策略分析
```
expireAfterWrite 写缓存后多久过期
expireAfterAccess 读写缓存后多久过期
refreshAfterWrite 写入数据后多久过期,只阻塞当前数据加载线程,其他线程返回旧值

这几个策略时间可以单独设置,也可以组合配置
```

expireAfterWrite与refreshAfterWrite单独使用与混合使用的策略分析

```
已知配置条件:
Thread-1 每 3 秒获取一次缓存id=1的数据
Thread-2 每 5 秒获取一次缓存id=1的数据
加载一次缓存加载数据耗时 8 秒
```

### 1、expireAfterWrite单独使用

expireAfterWrite=17

```
Thread-2 加载数据开始
Thread-2 加载数据结束
Thread-1 01:04:07 value:6798
Thread-2 01:04:07 value:6798
Thread-1 01:04:10 value:6798
Thread-2 01:04:12 value:6798
Thread-1 01:04:13 value:6798
Thread-1 01:04:16 value:6798
Thread-2 01:04:17 value:6798
Thread-1 01:04:19 value:6798
Thread-1 01:04:22 value:6798
Thread-2 01:04:22 value:6798
1 value:6798 被移除,原因:EXPIRED
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-1 01:04:33 value:7836
Thread-2 01:04:33 value:7836
Thread-1 01:04:36 value:7836
Thread-2 01:04:38 value:7836
Thread-1 01:04:39 value:7836
```

说明:

启动时Thread-2加载数据,此时缓存中无数据,Thread-1阻塞等待Thread-2加载完成数据.
在设置的时间数据过期后Thread-1加载数据,Thread-2本应该01:04:22后的5秒加载数据,但是Thread-1等待3秒后加载,数据加载耗时8秒,所以Thread-2在01:04:33时加载数据成功.

结论:

当其他线程在加载数据的时候,当前线程会一直阻塞等待其他线程加载数据完成.

### 2、refreshAfterWrite单独使用

refreshAfterWrite=17

```
Thread-2 加载数据开始
Thread-2 加载数据结束
Thread-1 01:13:32 value:551
Thread-2 01:13:32 value:551
Thread-1 01:13:35 value:551
Thread-2 01:13:37 value:551
Thread-1 01:13:38 value:551
Thread-1 01:13:41 value:551
Thread-2 01:13:42 value:551
Thread-1 01:13:44 value:551
Thread-1 01:13:47 value:551
Thread-2 01:13:47 value:551
Thread-1 加载数据开始
Thread-2 01:13:52 value:551
Thread-2 01:13:57 value:551
Thread-1 加载数据结束
1 value:551 被移除,原因:REPLACED
Thread-1 01:13:58 value:827
Thread-1 01:14:01 value:827
Thread-2 01:14:02 value:827
Thread-1 01:14:04 value:827
Thread-2 01:14:07 value:827
```

说明:

启动时Thread-2加载数据,此时缓存中无数据,Thread-1阻塞等待Thread-2加载完成数据.
在设置的时间数据过期后Thread-1加载数据,Thread-2仍然按照策略获取到旧数据成功.

结论:

当没有数据的时候,其他线程在加载数据的时候,当前线程会一直阻塞等待其他线程加载数据完成;如果有数据的情况下其他线程正在加载数据,当前线程返回旧数据.


### 3、expireAfterWrite与refreshAfterWrite一起使用情况一

expireAfterWrite=13

refreshAfterWrite=17

```
Thread-2 加载数据开始
Thread-2 加载数据结束
Thread-1 01:18:32 value:5901
Thread-2 01:18:32 value:5901
Thread-1 01:18:35 value:5901
Thread-2 01:18:37 value:5901
Thread-1 01:18:38 value:5901
Thread-1 01:18:41 value:5901
Thread-2 01:18:42 value:5901
Thread-1 01:18:44 value:5901
1 value:5901 被移除,原因:EXPIRED
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-2 01:18:55 value:1300
Thread-1 01:18:55 value:1300
Thread-1 01:18:58 value:1300
Thread-2 01:19:00 value:1300
Thread-1 01:19:01 value:1300
```

说明:

启动时Thread-2加载数据,此时缓存中无数据,Thread-1阻塞等待Thread-2加载完成数据.
在设置的时间数据过期后Thread-1加载数据,Thread-2本应该01:18:42后的5秒加载数据,但是Thread-1等待3秒后加载,数据加载耗时8秒,所以Thread-2在01:18:55时加载数据成功.

结论:

当其他线程在加载数据的时候,当前线程会一直阻塞等待其他线程加载数据完成,与单独使用expireAfterWrite一样的效果.

### 4、expireAfterWrite与refreshAfterWrite一起使用情况二

expireAfterWrite=17

refreshAfterWrite=13

```
Thread-2 加载数据开始
Thread-2 加载数据结束
Thread-1 01:20:25 value:1595
Thread-2 01:20:25 value:1595
Thread-1 01:20:28 value:1595
Thread-2 01:20:30 value:1595
Thread-1 01:20:31 value:1595
Thread-1 01:20:34 value:1595
Thread-2 01:20:35 value:1595
Thread-1 01:20:37 value:1595
Thread-2 加载数据开始
Thread-1 01:20:40 value:1595
Thread-2 加载数据结束
Thread-1 01:20:48 value:2277
1 value:1595 被移除,原因:EXPIRED
Thread-2 01:20:48 value:2277
Thread-1 01:20:51 value:2277
Thread-2 01:20:53 value:2277
Thread-1 01:20:54 value:2277
Thread-1 01:20:57 value:2277
Thread-2 01:20:58 value:2277
Thread-1 01:21:00 value:2277
Thread-1 加载数据开始
Thread-2 01:21:03 value:2277
Thread-1 加载数据结束
Thread-2 01:21:11 value:3750
1 value:2277 被移除,原因:EXPIRED
Thread-1 01:21:11 value:3750
Thread-1 01:21:14 value:3750
Thread-2 01:21:16 value:3750
Thread-1 01:21:17 value:3750
Thread-1 01:21:20 value:3750
Thread-2 01:21:21 value:3750
```

说明:

启动时Thread-2加载数据,此时缓存中无数据,Thread-1阻塞等待Thread-2加载完成数据.
在设置的时间数据过期后Thread-2加载数据,Thread-1仍然按照策略在01:20:40获取到旧数据成功,但是本应该01:20:45继续获取一次数据但是等到01:20:48才获取成功.

结论:

当没有数据的时候,其他线程在加载数据的时候,当前线程会一直阻塞等待其他线程加载数据完成;
如果有数据的情况下其他线程正在加载数据,已经超过refreshAfterWrite设置时间但是没有超过expireAfterWrite设置的时间时当前线程返回旧数据.
如果有数据的情况下其他线程正在加载数据,已经超过expireAfterWrite设置的时间时当前线程阻塞等待其他线程加载数据完成.
这种情况适合与设置一个加载缓冲区的情况,既能保证过期后加载数据,又能保证长时间没访问多个线程并发时获取到过期旧数据的情况.


## 三、注意点
由于google guava cache对于value=null的处理是抛出异常,使用缓存的时候如果存在旧值时会返回旧值，不能清除缓存。

推荐使用Java 8 Optional 将value包装一层，出现null的时候使用invalidate及时清理缓存

```
Optional<String> optional = cache.get(key);
if (optional.isPresent()) {
    return optional.get();
} else {
    cache.invalidate(key);
    return null;
}
```























