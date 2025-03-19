# Task 4.1.6: 服务部署与配置

## 描述

设计并实现阿瓦隆微信小游戏的服务部署和配置系统，包括容器化配置、环境变量管理、自动化部署流程、负载均衡设置、服务发现机制，以及灾备和扩展策略，确保服务高可用、易维护且可扩展。

## 验收标准

1. 完成 Docker 和 Docker Compose 配置文件
2. 实现基于环境的配置管理系统
3. 配置 Nginx 作为前端负载均衡器
4. 设计 CI/CD 自动化部署流程
5. 实现灰度发布和回滚机制
6. 配置服务健康检查和自动恢复
7. 设计服务扩展和缩容策略
8. 创建维护模式和无缝更新机制
9. 编写完整的部署和运维文档

## 详细内容

### 容器化配置

1. **Dockerfile 配置**

   - 为应用服务创建多阶段构建 Dockerfile
   - 优化镜像大小和构建速度
   - 配置适当的基础镜像和环境

   ```dockerfile
   # Dockerfile 示例
   # 构建阶段
   FROM node:16-alpine AS builder

   WORKDIR /app

   COPY package*.json ./
   RUN npm ci

   COPY tsconfig*.json ./
   COPY src ./src

   RUN npm run build

   # 生产阶段
   FROM node:16-alpine

   WORKDIR /app

   COPY package*.json ./
   RUN npm ci --only=production

   COPY --from=builder /app/dist ./dist

   # 创建非root用户
   RUN addgroup -g 1001 -S nodejs && \
       adduser -S nestjs -u 1001 -G nodejs

   # 切换到非root用户
   USER nestjs

   # 健康检查
   HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
     CMD wget -qO- http://localhost:3000/health || exit 1

   ENV NODE_ENV=production

   EXPOSE 3000

   CMD ["node", "dist/main"]
   ```

2. **Docker Compose 配置**

   - 创建用于开发和生产的 Docker Compose 文件
   - 配置服务依赖关系
   - 设置网络和卷配置
   - 配置资源限制和重启策略

   ```yaml
   # docker-compose.yml 示例
   version: "3.8"

   services:
     api:
       build:
         context: .
         dockerfile: Dockerfile
       image: avalon-api:${TAG:-latest}
       container_name: avalon-api
       restart: unless-stopped
       ports:
         - "3000:3000"
       environment:
         - NODE_ENV=production
         - MONGODB_URI=mongodb://mongodb:27017/avalon
         - REDIS_HOST=redis
         - REDIS_PORT=6379
         - JWT_SECRET=${JWT_SECRET}
         - WECHAT_APPID=${WECHAT_APPID}
         - WECHAT_SECRET=${WECHAT_SECRET}
       depends_on:
         - mongodb
         - redis
       deploy:
         resources:
           limits:
             cpus: "0.5"
             memory: 500M
       networks:
         - avalon-network
       healthcheck:
         test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
         interval: 30s
         timeout: 5s
         retries: 3
         start_period: 10s

     websocket:
       build:
         context: .
         dockerfile: Dockerfile.websocket
       image: avalon-websocket:${TAG:-latest}
       container_name: avalon-websocket
       restart: unless-stopped
       ports:
         - "3001:3001"
       environment:
         - NODE_ENV=production
         - MONGODB_URI=mongodb://mongodb:27017/avalon
         - REDIS_HOST=redis
         - REDIS_PORT=6379
         - JWT_SECRET=${JWT_SECRET}
       depends_on:
         - mongodb
         - redis
       deploy:
         resources:
           limits:
             cpus: "0.5"
             memory: 500M
       networks:
         - avalon-network
       healthcheck:
         test: ["CMD", "wget", "-qO-", "http://localhost:3001/health"]
         interval: 30s
         timeout: 5s
         retries: 3
         start_period: 10s

     mongodb:
       image: mongo:5.0
       container_name: avalon-mongodb
       restart: unless-stopped
       ports:
         - "27017:27017"
       volumes:
         - mongodb-data:/data/db
       networks:
         - avalon-network
       healthcheck:
         test: echo 'db.runCommand("ping").ok' | mongo mongodb:27017/admin --quiet
         interval: 30s
         timeout: 5s
         retries: 3
         start_period: 10s

     redis:
       image: redis:6.2-alpine
       container_name: avalon-redis
       restart: unless-stopped
       ports:
         - "6379:6379"
       volumes:
         - redis-data:/data
       networks:
         - avalon-network
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 30s
         timeout: 5s
         retries: 3
         start_period: 10s

     nginx:
       image: nginx:1.21-alpine
       container_name: avalon-nginx
       restart: unless-stopped
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - ./nginx/conf.d:/etc/nginx/conf.d
         - ./nginx/ssl:/etc/nginx/ssl
         - ./nginx/logs:/var/log/nginx
       depends_on:
         - api
         - websocket
       networks:
         - avalon-network
       healthcheck:
         test: ["CMD", "wget", "-qO-", "http://localhost/health"]
         interval: 30s
         timeout: 5s
         retries: 3
         start_period: 10s

   networks:
     avalon-network:
       driver: bridge

   volumes:
     mongodb-data:
     redis-data:
   ```

