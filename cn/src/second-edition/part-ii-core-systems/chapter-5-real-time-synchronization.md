# 第五章：实时同步

构建协作式 AI 编码助手需要让多个客户端保持实时同步。当一个开发者做出更改时，他们的队友需要立即看到更新。但与传统的实时应用不同，AI 助手面临独特的挑战：长时间运行的操作、大负载、不可靠的网络，以及最终一致性的需求。

本章探讨使用轮询、Observable 和智能批处理的同步模式，这些模式在 AI 系统中比传统 WebSocket 方案更加可靠。

## 同步的挑战

AI 助手的实时同步与典型的协作应用不同：

1. **大负载** - AI 响应可能是数兆字节的文本和代码
2. **长时间操作** - 工具执行可能需要数分钟才能完成
3. **不可靠的网络** - 开发者在咖啡厅、火车上和不稳定的 WiFi 环境中工作
4. **成本敏感** - 每次同步操作都会产生 API 调用费用
5. **一致性要求** - 代码变更必须按正确顺序应用

传统的 WebSocket 方案在这些约束条件下表现不佳。Amp 选择了另一条路径。

## WebSocket 在 AI 系统中的挑战

WebSocket 看起来是实时同步的理想选择，但 AI 系统提出了独特的挑战，使其变得棘手。

**识别模式**：当出现以下情况时，WebSocket 会变得有问题：
- 客户端频繁断开连接（移动网络、笔记本休眠）
- 消息大小差异巨大（小更新 vs. 大型 AI 响应）
- 操作持续时间长（多分钟的工具执行）
- 调试需要消息回放和检查

**WebSocket 的复杂性**：

- **有状态连接**需要精心的生命周期管理
- **消息排序**必须显式处理以确保正确性
- **重连风暴**在故障期间可能压垮服务器
- **调试**在没有适当消息日志的情况下很困难
- **负载均衡**需要粘性会话或复杂的路由
- 企业环境中的**防火墙问题**

**替代方案**：智能轮询配合 Observable 提供：
- **无状态交互**能够承受网络中断
- **自然的批处理**减少服务器负载
- **简单的调试**使用标准 HTTP 请求日志
- **简单的缓存**和 CDN 兼容性

## 基于 Observable 的架构

Amp 同步系统的核心是自定义 Observable 实现：

```typescript
export abstract class Observable<T> {
  abstract subscribe(observer: Observer<T>): Subscription<T>;
  
  pipe<Out>(...operators: Operator[]): Observable<Out> {
    return operators.reduce(
      (source, operator) => operator(source),
      this as Observable<any>
    );
  }
  
  // Convert various sources to Observables
  static from<T>(source: ObservableLike<T>): Observable<T> {
    if (source instanceof Observable) return source;
    
    if (isPromise(source)) {
      return new Observable(observer => {
        source.then(
          value => {
            observer.next(value);
            observer.complete();
          },
          error => observer.error(error)
        );
      });
    }
    
    if (isIterable(source)) {
      return new Observable(observer => {
        for (const value of source) {
          observer.next(value);
        }
        observer.complete();
      });
    }
    
    throw new Error('Invalid source');
  }
}
```

这为整个系统的响应式数据流提供了基础。

## 用于状态广播的 Subject

Amp 使用特定的 Subject 类型来满足不同的同步需求：

```typescript
// BehaviorSubject maintains current state
export class BehaviorSubject<T> extends Observable<T> {
  constructor(private currentValue: T) {
    super();
  }
  
  getValue(): T {
    return this.currentValue;
  }
  
  next(value: T): void {
    this.currentValue = value;
    this.observers.forEach(observer => observer.next(value));
  }
  
  subscribe(observer: Observer<T>): Subscription<T> {
    // New subscribers immediately receive current value
    observer.next(this.currentValue);
    return super.subscribe(observer);
  }
}

// SetSubject for managing collections
export function createSetSubject<T>(): SetSubject<T> {
  const set = new Set<T>();
  const subject = new BehaviorSubject<Set<T>>(set);
  
  return {
    add(value: T): void {
      set.add(value);
      subject.next(set);
    },
    
    delete(value: T): void {
      set.delete(value);
      subject.next(set);
    },
    
    has(value: T): boolean {
      return set.has(value);
    },
    
    clear(): void {
      set.clear();
      subject.next(set);
    },
    
    get size(): number {
      return set.size;
    },
    
    observable: subject.asObservable()
  };
}
```

这些模式实现了跨组件的高效状态同步。

