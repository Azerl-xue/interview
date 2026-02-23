# Java核心 - 基础知识

> 基于你的技术栈和项目经验整理的Java基础知识

---

## 📚 学习目标

- 掌握Java语言核心特性
- 理解面向对象设计原则
- 熟练使用集合框架及其底层实现
- 掌握异常处理与泛型反射
- 熟悉Java8+新特性
- 理解IO/NIO模型

---

## 1. 面向对象

### 1.1 封装、继承、多态

**封装**
- private修饰符隐藏实现细节
- getter/setter提供访问接口
- 遵循最小权限原则

```java
public class Device {
    private String deviceId;
    private String status;
    
    public Device getDeviceInfo() {
        return new DeviceDTO(this.deviceId, this.status);
    }
}
```

**继承**
- extends关键字
- super关键字调用父类
- 方法重写@Override

```java
public class SmartDevice extends Device {
    @Override
    public void activate() {
        super.activate();
        smartConnect();
    }
}
```

**多态**
- 编译时多态：方法重载
- 运行时多态：方法重写、接口实现
- 父类引用指向子类对象

```java
Device device = new SmartDevice();
device.activate();
```

### 1.2 抽象类与接口

**抽象类**
- 不能实例化
- 可以有抽象方法和普通方法
- 单继承

```java
public abstract class ApprovalProcess {
    public abstract void approve();
    public void notify() {}
}
```

**接口**
- 默认public abstract
- 可以有默认方法（Java8+）
- 可以多实现
- 可以有静态方法（Java8+）
- 可以有私有方法（Java9+）

```java
public interface DataSync {
    void sync();
    
    default void syncWithRetry() {
        int retry = 0;
        while (retry < 3) {
            try {
                sync();
                break;
            } catch (Exception e) {
                retry++;
            }
        }
    }
}
```

### 1.3 面向对象原则

- 单一职责原则（SRP）
- 开里闭原则（OCP）
- 里氏替换原则（LSP）
- 接口隔离原则（ISP）
- 依赖倒置原则（DIP）

---

## 2. 集合框架

### 2.1 List

**ArrayList**
- 底层：动态数组
- 特点：随机访问快，插入删除慢
- 扩容机制：默认10，每次扩容1.5倍

```java
List<Device> devices = new ArrayList<>();
devices.add(device);
Device d = devices.get(0);
```

**LinkedList**
- 底层：双向链表
- 特点：插入删除快，随机访问慢
- 可用作队列和栈

```java
LinkedList<String> queue = new LinkedList<>();
queue.offer("task1");
queue.poll();
```

**Vector**
- 线程安全（synchronized）
- 性能较差，一般不用

### 2.2 Set

**HashSet**
- 底层：HashMap
- 特点：无序、唯一
- 允许null

```java
Set<String> permissions = new HashSet<>();
permissions.add("READ");
permissions.add("WRITE");
```

**LinkedHashSet**
- 底层：LinkedHashMap
- 特点：有序（插入顺序）、唯一

**TreeSet**
- 底层：TreeMap（红黑树）
- 特点：有序（自然排序/自定义排序）
- 实现Comparable或Comparator

```java
TreeSet<Contract> contracts = new TreeSet<>((c1, c2) -> 
    c1.getCreateTime().compareTo(c2.getCreateTime())
);
```

### 2.3 Map

**HashMap**
- 底层：数组+链表+红黑树
- 线程不安全
- 键值允许null
- 扩容：默认16，负载因子0.75，链表转红黑树阈值8

```java
Map<String, String> cache = new HashMap<>();
cache.put("device:001", "status:active");
String status = cache.get("device:001");
```

**ConcurrentHashMap**
- 线程安全
- 分段锁（1.7）/CAS+synchronized（1.8）
- 更高性能

```java
ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();
cache.putIfAbsent(key, value);
```

**LinkedHashMap**
- 保持插入顺序或访问顺序
- 可用于实现LRU缓存

**TreeMap**
- 基于红黑树
- 有序存储

### 2.4 Queue/Deque

**ArrayDeque**
- 数组实现的双端队列
- 性能优于LinkedList

**PriorityQueue**
- 优先级队列
- 最小堆或最大堆

```java
PriorityQueue<Task> queue = new PriorityQueue<>((t1, t2) -> 
    t1.getPriority() - t2.getPriority()
);
```

---

## 3. 异常处理

### 3.1 异常体系

```
Throwable
├── Error（系统级错误，不可恢复）
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception（程序级异常，可恢复）
    ├── IOException（检查异常）
    └── RuntimeException（运行时异常）
        ├── NullPointerException
        ├── IndexOutOfBoundsException
        ├── ArithmeticException
        └── ClassCastException
```