3. **.dockerignore 配置**
   - 忽略不必要的文件和目录
   - 优化构建上下文
   - 避免敏感信息包含在镜像中

### 环境配置管理

1. **环境变量管理**

   - 创建各环境的环境变量模板
   - 设计敏感信息的安全存储方案
   - 配置运行时环境检测和验证

   ```
   # .env.example 示例
   # 应用配置
   NODE_ENV=production
   PORT=3000
   WS_PORT=3001
   API_PREFIX=/api

   # 数据库配置
   MONGODB_URI=mongodb://mongodb:27017/avalon
   MONGODB_USER=
   MONGODB_PASSWORD=

   # Redis配置
   REDIS_HOST=redis
   REDIS_PORT=6379
   REDIS_PASSWORD=
   REDIS_DB=0

   # JWT配置
   JWT_SECRET=your-jwt-secret-key
   JWT_EXPIRES_IN=3600
   REFRESH_TOKEN_EXPIRES_IN=604800

   # 微信配置
   WECHAT_APPID=your-appid
   WECHAT_SECRET=your-secret

   # 日志配置
   LOG_LEVEL=info

   # 性能配置
   MAX_CONNECTIONS=1000
   RATE_LIMIT_IP=60
   RATE_LIMIT_USER=30
   ```

2. **配置服务**
   - 实现基于环境的配置加载
   - 支持配置热更新
   - 添加配置验证逻辑

### 部署自动化

1. **CI/CD 流程配置**

   - 设置 GitHub Actions 工作流
   - 配置测试、构建和部署步骤
   - 实现自动版本管理

   ```yaml
   # .github/workflows/main.yml 示例
   name: CI/CD Pipeline

   on:
     push:
       branches: [main, dev]
     pull_request:
       branches: [main]

   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
         - name: Use Node.js
           uses: actions/setup-node@v2
           with:
             node-version: "16"
         - name: Install dependencies
           run: npm ci
         - name: Run tests
           run: npm test

     build:
       needs: test
       runs-on: ubuntu-latest
       if: github.event_name == 'push'
       steps:
         - uses: actions/checkout@v2
         - name: Set up Docker Buildx
           uses: docker/setup-buildx-action@v1
         - name: Login to Docker Hub
           uses: docker/login-action@v1
           with:
             username: ${{ secrets.DOCKER_USERNAME }}
             password: ${{ secrets.DOCKER_PASSWORD }}
         - name: Build and push API image
           uses: docker/build-push-action@v2
           with:
             context: .
             push: true
             tags: yourusername/avalon-api:latest
         - name: Build and push WebSocket image
           uses: docker/build-push-action@v2
           with:
             context: .
             file: ./Dockerfile.websocket
             push: true
             tags: yourusername/avalon-websocket:latest

     deploy:
       needs: build
       runs-on: ubuntu-latest
       if: github.ref == 'refs/heads/main'
       steps:
         - name: Deploy to production
           uses: appleboy/ssh-action@master
           with:
             host: ${{ secrets.HOST }}
             username: ${{ secrets.USERNAME }}
             key: ${{ secrets.SSH_KEY }}
             script: |
               cd /path/to/avalon
               git pull
               docker-compose pull
               docker-compose up -d
   ```

