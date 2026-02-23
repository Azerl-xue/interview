# Java核心 - 多线程与并发

> 基于你的项目经验（RabbitMQ异步、多线程文档处理）整理的并发知识

---

## 📚 学习目标

- 理解线程生命周期与ThreadLocal
- 掌握synchronized和volatile关键字
- 理解CAS与AQS原理
- 掌握线程池配置与使用
- 熟练使用并发工具类
- 理解JMM内存模型
- 掌握项目中的并发应用场景

---

## 1. 线程基础

### 1.1 线程生命周期

```
NEW（新建）→ start() → RUNNABLE（就绪/运行）
RUNNABLE → run()结束 → TERMINATED（终止）
RUNNABLE → sleep()/wait() → WAITING/TIMED_WAITING（等待）
RUNNABLE → 获取锁失败 → BLOCKED（阻塞）
```

**代码示例**

```java
public class ThreadLifecycle {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            System.out.println("Thread is running");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        System.out.println("State: " + thread.getState());
        thread.start();
        System.out.println("State: " + thread.getState());
        
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("State: " + thread.getState());
    }
}
```

### 1.2 创建线程的方式

**1. 继承Thread类**

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running");
    }
}
```

**2. 实现Runnable接口（推荐）**

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable running");
    }
}

Thread thread = new Thread(new MyRunnable());
```

**3. 使用线程池（最佳实践）**

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> {
    System.out.println("Task running");
});
```

### 1.3 ThreadLocal

**作用**：线程局部变量，每个线程独立一份副本

**应用场景**：数据库连接、Session、用户信息

```java
public class ThreadLocalExample {
    private static ThreadLocal<String> userContext = new ThreadLocal<>();
    
    public static void setUser(String user) {
        userContext.set(user);
    }
    
    public static String getUser() {
        return userContext.get();
    }
    
    public static void clear() {
        userContext.remove();
    }
}
```

**注意**：线程池中使用ThreadLocal要记得remove，避免内存泄漏

---

## 2. synchronized关键字

### 2.1 三种使用方式

**1. 修饰实例方法**

```java
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
}
```

**2. 修饰静态方法**

```java
public class StaticCounter {
    private static int count = 0;
    
    public static synchronized void increment() {
        count++;
    }
}
```

**3. 修饰代码块**

```java
public class BlockCounter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
}
```

### 2.2 synchronized特性

- 原子性：互斥访问
- 可见性：释放锁时刷新到主内存
- 可重入性：同一个线程可多次获取同一把锁

### 2.3 锁升级（JDK 1.6优化）

```
无锁（偏向锁）→ 轻量级锁（CAS自旋）→ 重量级锁（互斥）
```

---

## 3. volatile关键字

### 3.1 volatile特性

**可见性**：保证线程间可见

```java
public class VolatileExample {
    private volatile boolean flag = false;
    
    public void setFlag() {
        flag = true;
    }
    
    public void checkFlag() {
        while (!flag) {
        }
    }
}
```

**有序性**：禁止指令重排序

```java
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**不保证原子性**

```java
public class VolatileCounter {
    private volatile int count = 0;
    
    public void increment() {
        count++;
    }
}
```

`count++`不是原子操作，需要使用AtomicInteger

---

## 4. CAS与原子类

### 4.1 CAS（Compare And Swap）

**原理**：比较并交换，硬件级别的原子操作

**优点**：无锁，性能高
**`缺点`**：ABA问题、只能保证一个共享变量原子性

### 4.2 ABA问题

**解决方案**：使用版本号（AtomicStampedReference）

```java
AtomicStampedReference<Integer> ref = 
    new AtomicStampedReference<>(100, 1);

int stamp = ref.getStamp();
ref.compareAndSet(100, 101, stamp, stamp + 1);
```

### 4.3 原子类

**AtomicInteger**

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
count.getAndIncrement();
count.compareAndSet(100, 101);
count.addAndGet(10);
```

**AtomicReference**

```java
AtomicReference<User> userRef = new AtomicReference<>();
userRef.compareAndSet(null, newUser);
```

**AtomicIntegerArray**

```java
AtomicIntegerArray array = new AtomicIntegerArray(10);
array.getAndIncrement(0);
```

---

## 5. AQS原理

### 5.1 AQS（AbstractQueuedSynchronizer）

**核心思想**：基于CLH队列的变体实现的FIFO队列

**核心变量**

```java
private volatile int state;
private transient volatile Node head;
private transient volatile Node tail;
```

**核心方法**

- acquire(int arg)：获取锁
- release(int arg)：释放锁

### 5.2 ReentrantLock

**使用示例**

```java
ReentrantLock lock = new ReentrantLock();

