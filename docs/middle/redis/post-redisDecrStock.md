---
title: 抢购系统秒杀系统预扣减库存方案
date: 2020-04-08 12:41:58
categories: 
- 解决方案
tags:
- redis
- 高并发
---
![代码片段](/images/code-1.jpg)
# 秒杀系统中常见的问题

## 超卖
并发读的时候所有线程拿到的库存剩余数量都是一样的，假设目前有10个库存，100个并发抢。 所有的线程读到的都是剩余10个库存，结果都去更新数据库减少库存了，这样就发生了超卖。
```$xslt
void orderNow(){
   剩余库存 = 数据库.查询;
   if(剩余库存 > 0) {//这个地方并发场景下就超卖了
     数据库.减库存 
     生成订单
   }
}
```
## 少卖
仅使用分布式锁来锁定库存或者从数据库中使用where条件来控制库存的减少，会出现如果用户不想买了，或者系统出异常下单失败了。这个时候商品就没卖出去
```$xslt
if(预先减少库存.成功） return "没抢到";
剩余库存 = 数据库.查询;
   if(剩余库存 > 0) {//这个地方并发场景下就超卖了
     数据库.减库存 
     生成订单
   }
```

# 解决方案

## 基于预减少库存的解决方案
通过redis eval 实现检查剩余库存。库存的设置由发布抢购的功能进行添加。 对订单设置失效时间，下单失败后或订单过期后库存要加回到redis中（此时因为下单失败数据库库存已回退）。这样其他用户就可以接着抢购了。
```$xslt
thead{
    数据库.查询失效了的订单。
    数据库.回库存，回redis库存
}

void orderNow() {
if(预先减少库存.成功） return "没抢到";
剩余库存 = 数据库.查询;
   if(剩余库存 > 0) {//这个地方并发场景下就超卖了
        try{
             数据库.减库存 
             生成订单
        } catch(Exception e) {
           回库存   
           数据库.回滚事务
        }
   }
}
```

## redis预减库存工具类
[附Redis管理工具类](/2020/04/08/code-simpleredisutil/)
### SimpleRedisStockManager.java
```$xslt
import redis.clients.jedis.Jedis;

import java.util.ArrayList;
import java.util.List;

public class SimpleRedisStockManager {

    private SimpleRedisManager manager = SimpleRedisManager.getInstance();

    private static SimpleRedisStockManager stockManager = new SimpleRedisStockManager();

    public static final String PRE_STOCK = "preStock:";

    private SimpleRedisStockManager(){}

    public static SimpleRedisStockManager getInstance() {
        return stockManager;
    }


    public Long decrStock(String key, int buyNum){

        List<String> keys = new ArrayList<>();
        keys.add(key);
        List<String> argv = new ArrayList<>();
        argv.add(""+buyNum);
       Jedis jedis = null;
       try {
           jedis = manager.getJedis();
           Object obj = jedis.eval("local stock = redis.call('get', KEYS[1])\n" +
                   "if stock >= ARGV[1] then\n" +
                   "    return redis.call('decrBy',KEYS[1], ARGV[1])\n" +
                   "else\n" +
                   "    return -1\n" +
                   "end", keys, argv);
           System.out.println(key +":"+ obj.toString());
           return Long.parseLong(String.valueOf(obj));
       } finally {
           if(jedis!=null) jedis.close();
       }
    }

    public Long incrStock(String key, int buyNum){
        Jedis jedis = null;
        try {
            jedis = manager.getJedis();
            return jedis.incrBy(key, buyNum);
        } finally {
           if(jedis!=null) jedis.close();
        }
    }


    public String getPreStockKey() {
        return PRE_STOCK;
    }



}

```
## 附加模拟扣减库存的case 基于springboot和dubbo通信
### DecrStockFacadeImpl.java

```$xslt
import com.alibaba.dubbo.config.annotation.Service;
import com.alibaba.fastjson.JSONObject;
import com.sunhf.security.dao.VStockTableDao;
import com.sunhf.security.util.SimpleRedisStockManager;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

/**
 * 模拟减少库存和回库存的操作
 */
@Component
@Service(interfaceClass = DecrStockFacade.class)
public class DecrStockFacadeImpl implements DecrStockFacade {

    private static final Logger log = LoggerFactory.getLogger(DecrStockFacadeImpl.class);

    @Resource
    private VStockTableDao stockTableDao;

    private SimpleRedisStockManager stockManager = SimpleRedisStockManager.getInstance();


    @Override
    public String orderNow(String prodId, Integer buyNum) {
        if(stockManager.decrStock(stockManager.getPreStockKey()+prodId, buyNum) == -1) {
            return null;
        }
        log.warn("预先下单成功"+ prodId);
        int stock = stockTableDao.getStock(prodId);
        if(stock < buyNum) return null;
        log.info("当前库存数量:{}",stock);
        boolean decrStock = stockTableDao.updateStock(prodId, stock - buyNum);
        if(!decrStock) {
            log.warn("下单失败，原因，更新库存失败"+ prodId);
            return null;
        }
        return stockTableDao.createOrder(prodId, buyNum);
    }

    @Override
    public boolean cancelOrder(String orderId) {
        JSONObject orderInfo = stockTableDao.getOrder(orderId);
        int buyNum = orderInfo.getInteger("stock");
        String prodId = orderInfo.getString("prodId");
        if(buyNum == 0) {
            log.info("订单不存在"+orderId);
            return false;
        }
        int stock = stockTableDao.getStock(prodId);
        stockTableDao.updateStock(prodId, stock+buyNum);
        return true;
    }
}
```