## 同步服务架构

Amp 的同步系统提供 Observable 流和队列管理：

```typescript
// Core synchronization interface
export interface SyncService {
  // Observable data streams
  observeSyncStatus(threadId: ThreadID): Observable<SyncStatus>;
  observePendingItems(): Observable<Set<ThreadID>>;
  
  // Sync operations
  queueForSync(threadId: ThreadID): void;
  syncImmediately(threadId: ThreadID): Promise<void>;
  
  // Service lifecycle
  start(): void;
  stop(): void;
  dispose(): void;
}

// Factory function creates configured sync service
export function createSyncService(dependencies: {
  threadService: ThreadService;
  cloudAPI: CloudAPIClient;
  configuration: ConfigService;
}): SyncService {
  // Track items waiting for synchronization
  const pendingItems = createSetSubject<ThreadID>();
  
  // Per-thread sync status tracking
  const statusTracking = new Map<ThreadID, BehaviorSubject<SyncStatus>>();
  
  // Failure tracking for exponential backoff
  const failureHistory = new Map<ThreadID, number>();
  
  // Configurable sync parameters
  const SYNC_INTERVAL = 5000;         // 5 seconds
  const RETRY_BACKOFF = 60000;        // 1 minute
  const BATCH_SIZE = 50;              // Items per batch
  
  let syncTimer: NodeJS.Timer | null = null;
  let serviceRunning = false;
  
  return {
    observeSyncStatus(threadId: ThreadID): Observable<SyncStatus> {
      if (!statusTracking.has(threadId)) {
        statusTracking.set(threadId, new BehaviorSubject<SyncStatus>({
          state: 'unknown',
          lastSync: null
        }));
      }
      return statusTracking.get(threadId)!.asObservable();
    },
    
    observePendingItems(): Observable<Set<ThreadID>> {
      return pendingItems.observable;
    },
    
    queueForSync(threadId: ThreadID): void {
      pendingItems.add(threadId);
      updateSyncStatus(threadId, { state: 'pending' });
    },
    
    async syncImmediately(threadId: ThreadID): Promise<void> {
      // Bypass queue for high-priority sync
      await performThreadSync(threadId);
    },
    
    start(): void {
      if (serviceRunning) return;
      serviceRunning = true;
      
      // Begin periodic sync processing
      scheduleSyncLoop();
      
      // Set up reactive change detection
      setupChangeListeners();
    },
    
    stop(): void {
      serviceRunning = false;
      if (syncTimer) {
        clearTimeout(syncTimer);
        syncTimer = null;
      }
    },
    
    dispose(): void {
      this.stop();
      statusTracking.forEach(subject => subject.complete());
      statusTracking.clear();
    }
  };
  
  function scheduleSyncLoop(): void {
    if (!serviceRunning) return;
    
    syncTimer = setTimeout(async () => {
      await processQueuedItems();
      scheduleSyncLoop();
    }, SYNC_INTERVAL);
  }
  
  async function processQueuedItems(): Promise<void> {
    const queuedThreads = Array.from(pendingItems.set);
    if (queuedThreads.length === 0) return;
    
    // Filter items ready for sync (respecting backoff)
    const readyItems = queuedThreads.filter(shouldAttemptSync);
    if (readyItems.length === 0) return;
    
    // Process in manageable batches
    for (let i = 0; i < readyItems.length; i += BATCH_SIZE) {
      const batch = readyItems.slice(i, i + BATCH_SIZE);
      await processBatch(batch);
    }
  }
  
  function shouldAttemptSync(threadId: ThreadID): boolean {
    const lastFailure = failureHistory.get(threadId);
    if (!lastFailure) return true;
    
    const timeSinceFailure = Date.now() - lastFailure;
    return timeSinceFailure >= RETRY_BACKOFF;
  }
}
```

## 自适应轮询策略

Amp 不使用固定间隔轮询，而是根据用户活动进行自适应调整：

