## 权限系统

权限系统通过三部分模型构成关键的安全层：

1. **请求**：工具通过 `needsPermissions()` 声明需要的权限
2. **对话**：用户通过 `PermissionRequest` 组件看到附带上下文信息的显式权限请求
3. **持久化**：已批准的权限可以通过 `savePermission()` 保存以供后续使用

### TypeScript 实现

以下是实际工作方式：

```typescript
// Tool requesting permissions
const EditTool: Tool = {
  name: "Edit",
  /* other properties */
  
  // Each tool decides when it needs permission
  needsPermissions: (input: EditParams): boolean => {
    const { file_path } = input;
    return !hasPermissionForPath(file_path, "write");
  },
  
  async *call(input: EditParams, context: ToolContext) {
    const { file_path, old_string, new_string } = input;
    
    // Access will be automatically checked by the framework
    // If permission is needed but not granted, this code won't run
    
    // Perform the edit operation...
    const result = await modifyFile(file_path, old_string, new_string);
    yield { success: true, message: `Modified ${file_path}` };
  }
};

// Permission system implementation
function hasPermissionForPath(path: string, access: "read" | "write"): boolean {
  // Check cached permissions first
  const permissions = getPermissions();
  
  // Try to match permissions with path prefix
  for (const perm of permissions) {
    if (
      perm.type === "path" && 
      perm.access === access &&
      path.startsWith(perm.path)
    ) {
      return true;
    }
  }
  
  return false;
}

// Rendering permission requests to the user
function PermissionRequest({ 
  tool, 
  params,
  onApprove, 
  onDeny 
}: PermissionProps) {
  return (
    <Box flexDirection="column" borderStyle="round" padding={1}>
      <Text>Claude wants to use {tool.name} to modify</Text>
      <Text bold>{params.file_path}</Text>
      
      <Box marginTop={1}>
        <Button onPress={() => {
          // Save permission for future use
          savePermission({
            type: "path",
            path: params.file_path,
            access: "write",
            permanent: true 
          });
          onApprove();
        }}>
          Allow
        </Button>
        
        <Box marginLeft={2}>
          <Button onPress={onDeny}>Deny</Button>
        </Box>
      </Box>
    </Box>
  );
}
```

系统对不同的权限类型有专门的处理：

- **工具权限**：使用特定工具的通用权限
- **Bash 命令权限**：对 shell 命令的细粒度控制
- **文件系统权限**：针对目录的独立读/写权限

### 基于路径的权限模型

对于文件系统操作，目录权限会级联到子路径，在维护安全边界的同时减少权限疲劳：

```typescript
// Parent directory permissions cascade to children
if (hasPermissionForPath("/home/user/project", "write")) {
  // These will automatically be allowed without additional prompts
  editFile("/home/user/project/src/main.ts");
  createFile("/home/user/project/src/utils/helpers.ts");
  deleteFile("/home/user/project/tests/old-test.js");
}

// But operations outside that directory still need approval
editFile("/home/user/other-project/config.js"); // Will prompt for permission
```

这种模式平衡了安全性和可用性——用户不需要批准每一个文件操作，但仍然保持对智能体可以访问哪些目录的控制权。

### 安全措施

额外的安全特性包括：

- **命令注入检测**：分析 shell 命令中的可疑模式
- **路径规范化**：在检查前规范化路径，防止路径遍历攻击
- **风险评分**：根据操作的潜在影响为其分配风险等级
- **安全命令列表**：预先批准常见的开发操作（ls、git status 等）

权限系统是让用户能够自信地与一个直接访问其文件系统和终端的 AI 交互的主要安全机制。