### VStockTableDao.java
这个是模拟的数据库操作，可自己实现换成真实的数据库
```$xslt

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.sunhf.security.util.SimpleRedisCache;
import com.sunhf.security.util.SimpleRedisStockManager;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.Timer;
import java.util.TimerTask;
import java.util.UUID;

/**
 * 模拟库存表操作
 */
@Component
public class VStockTableDao {

    private static final Logger log = LoggerFactory.getLogger(VStockTableDao.class);
    String prodTable = "prodTable:";
    String orderTable = "orderTable:";
    String orderIdKey = "orderId:";


    @PostConstruct
    public void init() {
        JSONObject prodInfo1 = new JSONObject();
        prodInfo1.put("name", "小米10");
        prodInfo1.put("stock", "10");
        JSONObject prodInfo2 = new JSONObject();
        prodInfo2.put("name", "华为p40");
        prodInfo2.put("stock", "10");
        try {
            //临时使用redis作为数据存储工具
            SimpleRedisCache.setCache( prodTable+ 1, JSON.toJSONString(prodInfo1));
            SimpleRedisCache.setCache(prodTable + 2, JSON.toJSONString(prodInfo2));
            SimpleRedisStockManager stockManager = SimpleRedisStockManager.getInstance();
            Long v1 = stockManager.incrStock( stockManager.getPreStockKey()+1, 10);
            Long v2 = stockManager.incrStock( stockManager.getPreStockKey()+2, 10);
            log.info("预先设置库存成功,{},{},{}",stockManager.getPreStockKey()+1, v1,v2);

        } catch (Exception e) {
            log.error("初始化库存数据失败");
        }

        startOrderExpThread();
    }

    /**
     * 订单自动失效线程
     */
    private void startOrderExpThread() {
        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                //检查订单失效时间
            }
        };
        Timer timer = new Timer();
        timer.scheduleAtFixedRate(task, 0, 1000* 5);
    }

    public int getStock(String prodId) {
        String val = SimpleRedisCache.getCache(prodTable+prodId);
        if(val == null) return 0;
        Integer stock = JSON.parseObject(val).getInteger("stock");
        if(stock!=null && stock < 0) throw new RuntimeException("超卖了:"+stock);
        return stock == null ? 0 : stock;
    }

    public boolean updateStock(String prodId, int stock) {
        String val = SimpleRedisCache.getCache(prodTable+prodId);
        if(val == null) return false;
        JSONObject prodInfo = JSON.parseObject(val);
        prodInfo.put("stock", stock);
        SimpleRedisCache.setCache( prodTable+prodId, JSON.toJSONString(prodInfo));
        return true;
    }

    public String createOrder(String prodId, Integer buyNum) {
        JSONObject orderInfo = new JSONObject();
        orderInfo.put("prodId", prodId);
        orderInfo.put("buyNum", buyNum);
        String orderId = UUID.randomUUID().toString();
        SimpleRedisCache.setCache(orderTable+orderId, JSON.toJSONString(orderInfo));
        return orderId;
    }

    public JSONObject getOrder(String orderId) {
        String val = SimpleRedisCache.getCache(orderTable+orderId);
        if(val == null) return null;
        return  JSON.parseObject(val);
    }


}

```

### DecrStockTest.java
这个是调用端的并发测试代码
```$xslt

import com.alibaba.dubbo.config.annotation.Reference;
import com.sunhf.security.WebLanucher;
import com.sunhf.security.facade.DecrStockFacade;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = WebLanucher.class)
public class DecrStockTest {

    @Reference
    private DecrStockFacade decrStockFacade;

    @Test
    public void test() {
        ExecutorService pool = Executors.newFixedThreadPool(10);
        AtomicInteger count = new AtomicInteger();

        CyclicBarrier b = new CyclicBarrier(10);
        for(int i = 0;; i++) {
            pool.submit(new Thread(){
                @Override
                public void run() {
                    try {
                        b.await();
                       String orderId = decrStockFacade.orderNow("1", 2);
                       if(orderId != null && !"".equals(orderId)) {
                           System.out.println("抢到商品了，订单id:"+orderId);
                           System.out.println("总共抢到了:"+count.getAndIncrement());
                       }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }


    }

    @Test
    public void test2() {
        AtomicInteger count = new AtomicInteger();
        for(int i = 0;; i++) {
            String orderId = decrStockFacade.orderNow("1", 1);
            if(orderId != null && !"".equals(orderId)) {
                System.out.println("抢到商品了，订单id:"+orderId);
                System.out.println("总共抢到了:"+count.incrementAndGet());
            }
        }


    }
}

```
