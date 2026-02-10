# 第四章：大规模会话线程管理

大规模管理人类与 AI 之间的对话带来了独特的挑战。与传统聊天应用中消息只是简单文本不同，AI 编码助手必须处理涉及工具使用、文件修改、子代理生成和协作编辑的复杂交互，同时在分布式系统中保持一致性。

本章探讨数据建模、版本控制和同步模式，这些模式可以从单用户扩展到整个工程组织。

## 线程管理的挑战

AI 编码对话不仅仅是聊天记录。一个线程可能包含：

- 多轮人机交互
- 修改数百个文件的工具调用
- 为并行任务生成的子代理线程
- 成本追踪和使用指标
- 用于回滚的版本历史
- 与摘要线程和父线程的关系

管理这种复杂性需要重新思考传统的数据持久化和同步方案。

## 线程数据模型模式

AI 对话线程需要与传统聊天不同的数据模型。不应使用简单的线性消息数组，而应采用支持复杂工作流的版本化、层级化方案。

**识别模式**：当出现以下情况时，你需要结构化的线程建模：
- 对话涉及工具使用和文件修改
- 用户需要将对话分支为子任务
- 需要准确追踪资源使用和成本
- 协作编辑需要冲突解决

**核心设计原则**：

1. **不可变消息历史** - 消息永远不被修改，只能追加
2. **基于版本的并发控制** - 每次变更递增版本号
3. **层级化组织** - 线程可以为复杂任务生成子线程
4. **工具执行追踪** - 工具调用和结果被显式建模
5. **成本归因** - 按消息追踪资源使用以支持计费

**实现方式**：
```typescript
// Simplified thread structure focusing on key patterns
interface Thread {
  id: string;
  version: number;          // For optimistic concurrency control
  created: timestamp;       // Immutable creation time
  messages: Message[];      // Append-only message history
  
  // Hierarchical relationships
  parentThreadId?: string;  // Links to parent/source thread
  childThreadIds?: string[]; // Sub-threads spawned from this thread
  
  // Execution context
  environment?: Environment;
  metadata?: Metadata;
}

interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  
  // Tool interactions
  toolCalls?: ToolCall[];
  toolResults?: ToolResult[];
  
  // Resource tracking
  resourceUsage?: ResourceUsage;
}
```

**关键优势**：
- **冲突解决**：版本号支持乐观更新
- **审计追踪**：不可变历史提供完整的对话记录
- **可扩展性**：层级结构处理复杂工作流
- **成本追踪**：按消息的使用量支持精确计费

## 版本控制与乐观并发

Amp 使用乐观并发控制来处理并发更新，无需加锁：

```typescript
export class ThreadVersionControl {
  /**
   * Apply a delta to a thread, incrementing its version
   */
  applyDelta(thread: Thread, delta: ThreadDelta): Thread {
    // Create immutable copy
    const updated = structuredClone(thread);
    
    // Increment version for every change
    updated.v++;
    
    // Apply the specific delta
    switch (delta.type) {
      case 'user:message':
        updated.messages.push({
          id: generateMessageId(),
          role: 'user',
          content: delta.message.content,
          timestamp: Date.now(),
          ...delta.message
        });
        break;
        
      case 'assistant:message':
        updated.messages.push(delta.message);
        break;
        
      case 'title':
        updated.title = delta.value;
        break;
        
      case 'thread:truncate':
        updated.messages = updated.messages.slice(0, delta.fromIndex);
        break;
        
      // ... other delta types
    }
    
    return updated;
  }
  
  /**
   * Detect conflicts between versions
   */
  hasConflict(local: Thread, remote: Thread): boolean {
    // Simple version comparison
    return local.v !== remote.v;
  }
  
  /**
   * Merge concurrent changes
   */
  merge(base: Thread, local: Thread, remote: Thread): Thread {
    // If versions match, no conflict
    if (local.v === remote.v) {
      return local;
    }
    
    // If only one side changed, take that version
    if (local.v === base.v) {
      return remote;
    }
    if (remote.v === base.v) {
      return local;
    }
    
    // Both changed - need three-way merge
    return this.threeWayMerge(base, local, remote);
  }
  
  private threeWayMerge(
    base: Thread, 
    local: Thread, 
    remote: Thread
  ): Thread {
    const merged = structuredClone(remote);
    
    // Take the higher version
    merged.v = Math.max(local.v, remote.v) + 1;
    
    // Merge messages by timestamp
    const localNewMessages = local.messages.slice(base.messages.length);
    const remoteNewMessages = remote.messages.slice(base.messages.length);
    
    merged.messages = [
      ...base.messages,
      ...this.mergeMessagesByTimestamp(localNewMessages, remoteNewMessages)
    ];
    
    // Prefer local title if changed
    if (local.title !== base.title) {
      merged.title = local.title;
    }
    
    return merged;
  }
}
```

