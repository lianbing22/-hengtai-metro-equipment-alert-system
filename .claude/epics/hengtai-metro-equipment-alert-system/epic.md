---
name: hengtai-metro-equipment-alert-system
status: backlog
created: 2025-10-04T04:19:38Z
progress: 0%
prd: .claude/prds/hengtai-metro-equipment-alert-system.md
github: [Will be updated when synced to GitHub]
---

# Epic: hengtai-metro-equipment-alert-system

## Overview

基于BAS数据的地铁设备预警系统，采用微服务架构，通过离线数据导入方式实现对150个地铁车站的设备监控、预警分析和数据可视化。系统分为数据导入、监控中心、预警系统、数据分析、报表系统、设备管理、移动端支持和系统管理八大模块，使用Vue3+Spring Boot技术栈实现前后端分离，支持高并发访问和横向扩展。

## Architecture Decisions

### 核心技术决策
1. **微服务架构**：采用Spring Cloud微服务架构，各模块独立部署，便于扩展和维护
2. **前后端分离**：Vue3 + TypeScript前端，Spring Boot后端，RESTful API通信
3. **数据存储策略**：MySQL主库存储业务数据，Redis缓存热点数据，Elasticsearch支持日志检索
4. **文件存储**：使用MinIO分布式对象存储管理Excel导入文件和导出报表
5. **消息队列**：RabbitMQ处理异步任务（数据导入、预警通知、报表生成）
6. **地图服务**：集成高德地图API JavaScript 2.0，无需额外GIS服务器

### 技术选型理由
- **Vue3 + TypeScript**：提供更好的类型支持和组合式API，提升开发效率
- **Element Plus**：成熟的企业级UI组件库，降低开发成本
- **Spring Boot + MyBatis Plus**：简化数据库操作，提升开发速度
- **Redis**：缓存用户会话、设备状态等热点数据，提升系统性能
- **Docker + K8s**：容器化部署，支持弹性扩缩容

## Technical Approach

### Frontend Components

#### 1. 核心页面模块
- **登录/首页**：用户认证、系统概览、快捷入口
- **数据导入页**：文件上传、进度显示、历史导入记录
- **数据驾驶舱**：ECharts图表、实时数据刷新、自定义面板
- **设备监控中心**：设备列表、状态展示、筛选功能
- **地铁地图页**：高德地图集成、车站标注、交互操作
- **预警看板**：预警列表、详情查看、处理流程
- **数据分析页**：趋势图、对比分析、故障模式
- **报表中心**：报表生成、订阅管理、下载功能
- **设备管理**：设备档案、维护记录、备件管理
- **系统设置**：用户管理、权限配置、系统参数

#### 2. 共享组件
- **数据表格组件**：支持虚拟滚动、排序、筛选、导出
- **图表组件**：基于ECharts封装的通用图表组件
- **地图组件**：封装高德地图，支持标记、热力图、路径规划
- **文件上传组件**：支持拖拽上传、进度显示、格式校验
- **时间选择器**：支持快速选择时间范围
- **状态标签**：统一的设备状态、预警级别展示

#### 3. 状态管理
- 使用Pinia进行全局状态管理
- 模块化管理：用户、设备、预警、系统配置
- 持久化存储：用户偏好设置、本地缓存

### Backend Services

#### 1. 核心服务模块
```
hengtai-gateway          # 网关服务（路由、鉴权、限流）
hengtai-auth            # 认证服务（JWT、用户管理、权限）
hengtai-file            # 文件服务（上传、下载、存储）
hengtai-data            # 数据服务（导入、解析、校验）
hengtai-device          # 设备服务（设备管理、状态监控）
hengtai-alert           # 预警服务（规则引擎、通知推送）
hengtai-analysis        # 分析服务（趋势分析、预测模型）
hengtai-report          # 报表服务（生成、导出、订阅）
hengtai-notification    # 通知服务（钉钉、企业微信、短信、邮件）
```

#### 2. API设计
```
POST   /api/v1/files/upload          # 文件上传
GET    /api/v1/devices/stations      # 获取车站列表
GET    /api/v1/devices/status        # 获取设备状态
GET    /api/v1/alerts/list           # 获取预警列表
POST   /api/v1/alerts/handle         # 处理预警
GET    /api/v1/dashboard/kpi         # 获取KPI数据
GET    /api/v1/analysis/trends       # 获取趋势分析
POST   /api/v1/reports/generate      # 生成报表
```