### 3.2 异常处理原则

**try-catch-finally**

```java
try {
    processDocument();
} catch (DocumentParseException e) {
    log.error("文档解析失败", e);
    throw new BusinessException("文档格式错误");
} finally {
    closeResource();
}
```

**异常处理最佳实践**

1. 不要捕获Exception太宽泛
2. 不要吞掉异常
3. 不要在finally中return
4. 自定义业务异常
5. 异常链

```java
try {
    syncData();
} catch (SyncException e) {
    throw new BusinessException("数据同步失败", e);
}
```

### 3.3 自定义异常

```java
public class BusinessException extends RuntimeException {
    private String errorCode;
    
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

---

## 4. 泛型

### 4.1 泛型类与方法

**泛型类**

```java
public class Result<T> {
    private T data;
    private String code;
    private String message;
    
    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.setData(data);
        result.setCode("200");
        return result;
    }
}
```

**泛型方法**

```java
public <T> T find(Class<T> clazz, String id) {
    return mapper.selectById(id);
}
```

### 4.2 通配符

**? 无界通配符**

```java
void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}
```

**? extends T 上界通配符（读操作）**

```java
List<? extends Device> devices = new ArrayList<SmartDevice>();
Device d = devices.get(0);
devices.add(new SmartDevice());
```

**? super T 下界通配符（写操作）**

```java
void addDevices(List<? super SmartDevice> list) {
    list.add(new SmartDevice());
}
```

### 4.3 类型擦除

Java泛型是伪泛型，编译时类型擦除

```java
List<String> list1 = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();
System.out.println(list1.getClass() == list2.getClass());
```

---

## 5. 反射

### 5.1 基本使用

**获取Class对象**

```java
Class<?> clazz1 = User.class;
Class<?> clazz2 = user.getClass();
Class<?> clazz3 = Class.forName("com.example.User");
```

**获取构造方法**

```java
Constructor<?> constructor = clazz.getConstructor(String.class);
Object obj = constructor.newInstance("name");
```

**获取方法**

```java
Method method = clazz.getMethod("getId");
Object value = method.invoke(obj);

Method setMethod = clazz.getMethod("setName", String.class);
setMethod.invoke(obj, "newName");
```

**获取字段**

```java
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);
String value = (String) field.get(obj);
```

### 5.2 反射应用场景

- 框架（Spring、MyBatis）
- 注解处理器
- 动态代理
- 序列化/反序列化

### 5.3 反射性能

反射比直接调用慢10-100倍，注意：
- 缓存反射结果
- 避免在循环中频繁使用反射

---

## 6. 注解

### 6.1 内置注解

- @Override：重写方法
- @Deprecated：标记过时
- @SuppressWarnings：抑制警告
- @FunctionalInterface：函数式接口

### 6.2 元注解

- @Target：注解作用位置
- @Retention：注解保留策略
- @Documented：生成文档
- @Inherited：可继承

```java
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Cache {
    String key() default "";
    int timeout() default 3600;
}
```

### 6.3 注解处理

```java
@Cache(key = "user:${id}", timeout = 1800)
public User getUser(String id) {
    return userMapper.selectById(id);
}

@Aspect
public class CacheAspect {
    @Around("@annotation(cache)")
    public Object cache(ProceedingJoinPoint pjp, Cache cache) {
        String key = parseKey(cache.key(), pjp.getArgs());
        Object value = redis.get(key);
        if (value != null) {
            return value;
        }
        value = pjp.proceed();
        redis.set(key, value, cache.timeout());
        return value;
    }
}
```

---

## 7. Java 8+ 新特性

### 7.1 Lambda表达式

```java
List<Contract> contracts = getContracts();

List<Contract> approved = contracts.stream()
    .filter(c -> c.getStatus() == Status.APPROVED)
    .collect(Collectors.toList());

contracts.stream()
    .sorted((c1, c2) -> c1.getCreateTime().compareTo(c2.getCreateTime()))
    .forEach(c -> System.out.println(c.getId()));
```

### 7.2 Stream API

**常用操作**

```java
List<String> names = devices.stream()
    .filter(d -> d.isActive())
    .map(Device::getName)
    .distinct()
    .sorted()
    .collect(Collectors.toList());

long count = devices.stream()
    .filter(d -> d.getType() == Type.SMART)
    .count();

Optional<Device> first = devices.stream()
    .findFirst();
```

**并行流**

```java
List<ProcessedData> results = dataList.parallelStream()
    .map(this::process)
    .collect(Collectors.toList());
```

### 7.3 Optional

```java
Optional<User> user = userRepository.findById(id);

String name = user
    .map(User::getName)
    .orElse("unknown");

user.ifPresent(u -> System.out.println(u.getName()));