2. **部署脚本**
   - 编写部署和回滚脚本
   - 创建环境初始化脚本
   - 实现数据备份和恢复工具

### 负载均衡配置

1. **Nginx 配置**

   - 设置 HTTP 和 HTTPS 服务
   - 配置 WebSocket 代理
   - 设置负载均衡策略
   - 配置 SSL 终结和安全头部

   ```nginx
   # nginx/conf.d/default.conf 示例
   upstream api_servers {
       least_conn;
       server api:3000 max_fails=3 fail_timeout=30s;
       # 可以添加更多API服务器
   }

   upstream websocket_servers {
       ip_hash;
       server websocket:3001 max_fails=3 fail_timeout=30s;
       # 可以添加更多WebSocket服务器
   }

   server {
       listen 80;
       server_name example.com;

       # 重定向到HTTPS
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       server_name example.com;

       # SSL配置
       ssl_certificate /etc/nginx/ssl/server.crt;
       ssl_certificate_key /etc/nginx/ssl/server.key;
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_prefer_server_ciphers on;
       ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';

       # 安全头部
       add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
       add_header X-Content-Type-Options nosniff;
       add_header X-Frame-Options SAMEORIGIN;
       add_header X-XSS-Protection "1; mode=block";

       # API请求代理
       location /api/ {
           proxy_pass http://api_servers;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           # 超时设置
           proxy_connect_timeout 60s;
           proxy_send_timeout 60s;
           proxy_read_timeout 60s;
       }

       # WebSocket代理
       location /socket.io/ {
           proxy_pass http://websocket_servers;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

           # WebSocket特定设置
           proxy_socket_keepalive on;
           proxy_connect_timeout 60s;
           proxy_send_timeout 60s;
           proxy_read_timeout 300s; # 更长的读超时用于长连接
       }

       # 健康检查端点
       location /health {
           proxy_pass http://api_servers/health;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;

           # 不缓存健康检查结果
           add_header Cache-Control "no-cache, no-store, must-revalidate";
           add_header Pragma "no-cache";
           add_header Expires "0";
       }

       # 日志配置
       access_log /var/log/nginx/access.log;
       error_log /var/log/nginx/error.log;
   }
   ```

2. **会话粘性**
   - 为 WebSocket 连接配置会话粘性
   - 实现基于 Redis 的集中式会话存储
   - 确保请求路由到正确的服务实例

### 服务发现和弹性扩展

1. **服务发现机制**

   - 配置服务注册和发现
   - 实现健康检查和故障检测
   - 设置服务实例元数据

2. **自动扩展配置**
   - 基于 CPU 和内存使用率的自动扩展规则
   - 根据连接数的扩展策略
   - 配置扩展阈值和冷却期

### 灾备和故障恢复

1. **数据备份策略**

   - 配置数据库自动备份
   - 设置备份验证和测试恢复
   - 实现跨区域备份存储

2. **故障恢复自动化**
   - 设置故障检测和自动恢复
   - 配置回滚触发条件
   - 实现维护模式切换机制

## 工作量估计

3 人天

## 技术关键点

1. 确保容器配置的安全性和资源效率
2. 实现可靠的 CI/CD 流程，支持快速迭代
3. 优化负载均衡策略，确保流量平衡分配
4. 设计弹性扩展机制，应对流量波动
5. 建立完善的灾备和恢复机制

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.1: 基础服务架构](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Docker 文档](https://docs.docker.com/)
- [Docker Compose 文档](https://docs.docker.com/compose/)
- [Nginx 文档](https://nginx.org/en/docs/)
- [GitHub Actions 文档](https://docs.github.com/en/actions)

## 依赖关系

- 上游依赖:
  - Task 4.1.1: 架构设计与规划
  - Task 4.1.2: 核心服务框架搭建
  - Task 4.1.3: 数据库与缓存配置
  - Task 4.1.4: 认证与授权服务
  - Task 4.1.5: 日志与监控系统
