# 第十三章：构建你自己的协作式 AI 助手

所以你想构建一个协作式 AI 编码助手。也许你受到了我们探讨的架构模式的启发，或者你的团队有现有工具无法满足的特定需求。本章提供了构建自己系统的实用路线图，汲取了全书的经验教训。

## 从"为什么"开始

在深入技术选择之前，先明确你的目标。你是为了：
- 一个需要自定义集成的小团队？
- 一个有特定安全要求的企业？
- 一个面向开发者的 SaaS 产品？
- 一个需要与专有系统对接的内部工具？

你的答案决定了后续的每一个决策。为五个开发者构建的系统与服务数千人的系统截然不同。

## 架构决策清单

让我们按重要性和依赖顺序逐一梳理你将面临的关键架构决策。

### 1. 部署模型

**决策**：你的系统将运行在哪里？

选项：
- **本地优先带同步**：类似 Amp 的原始架构。每个开发者运行自己的实例，可选云端同步。
- **云原生**：一切运行在云端，通过 Web 或轻客户端访问。
- **混合模式**：本地执行配合云端功能（存储、协作、计算）。

权衡：
- 本地优先提供隐私保护且支持离线工作，但增加协作复杂性
- 云原生简化部署但需要稳定的网络连接
- 混合模式两者兼顾但增加复杂性

MVP 建议：如果隐私重要就选本地优先，如果协作是首要目标就选云原生。

### 2. 语言模型集成

**决策**：如何集成 LLM？

选项：
- **直接 API 集成**：直接调用 OpenAI、Anthropic 等
- **网关服务**：通过统一的 API 层路由
- **本地模型**：在内部运行开源模型
- **混合方案**：网关配合回退选项

权衡：
- 直接集成简单但锁定供应商
- 网关增加复杂性但提供灵活性
- 本地模型提供控制权但需要大量资源

MVP 建议：从一个供应商的直接集成开始，但在设计上预留抽象层。

### 3. 工具系统架构

**决策**：工具如何与系统交互？

选项：
- **仅内置工具**：固定的功能集
- **插件架构**：动态工具加载
- **进程级隔离**：工具在独立进程中运行
- **语言无关协议**：支持任何语言编写的工具

权衡：
- 内置方式实现最快但限制可扩展性
- 插件提供灵活性但需要精心的 API 设计
- 进程隔离提升安全性但增加开销
- 语言无关方式最大化灵活性但增加复杂度

MVP 建议：从内置工具开始，设计好接口以便未来扩展。

### 4. 状态管理

**决策**：如何管理对话和系统状态？

选项：
- **仅内存**：简单但重启后丢失状态
- **基于文件的持久化**：JSONLines、SQLite 或类似方案
- **数据库支撑**：PostgreSQL、MongoDB 等
- **事件溯源**：完整历史记录，支持重放

权衡：
- 纯内存方案微不足道但不适合实际使用
- 基于文件适用于单用户场景
- 数据库支持多用户但增加运维复杂性
- 事件溯源提供审计追踪但需要精心设计

MVP 建议：单用户用基于文件的方案，多用户用 PostgreSQL。

### 5. 实时通信

**决策**：组件之间如何通信？

选项：
- **REST API**：简单的请求-响应
- **WebSocket**：双向流式传输
- **Server-Sent Events**：单向流式传输
- **gRPC**：高性能 RPC
- **消息队列**：异步通信

权衡：
- REST 普遍支持但非实时
- WebSocket 支持实时但需要连接管理
- SSE 比 WebSocket 简单但只支持单向
- gRPC 性能优秀但生态支持较少
- 消息队列解耦组件但增加基础设施

MVP 建议：REST + SSE 用于流式响应。

### 6. 认证与授权

**决策**：如何处理身份和权限？

选项：
- **无认证**：单用户系统
- **基础认证**：简单的用户名/密码
- **OAuth/OIDC**：与现有提供商集成
- **API 密钥**：用于程序化访问
- **RBAC**：基于角色的访问控制

权衡：
- 无认证只适用于个人工具
- 基础认证简单但安全性较低
- OAuth 利用现有身份体系但增加复杂性
- API 密钥适合自动化场景
- RBAC 可扩展但需要精心设计

MVP 建议：从 API 密钥开始，需要时再添加 OAuth。

## 技术栈推荐

根据上述决策，以下是不同场景的推荐技术栈。

