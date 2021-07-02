# 2020 年度八股文

## Java基础篇

### 基础概念
1. Java面向对象的三大特性
2. Java默认有哪些方法级别
3. Java默认的方法级别是什么
4. 如果父类引用了子类对象，调用这个对象的方法是子类中的还是父类中的
5. 如果父类有一个同名同参数的方法，是private级别。子类可以声明这个方法吗。
6. 抽象方法的默认级别是什么
7. 如果父类方法是public级别，子类可以声明成private级别吗
8. 双亲委派的概念，如何破坏掉双亲委派。
9. 如果创建一个新的类，名字叫String，这个类可以被加载吗,为什么。
10. final关键字的作用是什么
11. 描述一下Java的内存模型(JMM)
12. Java的内存模型会产生什么问题
13. volatile关键字的作用
14. volatile能保证线程安全吗，什么场景下可以使用volatile
15. 描述一下happend-before原则
16. Java反射的作用，为什么不用JavaBean抽象

### HashMap
1. HashMap的数据结构
2. HashMap如何解决Hash冲突的，在java8中为什么要使用红黑树。
3. HashMap高并发场景下会产生什么问题
4. HashMap的环是怎么产生的，它的原理是什么，java8中还会出现吗，说说为什么, 生产环境会出现什么问题
5. HashMap除了会产生环还有什么其他问题，并说明原理.
5. HashMap和TreeMap的区别是什么，分别应用于什么场景
6. HashTable和HashMap在哪些地方有所不同
7. 解决hash冲突的方式有哪些，描述一下。

## 多线程篇

### 线程基础
1. start和run什么区别
2. future用过吗，什么原理
3. 描述一下BIO和NIO区别，描述NIO原理和AIO原理
4. 阻塞队列有哪些。
5. ArrayBlokingQueue和LinkedBlockingQueue的原理是什么

### 线程池
1. 线程池了解吗，Java中默认有哪些线程池
2. 介绍一下FixedThreadPool的实现原理
3. FixedThreadPool中使用的队列是有界的还是无界的
4. FixedThreadPool在高并发场景下会产生什么问题
5. CachedThreadPool和FixedThreadPool的区别
6. CachedThreadPool在高并发场景下有什么问题
7. 如何自定义一个线程池
8. 自定义线程池时都可以使用哪些队列，BlockingQueue都有哪些实现。
9. 如何创建一个线程,start和run的区别是什么
### LockSupport
1. 锁的种类有哪些、锁的状态有哪些
2. synchronized有什么作用
3. synchronized是公平锁吗
4. synchronized的实现原理
5. 什么是死锁，怎么产生的，来段可以引发死锁的代码。
6. 如何防止出现死锁，具体应该怎么操作。
7. ReentrantLock有什么用,它和synchronized有什么区别,它是公平锁吗
8. ReentrantLock实现原理,AQS原理和数据结构也描述一下

### JUC
1. AtomicInteger原理
2. CAS底层实现
3. CAS都有哪些问题，需要怎么解决这些问题
4. ConcurrentHashMap数据结构及原理
5. ConcurrentHashMap在java7中和java8中的区别，为什么java8放弃了分段锁, synchronized的优势是什么。
6. CountDownLatch原理及其主要结构
7. CountDownLatch应用场景及使用方法
8. Semaphore的作用是什么，应用场景是什么。知道怎么实现的吗?
9. CyclicBarrier是干什么的和CountDownLatch的区别是什么，它的应用场景是什么。
### Netty
1. 什么是零拷贝，原理是什么。零拷贝和正常的相比优势是什么。

## Jvm篇
### Java对象模型（Klass）
1. 描述一下Klass，里面都存了什么内容.
### Java的内存模型
1. 介绍一下Java内存模型
3. 虚拟机栈中存放了哪些内容
4. 本地方法栈和虚拟机栈的区别
5. 基本类型的全局变量存放在哪个区域
6. 描述一下Java内存分配机制
7. JVM栈什么时候创建的，怎么创建的。
8. 堆中的的对象结构是什么，都有哪些内容。
9. metaspace有gc吗
10. 堆的扩容机制是什么，堆是一点一点扩容的还是一下就扩到Xmx大小了
11. 类是怎么加载的，加载的过程是什么。
### 垃圾回收(GC)
1. Java都有哪些垃圾回收算法，特点是什么。
2. 描述一下分代垃圾回收
3. JVM都有哪些垃圾收集器，特点是什么。
4. Java堆是如何扩容的，是一点一点扩容还是一下就扩容到最大了。原理是什么，怎么发现的。

