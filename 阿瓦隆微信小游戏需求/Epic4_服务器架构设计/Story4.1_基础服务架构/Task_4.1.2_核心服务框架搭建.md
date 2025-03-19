# Task 4.1.2: 核心服务框架搭建

## 描述

基于 NestJS 框架搭建阿瓦隆微信小游戏的核心服务框架，实现模块化架构，创建基础服务和中间件，配置全局拦截器、过滤器和管道。

## 验收标准

1. 完成 NestJS 项目初始化和基础配置
2. 实现模块化的项目结构，包括核心模块、共享模块和功能模块
3. 配置环境变量和配置管理系统
4. 实现全局异常处理和响应转换
5. 集成基础中间件(如 CORS, Helmet, 压缩等)
6. 创建 API 文档(Swagger)
7. 实现健康检查端点
8. 搭建基础测试框架
9. 完成 CI/CD 配置文件

## 详细内容

### 项目初始化与结构

1. **基础项目创建**

   - 使用 Nest CLI 创建项目
   - 配置 TypeScript
   - 设置代码规范与格式化(ESLint, Prettier)
   - 配置环境变量管理(@nestjs/config)

2. **模块化设计实现**
   ```
   src/
   ├── main.ts                 # 应用入口
   ├── app.module.ts           # 根模块
   ├── core/                   # 核心模块
   │   ├── constants/          # 常量定义
   │   ├── decorators/         # 自定义装饰器
   │   ├── filters/            # 全局异常过滤器
   │   ├── guards/             # 全局守卫
   │   ├── interceptors/       # 全局拦截器
   │   ├── middleware/         # 中间件
   │   ├── pipes/              # 全局管道
   │   └── services/           # 共享服务
   ├── modules/                # 功能模块
   │   ├── auth/               # 认证模块
   │   ├── user/               # 用户模块
   │   ├── room/               # 房间模块
   │   ├── game/               # 游戏模块
   │   └── websocket/          # WebSocket模块
   ├── shared/                 # 共享模块
   │   ├── dto/                # 数据传输对象
   │   ├── entities/           # 实体定义
   │   ├── interfaces/         # 接口定义
   │   └── utils/              # 工具函数
   └── config/                 # 配置
       ├── app.config.ts       # 应用配置
       ├── database.config.ts  # 数据库配置
       └── redis.config.ts     # Redis配置
   ```

### 核心功能实现

1. **全局异常处理**

   - 实现 HTTP 异常过滤器
   - 实现 WebSocket 异常过滤器
   - 标准化错误响应格式

2. **响应转换**

   - 实现响应拦截器
   - 标准化成功响应格式

3. **中间件集成**

   - CORS 配置
   - Helmet 安全头配置
   - 请求日志中间件
   - 压缩中间件

4. **配置管理**

   - 使用@nestjs/config 集成环境变量
   - 实现基于环境的配置载入
   - 配置验证(使用 Joi)

5. **API 文档**

   - 集成 Swagger
   - 配置 API 分组和标签
   - 添加认证文档配置

6. **健康检查**
   - 集成@nestjs/terminus
   - 实现数据库连接检查
   - 实现 Redis 连接检查
   - 实现磁盘空间和内存检查

### 测试框架搭建

1. **单元测试**

   - 配置 Jest
   - 创建测试工具类和模拟数据

2. **集成测试**

   - 配置测试数据库
   - 创建测试模块和辅助函数

3. **E2E 测试**
   - 配置 SuperTest
   - 创建测试辅助工具

### CI/CD 配置

1. **GitHub Actions 配置**

   ```yaml
   # .github/workflows/main.yml 示例结构
   name: CI/CD Pipeline

   on:
     push:
       branches: [main, dev]
     pull_request:
       branches: [main]

   jobs:
     test:
       # 测试配置...

     build:
       # 构建配置...

     deploy:
       # 部署配置...
   ```

2. **Docker 配置**
   - 创建 Dockerfile
   - 创建.dockerignore
   - 创建 docker-compose.yml

## 工作量估计

3 人天

## 技术关键点

1. 确保模块间的低耦合高内聚
2. 实现可扩展的异常处理机制
3. 配置合理的 CORS 和安全设置
4. 确保配置管理的灵活性和安全性
5. 设计统一的响应格式规范

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.1: 基础服务架构](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [NestJS 官方文档](https://docs.nestjs.com/)

## 依赖关系

- 上游依赖:
  - Task 4.1.1: 架构设计与规划
- 下游依赖:
  - Task 4.1.3: 数据库与缓存配置
  - Task 4.1.4: 认证与授权服务
