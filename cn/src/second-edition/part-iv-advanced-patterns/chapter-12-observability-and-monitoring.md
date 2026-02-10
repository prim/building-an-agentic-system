# 第十二章：可观测性与监控模式

构建 AI 编码助手是一回事，理解它在生产环境中的实际行为则是另一个挑战。与可以追踪清晰执行路径的传统软件不同，AI 系统做出概率性决策、衍生并行操作，并以难以观测和调试的方式与外部模型交互。

本章探讨如何在 AI 编码助手中构建全面的可观测性。我们将研究跨智能体和工具的分布式追踪、多智能体系统中的错误聚合、真正重要的性能指标，以及如何利用行为分析来持续改进系统。

## 可观测性挑战

AI 编码助手面临独特的可观测性挑战：

1. **非确定性行为**：相同的输入可能因模型响应不同而产生不同的输出
2. **分布式执行**：工具并行运行、智能体衍生子智能体、操作跨越多个进程
3. **外部依赖**：LLM API、MCP 服务器和其他服务增加了延迟和潜在故障点
4. **上下文窗口**：需要理解做出决策时可用的上下文是什么
5. **用户意图**：需要映射用户请求和系统实际执行之间的关系

传统的 APM 工具不是为这些模式设计的。你需要理解 AI 系统独特特征的可观测性方案。

## AI 系统的分布式追踪

先从分布式追踪开始。在 AI 编码助手架构中，单个用户请求可能衍生多个工具执行，每个工具可能并行运行或触发专业化智能体。以下是实现全面追踪的方法：

```typescript
// Trace context that flows through the entire system
interface TraceContext {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  baggage: Map<string, string>;
}

// Span represents a unit of work
interface Span {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  operationName: string;
  startTime: number;
  endTime?: number;
  tags: Record<string, any>;
  logs: Array<{
    timestamp: number;
    fields: Record<string, any>;
  }>;
  status: 'ok' | 'error' | 'cancelled';
}

class TracingService {
  private spans: Map<string, Span> = new Map();
  private exporter: SpanExporter;

  startSpan(
    operationName: string,
    parent?: TraceContext
  ): { span: Span; context: TraceContext } {
    const span: Span = {
      traceId: parent?.traceId || generateTraceId(),
      spanId: generateSpanId(),
      parentSpanId: parent?.spanId,
      operationName,
      startTime: Date.now(),
      tags: {},
      logs: [],
      status: 'ok'
    };

    this.spans.set(span.spanId, span);

    const context: TraceContext = {
      traceId: span.traceId,
      spanId: span.spanId,
      parentSpanId: parent?.spanId,
      baggage: new Map(parent?.baggage || [])
    };

    return { span, context };
  }

  finishSpan(spanId: string, status: 'ok' | 'error' | 'cancelled' = 'ok') {
    const span = this.spans.get(spanId);
    if (!span) return;

    span.endTime = Date.now();
    span.status = status;

    // Export to your tracing backend
    this.exporter.export([span]);
    this.spans.delete(spanId);
  }

  addTags(spanId: string, tags: Record<string, any>) {
    const span = this.spans.get(spanId);
    if (span) {
      Object.assign(span.tags, tags);
    }
  }

  addLog(spanId: string, fields: Record<string, any>) {
    const span = this.spans.get(spanId);
    if (span) {
      span.logs.push({
        timestamp: Date.now(),
        fields
      });
    }
  }
}
```

接下来为工具执行添加追踪 instrumentation：

```typescript
class InstrumentedToolExecutor {
  constructor(
    private toolExecutor: ToolExecutor,
    private tracing: TracingService
  ) {}

  async executeTool(
    tool: Tool,
    params: any,
    context: TraceContext
  ): Promise<ToolResult> {
    const { span, context: childContext } = this.tracing.startSpan(
      `tool.${tool.name}`,
      context
    );

    // Add tool-specific tags
    this.tracing.addTags(span.spanId, {
      'tool.name': tool.name,
      'tool.params': JSON.stringify(params),
      'tool.parallel': tool.parallel || false
    });

    try {
      // Log tool execution start
      this.tracing.addLog(span.spanId, {
        event: 'tool.start',
        params: params
      });

      const result = await this.toolExecutor.execute(
        tool,
        params,
        childContext
      );

      // Log result
      this.tracing.addLog(span.spanId, {
        event: 'tool.complete',
        resultSize: JSON.stringify(result).length
      });

      this.tracing.finishSpan(span.spanId, 'ok');
      return result;

    } catch (error) {
      // Log error details
      this.tracing.addLog(span.spanId, {
        event: 'tool.error',
        error: error.message,
        stack: error.stack
      });

      this.tracing.addTags(span.spanId, {
        'error': true,
        'error.type': error.constructor.name
      });

      this.tracing.finishSpan(span.spanId, 'error');
      throw error;
    }
  }
}
```