### 小型团队（1-10 名开发者）

**后端栈**：
```
Language: TypeScript/Node.js or Python
Framework: Express + Socket.io or FastAPI
Database: SQLite or PostgreSQL
Cache: In-memory or Redis
Queue: Bull (Node) or Celery (Python)
```

**前端栈**：
```
CLI: Ink (React for terminals) or Click (Python)
Web UI: React or Vue with Tailwind
State: Zustand or Pinia
Real-time: Socket.io client or native WebSocket
```

**基础设施**：
```
Deployment: Docker Compose
CI/CD: GitHub Actions
Monitoring: Prometheus + Grafana
Logging: Loki or ELK stack
```

### 中型组织（10-100 名开发者）

**后端栈**：
```
Language: Go or Rust for performance
Framework: Gin (Go) or Axum (Rust)
Database: PostgreSQL with read replicas
Cache: Redis cluster
Queue: RabbitMQ or NATS
Search: Elasticsearch
```

**前端栈**：
```
CLI: Distributed as binary
Web UI: Next.js or SvelteKit
State: Redux Toolkit or MobX
Real-time: WebSocket with fallbacks
Mobile: React Native or Flutter
```

**基础设施**：
```
Orchestration: Kubernetes
Service Mesh: Istio or Linkerd
CI/CD: GitLab CI or Jenkins
Monitoring: Datadog or New Relic
Security: Vault for secrets
```

### SaaS 产品（100+ 名开发者）

**后端栈**：
```
Language: Multiple services in appropriate languages
API Gateway: Kong or AWS API Gateway
Database: PostgreSQL + DynamoDB
Cache: Redis + CDN
Queue: Kafka or AWS SQS
Search: Algolia or Elasticsearch
```

**前端栈**：
```
CLI: Multiple platform builds
Web UI: Micro-frontends architecture
State: Service-specific stores
Real-time: Managed WebSocket service
SDKs: Multiple language clients
```

**基础设施**：
```
Cloud: AWS, GCP, or Azure
Orchestration: Managed Kubernetes (EKS, GKE, AKS)
CI/CD: CircleCI or AWS CodePipeline
Monitoring: Full APM solution
Security: WAF, DDoS protection, SOC2 compliance
```

## MVP 功能集

以下是一个务实的 MVP，在保持范围可控的同时提供真正的价值。

### 核心功能（第 1-4 周）

1. **基本聊天界面**
   - 带消息历史的终端 UI
   - 响应的 Markdown 渲染
   - 文件路径检测和验证

2. **文件操作**
   - 带行号的文件读取
   - 创建新文件
   - 编辑现有文件（基于 diff）
   - 列出目录内容

3. **代码搜索**
   - Grep 功能
   - 文件模式匹配（glob）
   - 基本上下文提取

4. **Shell 集成**
   - 带审批的命令执行
   - 输出捕获
   - 工作目录管理

5. **对话管理**
   - 保存/加载对话
   - 清除历史
   - 导出记录

### 认证（第 5 周）

1. **API 密钥管理**
   - 生成/撤销密钥
   - 使用量追踪
   - 速率限制

2. **LLM 配置**
   - 供应商选择
   - 模型选择
   - Temperature 设置

### 增强功能（第 6-8 周）

1. **上下文感知**
   - Git 集成（status、diff）
   - 项目类型检测
   - 忽略文件处理

2. **工具扩展**
   - Web 搜索能力
   - 文档查找
   - 包管理器集成

3. **易用性**
   - 语法高亮
   - 自动保存对话
   - 键盘快捷键
   - 命令历史

### 协作功能（第 9-12 周）

1. **分享**
   - 通过链接分享对话
   - 公开/私有可见性
   - 过期控制

2. **团队功能**
   - 共享对话库
   - 团队成员权限
   - 使用量分析

3. **集成**
   - Slack 通知
   - GitHub 集成
   - IDE 扩展

## 实施路线图

### 阶段一：基础（第 1-4 周）

专注于获得一个能协助真实编码任务的工作系统。

```typescript
// Start with a simple tool interface
interface Tool {
  name: string;
  description: string;
  parameters: JSONSchema;
  execute(params: any): Promise<ToolResult>;
}

// Basic tools to implement first
const readFile: Tool = {
  name: "read_file",
  description: "Read contents of a file",
  parameters: {
    type: "object",
    properties: {
      path: { type: "string" }
    },
    required: ["path"]
  },
  async execute({ path }) {
    // Implementation
  }
};
```

