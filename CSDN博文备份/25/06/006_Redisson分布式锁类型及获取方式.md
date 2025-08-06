# Redisson分布式锁类型及获取方式

> 原创 已于 2025-06-13 16:22:48 修改 · 公开 · 318 阅读 · 5 · 8 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/148635248

## Redis分布式锁类型及获取方式（Redisson实现）

### 默认锁类型：可重入锁（Reentrant Lock）

```java
// 使用getLock获取的就是可重入锁
RLock lock = redisson.getLock("myLock");
```

**特点** ：

- 支持同一线程多次加锁（锁重入）

- 支持锁续期（Watchdog机制）

- 支持公平/非公平模式（默认非公平）

### Redisson支持的其他锁类型及获取方式

#### 1. 公平锁（Fair Lock）

```java
RLock fairLock = redisson.getFairLock("fairLock");
```

**特点** ：

- 按照请求锁的顺序分配锁

- 避免线程饥饿问题

- 性能略低于非公平锁

#### 2. 联锁（MultiLock）

```java
RLock lock1 = redisson.getLock("lock1");
RLock lock2 = redisson.getLock("lock2");
RLock multiLock = redisson.getMultiLock(lock1, lock2);
```

**特点** ：

- 同时锁定多个锁

- 所有锁都成功才算获取成功

- 适用于需要同时操作多个资源的场景

#### 3. 红锁（RedLock）

```java
RLock lock1 = redissonClient1.getLock("lock1");
RLock lock2 = redissonClient2.getLock("lock1");
RLock lock3 = redissonClient3.getLock("lock1");
RLock redLock = redisson.getRedLock(lock1, lock2, lock3);
```

**特点** ：

- 基于Redis多实例的分布式锁

- 遵循Redlock算法

- 需要大多数节点获取成功才算成功

#### 4.读写锁（ReadWriteLock）

```java
RReadWriteLock rwLock = redisson.getReadWriteLock("rwLock");
RLock readLock = rwLock.readLock();  // 读锁
RLock writeLock = rwLock.writeLock(); // 写锁
```

**特点** ：

- 读读不互斥

- 读写互斥

- 写写互斥

- 适合读多写少场景

#### 5. 信号量（Semaphore）

```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.acquire();    // 获取许可
semaphore.release();    // 释放许可
```

**特点** ：

- 限制同时访问资源的线程数

- 可用于限流场景

#### 6. 可过期锁（Spin Lock）

```java
// 尝试获取锁，最多等待10秒，锁持有时间5秒
boolean res = lock.tryLock(10, 5, TimeUnit.SECONDS);
```

**特点** ：

- 可设置明确的等待时间和持有时间

- 不会自动续期

### 锁类型选择校验表

| 场景需求 | 推荐锁类型 | 示例场景 |
|:---:|:---:|:---:|
| 通用分布式锁 | 可重入锁 | 商品秒杀、订单创建 |
| 需要公平获取 | 公平锁 | 按顺序处理任务 |
| 同时操作多个资源 | 联锁 | 跨账户转账 |
| 高可靠性要求 | 红锁 | 金融核心交易 |
| 读多写少 | 读写锁 | 商品详情页访问 |
| 资源池限制 | 信号量 | 限流控制 |
| 明确控制锁持有时间 | 可过期锁 | 短时任务处理 |


### 代码示例：抢券业务锁选择校验

```java
public Result grabCoupon(Long userId, Long couponId) {
    // 1. 选择锁类型 - 可重入锁最适合抢券场景
    RLock lock = redisson.getLock("coupon:lock:" + couponId);
  
    try {
        // 2. 参数校验 - 等待时间不宜过长，锁时间要覆盖业务执行
        if (!lock.tryLock(300, 3000, TimeUnit.MILLISECONDS)) {
            return Result.fail("抢券人数过多，请重试");
        }
      
        try {
            // 3. 业务校验
            Coupon coupon = couponMapper.selectById(couponId);
            if (coupon.getStock() <= 0) {
                return Result.fail("券已售罄");
            }
          
            // 4. 扣减库存
            if (couponMapper.reduceStock(couponId) > 0) {
                createOrder(userId, couponId);
                return Result.ok("抢券成功");
            }
            return Result.fail("抢券失败");
        } finally {
            // 5. 确保锁释放
            if (lock.isLocked() && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return Result.fail("系统繁忙");
    }
}
```

**校验要点** ：

1. 抢券场景选择可重入锁是最合适的

2. 设置合理的等待时间(300ms)和锁持有时间(3000ms)

3. 释放锁前检查锁状态和持有线程

4. 正确处理中断异常

