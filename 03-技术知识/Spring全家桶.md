# Spring全家桶

> 基于你的项目经验（Spring Boot、MyBatis、事务管理）整理的Spring框架知识

---

## 📚 学习目标

- 理解Spring IOC容器原理
- 掌握Bean生命周期
- 理解依赖注入方式
- 掌握Spring Boot自动配置
- 熟练使用MyBatis高级特性
- 理解事务管理与传播行为
- 掌握AOP原理与应用

---

## 1. Spring IOC

### 1.1 IOC（控制反转）

**概念**：将对象的创建和管理交给Spring容器

**DI（依赖注入）**：IOC的具体实现

### 1.2 Bean创建方式

**XML方式**

```xml
<bean id="userService" class="com.example.UserServiceImpl">
    <property name="userDao" ref="userDao"/>
</bean>
```

**注解方式**

```java
@Componemt
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

**Java Config方式**

```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService(UserDao userDao) {
        return new UserServiceImpl(userDao);
    }
}
```

### 1.3 Bean作用域

| 作用域 | 说明 |
|--------|------|
| singleton | 单例（默认） |
| prototype | 多例，每次getBean创建新实例 |
| request | 一次请求创建一个实例（Web） |
| session | 一次会话创建一个实例（Web） |

```java
@Scope("prototype")
@Component
public class PrototypeBean {
}
```

### 1.4 Bean生命周期

**完整生命周期**

```
实例化 → 设置属性值 → BeanNameAware → BeanFactoryAware 
→ ApplicationContextAware → BeanPostProcessor前置处理 
→ InitializingBean → init-method → BeanPostProcessor后置处理 
→ 使用 → DisposableBean → destroy-method
```

**代码示例**

```java
@Component
public class UserBean implements InitializingBean, DisposableBean {
    @Autowired
    private UserDao userDao;
    
    @PostConstruct
    public void init() {
        System.out.println("初始化");
    }
    