```typescript
// Dynamically adjusts polling frequency based on activity
export class AdaptivePoller {
  private baseInterval = 5000;    // 5 seconds baseline
  private maxInterval = 60000;    // 1 minute maximum
  private currentInterval = this.baseInterval;
  private activityLevel = 0;
  
  constructor(
    private syncService: SyncService,
    private threadService: ThreadService
  ) {
    this.setupActivityMonitoring();
  }
  
  private setupActivityMonitoring(): void {
    // Monitor thread modifications for user activity
    this.threadService.observeActiveThread().pipe(
      pairwise(),
      filter(([previous, current]) => previous?.v !== current?.v),
      tap(() => this.recordUserActivity())
    ).subscribe();
    
    // Monitor sync queue depth to adjust frequency
    this.syncService.observePendingItems().pipe(
      map(pending => pending.size),
      tap(queueDepth => {
        if (queueDepth > 10) this.increaseSyncFrequency();
        if (queueDepth === 0) this.decreaseSyncFrequency();
      })
    ).subscribe();
  }
  
  private recordUserActivity(): void {
    this.activityLevel = Math.min(100, this.activityLevel + 10);
    this.adjustPollingInterval();
  }
  
  private adjustPollingInterval(): void {
    // Higher activity leads to more frequent polling
    const scaleFactor = 1 - (this.activityLevel / 100) * 0.8;
    this.currentInterval = Math.floor(
      this.baseInterval + (this.maxInterval - this.baseInterval) * scaleFactor
    );
    
    // Schedule activity decay for gradual slow-down
    this.scheduleActivityDecay();
  }
  
  private scheduleActivityDecay(): void {
    setTimeout(() => {
      this.activityLevel = Math.max(0, this.activityLevel - 1);
      this.adjustPollingInterval();
    }, 1000);
  }
  
  getCurrentInterval(): number {
    return this.currentInterval;
  }
}
```

## 防抖与节流

Amp 实现了精密的流控机制以防止系统过载：

```typescript
// Debounce rapid changes
export function debounceTime<T>(
  duration: number
): OperatorFunction<T, T> {
  return (source: Observable<T>) => 
    new Observable<T>(observer => {
      let timeoutId: NodeJS.Timeout | null = null;
      let lastValue: T;
      let hasValue = false;
      
      const subscription = source.subscribe({
        next(value: T) {
          lastValue = value;
          hasValue = true;
          
          if (timeoutId) {
            clearTimeout(timeoutId);
          }
          
          timeoutId = setTimeout(() => {
            if (hasValue) {
              observer.next(lastValue);
              hasValue = false;
            }
            timeoutId = null;
          }, duration);
        },
        
        error(err) {
          observer.error(err);
        },
        
        complete() {
          if (timeoutId) {
            clearTimeout(timeoutId);
            if (hasValue) {
              observer.next(lastValue);
            }
          }
          observer.complete();
        }
      });
      
      return () => {
        if (timeoutId) {
          clearTimeout(timeoutId);
        }
        subscription.unsubscribe();
      };
    });
}

// Throttle with leading and trailing edges
export function throttleTime<T>(
  duration: number,
  { leading = true, trailing = true } = {}
): OperatorFunction<T, T> {
  return (source: Observable<T>) =>
    new Observable<T>(observer => {
      let lastEmitTime = 0;
      let trailingTimeout: NodeJS.Timeout | null = null;
      let lastValue: T;
      let hasTrailingValue = false;
      
      const emit = (value: T) => {
        lastEmitTime = Date.now();
        hasTrailingValue = false;
        observer.next(value);
      };
      
      const subscription = source.subscribe({
        next(value: T) {
          const now = Date.now();
          const elapsed = now - lastEmitTime;
          
          lastValue = value;
          
          if (elapsed >= duration) {
            // Enough time has passed
            if (leading) {
              emit(value);
            }
            
            if (trailing && !leading) {
              // Schedule trailing emit
              hasTrailingValue = true;
              trailingTimeout = setTimeout(() => {
                if (hasTrailingValue) {
                  emit(lastValue);
                }
                trailingTimeout = null;
              }, duration);
            }
          } else {
            // Still within throttle window
            if (trailing && !trailingTimeout) {
              hasTrailingValue = true;
              trailingTimeout = setTimeout(() => {
                if (hasTrailingValue) {
                  emit(lastValue);
                }
                trailingTimeout = null;
              }, duration - elapsed);
            }
          }
        }
      });
      
      return () => {
        if (trailingTimeout) {
          clearTimeout(trailingTimeout);
        }
        subscription.unsubscribe();
      };
    });
}
```

## 批量同步

Amp 将同步操作分组以提高网络效率：

