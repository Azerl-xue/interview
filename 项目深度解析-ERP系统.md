# 项目深度解析 - ERP系统

> 企业自有信息管理平台深度解析

---

## 📋 项目概述

**项目名称**：ERP企业自有信息管理平台  
**项目角色**：后端研发工程师  
**开发时间**：2024.06 - 至今  
**技术栈**：Java、Spring Boot、MyBatis、MySQL、Redis、RabbitMQ、Maven、Tomcat  
**用户规模**：200+名员工

---

## 🎯 项目背景与目标

### 背景

公司内部原有设备管理方式落后，缺乏系统化管理，导致：
- 设备利用率低（65%）
- 故障响应慢（6小时）
- 权限管理混乱
- 工单处理效率低

### 目标

- 构建设备全生命周期管理系统
- 实现精细化权限控制
- 优化设备维修流程
- 提升系统响应速度

---

## 🏗️ 系统架构设计

### 技术架构

```
┌─────────────────────────────────────────┐
│           Android App (测试端)            │
└─────────────────────────────────────────┘
                    ↑
                    │ HTTPS
┌─────────────────────────────────────────┐
│            Spring Boot应用               │
│  ┌─────────────────────────────────┐   │
│  │    Controller层                 │   │
│  │    Service层                    │   │
│  │    Dao层 (MyBatis)              │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
         ↓           ↓           ↓
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  MySQL  │  │  Redis  │  │RabbitMQ │
   └─────────┘  └─────────┘  └─────────┘
         ↓                      ↓
   ┌─────────────────────────────────┐
   │          OA系统 (异步对接)      │
   └─────────────────────────────────┘
```

### 模块划分

1. **设备管理模块**
   - 设备出入库管理
   - 设备信息库
   - 设备状态跟踪

2. **权限管理模块**
   - 用户权限配置
   - 角色管理
   - 设备访问控制

3. **维修管理模块**
   - 维修工单
   - 设备置换
   - 故障响应

4. **测试工单模块**
   - 工单创建
   - 工单流转
   - 工单统计

5. **Android端接口**
   - 设备管理API
   - 数据加密传输

---

## 🔑 核心功能实现

### 1. 设备全生命周期追踪

#### 功能描述
实现设备从采购、入库、领用、维修、报废的全过程追踪。

#### 数据库设计

```sql
CREATE TABLE `device` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `device_no` VARCHAR(50) NOT NULL COMMENT '设备编号',
  `name` VARCHAR(100) NOT NULL COMMENT '设备名称',
  `type` VARCHAR(50) COMMENT '设备类型',
  `status` VARCHAR(20) COMMENT '状态：AVAILABLE/IN_USE/REPAIRING/SCRAPPED',
  `user_id` BIGINT COMMENT '当前使用人',
  `purchase_time` DATETIME COMMENT '采购时间',
  `in_stock_time` DATETIME COMMENT '入库时间',
  `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP,
  `update_time` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX `idx_device_no` (`device_no`),
  INDEX `idx_status` (`status`),
  INDEX `idx_user` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `device_log` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `device_id` BIGINT NOT NULL,
  `action` VARCHAR(50) COMMENT '操作类型',
  `operator_id` BIGINT COMMENT '操作人',
  `remark` TEXT COMMENT '备注',
  `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX `idx_device` (`device_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 核心代码

```java
@Service
public class DeviceService {
    
    @Autowired
    private DeviceMapper deviceMapper;
    
    @Autowired
    private DeviceLogMapper deviceLogMapper;
    
    @Transactional(rollbackFor = Exception.class)
    public void inStock(DeviceInStockRequest request) {
        Device device = new Device();
        device.setDeviceNo(request.getDeviceNo());
        device.setName(request.getName());
        device.setType(request.getType());
        device.setStatus(DeviceStatus.AVAILABLE);
        device.setPurchaseTime(request.getPurchaseTime());
        device.setInStockTime(new Date());
        deviceMapper.insert(device);
        
        DeviceLog log = new DeviceLog();
        log.setDeviceId(device.getId());
        log.setAction("IN_STOCK");
        log.setOperatorId(request.getOperatorId());
        log.setRemark("设备入库");
        deviceLogMapper.insert(log);
    }
    
    @Transactional(rollbackFor = Exception.class)
    public void allocate(Long deviceId, Long userId, Long operatorId) {
        Device device = deviceMapper.selectById(deviceId);
        if (device == null || !DeviceStatus.AVAILABLE.equals(device.getStatus())) {
            throw new BusinessException("设备不可用");
        }
        
        device.setStatus(DeviceStatus.IN_USE);
        device.setUserId(userId);
        deviceMapper.updateById(device);
        
        DeviceLog log = new DeviceLog();
        log.setDeviceId(deviceId);
        log.setAction("ALLOCATE");
        log.setOperatorId(operatorId);
        log.setRemark("设备分配给用户: " + userId);
        deviceLogMapper.insert(log);
    }
}
```