    @PreDestroy
    public void destroy() {
        System.out.println("销毁");
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("属性设置后");
    }
}
```

---

## 2. 依赖注入

### 2.1 三种注入方式

**构造器注入（推荐）**

```java
@Component
public class UserService {
    private final UserDao userDao;
    
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

**Setter注入**

```java
@Component
public class UserService {
    private UserDao userDao;
    
    @Autowired
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

**字段注入**

```java
@Component
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

### 2.2 @Autowired与@Resource

| 注解 | 来源 | By类型 | By名称 |
|------|------|--------|--------|
| @Autowired | Spring | 是 | 是（配合@Qualifier） |
| @Resource | JDK | 否 | 是 |

```java
@Autowired
@Qualifier("userDaoImpl")
private UserDao userDao;
```

### 2.3 循环依赖

**循环依赖场景**：A依赖B，B依赖A

**Spring如何解决**：
- 单例Bean：使用三级缓存
- 原型Bean：抛出异常

**三级缓存**

```
一级缓存：singletonObjects（完全初始化的Bean）
二级缓存：earlySingletonObjects（提前曝光的Bean）
三级缓存：singletonFactories（Bean工厂）
```

---

## 3. Spring Boot

### 3.1 自动配置原理

**核心注解**

```java
@SpringBootApplication
= @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
```

**自动配置流程**

1. `@EnableAutoConfiguration`导入`AutoConfigurationImportSelector`
2. 加载`META-INF/spring.factories`中的自动配置类
3. 根据`@Conditional`条件过滤
4. 注册Bean到容器

**自定义自动配置**

```java
@Configuration
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnProperty(name = "redis.enabled", havingValue = "true")
public class RedisAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public RedisTemplate<String, String> redisTemplate(
        RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }
}
```

### 3.2 Starter原理

**Starter结构**

```
my-spring-boot-starter/
├── pom.xml
└── src/main/resources/
    └── META-INF/
        └── spring.factories
```

**spring.factories**

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

### 3.3 条件注解

| 注解 | 条件 |
|------|------|
| @ConditionalOnClass | 类路径存在 |
| @ConditionalOnMissingClass | 类路径不存在 |
| @ConditionalOnBean | 容器中有指定Bean |
| @ConditionalOnMissingBean | 容器中无指定Bean |
| @ConditionalOnProperty | 配置属性匹配 |
| @ConditionalOnExpression | SpEL表达式 |

```java
@Bean
@ConditionalOnProperty(name = "cache.type", havingValue = "redis")
public CacheManager redisCacheManager() {
    return new RedisCacheManager();
}
```

---

## 4. MyBatis

### 4.1 基础使用

**配置**

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

**Mapper接口**

```java
@Mapper
public interface UserMapper {
    User selectById(Long id);
    
    List<User> selectByCondition(User condition);
    
    int insert(User user);
    
    int updateById(User user);
}
```

### 4.2 动态SQL

**XML方式**

```xml
<select id="selectByCondition" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null and name != ''">
            AND name = #{name}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
    </where>
</select>

<update id="updateById" parameterType="User">
    UPDATE user
    <set>
        <if test="name != null">
            name = #{name},
        </if>
        <if test="status != null">
            status = #{status},
        </if>
    </set>
    WHERE id = #{id}
</update>
```

**注解方式**

```java
@Select("SELECT * FROM user WHERE id = #{id}")
User selectById(Long id);

@Update("UPDATE user SET name = #{name} WHERE id = #{id}")
int update(User user);
```

### 4.3 MyBatis缓存

**一级缓存（SqlSession级别）**

```java
User user1 = mapper.selectById(1);
User user2 = mapper.selectById(1);
```

**二级缓存（Mapper级别）**

```xml
<cache
  eviction="LRU"
  flushInterval="60000"
  size="1024"
  readOnly="true"/>
```

### 4.4 MyBatis插件

**自定义分件**

```java
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class}
    )
})
public class MyPlugin implements Interceptor {
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("执行前");
        Object result = invocation.proceed();
        System.out.println("执行后");
        return result;
    }
}
```

**配置插件**

```yaml
mybatis:
  plugins: com.example.MyPlugin
```

---

## 5. 事务管理

### 5.1 @Transactional注解

**基本使用**

```java
@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;
    
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        userMapper.debit(fromId, amount);
        userMapper.credit(toId, amount);
    }
}
```

**属性**

| 属性 | 默认值 | 说明 |
|------|--------|------|
| value | "" | 事务管理器 |
| propagation | REQUIRED | 传播行为 |
| isolation | DEFAULT | 隔离级别 |
| timeout | -1 | 超时时间（秒） |
| readOnly | false | 是否只读 |
| rollbackFor | RuntimeException.class | 回滚异常 |

### 5.2 事务传播行为

| 传播行为 | 说明 |
|----------|------|
| REQUIRED | 支持当前事务，没有则新建 |
| REQUIRES_NEW | 新建事务，挂起当前事务 |
| SUPPORTS | 支持当前事务，没有则以非事务执行 |
| NOT_SUPPORTED | 以非事务执行，挂起当前事务 |
| MANDATORY | 必须在事务中执行 |
| NEVER | 必须在非事务中执行 |
| NESTED | 嵌套事务，使用Savepoint |

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    methodB();
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
}
```

### 5.3 事务隔离级别

| 隔离级别 | 说明 |
|----------|------|
| DEFAULT | 使用数据库默认 |
| READ_UNCOMMITTED | 读未提交 |
| READ_COMMITTED | 读已提交 |
| REPEATABLE_READ | 可重复读 |
| SERIALIZABLE | 串行化 |

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void readData() {
}
```

### 5.4 事务失效场景

1. 方法不是public
2. 方法内部调用
3. 异常被捕获
4. 异常类型不匹配
5. 未被Spring管理