## 独占访问模式

为防止并发写入导致的数据损坏，Amp 实现了独占写入者模式：

```typescript
// Ensures single-writer semantics for thread modifications
export class ThreadService {
  private activeWriters = new Map<ThreadID, ThreadWriter>();
  
  async acquireWriter(id: ThreadID): Promise<ThreadWriter> {
    // Prevent multiple writers for the same thread
    if (this.activeWriters.has(id)) {
      throw new Error(`Thread ${id} is already being modified`);
    }
    
    // Load current thread state
    const thread = await this.storage.get(id) || this.createThread(id);
    const writer = new ThreadWriter(thread, this.storage);
    
    // Register active writer
    this.activeWriters.set(id, writer);
    
    // Set up auto-persistence with debouncing
    writer.enableAutosave({
      debounceMs: 1000,        // Wait for activity to settle
      onSave: (thread) => this.onThreadSaved(thread),
      onError: (error) => this.onSaveError(error)
    });
    
    return {
      // Read current state reactively
      observe: () => writer.asObservable(),
      
      // Apply atomic modifications
      modify: async (modifier: ThreadModifier) => {
        const current = writer.getCurrentState();
        const updated = modifier(current);
        
        // Enforce version increment for optimistic concurrency
        if (updated.v <= current.v) {
          throw new Error('Version must increment on modification');
        }
        
        writer.updateState(updated);
        return updated;
      },
      
      // Release writer and ensure final save
      dispose: async () => {
        await writer.finalSave();
        this.activeWriters.delete(id);
      }
    };
  }
}
```

## 存储架构

Amp 使用多层存储策略，在性能与持久性之间取得平衡：

```typescript
// Tiered storage provides performance through caching hierarchy
export class TieredThreadStorage {
  constructor(
    private memoryCache: MemoryStorage,
    private localStorage: PersistentStorage,
    private cloudStorage: RemoteStorage
  ) {}
  
  async get(id: ThreadID): Promise<Thread | null> {
    // L1: In-memory cache for active threads
    const cached = this.memoryCache.get(id);
    if (cached) {
      return cached;
    }
    
    // L2: Local persistence for offline access
    const local = await this.localStorage.get(id);
    if (local) {
      this.memoryCache.set(id, local, { ttl: 300000 });
      return local;
    }
    
    // L3: Remote storage for sync and backup
    const remote = await this.cloudStorage.get(id);
    if (remote) {
      // Populate lower tiers
      await this.localStorage.set(id, remote);
      this.memoryCache.set(id, remote, { ttl: 300000 });
      return remote;
    }
    
    return null;
  }
  
  async set(id: ThreadID, thread: Thread): Promise<void> {
    // Write-through strategy: update all tiers
    await Promise.all([
      this.memoryCache.set(id, thread),
      this.localStorage.set(id, thread),
      this.queueCloudSync(id, thread)  // Async to avoid blocking
    ]);
  }
  
  private async queueCloudSync(id: ThreadID, thread: Thread): Promise<void> {
    // Queue for eventual consistency with remote storage
    this.syncQueue.add({ id, thread, priority: this.getSyncPriority(thread) });
  }
}
```

### 持久化策略模式