#### 3. 数据模型
```sql
-- 车站表
CREATE TABLE stations (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    line_id VARCHAR(50),
    longitude DECIMAL(10,7),
    latitude DECIMAL(10,7),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 设备表
CREATE TABLE devices (
    id BIGINT PRIMARY KEY,
    station_id BIGINT,
    name VARCHAR(200),
    type ENUM('HVAC','VENT','ELEVATOR','LIGHT','DRAINAGE'),
    model VARCHAR(100),
    install_date DATE,
    status ENUM('NORMAL','WARNING','CRITICAL','EMERGENCY'),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    INDEX idx_station_type (station_id, type)
);

-- 设备数据表（分区表）
CREATE TABLE device_data (
    id BIGINT PRIMARY KEY,
    device_id BIGINT,
    metric_name VARCHAR(100),
    metric_value DECIMAL(10,2),
    unit VARCHAR(20),
    record_time TIMESTAMP,
    INDEX idx_device_time (device_id, record_time)
) PARTITION BY RANGE (TO_DAYS(record_time));

-- 预警表
CREATE TABLE alerts (
    id BIGINT PRIMARY KEY,
    device_id BIGINT,
    level ENUM('WARNING','CRITICAL','EMERGENCY'),
    title VARCHAR(200),
    content TEXT,
    status ENUM('PENDING','PROCESSING','RESOLVED'),
    created_at TIMESTAMP,
    resolved_at TIMESTAMP,
    INDEX idx_device_status (device_id, status)
);
```

### Infrastructure

#### 1. 部署架构
```
负载均衡层：Nginx
应用层：Spring Cloud微服务集群
缓存层：Redis Cluster
数据库层：MySQL主从复制 + 读写分离
存储层：MinIO分布式存储
消息队列：RabbitMQ集群
监控层：Prometheus + Grafana + ELK
```

#### 2. 容器化配置
```yaml
# docker-compose.yml 核心服务
version: '3.8'
services:
  gateway:
    image: hengtai/gateway:latest
    ports:
      - "80:8080"
    depends_on:
      - auth
      - device

  auth:
    image: hengtai/auth:latest
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis

  mysql:
    image: mysql:8.0
    volumes:
      - mysql_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=******

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
```

#### 3. 监控和日志
- **应用监控**：Spring Boot Actuator + Micrometer
- **链路追踪**：SkyWalking实现分布式追踪
- **日志聚合**：ELK Stack（Elasticsearch + Logstash + Kibana）
- **告警通知**：AlertManager + 钉钉/企业微信

## Implementation Strategy

### 开发阶段规划
1. **第一阶段（基础平台）**：搭建基础架构，实现用户认证、文件上传、基础设备管理
2. **第二阶段（核心功能）**：开发数据驾驶舱、地铁地图、预警系统
3. **第三阶段（高级功能）**：实现数据分析、报表系统、移动端优化

### 风险缓解
- **性能风险**：采用分库分表、缓存策略、异步处理
- **数据一致性**：使用分布式事务、消息队列保证最终一致性
- **可用性保障**：服务降级、熔断、限流机制
- **安全防护**：HTTPS、JWT、RBAC权限控制、SQL注入防护

### 测试策略
- **单元测试**：JUnit + Mockito，覆盖率>80%
- **集成测试**：TestContainers，测试API接口
- **性能测试**：JMeter压测，模拟100并发用户
- **E2E测试**：Cypress自动化测试

## Tasks Created
- [ ] 001.md - 项目初始化和基础框架搭建 (parallel: false)
- [ ] 002.md - 数据库设计和初始化 (parallel: true)
- [ ] 003.md - 认证服务开发 (parallel: true)
- [ ] 004.md - 文件存储服务开发 (parallel: true)
- [ ] 005.md - 数据导入模块开发 (parallel: false)
- [ ] 006.md - 设备监控中心开发 (parallel: true)
- [ ] 007.md - 数据驾驶舱开发 (parallel: true)
- [ ] 008.md - 地铁地图功能开发 (parallel: true)
- [ ] 009.md - 预警系统实现 (parallel: false)
- [ ] 010.md - 数据分析模块开发 (parallel: true)
- [ ] 011.md - 报表系统开发 (parallel: true)
- [ ] 012.md - 移动端适配和PWA支持 (parallel: false)