```java
@Service
public class UserService {
    @Transactional
    public void methodA() {
        this.methodB();
    }
    
    @Transactional
    public void methodB() {
    }
}
```

---

## 6. AOP

### 6.1 AOP概念

**核心概念**

- JoinPoint（连接点）：程序执行的某个位置
- Pointcut（切点）：匹配连接点的表达式
- Advice（通知）：在切点上执行的动作
- Aspect（切面）：切点+通知
- Weaving（织入）：将切面应用到目标对象

### 6.2 通知类型

| 通知类型 | 执行时机 |
|----------|----------|
| @Before | 方法执行前 |
| @After | 方法执行后（无论成功失败） |
| @AfterReturning | 方法返回结果后 |
| @AfterThrowing | 方法抛出异常后 |
| @Around | 环绕通知 |

```java
@Aspect
@Component
public class LogAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void before(JoinPoint joinPoint) {
        System.out.println("方法执行前");
    }
    
    @After("execution(* com.example.service.*.*(..))")
    public void after() {
        System.out.println("方法执行后");
    }
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("环绕通知前");
        Object result = pjp.proceed();
        System.out.println("环绕通知后");
        return result;
    }
}
```

### 6.3 切点表达式

**execution表达式**

```java
execution(modifier-pattern? return-type-pattern 
    declaring-type-pattern? method-name-pattern 
    (param-pattern) throws-pattern?)

// 所有public方法
execution(public * *(..))

// 所有返回String的方法
execution(String *.*(..))

// service包下所有方法
execution(* com.example.service.*.*(..))

// 带一个参数的方法
execution(* *(String))
```

### 6.4 AOP原理

**JDK动态代理**

```java
public class JDKProxy implements InvocationHandler {
    private Object target;
    
    public JDKProxy(Object target) {
        this.target = target;
    }
    
    public Object bind() {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this
        );
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        System.out.println("前置增强");
        Object result = method.invoke(target, args);
        System.out.println("后置增强");
        return result;
    }
}
```

**CGLIB代理**

```java
public class CGLIBProxy implements MethodInterceptor {
    private Object target;
    
    public Object bind() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }
    
    @Override
    public Object intercept(Object obj, Method method, 
        Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("前置增强");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("后置增强");
        return result;
    }
}
```

---

## 7. Spring MVC

### 7.1 Controller

**基础用法**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public Result<User> getUser(@PathVariable Long id) {
        User user = userService.getById(id);
        return Result.success(user);
    }
    
    @PostMapping
    public Result<User> createUser(@RequestBody User user) {
        User created = userService.create(user);
        return Result.success(created);
    }
    
    @PutMapping("/{id}")
    public Result<User> updateUser(
        @PathVariable Long id, 
        @RequestBody User user) {
        user.setId(id);
        User updated = userService.update(user);
        return Result.success(updated);
    }
}
```

### 7.2 参数绑定

**常用注解**

| 注解 | 说明 |
|------|------|
| @PathVariable | 路径变量 |
| @RequestParam | 请求参数 |
| @RequestBody | 请求体 |
| @RequestHeader | 请求头 |
| @CookieValue | Cookie值 |
| @ModelAttribute | 模型属性 |

```java
@GetMapping("/search")
public Result<List<User>> search(
    @RequestParam(required = false) String name,
    @RequestParam(defaultValue = "1") int page,
    @RequestHeader("token") String token) {
    return Result.success(userService.search(name, page, token));
}
```

### 7.3 拦截器

**自定义拦截器**

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
        HttpServletResponse response, Object handler) throws Exception {
        String token = request.getHeader("token");
        if (!isValidToken(token)) {
            response.setStatus(401);
            return false;
        }
        return true;
    }
}
```