不同类型的线程需要不同的持久化策略，取决于其生命周期和重要性：

```typescript
// Strategy pattern for different thread types
export class ThreadPersistenceStrategy {
  getStrategy(thread: Thread): PersistenceConfig {
    // Ephemeral sub-agent threads (short-lived, disposable)
    if (thread.mainThreadID) {
      return {
        memory: { ttl: 60000 },      // Keep in memory briefly
        local: { enabled: false },    // Skip local persistence
        cloud: { enabled: false }     // No cloud sync needed
      };
    }
    
    // Summary threads (archival, long-term reference)
    if (thread.originThreadID) {
      return {
        memory: { ttl: 3600000 },    // Cache for an hour
        local: { enabled: true },     // Always persist locally
        cloud: { 
          enabled: true,
          priority: 'low',            // Eventual consistency OK
          compression: true           // Optimize for storage
        }
      };
    }
    
    // Main threads (active, high-value)
    return {
      memory: { ttl: 300000 },       // 5-minute cache
      local: { enabled: true },       // Always persist
      cloud: { 
        enabled: true,
        priority: 'high',             // Immediate sync
        versioning: true              // Keep version history
      }
    };
  }
}
```

## 同步策略

线程同步使用基于队列的方式，配合智能批处理和重试逻辑：

```typescript
// Manages sync operations with configurable batching and retry policies
export class ThreadSyncService {
  private syncQueue = new Map<ThreadID, SyncRequest>();
  private processingBatch = false;
  private failureBackoff = new Map<ThreadID, number>();
  
  // Configurable sync parameters
  private readonly BATCH_SIZE = 50;
  private readonly SYNC_INTERVAL = 5000;
  private readonly RETRY_BACKOFF = 60000;
  
  constructor(
    private cloudAPI: CloudSyncAPI,
    private localStorage: LocalStorage
  ) {
    this.startSyncLoop();
  }
  
  private async startSyncLoop(): Promise<void> {
    while (true) {
      await this.processPendingSync();
      await this.sleep(this.SYNC_INTERVAL);
    }
  }
  
  async queueSync(id: ThreadID, thread: Thread): Promise<void> {
    // Determine if sync is needed based on version comparison
    if (!this.shouldSync(id)) {
      return;
    }
    
    // Check if local version is ahead of remote
    const remoteVersion = await this.getRemoteVersion(id);
    if (remoteVersion && remoteVersion >= thread.v) {
      return; // Already synchronized
    }
    
    // Add to sync queue with metadata
    this.syncQueue.set(id, {
      id,
      thread,
      remoteVersion: remoteVersion || 0,
      queuedAt: Date.now(),
      attempts: 0
    });
  }
  
  private shouldSync(id: ThreadID): boolean {
    // Check backoff
    const lastFailed = this.lastFailedSync.get(id);
    if (lastFailed) {
      const elapsed = Date.now() - lastFailed;
      if (elapsed < this.RETRY_BACKOFF) {
        return false;
      }
    }
    
    return true;
  }
  
  private async processPendingSync(): Promise<void> {
    if (this.processingBatch || this.syncQueue.size === 0) {
      return;
    }
    
    this.processingBatch = true;
    
    try {
      // Select threads ready for sync (respecting backoff)
      const readyItems = Array.from(this.syncQueue.values())
        .filter(item => this.isReadyForSync(item.id))
        .sort((a, b) => a.queuedAt - b.queuedAt)
        .slice(0, this.BATCH_SIZE);
      
      if (readyItems.length === 0) {
        return;
      }
      
      // Execute sync operations with controlled concurrency
      const syncResults = await Promise.allSettled(
        readyItems.map(item => this.performSync(item))
      );
      
      // Handle results and update queue state
      syncResults.forEach((result, index) => {
        const item = readyItems[index];
        
        if (result.status === 'fulfilled') {
          this.syncQueue.delete(item.id);
          this.failureBackoff.delete(item.id);
        } else {
          this.handleSyncFailure(item, result.reason);
        }
      });
      
    } finally {
      this.processingBatch = false;
    }
  }
  
  private async performSync(item: SyncRequest): Promise<void> {
    // Attempt synchronization with conflict detection
    const response = await this.cloudAPI.syncThread({
      id: item.thread.id,
      localThread: item.thread,
      baseVersion: item.remoteVersion
    });
    
    if (response.hasConflict) {
      // Resolve conflicts using three-way merge
      await this.resolveConflict(item.thread, response.remoteThread);
    }
  }
  
  private async resolveConflict(
    local: Thread,
    remote: Thread
  ): Promise<void> {
    // Find common ancestor for three-way merge
    const base = await this.findCommonAncestor(local, remote);
    
    // Use merge algorithm to combine changes
    const merged = this.mergeStrategy.merge(base, local, remote);
    
    // Persist merged result
    await this.localStorage.set(local.id, merged);
    
    // Update version tracking for future conflicts
    await this.updateVersionHistory(local.id, merged);
  }
}
```