Total tasks: 12
Parallel tasks: 7
Sequential tasks: 5
Estimated total effort: 120故事点（约240人天）

## Dependencies

### 外部依赖
- **高德地图API**：需要申请开发者账号和API Key
- **钉钉开放平台**：企业内部应用开发权限
- **企业微信API**：企业微信管理员权限
- **短信服务**：第三方短信平台（如阿里云短信）
- **邮件服务**：SMTP服务器配置

### 内部依赖
- **基础设施团队**：提供服务器、网络、数据库支持
- **运维团队**：部署、监控、备份策略制定
- **安全团队**：安全评估、渗透测试
- **测试团队**：功能测试、性能测试、用户验收测试

### 技术债务管理
- 定期代码重构和优化
- 技术栈版本升级计划
- 性能基准测试和优化

## Success Criteria (Technical)

### 性能指标
- **响应时间**：API平均响应时间<500ms，95分位<1000ms
- **吞吐量**：支持1000 QPS，峰值3000 QPS
- **并发用户**：支持500个并发用户在线操作
- **数据处理**：10MB Excel文件30秒内完成导入解析

### 质量标准
- **代码覆盖率**：单元测试覆盖率>80%，关键路径>95%
- **可用性**：系统可用性>99.9%，月度故障时间<45分钟
- **数据准确性**：预警准确率>85%，误报率<10%
- **安全等级**：通过安全渗透测试，无高危漏洞

### 扩展能力
- **横向扩展**：支持通过增加节点线性提升性能
- **存储扩展**：支持PB级数据存储，自动扩容
- **功能扩展**：模块化设计，快速集成新功能

## Estimated Effort

### 时间估算
- **总工期**：6个月（24周）
- **第一阶段**：8周（基础平台）
- **第二阶段**：10周（核心功能）
- **第三阶段**：6周（优化上线）

### 人员配置
- **项目经理**：1人（全程）
- **架构师**：1人（全程）
- **前端开发**：3人
- **后端开发**：4人
- **测试工程师**：2人
- **UI/UX设计师**：1人
- **运维工程师**：1人

### 关键路径
1. 基础架构搭建 → 数据导入模块 → 设备监控中心
2. 高德API集成 → 地铁地图功能 → 数据驾驶舱
3. 预警规则引擎 → 智能算法优化 → 通知推送

### 成本估算
- **开发成本**：约120人月
- **基础设施**：云服务器、数据库、CDN等月费用约2-3万元
- **第三方服务**：高德API、短信、邮件等月费用约5000元
- **维护成本**：开发成本的20%年维护费

## Risk Matrix

| 风险项 | 概率 | 影响 | 应对措施 |
|--------|------|------|----------|
| 高德API限流 | 中 | 高 | 实现本地缓存、备用地图方案 |
| 数据导入性能 | 中 | 中 | 分批处理、异步队列、优化SQL |
| 预警算法准确性 | 高 | 高 | 持续训练、A/B测试、人工校验 |
| 第三方服务不稳定 | 低 | 中 | 多服务商备份、降级方案 |
| 数据安全泄露 | 低 | 高 | 加密存储、访问控制、审计日志 |

## Quality Assurance

### 代码规范
- ESLint + Prettier统一代码风格
- Git分支管理：GitFlow工作流
- Code Review：所有代码必须经过同行评审
- 持续集成：GitHub Actions自动化构建测试

### 发布流程
1. 开发环境自测
2. 测试环境集成测试
3. 预生产环境验证
4. 生产环境灰度发布
5. 全量发布 + 监控

### 回滚策略
- 数据库版本管理（Flyway）
- 应用快速回滚（蓝绿部署）
- 配置热更新（Nacos配置中心）
- 应急响应预案

---

**文档版本**：1.0
**创建日期**：2025-10-04
**架构师**：技术团队
**审批状态**：待审批