对于并行工具执行，需要追踪父子关系：

```typescript
class ParallelToolTracer {
  async executeParallel(
    tools: Array<{ tool: Tool; params: any }>,
    parentContext: TraceContext
  ): Promise<ToolResult[]> {
    const { span, context } = this.tracing.startSpan(
      'tools.parallel_batch',
      parentContext
    );

    this.tracing.addTags(span.spanId, {
      'batch.size': tools.length,
      'batch.tools': tools.map(t => t.tool.name)
    });

    try {
      const results = await Promise.all(
        tools.map(({ tool, params }) =>
          this.instrumentedExecutor.executeTool(tool, params, context)
        )
      );

      this.tracing.finishSpan(span.spanId, 'ok');
      return results;

    } catch (error) {
      this.tracing.finishSpan(span.spanId, 'error');
      throw error;
    }
  }
}
```

## 错误聚合与调试

在多智能体系统中，错误可能以复杂的方式级联。工具故障可能导致智能体用不同参数重试、衍生子智能体或回退到替代方案。我们需要理解这些模式的错误聚合：

```typescript
interface ErrorContext {
  traceId: string;
  spanId: string;
  timestamp: number;
  error: {
    type: string;
    message: string;
    stack?: string;
  };
  context: {
    tool?: string;
    agent?: string;
    userId?: string;
    threadId?: string;
  };
  metadata: Record<string, any>;
}

class ErrorAggregator {
  private errors: ErrorContext[] = [];
  private patterns: Map<string, ErrorPattern> = new Map();

  recordError(error: Error, span: Span, context: Record<string, any>) {
    const errorContext: ErrorContext = {
      traceId: span.traceId,
      spanId: span.spanId,
      timestamp: Date.now(),
      error: {
        type: error.constructor.name,
        message: error.message,
        stack: error.stack
      },
      context: {
        tool: span.tags['tool.name'],
        agent: span.tags['agent.id'],
        userId: context.userId,
        threadId: context.threadId
      },
      metadata: { ...span.tags, ...context }
    };

    this.errors.push(errorContext);
    this.detectPatterns(errorContext);
    this.maybeAlert(errorContext);
  }

  private detectPatterns(error: ErrorContext) {
    // Group errors by type and context
    const key = `${error.error.type}:${error.context.tool || 'unknown'}`;
    
    if (!this.patterns.has(key)) {
      this.patterns.set(key, {
        count: 0,
        firstSeen: error.timestamp,
        lastSeen: error.timestamp,
        examples: []
      });
    }

    const pattern = this.patterns.get(key)!;
    pattern.count++;
    pattern.lastSeen = error.timestamp;
    
    // Keep recent examples
    if (pattern.examples.length < 10) {
      pattern.examples.push(error);
    }
  }

  private maybeAlert(error: ErrorContext) {
    const pattern = this.patterns.get(
      `${error.error.type}:${error.context.tool || 'unknown'}`
    );

    if (!pattern) return;

    // Alert on error spikes
    const recentErrors = this.errors.filter(
      e => e.timestamp > Date.now() - 60000 // Last minute
    );

    if (recentErrors.length > 10) {
      this.sendAlert({
        type: 'error_spike',
        count: recentErrors.length,
        pattern: pattern,
        example: error
      });
    }

    // Alert on new error types
    if (pattern.count === 1) {
      this.sendAlert({
        type: 'new_error_type',
        pattern: pattern,
        example: error
      });
    }
  }
}
```

对于调试 AI 特有的问题，需要捕获模型交互：