```typescript
// Collects individual sync requests into efficient batches
export class BatchSyncOrchestrator {
  private requestQueue = new Map<ThreadID, SyncRequest>();
  private batchTimer: NodeJS.Timeout | null = null;
  
  private readonly BATCH_WINDOW = 100;      // 100ms collection window
  private readonly MAX_BATCH_SIZE = 50;     // Maximum items per batch
  
  constructor(private cloudAPI: CloudAPIClient) {}
  
  queueRequest(threadId: ThreadID, request: SyncRequest): void {
    // Merge with any existing request for same thread
    const existing = this.requestQueue.get(threadId);
    if (existing) {
      request = this.mergeRequests(existing, request);
    }
    
    this.requestQueue.set(threadId, request);
    
    // Start batch timer if not already running
    if (!this.batchTimer) {
      this.batchTimer = setTimeout(() => {
        this.flushBatch();
      }, this.BATCH_WINDOW);
    }
  }
  
  private async flushBatch(): Promise<void> {
    this.batchTimer = null;
    
    if (this.requestQueue.size === 0) return;
    
    // Extract batch of requests up to size limit
    const batchEntries = Array.from(this.requestQueue.entries())
      .slice(0, this.MAX_BATCH_SIZE);
    
    // Remove processed items from queue
    batchEntries.forEach(([id]) => this.requestQueue.delete(id));
    
    // Format batch request for API
    const batchRequest: BatchSyncRequest = {
      items: batchEntries.map(([id, request]) => ({
        threadId: id,
        version: request.version,
        changes: request.operations
      }))
    };
    
    try {
      const response = await this.cloudAPI.syncBatch(batchRequest);
      this.handleBatchResponse(response);
    } catch (error) {
      // Retry failed requests with exponential backoff
      batchEntries.forEach(([id, request]) => {
        request.attempts = (request.attempts || 0) + 1;
        if (request.attempts < 3) {
          this.queueRequest(id, request);
        }
      });
    }
    
    // Continue processing if more items queued
    if (this.requestQueue.size > 0) {
      this.batchTimer = setTimeout(() => {
        this.flushBatch();
      }, this.BATCH_WINDOW);
    }
  }
  
  private mergeRequests(
    existing: SyncRequest,
    incoming: SyncRequest
  ): SyncRequest {
    return {
      version: Math.max(existing.version, incoming.version),
      operations: [...existing.operations, ...incoming.operations],
      attempts: existing.attempts || 0
    };
  }
}
```

## 冲突解决

当并发编辑发生时，Amp 智能地解决冲突：

```typescript
export class ConflictResolver {
  async resolveConflict(
    local: Thread,
    remote: Thread,
    base?: Thread
  ): Promise<Thread> {
    // Simple case: one side didn't change
    if (!base) {
      return this.resolveWithoutBase(local, remote);
    }
    
    // Three-way merge
    const merged: Thread = {
      id: local.id,
      created: base.created,
      v: Math.max(local.v, remote.v) + 1,
      messages: await this.mergeMessages(
        base.messages,
        local.messages,
        remote.messages
      ),
      title: this.mergeScalar(base.title, local.title, remote.title),
      env: base.env
    };
    
    return merged;
  }
  
  private async mergeMessages(
    base: Message[],
    local: Message[],
    remote: Message[]
  ): Promise<Message[]> {
    // Find divergence point
    let commonIndex = 0;
    while (
      commonIndex < base.length &&
      commonIndex < local.length &&
      commonIndex < remote.length &&
      this.messagesEqual(
        base[commonIndex],
        local[commonIndex],
        remote[commonIndex]
      )
    ) {
      commonIndex++;
    }
    
    // Common prefix
    const merged = base.slice(0, commonIndex);
    
    // Get new messages from each branch
    const localNew = local.slice(commonIndex);
    const remoteNew = remote.slice(commonIndex);
    
    // Merge by timestamp
    const allNew = [...localNew, ...remoteNew].sort(
      (a, b) => a.timestamp - b.timestamp
    );
    
    // Remove duplicates
    const seen = new Set<string>();
    for (const msg of allNew) {
      const key = this.messageKey(msg);
      if (!seen.has(key)) {
        seen.add(key);
        merged.push(msg);
      }
    }
    
    return merged;
  }
  
  private messageKey(msg: Message): string {
    // Create unique key for deduplication
    return `${msg.role}:${msg.timestamp}:${msg.content.slice(0, 50)}`;
  }
  
  private mergeScalar<T>(base: T, local: T, remote: T): T {
    // If both changed to same value, use it
    if (local === remote) return local;
    
    // If only one changed, use the change
    if (local === base) return remote;
    if (remote === base) return local;
    
    // Both changed differently - prefer local
    return local;
  }
}
```

## 网络韧性

Amp 优雅地处理网络故障：

