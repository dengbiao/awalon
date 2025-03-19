# Task 4.3.4: 支付与订阅

## 任务描述

设计并实现阿瓦隆微信小游戏的支付与订阅功能，包括微信支付集成、商品管理、订单处理、订阅服务等，为游戏提供可靠的变现能力和良好的用户付费体验。

## 详细需求

### 微信支付集成

1. **支付环境配置**

   - 完成微信支付商户号申请和配置
   - 设置支付安全参数和 API 密钥
   - 实现开发环境与生产环境的隔离

2. **支付接口对接**

   - 实现微信 JSAPI 支付接口调用
   - 开发预支付订单创建功能
   - 实现支付签名生成和验证

3. **支付安全措施**
   - 设计支付数据加密传输机制
   - 实现防重复支付和防篡改措施
   - 开发异常支付处理机制

### 商品系统设计

1. **商品管理**

   - 设计游戏内商品数据模型
   - 实现商品分类、上架、下架功能
   - 支持价格调整和促销活动

2. **游戏内购买流程**

   - 设计用户友好的购买界面
   - 实现商品展示和购买确认流程
   - 支持多商品同时购买

3. **道具系统集成**
   - 设计商品与游戏道具的关联机制
   - 实现购买成功后的道具发放
   - 开发道具使用和库存管理功能

### 订单管理系统

1. **订单生命周期**

   - 实现订单创建、支付、完成的全流程管理
   - 设计订单状态变更机制和事件通知
   - 支持订单查询、取消和退款操作

2. **支付结果处理**

   - 开发支付回调通知接收和处理
   - 实现支付结果验证和订单状态更新
   - 设计订单支付超时处理机制

3. **订单数据分析**
   - 设计订单数据统计模型
   - 实现销售报表和趋势分析
   - 支持异常订单监控和告警

### 订阅服务

1. **会员订阅机制**

   - 设计会员特权和订阅等级
   - 实现会员订阅、续订和取消流程
   - 开发会员状态管理和过期处理

2. **订阅权益发放**

   - 实现订阅用户特权发放机制
   - 设计权益自动更新和检查系统
   - 支持会员专属内容访问控制

3. **订阅数据同步**
   - 实现订阅状态在多端同步
   - 设计订阅数据备份和恢复机制
   - 支持订阅异常处理和客服介入

## 接口设计

### 客户端接口

```typescript
// 获取商品列表接口
GET /api/payment/products
Headers: { Authorization: "Bearer {token}" }
Query: { category?: string, limit?: number, offset?: number }
Response: {
  products: Product[],
  totalCount: number
}

// 创建订单接口
POST /api/payment/createOrder
Headers: { Authorization: "Bearer {token}" }
Request: {
  productIds: string[],
  quantities: number[],
  couponId?: string,
  channel: 'wechat' | 'other'
}
Response: {
  orderId: string,
  totalAmount: number,
  payParams: WxPayParams
}

// 查询订单接口
GET /api/payment/orders/{orderId}
Headers: { Authorization: "Bearer {token}" }
Response: {
  order: Order
}

// 订阅会员接口
POST /api/payment/subscribe
Headers: { Authorization: "Bearer {token}" }
Request: {
  planId: string,
  duration: number  // 单位：月
}
Response: {
  subscriptionId: string,
  orderId: string,
  payParams: WxPayParams
}

// 查询会员状态接口
GET /api/payment/subscription
Headers: { Authorization: "Bearer {token}" }
Response: {
  isActive: boolean,
  plan?: SubscriptionPlan,
  expireAt?: Date,
  autoRenew: boolean
}
```

### 内部服务接口