## 线程关系模式

Amp 支持层级化的线程关系，用于复杂工作流：

```typescript
// Manages parent-child relationships between threads
export class ThreadRelationshipManager {
  
  // Create summary threads that reference original conversations
  async createSummaryThread(
    sourceThreadId: ThreadID,
    summaryContent: string
  ): Promise<Thread> {
    const sourceThread = await this.threadService.getThread(sourceThreadId);
    if (!sourceThread) {
      throw new Error(`Source thread ${sourceThreadId} not found`);
    }
    
    // Build summary thread with proper linking
    const summaryThread: Thread = {
      id: this.generateThreadId(),
      created: Date.now(),
      v: 1,
      title: `Summary: ${sourceThread.title || 'Conversation'}`,
      messages: [{
        id: this.generateMessageId(),
        role: 'assistant',
        content: summaryContent,
        timestamp: Date.now()
      }],
      originThreadID: sourceThreadId  // Link back to source
    };
    
    // Update source thread to reference summary
    await this.threadService.modifyThread(sourceThreadId, thread => ({
      ...thread,
      v: thread.v + 1,
      summaryThreads: [...(thread.summaryThreads || []), summaryThread.id]
    }));
    
    // Persist the new summary thread
    await this.threadService.persistThread(summaryThread);
    
    return summaryThread;
  }
  
  // Spawn sub-agent threads for delegated tasks
  async spawnSubAgentThread(
    parentThreadId: ThreadID,
    taskDescription: string
  ): Promise<Thread> {
    const parentThread = await this.threadService.getThread(parentThreadId);
    
    // Create sub-thread with parent reference
    const subThread: Thread = {
      id: this.generateThreadId(),
      created: Date.now(),
      v: 1,
      title: `Task: ${taskDescription}`,
      messages: [{
        id: this.generateMessageId(),
        role: 'user',
        content: taskDescription,
        timestamp: Date.now()
      }],
      mainThreadID: parentThreadId,    // Link to parent
      env: parentThread?.env           // Inherit execution context
    };
    
    await this.threadService.persistThread(subThread);
    
    return subThread;
  }
  
  // Retrieve complete thread relationship graph
  async getRelatedThreads(
    threadId: ThreadID
  ): Promise<ThreadRelationships> {
    const thread = await this.threadService.getThread(threadId);
    if (!thread) {
      throw new Error(`Thread ${threadId} not found`);
    }
    
    const relationships: ThreadRelationships = {
      thread,
      parent: null,
      summaries: [],
      children: []
    };
    
    // Load parent thread if this is a sub-thread
    if (thread.mainThreadID) {
      relationships.parent = await this.threadService.getThread(
        thread.mainThreadID
      );
    }
    
    // Load linked summary threads
    if (thread.summaryThreads) {
      relationships.summaries = await Promise.all(
        thread.summaryThreads.map(id => 
          this.threadService.getThread(id)
        )
      );
    }
    
    // Find child threads spawned from this thread
    const childThreads = await this.threadService.findChildThreads(threadId);
    relationships.children = childThreads;
    
    return relationships;
  }
}
```

## 文件变更追踪