```typescript
class ModelInteractionLogger {
  logInference(request: InferenceRequest, response: InferenceResponse, span: Span) {
    this.tracing.addLog(span.spanId, {
      event: 'model.inference',
      model: request.model,
      promptTokens: response.usage?.promptTokens,
      completionTokens: response.usage?.completionTokens,
      temperature: request.temperature,
      maxTokens: request.maxTokens,
      stopReason: response.stopReason,
      // Store prompt hash for debugging without exposing content
      promptHash: this.hashPrompt(request.messages)
    });

    // Sample full prompts for debugging (with PII scrubbing)
    if (this.shouldSample(span.traceId)) {
      this.storeDebugSample({
        traceId: span.traceId,
        spanId: span.spanId,
        request: this.scrubPII(request),
        response: this.scrubPII(response),
        timestamp: Date.now()
      });
    }
  }

  private shouldSample(traceId: string): boolean {
    // Sample 1% of traces for detailed debugging
    return parseInt(traceId.substring(0, 4), 16) < 0xFFFF * 0.01;
  }
}
```

## 真正重要的性能指标

并非所有指标对 AI 编码助手都同等重要。以下是真正有价值的指标：

```typescript
class AIMetricsCollector {
  // User-facing latency metrics
  private latencyHistogram = new Histogram({
    name: 'ai_operation_duration_seconds',
    help: 'Duration of AI operations',
    labelNames: ['operation', 'model', 'status'],
    buckets: [0.1, 0.5, 1, 2, 5, 10, 30, 60]
  });

  // Token usage for cost tracking
  private tokenCounter = new Counter({
    name: 'ai_tokens_total',
    help: 'Total tokens used',
    labelNames: ['model', 'type'] // type: prompt or completion
  });

  // Tool execution metrics
  private toolExecutions = new Counter({
    name: 'tool_executions_total',
    help: 'Total tool executions',
    labelNames: ['tool', 'status', 'parallel']
  });

  // Context window utilization
  private contextUtilization = new Gauge({
    name: 'context_window_utilization_ratio',
    help: 'Ratio of context window used',
    labelNames: ['model']
  });

  recordOperation(
    operation: string,
    duration: number,
    model: string,
    status: 'success' | 'error' | 'timeout'
  ) {
    this.latencyHistogram
      .labels(operation, model, status)
      .observe(duration / 1000);
  }

  recordTokenUsage(
    model: string,
    promptTokens: number,
    completionTokens: number
  ) {
    this.tokenCounter.labels(model, 'prompt').inc(promptTokens);
    this.tokenCounter.labels(model, 'completion').inc(completionTokens);
  }

  recordToolExecution(
    tool: string,
    status: 'success' | 'error' | 'timeout',
    parallel: boolean
  ) {
    this.toolExecutions
      .labels(tool, status, parallel.toString())
      .inc();
  }

  recordContextUtilization(model: string, used: number, limit: number) {
    this.contextUtilization
      .labels(model)
      .set(used / limit);
  }
}
```

对于系统健康状况，追踪 AI 工作负载特有的资源使用模式：

```typescript
class AISystemHealthMonitor {
  private metrics = {
    // Concurrent operations
    concurrentTools: new Gauge({
      name: 'concurrent_tool_executions',
      help: 'Number of tools currently executing'
    }),
    
    // Queue depths
    pendingOperations: new Gauge({
      name: 'pending_operations',
      help: 'Operations waiting to be processed',
      labelNames: ['type']
    }),
    
    // Model API health
    modelApiErrors: new Counter({
      name: 'model_api_errors_total',
      help: 'Model API errors',
      labelNames: ['model', 'error_type']
    }),
    
    // Memory usage for context
    contextMemoryBytes: new Gauge({
      name: 'context_memory_bytes',
      help: 'Memory used for context storage'
    })
  };

  trackConcurrency(delta: number) {
    this.metrics.concurrentTools.inc(delta);
  }

  trackQueueDepth(type: string, depth: number) {
    this.metrics.pendingOperations.labels(type).set(depth);
  }

  trackModelError(model: string, errorType: string) {
    this.metrics.modelApiErrors.labels(model, errorType).inc();
  }

  trackContextMemory(bytes: number) {
    this.metrics.contextMemoryBytes.set(bytes);
  }
}
```

