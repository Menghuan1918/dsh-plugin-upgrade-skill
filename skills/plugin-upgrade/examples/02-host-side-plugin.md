# 示例 02：宿主侧插件（已更新为 RemoteResult）

**场景**: 插件需要调用宿主的 session 管理 API。

**影响触点**: #3 内部服务探测（APIProxy 调用 → Remote）

**复杂度**: ⭐⭐

---

## 升级前

```typescript
// src/service.ts
import { executeRemote } from '@deepseek-ai/dsh-host-apiproxy'

export class MyService {
  async listSessions() {
    return executeRemote('session', 'list', { limit: 10 })
  }
  
  async getSession(id: string) {
    return executeRemote('session', 'get', { id })
  }
}
```

---

## 升级后（alpha.2 RemoteResult 契约）

```typescript
// src/service.ts
import type { Context } from '@deepseek-ai/cordis'

export class MyService {
  constructor(private ctx: Context) {}

  async listSessions() {
    // Consumer 侧 Remote 方法返回 RemoteResult<T>，永不 reject
    const result = await this.ctx.remote.session.list({ limit: 10 })
    
    // 业务失败走 ok === false 分支
    if (!result.ok) {
      // result.error 是 typed RemoteFailure，code 直接可读
      if (result.error.code === 'session/permission-denied') {
        return []
      }
      throw result.error  // 真 Error，带 stack
    }
    return result.value
  }
  
  async getSession(id: string) {
    const result = await this.ctx.remote.session.get({ id })
    if (!result.ok) {
      if (result.error.code === 'session/not-found') {
        return null
      }
      throw result.error
    }
    return result.value
  }
}
```

---

## 迁移步骤

1. **删除 APIProxy 导入**:
   ```typescript
   // 删除这行
   import { executeRemote } from '@deepseek-ai/dsh-host-apiproxy'
   ```

2. **注入 Context**:
   ```typescript
   constructor(private ctx: Context) {}
   ```

3. **改写所有调用为 RemoteResult 流**:
   
   | 旧调用 | 新调用 | 错误处理 |
   |---|---|---|
   | `executeRemote('session', 'list', args)` | `ctx.remote.session.list(args)` | `if (!result.ok)` 分支 |
   | `executeRemote('session', 'get', args)` | `ctx.remote.session.get(args)` | 读 `result.error.code` |

   参考 [DSH-0.1.2-A1-01](../references/v0.1.2-alpha.1.md) 的 17 条操作映射表。

4. **错误处理三原则**:
   - Consumer 侧 Remote 方法**永不 reject**，返回 `RemoteResult<T>`
   - 业务失败 `result.ok === false`，读 `result.error.code`
   - catch 只用于 Gateway client 层（`gateway/internal` 等传输故障）

5. **更新 package.json**:
   ```sh
   pnpm remove @deepseek-ai/dsh-host-apiproxy
   ```

---

## 验证

```sh
# 1. 检查无残留引用
grep -r "dsh-host-apiproxy\|executeRemote" src/
# 预期：无输出

# 2. 构建
pnpm run build

# 3. 静态通过不等于运行时契约正确——Remote 描述符漂移在静态层是静默的
# 必须真实冷启动 + 完整对话

# 4. 观察日志
# - 无 service-unavailable 循环
# - 业务失败走 ok:false 分支
# - 错误码能读到且 typed
```

---

## 常见错误

### 错误 1: 仍用 try/catch 包 Remote 调用

**症状**: 业务失败被当异常处理。

**原因**: alpha.2 重构为 `RemoteResult`，Consumer 侧永不 reject。

**修正**:
```typescript
// 错误
try {
  const data = await ctx.remote.session.list({})
} catch (error) {
  // 永远不会进这里（业务失败不是异常）
}

// 正确
const result = await ctx.remote.session.list({})
if (!result.ok) {
  // 业务失败在这处理
}
```

### 错误 2: 读 `error.failure.code`

**原因**: alpha.1 时代 `TypertRemoteFailure` 的形状，alpha.2 已删除。

**修正**: `result.error.code`（直接访问）。

### 错误 3: Gateway client 层误用 RemoteResult

**场景**: 直接调 `gateway.invoke()` 而非 `ctx.remote.*`。

**说明**: Gateway client 层仍可能 reject（`gateway/cancelled`、`gateway/internal`），需 catch + `isRemoteFailure` 判别；但其返回值也是 `RemoteResult`，业务失败仍走 `ok: false`。两层都要处理。

**来源**: [ctx-remote-failure-vocabulary](https://github.com/deepseek-ai/deepseek-harness/blob/dsh-v0.1.2-alpha.2/.agents/notes/implemented/architecture/2026-08-28-ctx-remote-failure-vocabulary.md)（status: implemented）。
