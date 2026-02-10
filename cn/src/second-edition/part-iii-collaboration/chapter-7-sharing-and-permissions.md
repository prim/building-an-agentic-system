# 第七章：共享与权限模式

在构建协作式 AI 编码助手时，最棘手的方面之一不是 AI 本身，而是如何让人们共享工作成果而不会意外暴露不该暴露的内容。本章探讨实现共享和权限的模式，在安全性、易用性和实现复杂度之间取得平衡。

## 三层共享模型

协作式 AI 助手的常见模式是三层共享模型。这种方式通过两个布尔标志 `private` 和 `public` 创建三个不同的状态，在简单性和灵活性之间取得平衡：

```typescript
interface ShareableResource {
    private: boolean
    public: boolean
}

// Three sharing states:
// 1. Private (private: true, public: false) - Only creator access
// 2. Team (private: false, public: false) - Shared with team members  
// 3. Public (private: false, public: true) - Anyone with URL can access

async updateSharingState(
    resourceID: string,
    meta: Pick<ShareableResource, 'private' | 'public'>
): Promise<void> {
    // Validate state transition
    if (meta.private && meta.public) {
        throw new Error('Invalid state: cannot be both private and public')
    }
    
    // Optimistic update for UI responsiveness
    this.updateLocalState(resourceID, meta)
    
    try {
        // Sync with server
        await this.syncToServer(resourceID, meta)
    } catch (error) {
        // Rollback on failure
        this.revertLocalState(resourceID)
        throw error
    }
}
```

这个设计使用两个布尔值而非枚举，原因如下：
- 状态转换更加明确
- 防止通过单字段更新意外改变可见性
- 创建了一个可以检测和拒绝的无效第四状态
- 自然地映射到用户界面控件

## 权限继承模式

在为层级化资源设计权限系统时，你面临一个基本选择：继承还是独立。复杂的权限继承可能在父级权限变更时导致意外暴露。更简单的方法是让每个资源独立管理。

```typescript
interface HierarchicalResource {
    id: string
    parentID?: string
    childIDs: string[]
    permissions: ResourcePermissions
}

// Independent permissions - each resource manages its own access
class IndependentPermissionModel {
    async updatePermissions(
        resourceID: string, 
        newPermissions: ResourcePermissions
    ): Promise<void> {
        // Only affects this specific resource
        await this.permissionStore.update(resourceID, newPermissions)
        
        // No cascading to children or parents
        // Users must explicitly manage each resource
    }
    
    async getEffectivePermissions(
        resourceID: string, 
        userID: string
    ): Promise<EffectivePermissions> {
        // Only check the resource itself
        const resource = await this.getResource(resourceID)
        return this.evaluatePermissions(resource.permissions, userID)
    }
}

// When syncing resources, treat each independently
for (const resource of resourcesToSync) {
    if (processed.has(resource.id)) {
        continue
    }
    processed.add(resource.id)
    
    // Each resource carries its own permission metadata
    syncRequest.resources.push({
        id: resource.id,
        permissions: resource.permissions,
        // No inheritance from parents
    })
}
```

这种方法保持权限模型简单且可预测。用户能准确理解更改共享设置时会发生什么，无需担心级联效应。

## 基于 URL 的共享实现

基于 URL 的共享创建了一个能力系统，其中知道 URL 就获得了访问权限。这种模式在现代应用中被广泛使用。