## 用户行为分析

理解用户如何与 AI 助手交互有助于持续改进系统。追踪揭示用户意图和满意度的模式：

```typescript
interface UserInteraction {
  userId: string;
  threadId: string;
  timestamp: number;
  action: string;
  metadata: Record<string, any>;
}

class UserAnalytics {
  private interactions: UserInteraction[] = [];
  
  // Track user actions
  trackInteraction(action: string, metadata: Record<string, any>) {
    this.interactions.push({
      userId: metadata.userId,
      threadId: metadata.threadId,
      timestamp: Date.now(),
      action,
      metadata
    });
    
    this.analyzePatterns();
  }

  // Common patterns to track
  trackToolUsage(userId: string, tool: string, success: boolean) {
    this.trackInteraction('tool_used', {
      userId,
      tool,
      success,
      // Track if user immediately uses a different tool
      followedBy: this.getNextTool(userId)
    });
  }

  trackRetry(userId: string, originalRequest: string, retryRequest: string) {
    this.trackInteraction('user_retry', {
      userId,
      originalRequest,
      retryRequest,
      // Calculate similarity to understand if it's a clarification
      similarity: this.calculateSimilarity(originalRequest, retryRequest)
    });
  }

  trackContextSwitch(userId: string, fromContext: string, toContext: string) {
    this.trackInteraction('context_switch', {
      userId,
      fromContext,
      toContext,
      // Track if user returns to previous context
      switchDuration: this.getContextDuration(userId, fromContext)
    });
  }

  private analyzePatterns() {
    // Detect frustration signals
    const recentRetries = this.interactions.filter(
      i => i.action === 'user_retry' && 
           i.timestamp > Date.now() - 300000 // Last 5 minutes
    );
    
    if (recentRetries.length > 3) {
      this.alertOnPattern('user_frustration', {
        userId: recentRetries[0].userId,
        retryCount: recentRetries.length
      });
    }

    // Detect successful workflows
    const toolSequences = this.extractToolSequences();
    const commonSequences = this.findCommonSequences(toolSequences);
    
    // These could become suggested workflows or macros
    if (commonSequences.length > 0) {
      this.storeWorkflowPattern(commonSequences);
    }
  }
}
```

追踪决策点以理解 AI 为何做出特定选择：

```typescript
class DecisionTracker {
  trackDecision(
    context: TraceContext,
    decision: {
      type: string;
      options: any[];
      selected: any;
      reasoning?: string;
      confidence?: number;
    }
  ) {
    this.tracing.addLog(context.spanId, {
      event: 'ai.decision',
      decisionType: decision.type,
      optionCount: decision.options.length,
      selectedIndex: decision.options.indexOf(decision.selected),
      confidence: decision.confidence,
      // Hash reasoning to track patterns without storing full text
      reasoningHash: decision.reasoning ? 
        this.hashText(decision.reasoning) : null
    });

    // Track decision patterns
    this.aggregateDecisionPatterns({
      type: decision.type,
      contextSize: this.estimateContextSize(context),
      confidence: decision.confidence,
      timestamp: Date.now()
    });
  }

  private aggregateDecisionPatterns(pattern: DecisionPattern) {
    // Group by decision type and context size buckets
    const bucket = Math.floor(pattern.contextSize / 1000) * 1000;
    const key = `${pattern.type}:${bucket}`;
    
    if (!this.patterns.has(key)) {
      this.patterns.set(key, {
        count: 0,
        totalConfidence: 0,
        contextSizeBucket: bucket
      });
    }
    
    const agg = this.patterns.get(key)!;
    agg.count++;
    agg.totalConfidence += pattern.confidence || 0;
  }
}
```

## 构建有意义的仪表盘

有了所有这些数据，你需要能呈现可操作洞察的仪表盘。以下是需要关注的重点：