```typescript
interface PaymentService {
  // 创建支付订单
  createOrder(
    userId: string,
    products: OrderItem[],
    options?: OrderOptions
  ): Promise<Order>;

  // 处理支付结果
  handlePaymentCallback(data: WxPaymentCallback): Promise<PaymentResult>;

  // 查询订单状态
  queryOrder(orderId: string): Promise<Order>;

  // 取消订单
  cancelOrder(orderId: string, reason?: string): Promise<boolean>;

  // 申请退款
  refundOrder(
    orderId: string,
    amount?: number,
    reason?: string
  ): Promise<RefundResult>;

  // 创建订阅
  createSubscription(
    userId: string,
    planId: string,
    duration: number
  ): Promise<Subscription>;

  // 查询订阅状态
  getSubscriptionStatus(userId: string): Promise<SubscriptionStatus>;

  // 取消订阅
  cancelSubscription(subscriptionId: string): Promise<boolean>;
}

interface Product {
  _id: string;
  productId: string;
  name: string;
  description: string;
  category: string;
  price: number;
  originalPrice?: number;
  imageUrl?: string;
  properties?: Record<string, any>;
  stock: number;
  status: "active" | "inactive";
}

interface Order {
  _id: string;
  orderId: string;
  userId: string;
  products: OrderItem[];
  totalAmount: number;
  discountAmount?: number;
  finalAmount: number;
  status: "pending" | "paid" | "completed" | "cancelled" | "refunded";
  paymentChannel: string;
  paymentId?: string;
  createdAt: Date;
  paidAt?: Date;
  completedAt?: Date;
  cancelledAt?: Date;
}

interface OrderItem {
  productId: string;
  productName: string;
  quantity: number;
  unitPrice: number;
  totalPrice: number;
}

interface WxPayParams {
  timeStamp: string;
  nonceStr: string;
  package: string;
  signType: string;
  paySign: string;
}

interface SubscriptionPlan {
  _id: string;
  planId: string;
  name: string;
  description: string;
  price: number;
  duration: number; // 单位：月
  benefits: string[];
  status: "active" | "inactive";
}

interface Subscription {
  _id: string;
  subscriptionId: string;
  userId: string;
  planId: string;
  startAt: Date;
  expireAt: Date;
  autoRenew: boolean;
  status: "active" | "expired" | "cancelled";
  paymentRecords: PaymentRecord[];
}

interface PaymentRecord {
  orderId: string;
  amount: number;
  paidAt: Date;
  period: {
    from: Date;
    to: Date;
  };
}
```

## 数据模型

### 商品数据模型

```typescript
interface Product {
  _id: string;
  productId: string;
  name: string;
  description: string;
  category: string;
  price: number;
  originalPrice?: number;
  discount?: {
    type: "percentage" | "fixed";
    value: number;
    startAt?: Date;
    endAt?: Date;
  };
  imageUrl?: string;
  detailImages?: string[];
  properties?: Record<string, any>;
  stock: number;
  limitPerUser?: number;
  status: "active" | "inactive" | "deleted";
  tags?: string[];
  sortOrder: number;
  createdAt: Date;
  updatedAt: Date;
}

// MongoDB索引
// { productId: 1 }
// { category: 1, status: 1, sortOrder: 1 }
// { status: 1, price: 1 }
```

### 订单数据模型

```typescript
interface Order {
  _id: string;
  orderId: string;
  userId: string;
  openId: string;
  products: {
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    totalPrice: number;
    properties?: Record<string, any>;
  }[];
  totalAmount: number;
  discountAmount?: number;
  finalAmount: number;
  status: "pending" | "paid" | "completed" | "cancelled" | "refunded";
  paymentChannel: "wechat" | "other";
  paymentInfo: {
    prepayId?: string;
    transactionId?: string;
    payTime?: Date;
  };
  refundInfo?: {
    refundId: string;
    refundAmount: number;
    reason: string;
    refundTime: Date;
    status: "pending" | "success" | "failed";
  };
  deliveryInfo?: {
    itemIds: string[];
    deliveryTime: Date;
    status: "pending" | "delivered" | "failed";
  };
  remarks?: string;
  createdAt: Date;
  updatedAt: Date;
  expireAt?: Date; // 订单过期时间
}

// MongoDB索引
// { orderId: 1 }
// { userId: 1, status: 1 }
// { status: 1, createdAt: 1 }
// { expireAt: 1 }, { expireAfterSeconds: 0 }  // TTL索引，自动清理过期订单
```

### 订阅数据模型

```typescript
interface SubscriptionPlan {
  _id: string;
  planId: string;
  name: string;
  description: string;
  price: number;
  duration: number; // 单位：月
  benefits: string[];
  status: "active" | "inactive" | "deleted";
  sortOrder: number;
  createdAt: Date;
  updatedAt: Date;
}

interface Subscription {
  _id: string;
  subscriptionId: string;
  userId: string;
  openId: string;
  planId: string;
  planName: string;
  price: number;
  startAt: Date;
  expireAt: Date;
  autoRenew: boolean;
  status: "active" | "expired" | "cancelled";
  paymentRecords: {
    orderId: string;
    amount: number;
    paidAt: Date;
    period: {
      from: Date;
      to: Date;
    };
  }[];
  renewalNotified: boolean;
  createdAt: Date;
  updatedAt: Date;
}

// MongoDB索引
// { subscriptionId: 1 }
// { userId: 1, status: 1 }
// { status: 1, expireAt: 1 }
// { autoRenew: 1, expireAt: 1 }  // 用于查找需要自动续费的订阅
```