public void doSomething() {
    lock.lock();
    try {
        System.out.println("Do something");
    } finally {
        lock.unlock();
    }
}
```

**公平锁与非公平锁**

```java
ReentrantLock fairLock = new ReentrantLock(true);
ReentrantLock unfairLock = new ReentrantLock(false);
```

**tryLock**

```java
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try {
    } finally {
        lock.unlock();
    }
}
```

### 5.3 读写锁ReentrantReadWriteLock

**应用场景**：读多写少

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
ReentrantReadWriteLock.ReadLock readLock = rwLock.readLock();
ReentrantReadWriteLock.WriteLock writeLock = rwLock.readLock();

public void read() {
    readLock.lock();
    try {
    } finally {
        readLock.unlock();
    }
}

public void write() {
    writeLock.lock();
    try {
    } finally {
        writeLock.unlock();
    }
}
```

---

## 6. 线程池

### 6.1 ThreadPoolExecutor参数

```java
public ThreadPoolExecutor(
    int corePoolSize,      // 核心线程数
    int maximumPoolSize,   // 最大线程数
    long keepAliveTime,    // 线程空闲存活时间
    TimeUnit unit,         // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,         // 线程工厂
    RejectedExecutionHandler handler      // 拒绝策略
)
```

### 6.2 参数配置原则

**核心线程数**：CPU密集型 = CPU核心数 + 1；IO密集型 = CPU核心数 * 2

**最大线程数**：核心线程数 + 备用线程数

**任务队列**：
- ArrayBlockingQueue：有界队列
- LinkedBlockingQueue：无界队列（慎用）
- SynchronousQueue：不存储任务

**拒绝策略**：
- AbortPolicy（默认）：抛出异常
- CallerRunsPolicy：调用者线程执行
- DiscardPolicy：丢弃任务
- DiscardOldestPolicy：丢弃最老任务

### 6.3 线程池使用示例

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                      // 核心线程数
    10,                     // 最大线程数
    60L,                    // 空闲存活时间
    TimeUnit.SECONDS,       // 时间单位
    new LinkedBlockingQueue<>(100),  // 任务队列
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);

Future<String> future = executor.submit(() -> {
    return "task result";
});

String result = future.get();
```

### 6.4 常用线程池

```java
// 固定大小线程池
ExecutorService fixedPool = Executors.newFixedThreadPool(10);

// 缓存线程池（不推荐生产使用）
ExecutorService cachedPool = Executors.newCachedThreadPool();

// 单线程池
ExecutorService singlePool = Executors.newSingleThreadExecutor();

// 定时任务线程池
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(5);