User u = user.orElseThrow(() -> new NotFoundException("用户不存在"));
```

### 7.4 其他新特性

**默认方法**

```java
interface DataSync {
    void sync();
    
    default void syncWithRetry(int maxRetry) {
        int retry = 0;
        while (retry < maxRetry) {
            try {
                sync();
                return;
            } catch (Exception e) {
                retry++;
            }
        }
    }
}
```

**本地变量类型推断（Java10+）**

```java
var device = getDevice();
var list = new ArrayList<Device>();
```

---

## 8. IO与NIO

### 8.1 IO模型

**BIO（阻塞IO）**
- 传统IO
- 一个线程处理一个连接
- 适合连接数不多的场景

**NIO（非阻塞IO）**
- 同步非阻塞
- 多路复用
- 适合高并发场景

**AIO（异步IO）**
- 异步非阻塞
- Proactor模式

### 8.2 NIO核心组件

**Buffer（缓冲区）**

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.put("hello".getBytes());
buffer.flip();
byte[] data = new byte[buffer.remaining()];
buffer.get(data);
```

**Channel（通道）**

```java
FileChannel channel = new FileInputStream("file.txt").getChannel();
ByteBuffer buffer = ByteBuffer.allocate(1024);
channel.read(buffer);
buffer.flip();
channel.write(buffer);
```

**Selector（选择器）**

```java
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.socket().bind(new InetSocketAddress(8080));
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable()) {
        }
    }
}
```

---

## 9. String

### 9.1 String特性

- 不可变（final类，final字段）
- 线程安全
- 字符常量池

### 9.2 String vs StringBuilder vs StringBuffer

| 类型 | 可变性 | 线程安全 | 性能 |
|------|--------|---------|------|
| String | 不可变 | 安全 | 低 |
| StringBuilder | 可变 | 不安全 | 高 |
| StringBuffer | 可变 | 安全 | 中（synchronized） |

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append("data");
}
String result = sb.toString();
```

### 9.3 常用方法

```java
String s = "hello world";

boolean starts = s.startsWith("hello");
boolean ends = s.endsWith("world");
boolean contains = s.contains("lo wo");

String replaced = s.replace("world", "java");
String substring = s.substring(6, 11);
String[] split = s.split(" ");

boolean empty = s.isEmpty();
int length = s.length();
char[] chars = s.toCharArray();

String upper = s.toUpperCase();
String lower = s.toLowerCase();
String trim = s.trim();
```

---

## 10. 基本数据类型与包装类

### 10.1 基本类型

| 类型 | 字节 | 包装类 | 默认值 |
|------|------|--------|--------|
| byte | 1 | Byte | 0 |
| short | 2 | Short | 0 |
| int | 4 | Integer | 0 |
| long | 8 | Long | 0L |
| float | 4 | Float | 0.0f |
| double | 8 | Double | 0.0d |
| char | 2 | Character | '\u0000' |
| boolean | - | Boolean | false |

### 10.2 自动装箱与拆箱

```java
Integer a = 100;
int b = a;
```

**注意：**
- -128~127之间缓存
- 包装类比较用equals
- NPE风险

```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b);

Integer c = 200;
Integer d = 200;
System.out.println(c == d);
```

---

## 面试高频题

### 基础题

1. Java有哪些基本数据类型？各占多少字节？
2. ==和equals的区别？
3. String、StringBuilder、StringBuffer的区别？
4. 接口和抽象类的区别？
5. 重载和重写的区别？
6. Java集合框架有哪些？
7. ArrayList和LinkedList的区别？
8. HashMap和ConcurrentHashMap的区别？
9. 异常处理的最佳实践？
10. 泛型的通配符？

### 中等题

1. HashMap的底层实现原理？
2. HashMap扩容机制？
3. 为什么HashMap线程不安全？
4. 泛型类型擦除？
5. 反射的应用场景和性能问题？
6. Java8新特性有哪些？
7. Stream API常用操作？
8. Optional的使用场景？
9. IO/NIO的区别？
10. BIO、NIO、AIO的区别？

### 高级题

1. 自定义一个简单的HashMap？
2. 实现一个LRU缓存？
3. 如何实现线程安全的单例？
4. 自定义注解处理器？
5. 反射实现简单的ORM？
6. 实现一个简单的Spring IOC容器？

---

## 学习建议

1. **重点掌握**：HashMap、ArrayList、多线程、反射、Stream
2. **理解原理**：HashMap底层、ArrayList扩容、String不可变
3. **实践应用**：实际项目中使用Stream、Optional、Lambda
4. **准备代码**：手写简单集合类、单例、工具类

**参考文档**：
- Java官方文档
- JDK源码
- Effective Java

---

**更新进度到学习进度.md**