关键里程碑：
- 第 1 周：带 LLM 集成的基本聊天循环
- 第 2 周：文件操作就绪
- 第 3 周：搜索和 Shell 命令
- 第 4 周：持久化和错误处理

### 阶段二：易用性（第 5-8 周）

让系统适合日常使用。

- 改善响应流式传输
- 添加进度指示器
- 实现编辑的撤销/重做
- 打磨错误消息
- 添加配置选项

### 阶段三：协作（第 9-12 周）

支持团队使用。

- 构建分享基础设施
- 添加访问控制
- 实现使用量追踪
- 创建管理界面
- 编写部署文档

### 阶段四：扩展（第 4-6 月）

为增长做准备。

- 性能优化
- 水平扩展
- 监控和告警
- 安全加固
- 合规功能

## 扩展性考量

从第一天就为扩展性设计，即使你暂时不需要。

### 数据架构

**对话存储**：
- 从一开始就按用户/团队分区
- 使用 UUID，不用自增 ID
- 为最终分片设计
- 将热数据与冷数据分离

**文件处理**：
- 大文件用流式处理，不要加载到内存
- 缓存频繁访问的内容
- 共享资源使用 CDN
- 实现渐进式加载

### 性能模式

**工具执行**：
```typescript
// Design for parallel execution from the start
class ToolExecutor {
  async executeBatch(tools: ToolCall[]): Promise<ToolResult[]> {
    // Group by dependency
    const groups = this.groupByDependency(tools);
    
    const results: ToolResult[] = [];
    for (const group of groups) {
      // Execute independent tools in parallel
      const groupResults = await Promise.all(
        group.map(tool => this.execute(tool))
      );
      results.push(...groupResults);
    }
    
    return results;
  }
}
```

**响应流式传输**：
- 使用 Server-Sent Events 或 WebSocket
- Token 到达即流式传输
- 缓冲至最优块大小
- 处理连接中断

### 安全考量

**输入验证**：
- 清理所有文件路径
- 验证命令输入
- 按用户和端点限速
- 实现请求签名

**隔离**：
- 在沙盒环境中运行工具
- 使用独立的服务账号
- 实现最小权限原则
- 审计所有操作

### 运维卓越

**监控**：
```yaml
# Key metrics to track from day one
metrics:
  - api_request_duration
  - llm_token_usage
  - tool_execution_time
  - error_rates
  - active_users
  - conversation_length
```

**部署**：
- 自动化一切
- 使用功能标志
- 实现灰度发布
- 规划回滚方案
- 编写运维手册

## 常见陷阱

1. **过度设计 MVP**：在核心功能完善之前抵制添加功能的冲动。

2. **忽视运维**：日志、监控和部署自动化的投入回报丰厚。

3. **与 LLM 供应商紧耦合**：即使只用一个供应商也要尽早抽象。

4. **低估 UI/UX**：开发者工具也需要良好的设计。

5. **跳过测试**：工具的集成测试能节省大量调试时间。

6. **过早优化**：先做性能分析，优化真正重要的部分。

7. **忽视安全**：从一开始就把安全融入设计，而不是事后补救。

## 入门清单

准备好开始了？这是你第一周的清单：

- [ ] 搭建带 CI/CD 流水线的代码仓库
- [ ] 选择并配置 LLM 供应商
- [ ] 实现基本聊天循环
- [ ] 添加文件读取功能
- [ ] 创建简单的 CLI 界面
- [ ] 搭建开发环境
- [ ] 编写第一个集成测试
- [ ] 部署 Hello World 版本
- [ ] 编写搭建文档
- [ ] 获取第一个用户反馈

## 结论

构建协作式 AI 编码助手是一项雄心勃勃的任务，但本书中的模式和经验教训提供了坚实的基础。从简单开始，专注核心价值，根据用户反馈迭代。

记住：目标不是完全复制任何现有系统，而是创造满足你特定需求的东西。把这些模式当作灵感而非处方。最好的系统是你的团队实际使用的、能自然融入开发工作流的系统。

软件开发的未来涉及 AI 协作。通过构建自己的系统，你不仅仅在创造一个工具——你在塑造团队未来的工作方式。无论你是为小团队还是大型企业组织构建，这些架构模式都为创建真正提升开发者生产力的 AI 编码助手提供了基础。