scheduledPool.scheduleAtFixedRate(() -> {
}, 0, 1, TimeUnit.SECONDS);
```

---

## 7. 并发工具类

### 7.1 CountDownLatch

**作用**：等待多个线程完成

```java
public class DocumentProcessor {
    public void process(List<Document> documents) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(documents.size());
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        for (Document doc : documents) {
            executor.submit(() -> {
                try {
                    processDocument(doc);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        executor.shutdown();
    }
}
```

### 7.2 CyclicBarrier

**作用**：线程同步，到达屏障点后同时执行

```java
public class BatchProcessor {
    public void process() {
        CyclicBarrier barrier = new CyclicBarrier(5, () -> {
            System.out.println("所有线程到达屏障点");
        });
        
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    System.out.println("线程执行任务");
                    barrier.await();
                    System.out.println("继续执行");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### 7.3 Semaphore

**作用**：控制并发访问资源数量

```java
public class ConnectionPool {
    private Semaphore semaphore = new Semaphore(10);
    
    public Connection getConnection() {
        semaphore.acquire();
        return createConnection();
    }
    
    public void releaseConnection(Connection conn) {
        releaseConnection(conn);
        semaphore.release();
    }
}
```

### 7.4 Exchanger

**作用**：线程间交换数据

```java
Exchanger<String> exchanger = new Exchanger<>();

new Thread(() -> {
    try {
        String data = "Thread A data";
        String received = exchanger.exchange(data);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}).start();
```

---

## 8. CompletableFuture

### 8.1 异步编程

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});

CompletableFuture<String> result = future
    .thenApply(s -> s + " World")
    .thenApplyAsync(s -> s + "!")
    .exceptionally(e -> {
        log.error("Error", e);
        return "Default";
    });

result.thenAccept(s -> System.out.println(s));
```

### 8.2 组合多个Future

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");

// allOf：等待所有完成
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);

// anyOf：等待任意一个完成
CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2);

// thenCombine：组合两个Future
CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + b);
```

### 8.3 项目应用：文档异步处理

```java
public class DocumentService {
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public CompletableFuture<ProcessResult> processAsync(Document doc) {
        return CompletableFuture.supplyAsync(() -> {
            return parseDocument(doc);
        }, executor).thenApplyAsync(result -> {
            return analyzeContent(result);
        }, executor).thenApplyAsync(result -> {
            return saveToDatabase(result);
        }, executor);
    }
    
    public List<ProcessResult> batchProcess(List<Document> docs) {
        List<CompletableFuture<ProcessResult>> futures = docs.stream()
            .map(this::processAsync)
            .collect(Collectors.toList());
        
        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }
}
```

---

## 9. 死锁与活锁

### 9.1 死锁条件

1. 互斥条件
2. 请求与保持
3. 不剥夺条件
4. 循环等待

### 9.2 避免死锁

**1. 锁顺序**

```java
public void transfer(Account from, Account to, int amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    
    synchronized (first) {
        second.lock();
        try {
            first.debit(amount);
            second.credit(amount);
        } finally {
            second.unlock();
        }
    }
}
```

**2. tryLock**

```java
if (lock1.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        if (lock2.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
            } finally {
                lock2.unlock();
            }
        }
    } finally {
        lock1.unlock();
    }
}
```

### 9.3 检测死锁

```java
public class DeadlockDetector {
    public void detectDeadlock() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        
        if (deadlockedThreads != null) {
            ThreadInfo[] threadInfos = 
                threadMXBean.getThreadInfo(deadlockedThreads);
            for (ThreadInfo info : threadInfos) {
                System.out.println(info);
            }
        }
    }
}
```

---

## 10. JMM（Java内存模型）

### 10.1 主内存与工作内存

```
主内存
├── 线程1工作内存
├── 线程2工作内存
└── ...
```

### 10.2 三大特性

**原子性**：synchronized、Lock、Atomic

**可见性**：volatile、synchronized、final

**有序性**：volatile、synchronized、happens-before

### 10.3 happens-before

- 程序顺序规则
- 管程锁定规则
- volatile规则
- 线程启动规则
- 线程终止规则
- 线程中断规则
- 对象终结规则
- 传递性

---

## 11. 项目中的应用场景

### 11.1 ERP系统：异步审批（RabbitMQ）

```java
public class ApprovalService {
    private RabbitTemplate rabbitTemplate;
    private ExecutorService executor = Executors.newFixedThreadPool(5);
    
    public void submitApproval(ApprovalRequest request) {
        executor.submit(() -> {
            rabbitTemplate.convertAndSend("approval.queue", request);
        });
    }
    
    @RabbitListener(queues = "approval.queue")
    public void processApproval(ApprovalRequest request) {
        ApprovalProcess process = new ApprovalProcess(request);
        process.start();
    }
}
```

### 11.2 文档处理：多线程并发解析

```java
public class BatchDocumentProcessor {
    private ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4, 10, 60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(100)
    );
    
    public List<ProcessResult> process(List<Document> documents) {
        CountDownLatch latch = new CountDownLatch(documents.size());
        List<ProcessResult> results = Collections.synchronizedList(new ArrayList<>());
        
        for (Document doc : documents) {
            executor.submit(() -> {
                try {
                    ProcessResult result = processDocument(doc);
                    results.add(result);
                } catch (Exception e) {
                    log.error("处理失败", e);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        return results;
    }
    
    private ProcessResult processDocument(Document doc) {
        return CompletableFuture.supplyAsync(() -> parse(doc))
            .thenApply(this::analyze)
            .thenApply(this::save)
            .join();
    }
}
```

### 11.3 设备权限管理：并发安全

```java
public class DevicePermissionService {
    private ConcurrentHashMap<String, Set<String>> devicePermissions = 
        new ConcurrentHashMap<>();
    private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    
    public boolean hasPermission(String deviceId, String userId) {
        rwLock.readLock().lock();
        try {
            Set<String> users = devicePermissions.get(deviceId);
            return users != null && users.contains(userId);
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    public void grantPermission(String deviceId, String userId) {
        rwLock.writeLock().lock();
        try {
            devicePermissions.computeIfAbsent(deviceId, k -> new HashSet<>())
                .add(userId);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

---

## 面试高频题

### 基础题

1. 创建线程的方式有哪些？推荐哪种？
2. ThreadLocal的作用和内存泄漏问题？
3. synchronized和volatile的区别？
4. synchronized的三种使用方式？
5. volatile的特性是什么？
6. AtomicInteger如何保证原子性？
7. CAS的原理和ABA问题？
8. ReentrantLock和synchronized的区别？

### 中等题

1. 线程池的核心参数及其含义？
2. 线程池的拒绝策略有哪些？
3. CountDownLatch和CyclicBarrier的区别？
4. Semaphore的应用场景？
5. CompletableFuture的使用场景？
6. 死锁的四个必要条件？
7. 如何避免死锁？
8. JMM的三大特性？

### 高级题

1. AQS的原理是什么？
2. ReentrantReadWriteLock的实现原理？
3. 手写一个简单的线程池？
4. 实现一个阻塞队列？
5. 实现生产者-消费者模型？
6. 手写一个简单的分布式锁？

---

## 学习建议

1. **重点掌握**：synchronized、volatile、线程池、并发工具类
2. **理解原理**：AQS、CAS、JMM
3. **实践应用**：RabbitMQ异步、文档并发处理
4. **准备代码**：手写线程池、阻塞队列、生产者消费者

**参考文档**：
- Java并发编程实战
- 深入理解Java虚拟机
- JDK源码

---

**更新进度到学习进度.md**