```typescript
// Generate unguessable resource identifiers
type ResourceID = `R-${string}`

function generateResourceID(): ResourceID {
    return `R-${crypto.randomUUID()}`
}

function buildResourceURL(baseURL: URL, resourceID: ResourceID): URL {
    return new URL(`/shared/${resourceID}`, baseURL)
}

// Security considerations for URL-based sharing
class URLSharingService {
    async createShareableLink(
        resourceID: ResourceID,
        permissions: SharePermissions
    ): Promise<ShareableLink> {
        // Generate unguessable token
        const shareToken = crypto.randomUUID()
        
        // Store mapping with expiration
        await this.shareStore.create({
            token: shareToken,
            resourceID,
            permissions,
            expiresAt: new Date(Date.now() + permissions.validForMs),
            createdBy: permissions.creatorID
        })
        
        return {
            url: new URL(`/share/${shareToken}`, this.baseURL),
            expiresAt: new Date(Date.now() + permissions.validForMs),
            permissions
        }
    }
    
    async validateShareAccess(
        shareToken: string,
        requesterID: string
    ): Promise<AccessResult> {
        const share = await this.shareStore.get(shareToken)
        
        if (!share || share.expiresAt < new Date()) {
            return { allowed: false, reason: 'Link expired or invalid' }
        }
        
        // Check if additional authentication is required
        if (share.permissions.requiresAuth && !requesterID) {
            return { allowed: false, reason: 'Authentication required' }
        }
        
        return { 
            allowed: true, 
            resourceID: share.resourceID,
            effectivePermissions: share.permissions
        }
    }
}

// Defense in depth: URL capability + authentication
class SecureAPIClient {
    async makeRequest(endpoint: string, options: RequestOptions): Promise<Response> {
        return fetch(new URL(endpoint, this.baseURL), {
            ...options,
            headers: {
                ...options.headers,
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.apiKey}`,
                'X-Client-ID': this.clientID,
            },
        })
    }
}
```

这种双重方式提供了纵深防御：URL 授予能力，而身份认证验证身份。即使有人发现了共享 URL，他们仍然需要有效的凭据才能执行敏感操作。

## 安全考量

实现安全共享需要多种防御性模式：

### 带回滚的乐观更新

为了响应式 UI，乐观更新在后台同步的同时立即显示变更：

```typescript
class SecurePermissionService {
    async updatePermissions(
        resourceID: string, 
        newPermissions: ResourcePermissions
    ): Promise<void> {
        // Capture current state for rollback
        const previousState = this.localState.get(resourceID)
        
        try {
            // Optimistic update for immediate UI feedback
            this.localState.set(resourceID, {
                status: 'syncing',
                permissions: newPermissions,
                lastUpdated: Date.now()
            })
            this.notifyStateChange(resourceID)
            
            // Sync with server
            await this.syncToServer(resourceID, newPermissions)
            
            // Mark as synced
            this.localState.set(resourceID, {
                status: 'synced',
                permissions: newPermissions,
                lastUpdated: Date.now()
            })
            
        } catch (error) {
            // Rollback on failure
            if (previousState) {
                this.localState.set(resourceID, previousState)
            } else {
                this.localState.delete(resourceID)
            }
            this.notifyStateChange(resourceID)
            throw error
        }
    }
}
```

### 智能重试逻辑

网络故障不应导致永久的不一致：

```typescript
class ResilientSyncService {
    private readonly RETRY_BACKOFF_MS = 60000 // 1 minute
    private failedAttempts = new Map<string, number>()
    
    shouldRetrySync(resourceID: string): boolean {
        const lastFailed = this.failedAttempts.get(resourceID)
        if (!lastFailed) {
            return true // Never failed, okay to try
        }
        
        const elapsed = Date.now() - lastFailed
        return elapsed >= this.RETRY_BACKOFF_MS
    }
    
    async attemptSync(resourceID: string): Promise<void> {
        try {
            await this.performSync(resourceID)
            // Clear failure record on success
            this.failedAttempts.delete(resourceID)
        } catch (error) {
            // Record failure time
            this.failedAttempts.set(resourceID, Date.now())
            throw error
        }
    }
}
```

### 支持访问模式

独立的支持访问机制维持清晰的边界：

```typescript
class SupportAccessService {
    async grantSupportAccess(
        resourceID: string,
        userID: string,
        reason: string
    ): Promise<SupportAccessGrant> {
        // Validate user can grant support access
        const resource = await this.getResource(resourceID)
        if (!this.canGrantSupportAccess(resource, userID)) {
            throw new Error('Insufficient permissions to grant support access')
        }
        
        // Create time-limited support access
        const grant: SupportAccessGrant = {
            id: crypto.randomUUID(),
            resourceID,
            grantedBy: userID,
            reason,
            expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
            permissions: { read: true, debug: true }
        }
        
        await this.supportAccessStore.create(grant)
        
        // Audit log
        await this.auditLogger.log({
            action: 'support_access_granted',
            resourceID,
            grantedBy: userID,
            grantID: grant.id,
            reason
        })
        
        return grant
    }
}
```

这些模式在保持可用性和支持合法运维需求的同时提供了多层保护。

## 真实世界的实现细节

生产系统需要针对常见挑战的务实解决方案：

### API 版本控制与降级

在演进 API 时，优雅降级确保系统可靠性：

```typescript
class VersionedAPIClient {
    private useNewAPI: boolean = true
    