线程维护所有文件修改的审计追踪，用于回滚和问责：

```typescript
// Represents a single file modification event
export interface FileChangeRecord {
  path: string;
  type: 'create' | 'modify' | 'delete';
  beforeContent?: string;
  afterContent?: string;
  timestamp: number;
  operationId: string;  // Links to specific tool execution
}

// Tracks file changes across thread execution
export class ThreadFileTracker {
  private changeLog = new Map<ThreadID, Map<string, FileChangeRecord[]>>();
  
  async recordFileChange(
    threadId: ThreadID,
    operationId: string,
    change: FileModification
  ): Promise<void> {
    // Initialize change tracking for thread if needed
    if (!this.changeLog.has(threadId)) {
      this.changeLog.set(threadId, new Map());
    }
    
    const threadChanges = this.changeLog.get(threadId)!;
    const fileHistory = threadChanges.get(change.path) || [];
    
    // Capture file state before change
    const beforeState = await this.captureFileState(change.path);
    
    // Record the modification
    fileHistory.push({
      path: change.path,
      type: change.type,
      beforeContent: beforeState,
      afterContent: change.type !== 'delete' ? change.newContent : undefined,
      timestamp: Date.now(),
      operationId
    });
    
    threadChanges.set(change.path, fileHistory);
    
    // Persist change log for crash recovery
    await this.persistChangeLog(threadId);
  }
  
  async rollbackOperation(
    threadId: ThreadID,
    operationId: string
  ): Promise<void> {
    const threadChanges = this.changeLog.get(threadId);
    if (!threadChanges) return;
    
    // Collect all changes from this operation
    const changesToRevert: FileChangeRecord[] = [];
    
    for (const [path, history] of threadChanges) {
      const operationChanges = history.filter(
        record => record.operationId === operationId
      );
      changesToRevert.push(...operationChanges);
    }
    
    // Sort by timestamp (newest first) for proper rollback order
    changesToRevert.sort((a, b) => b.timestamp - a.timestamp);
    
    // Apply rollback in reverse chronological order
    for (const change of changesToRevert) {
      await this.revertFileChange(change);
    }
  }
  
  private async revertFileChange(change: FileChangeRecord): Promise<void> {
    try {
      switch (change.type) {
        case 'create':
          // Remove file that was created
          await this.fileSystem.deleteFile(change.path);
          break;
          
        case 'modify':
          // Restore previous content
          if (change.beforeContent !== undefined) {
            await this.fileSystem.writeFile(change.path, change.beforeContent);
          }
          break;
          
        case 'delete':
          // Recreate deleted file
          if (change.beforeContent !== undefined) {
            await this.fileSystem.writeFile(change.path, change.beforeContent);
          }
          break;
      }
    } catch (error) {
      // Log rollback failures but continue with other changes
      this.logger.error(`Failed to rollback ${change.path}:`, error);
    }
  }
}
```

## 线程生命周期管理

线程遵循从创建到归档的受管生命周期：

