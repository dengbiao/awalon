# Task 4.1.5: 日志与监控系统

## 描述

设计并实现阿瓦隆微信小游戏的日志记录和系统监控框架，包括应用日志、性能指标收集、告警系统和可视化监控面板，确保系统运行状态可观察性和问题的快速定位。

## 验收标准

1. 实现多级别、结构化的日志系统
2. 配置集中式日志收集和存储方案
3. 实现系统关键性能指标的收集
4. 创建监控指标的可视化仪表板
5. 设置基于阈值的告警规则
6. 实现异常事件通知机制
7. 配置日志和指标的保留策略
8. 创建运维相关的 API 端点
9. 编写监控与日志系统的使用文档

## 详细内容

### 日志系统实现

1. **日志框架集成**

   - 使用 Winston 作为主要日志库
   - 配置多目标输出(控制台、文件、远程服务)
   - 设置不同环境的日志级别
   - 实现结构化 JSON 日志格式

   ```typescript
   // logger.service.ts 示例
   @Injectable()
   export class LoggerService implements LoggerService {
     private logger: winston.Logger;

     constructor(private configService: ConfigService) {
       const environment = this.configService.get("NODE_ENV", "development");
       const logLevel = environment === "production" ? "info" : "debug";

       this.logger = winston.createLogger({
         level: logLevel,
         format: winston.format.combine(
           winston.format.timestamp(),
           winston.format.json()
         ),
         defaultMeta: { service: "avalon-game" },
         transports: [
           new winston.transports.Console({
             format: winston.format.combine(
               winston.format.colorize(),
               winston.format.simple()
             ),
           }),
           new winston.transports.File({
             filename: "logs/error.log",
             level: "error",
             maxsize: 10485760, // 10MB
             maxFiles: 5,
           }),
           new winston.transports.File({
             filename: "logs/combined.log",
             maxsize: 10485760, // 10MB
             maxFiles: 5,
           }),
         ],
       });

       // 生产环境添加远程日志收集
       if (environment === "production") {
         // 可添加Logstash, Elasticsearch等传输
       }
     }

     log(message: string, context?: string) {
       this.logger.info(message, { context });
     }

     error(message: string, trace?: string, context?: string) {
       this.logger.error(message, { trace, context });
     }

     warn(message: string, context?: string) {
       this.logger.warn(message, { context });
     }

     debug(message: string, context?: string) {
       this.logger.debug(message, { context });
     }

     verbose(message: string, context?: string) {
       this.logger.verbose(message, { context });
     }
   }
   ```

2. **全局日志拦截器**

   - 记录请求和响应信息
   - 捕获并记录异常
   - 添加请求跟踪 ID
   - 记录性能计时数据

   ```typescript
   // logging.interceptor.ts 示例
   @Injectable()
   export class LoggingInterceptor implements NestInterceptor {
     constructor(private readonly loggerService: LoggerService) {}

     intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
       const request = context.switchToHttp().getRequest();
       const { method, url, body, ip, headers } = request;
       const userAgent = headers["user-agent"] || "";
       const requestId = headers["x-request-id"] || uuid();

       const startTime = Date.now();

       request.requestId = requestId;

       this.loggerService.log(`请求开始 - ${method} ${url}`, {
         requestId,
         ip,
         body,
         userAgent,
       });

       return next.handle().pipe(
         tap({
           next: (data) => {
             const responseTime = Date.now() - startTime;
             this.loggerService.log(
               `请求结束 - ${method} ${url} - ${responseTime}ms`,
               { requestId, responseTime, response: data }
             );
           },
           error: (error) => {
             const responseTime = Date.now() - startTime;
             this.loggerService.error(
               `请求错误 - ${method} ${url} - ${error.message}`,
               error.stack,
               { requestId, responseTime }
             );
           },
         })
       );
     }
   }
   ```

3. **日志聚合与分析**
   - 配置日志收集管道
   - 设置日志分析工具
   - 创建常用日志查询和可视化

### 性能监控系统