```typescript
class AIDashboardMetrics {
  // Real-time health indicators
  getHealthMetrics() {
    return {
      // Is the system responsive?
      p95Latency: this.getPercentileLatency(95),
      errorRate: this.getErrorRate(300), // Last 5 minutes
      
      // Are we hitting limits?
      tokenBurnRate: this.getTokensPerMinute(),
      contextUtilization: this.getAvgContextUtilization(),
      
      // Are tools working?
      toolSuccessRate: this.getToolSuccessRate(),
      parallelExecutionRatio: this.getParallelRatio()
    };
  }

  // User experience metrics
  getUserExperienceMetrics() {
    return {
      // Task completion
      taskCompletionRate: this.getTaskCompletionRate(),
      averageRetriesPerTask: this.getAvgRetries(),
      
      // User satisfaction proxies
      sessionLength: this.getAvgSessionLength(),
      returnUserRate: this.getReturnRate(7), // 7-day return
      
      // Feature adoption
      toolUsageDistribution: this.getToolUsageStats(),
      advancedFeatureAdoption: this.getFeatureAdoption()
    };
  }

  // Cost and efficiency metrics
  getCostMetrics() {
    return {
      // Token costs
      tokensPerUser: this.getAvgTokensPerUser(),
      costPerOperation: this.getAvgCostPerOperation(),
      
      // Efficiency
      cacheHitRate: this.getCacheHitRate(),
      duplicateRequestRate: this.getDuplicateRate(),
      
      // Resource usage
      cpuPerRequest: this.getAvgCPUPerRequest(),
      memoryPerContext: this.getAvgMemoryPerContext()
    };
  }
}
```

## 针对重要事项的告警

不是每个异常都需要告警。专注于真正影响用户的条件：

```typescript
class AIAlertingRules {
  defineAlerts() {
    return [
      {
        name: 'high_error_rate',
        condition: () => this.metrics.errorRate > 0.05, // 5% errors
        severity: 'critical',
        message: 'Error rate exceeds 5%'
      },
      {
        name: 'token_budget_exceeded',
        condition: () => this.metrics.tokenBurnRate > this.budgetLimit,
        severity: 'warning',
        message: 'Token usage exceeding budget'
      },
      {
        name: 'context_overflow',
        condition: () => this.metrics.contextOverflows > 10,
        severity: 'warning',
        message: 'Multiple context window overflows'
      },
      {
        name: 'tool_degradation',
        condition: () => this.metrics.toolSuccessRate < 0.8,
        severity: 'critical',
        message: 'Tool success rate below 80%'
      },
      {
        name: 'user_frustration_spike',
        condition: () => this.metrics.retryRate > 0.3,
        severity: 'warning',
        message: 'High user retry rate indicates confusion'
      }
    ];
  }
}
```

## 实践建议

在 AI 系统中构建可观测性需要一些特殊考量：

1. **从追踪开始**：每个用户请求都应生成一条追踪。这为你提供了完整的事件全貌。

2. **智能采样**：你无法存储每个提示词和响应。基于错误、高延迟或特定用户群体进行采样。

3. **哈希敏感数据**：存储提示词和响应的哈希值用于模式匹配，而不暴露用户数据。

4. **追踪决策而非仅追踪结果**：理解 AI 为何选择特定路径与知道它做了什么同样重要。

5. **构建反馈循环**：利用分析来识别常见模式，并将其作为优化构建到系统中。

6. **监控成本**：Token 使用量可能快速飙升。在用户和操作级别追踪成本。

7. **渐进式 instrumentation**：从基本的追踪和指标开始，随着你了解哪些指标重要，再添加更详细的 instrumentation。

## 总结

AI 系统中的可观测性不仅仅是追踪错误和延迟。它关乎理解系统做出的概率性决策、用户如何与这些决策交互，以及系统可以在哪里改进。

关键在于构建理解 AI 特有模式的可观测性：并行工具执行、模型交互、上下文管理和用户意图。通过适当的 instrumentation，你可以调试复杂的多智能体交互、在关键位置优化性能，并基于真实使用模式持续改进。

记住你的可观测性系统也是一个产品。它需要快速、可靠，并且对运维系统的工程师真正有用。不要仅仅收集指标——构建帮助你理解和改进 AI 助手的工具。

这些可观测性模式为理解生产环境中的复杂 AI 系统提供了基础。它们让你能够在通过数据驱动的洞察持续改进用户体验的同时保持可靠性。

我们在本章探讨的这些模式代表了来自生产系统的经过验证的方法。它们通过无数次调试会话和性能调查得到了打磨。将它们作为起点，但始终根据你特定系统的需求和约束进行调整。