```typescript
export class ResilientSyncClient {
  private online$ = new BehaviorSubject(navigator.onLine);
  private retryDelays = [1000, 2000, 5000, 10000, 30000]; // Exponential backoff
  
  constructor(private api: ServerAPIClient) {
    // Monitor network status
    window.addEventListener('online', () => this.online$.next(true));
    window.addEventListener('offline', () => this.online$.next(false));
    
    // Test connectivity periodically
    this.startConnectivityCheck();
  }
  
  async syncWithRetry(
    request: SyncRequest,
    attempt = 0
  ): Promise<SyncResponse> {
    try {
      // Wait for network if offline
      await this.waitForNetwork();
      
      // Make request with timeout
      const response = await this.withTimeout(
        this.api.sync(request),
        10000 // 10 second timeout
      );
      
      return response;
      
    } catch (error) {
      if (this.isRetryable(error) && attempt < this.retryDelays.length) {
        const delay = this.retryDelays[attempt];
        
        logger.debug(
          `Sync failed, retrying in ${delay}ms (attempt ${attempt + 1})`
        );
        
        await this.delay(delay);
        return this.syncWithRetry(request, attempt + 1);
      }
      
      throw error;
    }
  }
  
  private async waitForNetwork(): Promise<void> {
    if (this.online$.getValue()) return;
    
    return new Promise(resolve => {
      const sub = this.online$.subscribe(online => {
        if (online) {
          sub.unsubscribe();
          resolve();
        }
      });
    });
  }
  
  private isRetryable(error: unknown): boolean {
    if (error instanceof NetworkError) return true;
    if (error instanceof TimeoutError) return true;
    if (error instanceof HTTPError) {
      return error.status >= 500 || error.status === 429;
    }
    return false;
  }
  
  private async startConnectivityCheck(): Promise<void> {
    while (true) {
      if (!this.online$.getValue()) {
        // Try to ping server
        try {
          await this.api.ping();
          this.online$.next(true);
        } catch {
          // Still offline
        }
      }
      
      await this.delay(30000); // Check every 30 seconds
    }
  }
}
```

## 乐观更新

为了保持响应速度，Amp 采用乐观更新策略：

```typescript
export class OptimisticSyncManager {
  private pendingUpdates = new Map<string, PendingUpdate>();
  
  async applyOptimisticUpdate<T>(
    key: string,
    currentValue: T,
    update: (value: T) => T,
    persist: (value: T) => Promise<void>
  ): Promise<T> {
    // Apply update locally immediately
    const optimisticValue = update(currentValue);
    
    // Track pending update
    const pendingUpdate: PendingUpdate<T> = {
      key,
      originalValue: currentValue,
      optimisticValue,
      promise: null
    };
    
    this.pendingUpdates.set(key, pendingUpdate);
    
    // Persist asynchronously
    pendingUpdate.promise = persist(optimisticValue)
      .then(() => {
        // Success - remove from pending
        this.pendingUpdates.delete(key);
      })
      .catch(error => {
        // Failure - prepare for rollback
        pendingUpdate.error = error;
        throw error;
      });
    
    return optimisticValue;
  }
  
  async rollback(key: string): Promise<void> {
    const pending = this.pendingUpdates.get(key);
    if (!pending) return;
    
    // Wait for pending operation to complete
    try {
      await pending.promise;
    } catch {
      // Expected to fail
    }
    
    // Rollback if it failed
    if (pending.error) {
      // Notify UI to revert to original value
      this.onRollback?.(key, pending.originalValue);
    }
    
    this.pendingUpdates.delete(key);
  }
  
  hasPendingUpdates(): boolean {
    return this.pendingUpdates.size > 0;
  }
  
  async waitForPendingUpdates(): Promise<void> {
    const promises = Array.from(this.pendingUpdates.values())
      .map(update => update.promise);
    
    await Promise.allSettled(promises);
  }
}
```

## 性能监控

Amp 追踪同步性能以优化行为：