**配置拦截器**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private LoginInterceptor loginInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/auth/**");
    }
}
```

---

## 8. 项目应用场景

### 8.1 智能合同：事务管理

```java
@Service
public class ContractService {
    @Autowired
    private ContractMapper contractMapper;
    
    @Autowired
    private ApprovalMapper approvalMapper;
    
    @Autowired
    private RecordMapper recordMapper;
    
    @Transactional(rollbackFor = Exception.class)
    public void approveContract(Long contractId, String approver) {
        Contract contract = contractMapper.selectById(contractId);
        contract.setStatus(Status.APPROVED);
        contract.setApprover(approver);
        contractMapper.updateById(contract);
        
        Approval approval = new Approval();
        approval.setContractId(contractId);
        approval.setApprover(approver);
        approvalMapper.insert(approval);
        
        Record record = new Record();
        record.setContractId(contractId);
        record.setAction("APPROVE");
        recordMapper.insert(record);
    }
}
```

### 8.2 ERP系统：AOP日志

```java
@Aspect
@Component
public class OperationLogAspect {
    
    @Autowired
    private OperationLogMapper logMapper;
    
    @Around("@annotation(operationLog)")
    public Object log(ProceedingJoinPoint pjp, OperationLog operationLog) {
        long start = System.currentTimeMillis();
        Object result = null;
        try {
            result = pjp.proceed();
            saveLog(pjp, operationLog, null, start);
            return result;
        } catch (Throwable e) {
            saveLog(pjp, operationLog, e, start);
            throw e;
        }
    }
    
    private void saveLog(ProceedingJoinPoint pjp, 
        OperationLog operationLog, Throwable error, long start) {
        OperationLogEntity log = new OperationLogEntity();
        log.setModule(operationLog.module());
        log.setOperation(operationLog.operation());
        log.setCost(System.currentTimeMillis() - start);
        log.setSuccess(error == null);
        logMapper.insert(log);
    }
}

@OperationLog(module = "DEVICE", operation = "UPDATE")
public void updateDevice(Device device) {
}
```

### 8.3 万物云对接：配置管理

```java
@Configuration
@ConditionalOnProperty(name = "wanyun.enabled", havingValue = "true")
public class WanyunConfig {
    
    @Value("${wanyun.api.url}")
    private String apiUrl;
    
    @Value("${wanyun.api.app-id}")
    private String appId;
    
    @Value("${wanyun.api.secret}")
    private String secret;
    
    @Bean
    public WanyunApiClient wanyunApiClient() {
        return new WanyunApiClient(apiUrl, appId, secret);
    }
    
    @Bean
    public WanyunDataSync wanyunDataSync(WanyunApiClient client) {
        return new WanyunDataSync(client);
    }
}
```

---

## 面试高频题

### 基础题

1. IOC和DI的区别？
2. Bean的生命周期？
3. @Autowired和@Resource的区别？
4. Bean的作用域有哪些？
5. Spring如何解决循环依赖？
6. Spring Boot自动配置原理？
7. MyBatis动态SQL的使用？
8. @Transactional的使用？

### 中等题

1. Spring三级缓存的作用？
2. Spring Boot启动流程？
3. 条件注解有哪些？
4. MyBatis缓存机制？
5. 事务传播行为有哪些？
6. 事务失效的场景？
7. AOP的五种通知类型？
8. JDK动态代理和CGLIB代理的区别？

### 高级题

1. 手写一个简单的Spring容器？
2. 实现一个简单的AOP代理？
3. MyBatis插件原理？
4. 手写一个简单的分布式事务？
5. Spring事务管理原理？
6. BeanFactory和ApplicationContext的区别？

---

## 学习建议

1. **重点掌握**：IOC、Bean生命周期、MyBatis、事务管理、AOP
2. **理解原理**：自动配置、AOP代理、事务传播
3. **实践应用**：项目中的事务、日志、配置管理
4. **准备代码**：手写简单IOC、AOP代理、动态代理

**参考文档**：
- Spring官方文档
- Spring Boot实战
- MyBatis官方文档
- 深入理解Spring

---

**更新进度到学习进度.md**