---

### 2. 设备权限管理

#### 功能描述
为200+名员工提供精细化权限控制，支持用户、部门、角色等多维度权限。

#### 权限模型设计

```java
public class Permission {
    private Long id;
    private String resourceType;  // 资源类型：DEVICE/OPERATION
    private Long resourceId;      // 资源ID
    private String permission;   // 权限：READ/WRITE/DELETE
    private Long userId;          // 用户ID
    private Long roleId;          // 角色ID
    private Long deptId;          // 部门ID
}

public class DevicePermission {
    private Long deviceId;
    private Set<Long> userIds;    // 可访问的用户ID集合
}
```

#### 核心代码

```java
@Service
public class DevicePermissionService {
    
    @Autowired
    private PermissionMapper permissionMapper;
    
    @Autowired
    private RedisTemplate<String, Set<String>> redisTemplate;
    
    private static final String PERMISSION_CACHE_PREFIX = "device:permission:";
    
    public boolean hasPermission(Long userId, Long deviceId, String permission) {
        String key = PERMISSION_CACHE_PREFIX + deviceId;
        Set<String> permissions = redisTemplate.opsForValue().get(key);
        
        if (permissions == null) {
            permissions = loadPermissionsFromDB(deviceId);
            redisTemplate.opsForValue().set(key, permissions, 3600, TimeUnit.SECONDS);
        }
        
        String permissionKey = userId + ":" + permission;
        return permissions.contains(permissionKey);
    }
    
    private Set<String> loadPermissionsFromDB(Long deviceId) {
        List<Permission> perms = permissionMapper.selectByResourceId(
            "DEVICE", deviceId);
        
        Set<String> result = new HashSet<>();
        for (Permission perm : perms) {
            String key = perm.getUserId() + ":" + perm.getPermission();
            result.add(key);
        }
        
        return result;
    }
    
    public void grantPermission(Long deviceId, Long userId, String permission) {
        Permission perm = new Permission();
        perm.setResourceType("DEVICE");
        perm.setResourceId(deviceId);
        perm.setPermission(permission);
        perm.setUserId(userId);
        permissionMapper.insert(perm);
        
        String key = PERMISSION_CACHE_PREFIX + deviceId;
        redisTemplate.delete(key);
    }
}
```

---

### 3. 异步审批流程（RabbitMQ）

#### 功能描述
通过RabbitMQ消息队列实现与OA系统的异步对接，自动化审批流程。

#### 消息设计

```java
public class ApprovalMessage {
    private String messageId;      // 消息ID
    private String businessType;   // 业务类型：DEVICE_ALLOCATE
    private Long businessId;        // 业务ID
    private Long applicantId;      // 申请人ID
    private String approver;       // 审批人
    private String status;         // 状态
    private Date createTime;
}
```

#### 生产者

```java
@Service
public class ApprovalProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void submitApproval(ApprovalMessage message) {
        message.setMessageId(UUID.randomUUID().toString());
        message.setCreateTime(new Date());
        message.setStatus("PENDING");
        
        rabbitTemplate.convertAndSend(
            "approval.exchange",
            "approval.routing.key",
            message
        );
    }
}
```

#### 消费者

```java
@RabbitListener(queues = "approval.queue")
public class ApprovalConsumer {
    
    @Autowired
    private OaApiClient oaApiClient;
    
    @Autowired
    private ApprovalMapper approvalMapper;
    
    @RabbitHandler
    public void handleApproval(ApprovalMessage message) {
        try {
            syncToOA(message);
            message.setStatus("APPROVED");
            approvalMapper.updateMessageStatus(
                message.getMessageId(), "APPROVED");
        } catch (Exception e) {
            log.error("审批同步失败", e);
            message.setStatus("FAILED");
            approvalMapper.updateMessageStatus(
                message.getMessageId(), "FAILED");
            throw new AmqpRejectAndDontRequeueException("审批失败");
        }
    }
    
    private void syncToOA(ApprovalMessage message) {
        ApprovalRequest request = new ApprovalRequest();
        request.setMessageId(message.getMessageId());
        request.setBusinessType(message.getBusinessType());
        request.setBusinessId(message.getBusinessId());
        request.setApplicantId(message.getApplicantId());
        
        oaApiClient.submitApproval(request);
    }
}
```

---

### 4. Redis缓存优化

#### 功能描述
通过Redis缓存优化MySQL查询，减少数据库压力。

#### 缓存策略

