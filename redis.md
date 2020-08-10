### redis缓存雪崩，缓存穿透，缓存击穿的解决方法

#### 缓存击穿
> 缓存击穿表示某个key的缓存非常热门，有很高的并发一直在访问，如果该缓存失效，那同时会走数据库，压垮数据库。

解决方案：
1. 让该热门key的缓存永不过期。
2. 使用互斥锁，通过redis的setnx实现互斥锁。
```php
function getRedis()
{
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379, 60);
    return $redis;
}
 
//加锁
function lock($key, $random)
{
    $redis = getRedis();
    //设置锁的超时时间，避免释放锁失败，del()操作失败，产生死锁。
    $ret = $redis->set($key, $random, ['nx', 'ex' => 3 * 60]);
    return $ret;
}
 
//解锁
function unLock($key, $random)
{
    $redis = getRedis();
    //这里的随机数作用是，防止更新缓存操作时间过长，超过了锁的有效时间，导致其他请求拿到了锁。
    //但上一个请求更新缓存完毕后，如果不加判断直接删除锁，就会误删其他请求创建的锁。
    if ($redis->get($key) == $random) {
        $redis->del($key);
    }
}
 
//从缓存中获取文章数据
function getArticleInCache($id)
{
    $redis = getRedis();
    $key = 'article_content_' . $id;
    $ret = $redis->get($key);
    if ($ret === false) {
        //生成锁的key
        $lockKey = $key . '_lock';
        //生成随机数，用于设置锁的值，后面释放锁时会用到
        $random = mt_rand();
        //拿到互斥锁
        if (lock($lockKey, $random)) {
            //这里是伪代码，表示从数据库中获取文章数据
            $value = $db->getArticle($id);
            //更新缓存，过期时间可以根据情况自已调整
            $redis->set($key, $value, 2 * 60);
            //释放锁
            unLock($lockKey, $random);
        } else {
            //等待200毫秒，然后重新获取缓存值，让其他获取到锁的进程取得数据并设置缓存
            usleep(200);
            getArticleInCache($id);
        }
    } else {
        return $ret;
    }
}
```
#### 缓存雪崩

> 缓存雪崩表示在某一时间段，缓存集中失效，导致请求全部走数据库，有可能搞垮数据库，使整个服务瘫痪

使缓存集中失效的原因:
1. redis服务器挂掉了
2. 对缓存数据设置了相同的过期时间，导致某时间段内缓存集中失效

**缓存雪崩与缓存击穿的区别: 缓存击穿针对的是某一热门key缓存，而雪崩针对的是大量缓存的集中失效。**

如何解决缓存集中失效：
1. 针对原因1， 可以实现redis的高可用方案（Sentinel方案、Cluster方案）
2. 在缓存的过期时间之后，再加上一个随机值。避免缓存在同一时间过期
```php
$redis = new Redis();
$redis->connect('127.0.0.1', 6379, 60);
$redis->auth('');
 
//设置过期时间加上一个随机值
$redis->set('article_content_1', '文章内容', 60 + mt_rand(1, 60));
$redis->set('article_content_2', '文章内容', 60 + mt_rand(1, 60));
```
3. 使用双缓存策略，设置两个缓存，原始缓存和备用缓存，原始缓存失效时，访问备用缓存，备用缓存失效时间设置长点。
```php
//原始缓存
$redis->set('article_content_2', '文章内容', 60);
//设置备用缓存，失效时间设置长点
$redis->set('article_content_backup_2', '文章内容', 1800);
```

#### 缓存穿透
> 缓存穿透表示查询一个一定不存在的数据，由于没有获取到缓存，所以没写入缓存，导致这个不存在的数据每次都需要去数据库查询，失去了缓存的意义

请求的数据大量的没有获取到缓存，导致走数据库，有可能搞垮数据库，使整个服务瘫痪。

比如文章表，一般我们的主键ID都是无符号的自增类型，有些人想要搞垮你的数据库，每次请求都用负数ID，而ID为负数的记录在数据库根本就没有

解决方案：
1. 对于像ID为负数的非法请求直接过滤掉，采用布隆过滤器(Bloom Filter)。
2. 针对在数据库中找不到记录的，我们仍然将该空数据存入缓存中，当然一般会设置一个较短的过期时间。
```php
//设置文章ID为-10000的缓存为空
$id = -10000;
$redis->set('article_content_' . $id, '', 60);
 
var_dump($redis->get('article_content_' . $id));
```