```typescript
// Manages thread lifecycle stages and transitions
export class ThreadLifecycleManager {
  
  // Initialize new thread with proper setup
  async createThread(options: ThreadCreationOptions = {}): Promise<Thread> {
    const thread: Thread = {
      id: options.id || this.generateThreadId(),
      created: Date.now(),
      v: 1,
      title: options.title,
      messages: [],
      env: options.captureEnvironment ? {
        initial: await this.captureCurrentEnvironment()
      } : undefined
    };
    
    // Persist immediately for durability
    await this.storage.persistThread(thread);
    
    // Queue for cloud synchronization
    await this.syncService.scheduleSync(thread.id, thread);
    
    // Broadcast creation event
    this.eventBus.publish('thread:created', { thread });
    
    return thread;
  }
  
  // Archive inactive threads to cold storage
  async archiveInactiveThreads(): Promise<void> {
    const archiveThreshold = Date.now() - (30 * 24 * 60 * 60 * 1000); // 30 days
    
    const activeThreads = await this.storage.getAllThreads();
    
    for (const thread of activeThreads) {
      // Determine last activity time
      const lastMessage = thread.messages[thread.messages.length - 1];
      const lastActivity = lastMessage?.timestamp || thread.created;
      
      if (lastActivity < archiveThreshold) {
        await this.moveToArchive(thread);
      }
    }
  }
  
  private async moveToArchive(thread: Thread): Promise<void> {
    // Transfer to cold storage
    await this.coldStorage.archive(thread.id, thread);
    
    // Remove from active storage, keep metadata for indexing
    await this.storage.deleteThread(thread.id);
    await this.storage.storeMetadata(`${thread.id}:meta`, {
      id: thread.id,
      title: thread.title,
      created: thread.created,
      archived: Date.now(),
      messageCount: thread.messages.length
    });
    
    this.logger.info(`Archived thread ${thread.id}`);
  }
  
  // Restore archived thread to active storage
  async restoreThread(id: ThreadID): Promise<Thread> {
    const thread = await this.coldStorage.retrieve(id);
    if (!thread) {
      throw new Error(`Archived thread ${id} not found`);
    }
    
    // Move back to active storage
    await this.storage.persistThread(thread);
    
    // Clean up archive metadata
    await this.storage.deleteMetadata(`${id}:meta`);
    
    return thread;
  }
}
```

## 性能优化策略

Amp 采用了多种技术来保持线程数据增长时的性能表现：

### 1. 消息分页

大型对话增量加载以避免内存问题：

```typescript
export class PaginatedThreadLoader {
  async loadThread(
    id: ThreadID,
    options: { limit?: number; offset?: number } = {}
  ): Promise<PaginatedThread> {
    const limit = options.limit || 50;
    const offset = options.offset || 0;
    
    // Load thread metadata
    const metadata = await this.storage.getMetadata(id);
    
    // Load only requested messages
    const messages = await this.storage.getMessages(id, {
      limit,
      offset,
      // Load newest messages first
      order: 'desc'
    });
    
    return {
      id,
      created: metadata.created,
      v: metadata.v,
      title: metadata.title,
      messages: messages.reverse(), // Return in chronological order
      totalMessages: metadata.messageCount,
      hasMore: offset + limit < metadata.messageCount
    };
  }
}
```

### 2. 增量压缩

只通过网络传输变更部分：

```typescript
export class ThreadDeltaCompressor {
  compress(
    oldThread: Thread,
    newThread: Thread
  ): CompressedDelta {
    const delta: CompressedDelta = {
      id: newThread.id,
      fromVersion: oldThread.v,
      toVersion: newThread.v,
      changes: []
    };
    
    // Compare messages
    const messagesDiff = this.diffMessages(
      oldThread.messages,
      newThread.messages
    );
    
    if (messagesDiff.added.length > 0) {
      delta.changes.push({
        type: 'messages:add',
        messages: messagesDiff.added
      });
    }
    
    // Compare metadata
    if (oldThread.title !== newThread.title) {
      delta.changes.push({
        type: 'metadata:update',
        title: newThread.title
      });
    }
    
    return delta;
  }
  
  decompress(
    thread: Thread,
    delta: CompressedDelta
  ): Thread {
    let result = structuredClone(thread);
    
    for (const change of delta.changes) {
      switch (change.type) {
        case 'messages:add':
          result.messages.push(...change.messages);
          break;
          
        case 'metadata:update':
          if (change.title !== undefined) {
            result.title = change.title;
          }
          break;
      }
    }
    
    result.v = delta.toVersion;
    return result;
  }
}
```

### 3. 批量操作

多个线程操作被批量处理：