    async updateResource(
        resourceID: string, 
        updates: ResourceUpdates
    ): Promise<void> {
        let newAPISucceeded = false
        
        if (this.useNewAPI) {
            try {
                const response = await this.callNewAPI(resourceID, updates)
                if (response.ok) {
                    newAPISucceeded = true
                }
            } catch (error) {
                // Log but don't fail - will try fallback
                this.logAPIError('new_api_failed', error)
            }
        }
        
        if (!newAPISucceeded) {
            // Fallback to older API format
            await this.callLegacyAPI(resourceID, this.transformToLegacy(updates))
        }
    }
    
    private transformToLegacy(updates: ResourceUpdates): LegacyUpdates {
        // Transform new format to legacy API expectations
        return {
            private: updates.visibility === 'private',
            public: updates.visibility === 'public',
            // Map other fields...
        }
    }
}
```

### 避免空状态同步

不要同步没有价值的资源：

```typescript
class IntelligentSyncService {
    shouldSyncResource(resource: SyncableResource): boolean {
        // Skip empty or placeholder resources
        if (this.isEmpty(resource)) {
            return false
        }
        
        // Skip resources that haven't been meaningfully used
        if (this.isUnused(resource)) {
            return false
        }
        
        // Skip resources with only metadata
        if (this.hasOnlyMetadata(resource)) {
            return false
        }
        
        return true
    }
    
    private isEmpty(resource: SyncableResource): boolean {
        return (
            !resource.content?.length &&
            !resource.interactions?.length &&
            !resource.modifications?.length
        )
    }
    
    private isUnused(resource: SyncableResource): boolean {
        const timeSinceCreation = Date.now() - resource.createdAt
        const hasMinimalUsage = resource.interactionCount < 3
        
        // Created recently but barely used
        return timeSinceCreation < 5 * 60 * 1000 && hasMinimalUsage
    }
}
```

### 配置驱动的行为

使用功能开关进行渐进式发布和紧急回滚：

```typescript
interface FeatureFlags {
    enableNewPermissionSystem: boolean
    strictPermissionValidation: boolean
    allowCrossTeamSharing: boolean
    enableAuditLogging: boolean
}

class ConfigurablePermissionService {
    constructor(
        private config: FeatureFlags,
        private legacyService: LegacyPermissionService,
        private newService: NewPermissionService
    ) {}
    
    async checkPermissions(
        resourceID: string, 
        userID: string
    ): Promise<PermissionResult> {
        if (this.config.enableNewPermissionSystem) {
            const result = await this.newService.check(resourceID, userID)
            
            if (this.config.strictPermissionValidation) {
                // Also validate with legacy system for comparison
                const legacyResult = await this.legacyService.check(resourceID, userID)
                this.compareResults(result, legacyResult, resourceID, userID)
            }
            
            return result
        } else {
            return this.legacyService.check(resourceID, userID)
        }
    }
}
```

这些模式承认生产系统是渐进演进的，需要安全过渡的机制。

## 性能优化

权限系统在没有精心优化的情况下可能成为性能瓶颈：

### 批处理与防抖

将快速变更分组以减少服务器负载：

```typescript
class OptimizedSyncService {
    private pendingUpdates = new BehaviorSubject<Set<string>>(new Set())
    
    constructor() {
        // Batch updates with debouncing
        this.pendingUpdates.pipe(
            filter(updates => updates.size > 0),
            debounceTime(3000), // Wait 3 seconds for additional changes
            map(updates => Array.from(updates))
        ).subscribe(resourceIDs => {
            this.processBatch(resourceIDs).catch(error => {
                this.logger.error('Batch sync failed:', error)
            })
        })
    }
    
    queueUpdate(resourceID: string): void {
        const current = this.pendingUpdates.value
        current.add(resourceID)
        this.pendingUpdates.next(current)
    }
    
    private async processBatch(resourceIDs: string[]): Promise<void> {
        // Batch API call instead of individual requests
        const updates = await this.gatherUpdates(resourceIDs)
        await this.apiClient.batchUpdate(updates)
        
        // Clear processed items
        const remaining = this.pendingUpdates.value
        resourceIDs.forEach(id => remaining.delete(id))
        this.pendingUpdates.next(remaining)
    }
}
```

### 本地缓存策略

本地缓存权限状态以实现即时 UI 响应：

```typescript
class CachedPermissionService {
    private permissionCache = new Map<string, CachedPermission>()
    private readonly CACHE_TTL = 5 * 60 * 1000 // 5 minutes
    