## 实现步骤

1. **配置微信支付环境**

   - 申请微信支付商户号
   - 配置 API 密钥和证书
   - 设置支付回调域名

2. **开发商品管理功能**

   - 实现商品数据模型和数据库操作
   - 开发商品管理接口
   - 实现商品展示和查询功能

3. **实现订单系统**

   - 开发订单创建和管理功能
   - 实现订单状态流转和事件处理
   - 开发订单查询和统计接口

4. **集成微信支付**

   - 实现预支付订单创建
   - 开发支付参数生成和签名
   - 实现支付结果回调处理

5. **开发订阅功能**

   - 实现订阅计划管理
   - 开发订阅创建和续订流程
   - 实现会员权益发放机制

6. **实现支付安全措施**

   - 开发防重复支付机制
   - 实现支付数据加密和验证
   - 开发异常交易监控系统

7. **进行支付测试**
   - 编写支付流程单元测试
   - 进行模拟支付测试
   - 执行真实环境支付测试

## 技术选型

1. **客户端技术**

   - 微信小游戏支付 API
   - TypeScript
   - 状态管理库（如 MobX）

2. **服务端技术**

   - NestJS 框架
   - MongoDB（数据存储）
   - Redis（缓存和分布式锁）
   - Node.js 加密库

3. **支付工具**
   - 微信支付 SDK
   - 加密工具（crypto）
   - 签名验证工具

## 测试要点

1. **功能测试**

   - 商品展示和购买测试
   - 订单创建和状态更新测试
   - 支付流程完整性测试
   - 订阅功能和权益发放测试

2. **异常测试**

   - 支付超时和失败处理测试
   - 订单取消和退款流程测试
   - 网络异常和并发请求测试
   - 订阅特殊场景测试（如手动取消、自动续费失败）

3. **安全测试**

   - 支付签名验证测试
   - 防重复支付和重放攻击测试
   - 支付数据加密和传输安全测试
   - 权限控制和越权操作测试

4. **性能测试**
   - 高并发订单创建测试
   - 大数据量订单查询性能测试
   - 支付回调处理性能测试

## 验收标准

1. 微信支付集成完成，支持正常的支付流程和回调处理
2. 商品系统功能完善，支持商品管理和展示
3. 订单管理系统稳定可靠，能够正确处理订单的全生命周期
4. 订阅服务正常工作，支持会员特权和自动续费
5. 支付安全措施有效，能够防止常见的支付安全问题
6. 所有测试用例通过，包括功能、异常、安全和性能测试
7. 代码符合项目编码规范，并通过代码审查
8. 提供完整的支付和订阅相关文档

## 工作量估计

- 微信支付环境配置：1 人天
- 商品管理功能开发：1.5 人天
- 订单系统实现：2 人天
- 微信支付集成：2 人天
- 订阅功能开发：1.5 人天
- 支付安全措施：1 人天
- 测试和调优：2 人天

总计：**11 人天**

## 依赖关系

- 依赖 Task 4.1.3（数据库与缓存配置）
- 依赖 Task 4.3.1（微信登录与授权）
- 依赖 Task 4.3.2（用户信息处理）

## 风险与应对

| 风险               | 可能性 | 影响 | 应对措施                                     |
| ------------------ | ------ | ---- | -------------------------------------------- |
| 微信支付接口变更   | 低     | 高   | 密切关注微信支付文档更新，设计松耦合支付接口 |
| 支付安全漏洞       | 低     | 高   | 严格遵循微信支付安全规范，实施多重安全验证   |
| 订单数据一致性问题 | 中     | 高   | 实现事务处理和补偿机制，定期数据对账         |
| 高并发支付性能问题 | 中     | 中   | 优化数据库访问，实现请求队列和限流机制       |
| 退款纠纷问题       | 低     | 中   | 完善退款流程和记录，保留足够的日志和证据     |

## 相关文档

- [微信支付开发文档](https://pay.weixin.qq.com/wiki/doc/api/index.html)
- [微信小游戏支付接入指南](https://developers.weixin.qq.com/minigame/dev/guide/open-ability/payment.html)
- [NestJS 事务管理](https://docs.nestjs.com/techniques/database#transactions)
- [MongoDB 事务](https://docs.mongodb.com/manual/core/transactions/)
- [支付安全最佳实践](https://pay.weixin.qq.com/wiki/doc/api/micropay.php?chapter=4_3)