```typescript
export class BatchThreadOperations {
  private pendingReads = new Map<ThreadID, Promise<Thread>>();
  private writeQueue: WriteOperation[] = [];
  private flushTimer?: NodeJS.Timeout;
  
  async batchRead(ids: ThreadID[]): Promise<Map<ThreadID, Thread>> {
    const results = new Map<ThreadID, Thread>();
    const toFetch: ThreadID[] = [];
    
    // Check for in-flight reads
    for (const id of ids) {
      const pending = this.pendingReads.get(id);
      if (pending) {
        results.set(id, await pending);
      } else {
        toFetch.push(id);
      }
    }
    
    if (toFetch.length > 0) {
      // Batch fetch
      const promise = this.storage.batchGet(toFetch);
      
      // Track in-flight
      for (const id of toFetch) {
        this.pendingReads.set(id, promise.then(
          batch => batch.get(id)!
        ));
      }
      
      const batch = await promise;
      
      // Clear tracking
      for (const id of toFetch) {
        this.pendingReads.delete(id);
        const thread = batch.get(id);
        if (thread) {
          results.set(id, thread);
        }
      }
    }
    
    return results;
  }
  
  async batchWrite(operation: WriteOperation): Promise<void> {
    this.writeQueue.push(operation);
    
    // Schedule flush
    if (!this.flushTimer) {
      this.flushTimer = setTimeout(() => {
        this.flushWrites();
      }, 100); // 100ms batching window
    }
  }
  
  private async flushWrites(): Promise<void> {
    const operations = this.writeQueue.splice(0);
    this.flushTimer = undefined;
    
    if (operations.length === 0) return;
    
    // Group by operation type
    const creates = operations.filter(op => op.type === 'create');
    const updates = operations.filter(op => op.type === 'update');
    const deletes = operations.filter(op => op.type === 'delete');
    
    // Execute in parallel
    await Promise.all([
      creates.length > 0 && this.storage.batchCreate(creates),
      updates.length > 0 && this.storage.batchUpdate(updates),
      deletes.length > 0 && this.storage.batchDelete(deletes)
    ]);
  }
}
```

## 错误恢复与韧性

线程管理必须处理各种故障场景：

```typescript
export class ResilientThreadService {
  async withRetry<T>(
    operation: () => Promise<T>,
    options: RetryOptions = {}
  ): Promise<T> {
    const maxAttempts = options.maxAttempts || 3;
    const backoff = options.backoff || 1000;
    
    let lastError: Error;
    
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error as Error;
        
        if (!this.isRetryable(error)) {
          throw error;
        }
        
        if (attempt < maxAttempts) {
          const delay = backoff * Math.pow(2, attempt - 1);
          logger.warn(
            `Operation failed (attempt ${attempt}/${maxAttempts}), ` +
            `retrying in ${delay}ms:`,
            error
          );
          await sleep(delay);
        }
      }
    }
    
    throw lastError!;
  }
  
  private isRetryable(error: unknown): boolean {
    if (error instanceof NetworkError) return true;
    if (error instanceof TimeoutError) return true;
    if (error instanceof ServerError && error.status >= 500) return true;
    return false;
  }
  
  async recoverFromCrash(): Promise<void> {
    logger.info('Recovering thread state after crash');
    
    // Find threads that were being modified
    const dirtyThreads = await this.storage.findDirtyThreads();
    
    for (const threadId of dirtyThreads) {
      try {
        // Restore from write-ahead log
        const wal = await this.storage.getWriteAheadLog(threadId);
        if (wal.length > 0) {
          await this.replayWriteAheadLog(threadId, wal);
        }
        
        // Mark as clean
        await this.storage.markClean(threadId);
      } catch (error) {
        logger.error(`Failed to recover thread ${threadId}:`, error);
      }
    }
  }
}
```

## 总结

本章探讨了构建可扩展线程管理系统的架构模式：

- **版本化数据模型**支持无锁的乐观并发
- **独占写入者模式**在保持性能的同时防止数据损坏
- **多层存储策略**在速度、持久性和成本之间取得平衡
- **智能同步**通过合并策略解决冲突
- **层级关系**支持复杂的多代理工作流
- **审计追踪系统**支持回滚和问责
- **性能优化**在数据增长时保持响应速度

这些模式提供了一个可以从单用户扩展到大型团队的基础，同时保持数据完整性和系统性能。下一章将探讨实时同步策略，这些策略在不依赖传统 WebSocket 复杂性的情况下保持分布式客户端的协调一致。