1. **指标收集**

   - 集成 Prometheus 客户端
   - 定义和收集关键业务指标
   - 实现自定义指标和标签

   ```typescript
   // metrics.service.ts 示例
   @Injectable()
   export class MetricsService {
     private register: Registry;
     private httpRequestsCounter: Counter;
     private httpRequestDurationHistogram: Histogram;
     private wsConnectionsGauge: Gauge;
     private activeGamesGauge: Gauge;

     constructor() {
       this.register = new Registry();

       // 定义指标
       this.httpRequestsCounter = new Counter({
         name: "http_requests_total",
         help: "Total number of HTTP requests",
         labelNames: ["method", "endpoint", "status"],
         registers: [this.register],
       });

       this.httpRequestDurationHistogram = new Histogram({
         name: "http_request_duration_seconds",
         help: "HTTP request duration in seconds",
         labelNames: ["method", "endpoint"],
         buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
         registers: [this.register],
       });

       this.wsConnectionsGauge = new Gauge({
         name: "websocket_connections",
         help: "Current number of WebSocket connections",
         registers: [this.register],
       });

       this.activeGamesGauge = new Gauge({
         name: "active_games",
         help: "Current number of active games",
         registers: [this.register],
       });
     }

     // 指标记录方法
     recordHttpRequest(method: string, endpoint: string, status: number): void {
       this.httpRequestsCounter.inc({ method, endpoint, status });
     }

     startHttpRequestTimer(method: string, endpoint: string): Histogram.Timer {
       return this.httpRequestDurationHistogram.startTimer({
         method,
         endpoint,
       });
     }

     setWsConnections(count: number): void {
       this.wsConnectionsGauge.set(count);
     }

     incrementWsConnections(): void {
       this.wsConnectionsGauge.inc();
     }

     decrementWsConnections(): void {
       this.wsConnectionsGauge.dec();
     }

     setActiveGames(count: number): void {
       this.activeGamesGauge.set(count);
     }

     // 指标公开
     getMetrics(): Promise<string> {
       return this.register.metrics();
     }

     getContentType(): string {
       return this.register.contentType;
     }
   }
   ```

2. **健康检查 API**

   - 实现系统组件健康检查
   - 数据库连接状态检查
   - Redis 连接状态检查
   - 磁盘空间和内存使用检查
   - 外部服务依赖检查

   ```typescript
   // health.controller.ts 示例
   @Controller("health")
   export class HealthController {
     constructor(
       private readonly health: HealthCheckService,
       private readonly db: MongooseHealthIndicator,
       private readonly redis: RedisHealthIndicator,
       private readonly disk: DiskHealthIndicator,
       private readonly memory: MemoryHealthIndicator
     ) {}

     @Get()
     @HealthCheck()
     check() {
       return this.health.check([
         // MongoDB健康检查
         () => this.db.pingCheck("mongodb"),

         // Redis健康检查
         () => this.redis.pingCheck("redis"),

         // 磁盘空间检查
         () =>
           this.disk.checkStorage("storage", {
             path: "/",
             thresholdPercent: 0.9,
           }),

         // 内存使用检查
         () => this.memory.checkHeap("memory_heap", 300 * 1024 * 1024), // 300MB

         // 外部服务健康检查
         // ...
       ]);
     }
   }
   ```

3. **指标可视化**
   - 使用 Grafana 配置指标仪表板
   - 创建业务指标和技术指标视图
   - 设置趋势分析和异常检测图表

### 告警系统

1. **告警规则配置**

   - 基于 Prometheus Alert Manager 配置
   - 定义关键指标的告警阈值
   - 设置告警规则的频率和严重性级别

2. **通知渠道**

   - 配置邮件告警
   - 设置微信/钉钉等即时通讯工具告警
   - 配置 PagerDuty 等专业告警工具
   - 实现告警升级机制

3. **告警分组和路由**
   - 根据告警类型分组
   - 设置不同告警的处理人员
   - 配置告警静默和抑制规则

### 运维 API 与工具

1. **运维 API 端点**

   - 创建日志级别动态调整 API
   - 实现指标查询和分析 API
   - 添加系统状态检查和管理 API

2. **运维工具脚本**
   - 编写日志轮转和归档脚本
   - 创建性能分析和问题定位工具
   - 实现自动化监控检查脚本

## 工作量估计

3 人天

## 技术关键点

1. 设计既全面又不过度耗费资源的日志系统
2. 选择和收集最有价值的系统和业务指标
3. 确保告警规则的有效性，避免过多误报
4. 优化监控系统自身的性能，避免成为性能瓶颈
5. 实现对分布式系统的全面监控

## 相关文档

- [技术方案文档](./技术方案.md)
- [Story 4.1: 基础服务架构](./README.md)
- [Epic 4: 服务器架构设计](../README.md)
- [Winston 文档](https://github.com/winstonjs/winston)
- [Prometheus 文档](https://prometheus.io/docs/introduction/overview/)
- [Grafana 文档](https://grafana.com/docs/)
- [NestJS 终端健康检查](https://docs.nestjs.com/recipes/terminus)

## 依赖关系

- 上游依赖:
  - Task 4.1.1: 架构设计与规划
  - Task 4.1.2: 核心服务框架搭建
  - Task 4.1.3: 数据库与缓存配置
  - Task 4.1.4: 认证与授权服务
- 下游依赖:
  - Task 4.1.6: 服务部署与配置