## mysql篇
1. Innodb表数据结构
2. Innodb中都有哪些索引，原理是什么
3. 聚集索引和辅助索引的原理、特点和区别
4. B+树与B树的区别，优点是什么。
5. 如何优化SQL，具体的策略有哪些
6. 组合索引基本原理，及组合索引的规则。
7. 使用mysql时怎么确定索引列，有具体量化的指标吗
8. mvcc原理
9. 描述数据库隔离级别，可重复读的隔离级别会出现幻读吗。
10. 什么是间隙锁，为什么需要间隙锁
11. 什么是回表，什么情况下会回表
12. MySQL的可重复读级别能解决幻读吗

## redis篇
1. redis用过没，干什么用的，了解redis实现原理吗?
2. sds的数据结构，sds有什么特点
3. dict的数据结构，它的特点是什么
4. dict如何进行扩容的，描述一下，扩容期间会产生什么问题
5. 用redis怎么实现分布式锁，setnx有什么问题。
6. 分布式锁有超时时间吗，如果超时时间过了怎么办。
7. redis如何实现lru的。

## dubbo篇
1. dubbo中实现了哪些通信协议
2. dubbo默认的通信协议是哪个原理是什么
3. dubbo用的哪个注册中心，为什么用它当数据中心。有没有出现过问题.
4. zookeeper当注册中心会产生的问题。
5. dubbo心跳检测机制，以及如何发现其他应用宕机的。
6. dubbo中有哪些负载均衡算法
7. dubbo的一致性hash用来解决负载均衡中的哪些问题。

## zookeeper篇
1. zookeeper如何实现高可用的
2. zookeeper会出现雪崩吗，什么情况下会雪崩。
3. zookeeper选举策略

## rocketMQ篇
1. 如何保证消息幂等,出现了重复发送的消息怎么办.
2. RocketMQ事务消息原理
3. RocketMQ延迟消息原理

## spring篇
1. spring的事务传播级别
2. spring的IOC与AOP，描述动态代理

## 架构篇
1. 描述一下分布式事务，和具体的应用场景。
2. TCC的实际应用怎么实现的，与TC的区别是什么。
3. 限流是什么，什么情况下会限流，限流只有流量访问频率超过阈值才会触发吗
4. 描述一下CAP，可以是CA吗，为什么。如果需要保证一致性有什么方案吗。
5. 怎么实现限流熔断降级的，用过相关框架吗？
6. 描述一下两阶段提交，如果不止有应用A调应用B，又出现了应用C的情况呢？
7. 什么是反向代理，什么是正向代理。
8. 秒杀系统中超卖少卖的问题和解决方案。
9. 如何在java中设计一个lru缓存。 linkedhashmap 不是线程安全的，有线程安全的lru吗。
10. 100w用户设置积分排行榜，怎么实现。有没有实时性比较高的解决方案。
11. 如何用redis实现一个可重入的锁
12. 什么是缓存穿透，什么是缓存击穿。如何解决。

## mybatis篇
1. 描述一下mybatis的执行原理
2. mybaits拦截器原理

## 算法篇
1. （链表、树、动态规划三选一，选了链表）快速排序单向链表，要求时间复杂度n(logn)，空间复杂度o(1).lettcode 148
答案：
```java
class Solution {
     public ListNode sortList(ListNode head) {
        int n = 0;
        ListNode tempHead = new ListNode(0);
        tempHead.next = head;
        while (head != null) {
            head = head.next;
            n++;
        }
        for (int step = 1; step < n; step = step << 1) {
            ListNode prev = tempHead;
            ListNode cur = tempHead.next;
            while (cur != null) {
                ListNode left = cur;
                ListNode right = split(left, step);
                cur = split(right, step);
                prev = merge(prev, left, right);
            }
        }
        return tempHead.next;
    }

    private ListNode split(ListNode head, int step) {
        if (head == null) {
            return null;
        }
        for (int i = 1; head.next != null && i < step; i++) {
            head = head.next;
        }

        ListNode right = head.next;
        head.next = null;
        return right;
    }

    private ListNode merge(ListNode head, ListNode left, ListNode right) {
        while (left != null && right != null) {
            if (left.val <= right.val) {
                head.next = left;
                left = left.next;
            } else{
                head.next = right;
                right = right.next;
            }
            head = head.next;
        }
        head.next = left == null ? right : left;
        while (head.next != null) {
            head = head.next;
        }
        return head;
    }
}
```
2. 二分查找
3. 查找重复次数最多的字符串