```typescript
export class SyncPerformanceMonitor {
  private metrics = new Map<string, MetricHistory>();
  
  recordSyncTime(
    threadId: string,
    duration: number,
    size: number
  ): void {
    const history = this.getHistory('sync-time');
    history.add({
      timestamp: Date.now(),
      value: duration,
      metadata: { threadId, size }
    });
    
    // Analyze for anomalies
    if (duration > this.getP95(history)) {
      logger.warn(`Slow sync detected: ${duration}ms for thread ${threadId}`);
    }
  }
  
  recordBatchSize(size: number): void {
    this.getHistory('batch-size').add({
      timestamp: Date.now(),
      value: size
    });
  }
  
  recordConflictRate(hadConflict: boolean): void {
    this.getHistory('conflicts').add({
      timestamp: Date.now(),
      value: hadConflict ? 1 : 0
    });
  }
  
  getOptimalBatchSize(): number {
    const history = this.getHistory('batch-size');
    const recentSizes = history.getRecent(100);
    
    // Find size that minimizes sync time
    const sizeToTime = new Map<number, number[]>();
    
    for (const entry of this.getHistory('sync-time').getRecent(100)) {
      const size = entry.metadata?.size || 1;
      if (!sizeToTime.has(size)) {
        sizeToTime.set(size, []);
      }
      sizeToTime.get(size)!.push(entry.value);
    }
    
    // Calculate average time per size
    let optimalSize = 50;
    let minAvgTime = Infinity;
    
    for (const [size, times] of sizeToTime) {
      const avgTime = times.reduce((a, b) => a + b) / times.length;
      if (avgTime < minAvgTime) {
        minAvgTime = avgTime;
        optimalSize = size;
      }
    }
    
    return Math.max(10, Math.min(100, optimalSize));
  }
  
  private getP95(history: MetricHistory): number {
    const values = history.getRecent(100)
      .map(entry => entry.value)
      .sort((a, b) => a - b);
    
    const index = Math.floor(values.length * 0.95);
    return values[index] || 0;
  }
}
```

## 同步测试

Amp 包含全面的同步测试工具：

```typescript
export class SyncTestHarness {
  private mockServer = new MockSyncServer();
  private clients: TestClient[] = [];
  
  async testConcurrentEdits(): Promise<void> {
    // Create multiple clients
    const client1 = this.createClient('user1');
    const client2 = this.createClient('user2');
    
    // Both edit same thread
    const threadId = 'test-thread';
    
    await Promise.all([
      client1.addMessage(threadId, 'Hello from user 1'),
      client2.addMessage(threadId, 'Hello from user 2')
    ]);
    
    // Let sync complete
    await this.waitForSync();
    
    // Both clients should have both messages
    const thread1 = await client1.getThread(threadId);
    const thread2 = await client2.getThread(threadId);
    
    assert.equal(thread1.messages.length, 2);
    assert.equal(thread2.messages.length, 2);
    assert.deepEqual(thread1, thread2);
  }
  
  async testNetworkPartition(): Promise<void> {
    const client = this.createClient('user1');
    
    // Make changes while online
    await client.addMessage('thread1', 'Online message');
    
    // Go offline
    this.mockServer.disconnect(client);
    
    // Make offline changes
    await client.addMessage('thread1', 'Offline message 1');
    await client.addMessage('thread1', 'Offline message 2');
    
    // Verify changes are queued
    assert.equal(client.getPendingSyncCount(), 1);
    
    // Reconnect
    this.mockServer.connect(client);
    
    // Wait for sync
    await this.waitForSync();
    
    // Verify all changes synced
    assert.equal(client.getPendingSyncCount(), 0);
    
    const serverThread = this.mockServer.getThread('thread1');
    assert.equal(serverThread.messages.length, 3);
  }
  
  async testSyncPerformance(): Promise<void> {
    const client = this.createClient('user1');
    const messageCount = 1000;
    
    // Add many messages
    const startTime = Date.now();
    
    for (let i = 0; i < messageCount; i++) {
      await client.addMessage('perf-thread', `Message ${i}`);
    }
    
    await this.waitForSync();
    
    const duration = Date.now() - startTime;
    const throughput = messageCount / (duration / 1000);
    
    console.log(`Synced ${messageCount} messages in ${duration}ms`);
    console.log(`Throughput: ${throughput.toFixed(2)} messages/second`);
    
    // Should sync within reasonable time
    assert(throughput > 100, 'Sync throughput too low');
  }
}
```

## 总结

本章展示了实时同步不一定需要 WebSocket：

- **自适应轮询**根据活动模式调整频率
- **Observable 架构**提供响应式的本地状态管理
- **智能批处理**优化网络效率
- **乐观更新**保持响应式的用户界面
- **弹性重试逻辑**优雅地处理网络故障
- **冲突解决策略**确保最终一致性

这种方案在保持实时用户体验的同时，比传统 WebSocket 方案更加可靠和易于调试。核心洞察是：对于 AI 系统，带有智能冲突解决的最终一致性往往优于复杂的实时协议。

下一章将探讨面向分布式执行的工具系统架构，兼顾安全性和大规模性能。