```java
@Service
public class DeviceCacheService {
    
    @Autowired
    private DeviceMapper deviceMapper;
    
    @Autowired
    private RedisTemplate<String, Device> redisTemplate;
    
    private static final String DEVICE_CACHE_PREFIX = "device:";
    
    public Device getDeviceWithCache(Long deviceId) {
        String key = DEVICE_CACHE_PREFIX + deviceId;
        Device device = redisTemplate.opsForValue().get(key);
        
        if (device == null) {
            device = deviceMapper.selectById(deviceId);
            if (device != null) {
                long randomExpire = 3600 + RandomUtils.nextLong(0, 600);
                redisTemplate.opsForValue().set(key, device, 
                    randomExpire, TimeUnit.SECONDS);
            }
        }
        
        return device;
    }
    
    public List<Device> getUserDeviceList(Long userId) {
        String key = "user:devices:" + userId;
        List<Device> devices = redisTemplate.opsForValue().get(key);
        
        if (devices == null) {
            devices = deviceMapper.selectByUserId(userId);
            redisTemplate.opsForValue().set(key, devices, 
                300, TimeUnit.SECONDS);
        }
        
        return devices;
    }
    
    public void updateDevice(Device device) {
        deviceMapper.updateById(device);
        String key = DEVICE_CACHE_PREFIX + device.getId();
        redisTemplate.delete(key);
    }
}
```

---

### 5. MySQL查询优化

#### 优化前

```sql
-- 慢查询（全表扫描）
SELECT * FROM device 
WHERE user_id = 123 
  AND status = 'IN_USE'
ORDER BY update_time DESC
LIMIT 10;
```

#### Explain分析

```
type: ALL
rows: 10000+
Extra: Using where; filesort
```

#### 优化后

```sql
-- 添加联合索引
CREATE INDEX idx_user_status_time 
ON device(user_id, status, update_time DESC);

-- 优化后的查询
SELECT * FROM device 
WHERE user_id = 123 
  AND status = 'IN_USE'
ORDER BY update_time DESC
LIMIT 10;
```

#### Explain分析

```
type: ref
rows: 10
Extra: Using index
```

---

### 6. 接口版本控制

#### 功能描述
通过接口版本控制策略确保100%设备兼容性。

#### 版本控制实现

```java
@RestController
@RequestMapping("/api/v{version}/device")
public class DeviceController {
    
    @GetMapping("/{deviceId}")
    public Result<Device> getDevice(
        @PathVariable String version,
        @PathVariable Long deviceId) {
        
        Device device;
        if ("1".equals(version)) {
            device = deviceService.getDeviceV1(deviceId);
        } else if ("2".equals(version)) {
            device = deviceService.getDeviceV2(deviceId);
        } else {
            device = deviceService.getDevice(deviceId);
        }
        
        return Result.success(device);
    }
}
```

---

## 📊 核心成果

### 性能提升

| 指标 | 优化前 | 优化后 | 提升幅度 |
|------|--------|--------|----------|
| 故障响应时间 | 6小时 | 2小时 | ↓66% |
| 设备利用率 | 65% | 85% | ↑20% |
| 工单处理效率 | 15单/天 | 21单/天 | ↑40% |
| 平均故障响应时间 | 6小时 | 2小时 | ↓66% |
| 审批流程效率 | - | - | ↑60% |

### 系统能力

- 支持200+名员工日常使用
- 月均处理工单量50+份
- 测试人员设备管理效率提升45%
- 100%设备兼容性
- 系统可用性达到99.9%

---

## 🎓 面试重点

### 技术亮点

1. **异步处理**
   - RabbitMQ实现异步审批
   - 提升系统吞吐量

2. **缓存优化**
   - Redis缓存减少数据库压力
   - 随机过期时间避免缓存雪崩

3. **查询优化**
   - MySQL索引优化
   - 慢查询性能提升

4. **权限设计**
   - RBAC模型
   - 精细化权限控制

### 常见面试题

1. 你是如何设计设备权限系统的？
2. RabbitMQ异步处理是如何保证消息可靠性的？
3. Redis缓存策略是如何设计的？
4. 如何解决MySQL的慢查询问题？
5. 接口版本控制是如何实现的？
6. 设备全生命周期追踪是如何设计的？

### STAR回答要点

**最有挑战的功能**
- S：设备维修流程性能差，故障响应时间长
- T：将故障响应时间从6小时缩短到2小时
- A：优化SQL、添加索引、引入Redis缓存
- R：故障响应时间缩短66%，设备利用率提升30%

**与OA系统集成**
- S：需要与OA系统对接实现审批自动化
- T：设计异步对接方案，保证系统稳定性
- A：使用RabbitMQ消息队列，实现异步处理
- R：审批流程效率提升60%，系统稳定性提升

---

## 📝 总结

**项目亮点**：
- 综合运用多种技术：Spring Boot、MySQL、Redis、RabbitMQ
- 性能优化显著：多个指标提升40-66%
- 系统设计合理：模块化、可扩展
- 实战经验丰富：200+用户、月均50+工单

**技术收获**：
- 异步处理与消息队列应用
- 缓存策略与性能优化
- 数据库索引优化
- 权限系统设计
- 接口版本控制

---

**更新进度到学习进度.md**