    async checkPermission(
        resourceID: string, 
        userID: string
    ): Promise<PermissionResult> {
        const cacheKey = `${resourceID}:${userID}`
        const cached = this.permissionCache.get(cacheKey)
        
        // Return cached result if fresh
        if (cached && this.isFresh(cached)) {
            return cached.result
        }
        
        // Fetch from server
        const result = await this.fetchPermission(resourceID, userID)
        
        // Cache for future use
        this.permissionCache.set(cacheKey, {
            result,
            timestamp: Date.now()
        })
        
        return result
    }
    
    private isFresh(cached: CachedPermission): boolean {
        return Date.now() - cached.timestamp < this.CACHE_TTL
    }
    
    // Invalidate cache when permissions change
    invalidateUser(userID: string): void {
        for (const [key, _] of this.permissionCache) {
            if (key.endsWith(`:${userID}`)) {
                this.permissionCache.delete(key)
            }
        }
    }
    
    invalidateResource(resourceID: string): void {
        for (const [key, _] of this.permissionCache) {
            if (key.startsWith(`${resourceID}:`)) {
                this.permissionCache.delete(key)
            }
        }
    }
}
```

### 预加载权限

为可能需要的资源预加载权限：

```typescript
class PreemptivePermissionLoader {
    async preloadPermissions(context: UserContext): Promise<void> {
        // Load permissions for recently accessed resources
        const recentResources = await this.getRecentResources(context.userID)
        
        // Load permissions for team resources
        const teamResources = await this.getTeamResources(context.teamIDs)
        
        // Batch load to minimize API calls
        const allResources = [...recentResources, ...teamResources]
        const permissions = await this.batchLoadPermissions(
            allResources, 
            context.userID
        )
        
        // Populate cache
        permissions.forEach(perm => {
            this.cache.set(`${perm.resourceID}:${context.userID}`, {
                result: perm,
                timestamp: Date.now()
            })
        })
    }
}
```

这些优化确保权限检查不会成为用户体验瓶颈，同时保持安全保障。

## 设计权衡

实现过程中揭示了几个有趣的权衡：

**简单性 vs. 灵活性**：三层模型简单易懂且易于实现，但不支持细粒度权限，如"与特定用户共享"或"只读访问"。对于面向个人开发者和小团队的工具，这可能是正确的选择。

**安全性 vs. 便利性**：基于 URL 的共享让分享线程变得简单（只需发送链接！），但意味着任何拥有 URL 的人都可以访问公共线程。UUID 的随机性提供了安全保障，但它仍然是基于能力的模型。

**一致性 vs. 性能**：乐观更新使 UI 感觉更响应，但创建了本地状态可能与服务器状态不匹配的窗口期。实现通过回滚优雅地处理了这个问题，但增加了复杂性。

**向后兼容 vs. 简洁代码**：降级 API 机制增加了代码复杂度，但确保了平滑的部署和回滚。这是生产系统需要的务实决策。

## 实现原则

在为协作式 AI 工具构建共享系统时，考虑以下关键原则：

### 1. 从简单开始
三层模型（私有/团队/公共）覆盖了大多数用例，无需复杂的 ACL 系统。有需要时可以随时增加复杂性。

### 2. 使状态转换明确
使用独立标志而非枚举使权限变更更有意图性，防止意外暴露。

### 3. 为故障而设计
实现带回滚的乐观更新、带退避的重试逻辑和优雅降级模式。

### 4. 策略性缓存
本地缓存防止权限检查阻塞 UI 交互，同时保持安全性。

### 5. 支持运维需求
从一开始就规划支持工作流、调试访问和管理覆盖。

### 6. 为常见模式优化
大多数开发者遵循可预测的共享模式：
- 开发期间使用私有模式
- 代码审查时使用团队共享
- 教学或文档使用公共共享

围绕这些自然工作流设计系统，而不是试图支持每种可能的权限组合。

### 7. 维护审计追踪
追踪权限变更用于调试、合规和安全分析。

```typescript
interface PermissionAuditEvent {
    timestamp: Date
    resourceID: string
    userID: string
    action: 'granted' | 'revoked' | 'modified'
    previousState?: PermissionState
    newState: PermissionState
    reason?: string
}
```

### 8. 考虑隐私设计
默认使用私有共享，要求明确操作才能提高可见性。向用户清楚说明每个共享级别的含义。

最重要的洞察是，有效的权限系统应与人类的信任模式和工作流保持一致。技术复杂性应服务于用户需求，而不是为协作制造障碍。
