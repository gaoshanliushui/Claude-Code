# 异步生成器（Async Generator）与 Promise 的深度对比

> 本文深入解释 JavaScript/TypeScript 中的异步编程抽象。
> 阅读 Claude Code 这样的工业级代码，理解 Async Generator 是基础。

---

## 一、Promise：异步的"完成事件"

### 1.1 本质

Promise 是一个**未来会完成的单一事件**。

```ts
// Promise = 一个"未来会完成的事件"
// 它是一个**单一**的值，可以是成功或失败
const promise: Promise<string> = fetch('https://api.example.com/data')
  .then(response => response.json())
  .then(data => data.name)

// 一次性消费
promise.then(name => console.log(name))
//       ↑ 只返回一个值
```

### 1.2 关键特征

| 特征 | 说明 |
| --- | --- |
| **值的数量** | 1 个值（成功或失败） |
| **消费方式** | `.then()` / `.await` 只能消费一次 |
| **完成时机** | 单次 resolve 或 reject |
| **内存占用** | 小（只有一个值） |
| **适用场景** | 单次异步操作（API 调用、文件读取） |

### 1.3 生活类比

```
Promise 就像点一份外卖：
- 下单（创建 Promise）
- 等待配送（pending）
- 收到一份外卖（resolved）
- 或配送失败（rejected）

你只会收到**一份**外卖。
```

---

## 二、Generator：可暂停的函数

### 2.1 本质

Generator 是一个可以**暂停和恢复**的函数，使用 `function*` 和 `yield` 关键字。

```ts
function* numberGenerator(): Generator<number> {
  console.log('开始')
  yield 1                    // 暂停，返回 1
  console.log('继续')
  yield 2                    // 暂停，返回 2
  console.log('继续')
  yield 3                    // 暂停，返回 3
  console.log('结束')
}

// 使用
const gen = numberGenerator()  // 创建 Generator（不执行）
console.log(gen.next())         // { value: 1, done: false }
console.log(gen.next())         // { value: 2, done: false }
console.log(gen.next())         // { value: 3, done: false }
console.log(gen.next())         // { value: undefined, done: true }
```

### 2.2 关键特征

| 特征 | 说明 |
| --- | --- |
| **值的数量** | 任意多个值（按需 yield） |
| **消费方式** | `.next()` 逐个消费 |
| **暂停/恢复** | 可以在 yield 处暂停 |
| **双向通信** | 可以通过 `.next(value)` 向 generator 传值 |
| **内存占用** | 小（不缓存所有值） |
| **适用场景** | 惰性序列、无限流、协作式多任务 |

### 2.3 生活类比

```
Generator 就像自助餐：
- 你拿一个盘子（创建 Generator）
- 每次取一道菜（yield 一个值）
- 可以暂停吃饭（暂停 Generator）
- 可以继续吃（恢复 Generator）

你可以取**任意多**道菜。
```

---

## 三、Async Generator：异步 + 可暂停

### 3.1 本质

Async Generator 是 Generator 和 Async 的组合，使用 `async function*` 和 `yield` 关键字。

```ts
async function* streamData(): AsyncGenerator<string> {
  console.log('开始')
  yield await fetchChunk(1)  // 异步暂停，等待数据
  console.log('继续')
  yield await fetchChunk(2)  // 异步暂停，等待数据
  console.log('结束')
}

// 使用
for await (const chunk of streamData()) {
  console.log(chunk)
  // 可以随时 break 退出
}
```

### 3.2 关键特征

| 特征 | 说明 |
| --- | --- |
| **值的数量** | 任意多个异步值 |
| **消费方式** | `for await ... of` 逐个消费 |
| **异步暂停** | 可以在 `await` 处等待异步操作 |
| **流式输出** | 不必等待所有值就绪 |
| **可中断** | 可以通过 AbortSignal 中断 |
| **内存占用** | 极小（不缓存所有值） |
| **适用场景** | 流式响应、SSE、WebSocket、实时数据 |

### 3.3 生活类比

```
Async Generator 就像 Netflix 流媒体：
- 开始播放（启动）
- 持续接收数据包（yield 数据）
- 网络慢时缓冲（await 等待）
- 可以随时暂停/恢复/停止

你可以观看**任意时长**的内容。
```

---

## 四、核心对比表

### 4.1 语法对比

```ts
// ===== Promise =====
const promise: Promise<string> = asyncFunc()

// ===== Generator =====
function* genFunc(): Generator<number> {
  yield 1
  yield 2
}

// ===== Async Generator =====
async function* asyncGenFunc(): AsyncGenerator<string> {
  yield await fetchData()
  yield await fetchData()
}
```

### 4.2 行为对比

| 维度 | Promise | Generator | Async Generator |
| --- | --- | --- | --- |
| **异步** | ✅ | ❌ | ✅ |
| **多个值** | ❌（只有 1 个） | ✅ | ✅ |
| **可暂停** | ❌ | ✅ | ✅ |
| **可中断** | 有限（AbortController） | ✅（return） | ✅（return + AbortController） |
| **流式输出** | ❌ | ✅ | ✅ |
| **消费语法** | `.then()` / `await` | `.next()` | `for await ... of` |
| **内存占用** | 小 | 小 | 极小（适合无限流） |

### 4.3 时间线对比

```
Promise 模式：
┌────────────────────────────────────────────────┐
│  ───────(await)─────────[resolved value]─────  │
│  一次性产生一个值                                    │
└────────────────────────────────────────────────┘

Async Generator 模式：
┌────────────────────────────────────────────────┐
│  ──await──[v1]──await──[v2]──await──[v3]──...    │
│  持续产生多个值                                      │
└────────────────────────────────────────────────┘
```

---

## 五、Claude Code 中的实际例子

### 5.1 Promise 风格（API 调用）

```ts
// src/services/api/claude.ts
// API 调用返回 Promise
async function queryModelWithStreaming(params): Promise<StreamResponse> {
  const response = await fetch(API_URL, {...})
  return processStreamResponse(response)
}

// 调用方
const result = await queryModelWithStreaming(params)
// 一次性获得完整结果
```

**问题**：在流式场景下，必须等待整个响应完成才能拿到结果。

### 5.2 Async Generator 风格（query 主循环）

```ts
// src/query.ts
// 异步生成器：逐个 yield 流式事件
async function* query(params): AsyncGenerator<StreamEvent | Message | ToolUseSummary | ...> {
  // ...
  while (true) {
    // 阶段 1：模型流式调用
    for await (const event of callModel(...)) {
      yield event  // 每个流式事件立即 yield 出去
    }
    
    // 阶段 2：工具执行
    const toolResults = await runTools(...)
    yield* generateToolResultMessages(toolResults)
    
    // 阶段 3：判断是否继续
    if (decision === 'end_turn') return
  }
}

// 调用方：流式消费
for await (const event of query(params)) {
  if (event.type === 'text') {
    process.stdout.write(event.text)  // 立即打印
  } else if (event.type === 'tool_use') {
    console.log('工具调用：', event.name)
  }
  // 可以随时 break 中断
}
```

### 5.3 QueryEngine 的 Async Generator

```ts
// src/QueryEngine.ts
class QueryEngine {
  // 异步生成器方法
  async* submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // 系统 prompt 构建
    yield { type: 'system', subtype: 'init', ... }
    
    // 用户输入处理
    yield { type: 'user', ... }
    
    // 模型响应
    for await (const message of query(...)) {
      yield this.normalizeMessage(message)  // 转换为 SDK 消息
    }
    
    // 最终结果
    yield { type: 'result', subtype: 'success', ... }
  }
}

// SDK 用户使用
import { ask } from 'src/QueryEngine.js'

for await (const message of ask({
  prompt: '你好',
  // ...
})) {
  console.log(message)
  // SDK 客户端可以实时接收每个消息
}
```

### 5.4 在 print.ts 中的消费

```ts
// src/cli/print.ts
// 流式消费 ask() 的输出
for await (const message of runHeadlessStreaming(...)) {
  // 每个消息立即 yield 给 SDK 客户端
  await structuredIO.write(message)
  // 用户可以在响应完成前就看到部分内容
}
```

### 5.5 流式工具执行

```ts
// src/services/tools/StreamingToolExecutor.ts
async function* executeToolsStreaming(
  toolUses: ToolUseBlock[]
): AsyncGenerator<ToolResult> {
  // 并行启动所有工具
  const promises = toolUses.map(toolUse => executeOne(toolUse))
  
  // 流式 yield 每个完成的工具结果（谁先完成谁先 yield）
  for await (const result of racePromises(promises)) {
    yield result
  }
}
```

---

## 六、为什么要用 Async Generator？

### 6.1 流式响应（Streaming）

```ts
// ❌ 用 Promise：必须等待整个响应
async function generateResponse(): Promise<string> {
  const chunks = await Promise.all([
    fetchChunk(1),  // 等待
    fetchChunk(2),  // 等待
    fetchChunk(3),  // 等待
  ])
  return chunks.join('')  // 全部就绪后才返回
}

// ✅ 用 Async Generator：逐个 yield
async function* generateResponse(): AsyncGenerator<string> {
  yield await fetchChunk(1)  // 立即返回
  yield await fetchChunk(2)  // 立即返回
  yield await fetchChunk(3)  // 立即返回
}

// SDK 用户实时接收
for await (const chunk of generateResponse()) {
  process.stdout.write(chunk)  // 实时打印
}
```

### 6.2 可中断（Interruptible）

```ts
// ✅ Async Generator 可以随时中断
const stream = generateResponse()

for await (const chunk of stream) {
  process.stdout.write(chunk)
  if (userPressedCtrlC) {
    break  // 立即停止，停止所有未完成的 await
  }
}
```

### 6.3 内存效率（Memory Efficient）

```ts
// ❌ Promise.all 需要缓存所有结果
const results = await Promise.all(
  Array.from({ length: 1000000 }, (_, i) => fetchChunk(i))
)
// 内存：100 万个结果全部在内存中

// ✅ Async Generator 一次只持有一个值
async function* streamResults() {
  for (let i = 0; i < 1000000; i++) {
    yield await fetchChunk(i)
    // 每次只有一个值在内存中
  }
}
```

### 6.4 协作式多任务（Cooperative Multitasking）

```ts
// ✅ 在 yield 处可以插入其他逻辑
async function* processQueue(items: Item[]) {
  for (const item of items) {
    const result = await processItem(item)
    
    // 在每次 yield 之前，让出控制权
    // 其它异步任务可以插入执行
    yield result
    
    // 检查是否需要取消
    if (abortSignal.aborted) {
      console.log('已取消')
      return  // 优雅退出
    }
  }
}
```

---

## 七、Async Generator 的核心机制

### 7.1 双向通信

```ts
async function* chat(): AsyncGenerator<string, void, string> {
  // 第一次 next() 没有传值
  const name = yield '你叫什么名字？'
  
  // 第二次 next('Alice') 传入值
  yield `你好，${name}！`
  
  // 第三次 next('我很好') 传入值
  const mood = yield '你今天怎么样？'
  yield `听起来${mood}`
}

// 使用
const gen = chat()
console.log(await gen.next())              // { value: '你叫什么名字？', done: false }
console.log(await gen.next('Alice'))         // { value: '你好，Alice！', done: false }
console.log(await gen.next('我很好'))        // { value: '你今天怎么样？', done: false }
console.log(await gen.next('开心'))          // { value: '听起来开心', done: false }
console.log(await gen.next())               // { value: undefined, done: true }
```

### 7.2 错误传播

```ts
async function* riskyOperation(): AsyncGenerator<string> {
  try {
    yield '开始'
    yield await mightFail()
    yield '完成'
  } catch (error) {
    yield `出错：${error.message}`
  } finally {
    console.log('清理资源')
  }
}

// 使用
const gen = riskyOperation()
console.log(await gen.next())  // { value: '开始', done: false }

try {
  await gen.throw(new Error('手动抛出'))
} catch (e) {
  // generator 内部 catch 到错误
}
```

### 7.3 return 提前终止

```ts
async function* infiniteStream(): AsyncGenerator<number> {
  let i = 0
  while (true) {
    yield i++
  }
}

// 使用
const stream = infiniteStream()
const reader = stream[Symbol.asyncIterator]()

await reader.next()  // 0
await reader.next()  // 1
await reader.next()  // 2
await reader.return()  // 提前终止，触发 finally
```

### 7.4 yield* 委托给另一个生成器

```ts
async function* inner(): AsyncGenerator<number> {
  yield 1
  yield 2
}

async function* outer(): AsyncGenerator<number> {
  yield* inner()  // 委托给 inner，自动 yield 每个值
  yield 3
  yield 4
}

// 等价于
async function* outerEquivalent(): AsyncGenerator<number> {
  for await (const value of inner()) {
    yield value
  }
  yield 3
  yield 4
}
```

---

## 八、实际应用场景对比

### 8.1 适用场景

| 场景 | 推荐 | 原因 |
| --- | --- | --- |
| 单次 API 调用 | Promise | 简单，不需要流式 |
| 文件读取 | Promise | 单次操作 |
| WebSocket 接收 | Async Generator | 持续流式 |
| SSE 流式响应 | Async Generator | 服务器持续推送 |
| 大文件读取 | Async Generator | 避免一次性读入内存 |
| 实时数据处理 | Async Generator | 持续输入 |
| Agent 工具调用循环 | Async Generator | 多步骤、有状态 |

### 8.2 Claude Code 中的具体应用

```ts
// ✅ 1. 流式 API 响应
async function* queryModel(params): AsyncGenerator<StreamEvent> {
  const stream = await fetch(API_URL, {...})
  for await (const chunk of stream) {
    yield chunk  // 立即 yield
  }
}

// ✅ 2. 多步骤 Agent 循环
async function* query(params): AsyncGenerator<Message> {
  while (true) {
    yield* streamModelResponse(...)  // yield 每个消息
    
    if (shouldContinue()) {
      yield* executeTools(...)  // yield 每个工具结果
    } else {
      return  // 结束
    }
  }
}

// ✅ 3. SDK 消息流
async function* submitMessage(prompt): AsyncGenerator<SDKMessage> {
  yield { type: 'system', subtype: 'init', ... }
  yield* this.executeQuery(...)
  yield { type: 'result', ... }
}

// ✅ 4. 文件历史的流式快照
async function* streamSnapshots(messages: Message[]) {
  for (const msg of messages) {
    const snapshot = await createSnapshot(msg)
    yield snapshot
  }
}
```

---

## 九、性能对比

### 9.1 内存占用

```
处理 100 万条记录：

Promise.all：
┌────────────────────────────────────────┐
│ 内存：[1][2][3]...[1000000]            │
│ 大小：~80MB                             │
└────────────────────────────────────────┘

Async Generator：
┌────────────────────────────────────────┐
│ 内存：[当前一条]                        │
│ 大小：~80B                              │
└────────────────────────────────────────┘
```

### 9.2 延迟

```
Promise.all：
开始 → ────────────等待所有─────────── → 全部完成
       0ms                              5000ms
       │                                │
       用户看不到任何东西 5000ms         │

Async Generator：
开始 → [1] → [2] → [3] → ...
       0ms  50ms  100ms  150ms
       │    │    │    │
       用户能立即看到第一批数据
```

### 9.3 用户体验对比

| 维度 | Promise | Async Generator |
| --- | --- | --- |
| 首字节延迟 | 高（全部完成） | 低（立即可见） |
| 内存占用 | 高 | 低 |
| 可中断 | 弱 | 强 |
| 实时性 | 差 | 优 |
| 复杂度 | 低 | 中 |

---

## 十、常见误区

### 10.1 误区 1：Async Generator 和 Promise 互斥

**❌ 错误**：以为只能选其一

**✅ 正确**：两者可以组合使用

```ts
async function* streamChunks(): AsyncGenerator<string> {
  // 在 async generator 内部使用 Promise
  const chunk = await fetchChunk()  // Promise → 转异步
  yield chunk
  
  // 还可以 yield Promise（会被自动 await）
  yield fetchChunk()
}
```

### 10.2 误区 2：以为 Async Generator 是多线程

**❌ 错误**：以为可以并行执行多个 yield

**✅ 正确**：仍然是单线程协作式并发

```ts
// ❌ 这样不会并行
async function* notParallel(): AsyncGenerator<string> {
  yield await fetchA()  // 顺序执行
  yield await fetchB()
}

// ✅ 用 Promise.all 实现并行
async function* parallel(): AsyncGenerator<string> {
  const [a, b] = await Promise.all([fetchA(), fetchB()])
  yield a
  yield b
}
```

### 10.3 误区 3：以为 return 等同于 break

**✅ 这是对的**：在 `for await...of` 中，`break` 会调用 generator 的 `return()` 方法。

```ts
for await (const item of stream) {
  if (someCondition) break  // 触发 generator 的 return()
}
```

### 10.4 误区 4：忘记 try/finally 中的资源清理

**❌ 错误**：认为 generator 一定会执行完

**✅ 正确**：可能在任何时候被中断

```ts
// ❌ 危险
async function* leakResource(): AsyncGenerator<Data> {
  const file = await fs.open(path, 'r')
  while (true) {
    yield await file.read()
  }
  // 如果 break，文件句柄泄漏
}

// ✅ 安全
async function* safeResource(): AsyncGenerator<Data> {
  const file = await fs.open(path, 'r')
  try {
    while (true) {
      yield await file.read()
    }
  } finally {
    await file.close()  // 一定会执行
  }
}
```

### 10.5 误区 5：在 async generator 中错误地使用 await

```ts
// ❌ await 不必要（失去并发性）
async function* bad(): AsyncGenerator<string> {
  const result = await somePromise  // 阻塞
  yield result
}

// ✅ 直接 yield Promise
async function* good(): AsyncGenerator<string> {
  yield somePromise  // 自动 await
}

// ✅ 或用 Promise.all 实现并行
async function* parallel(): AsyncGenerator<string> {
  const [a, b] = await Promise.all([
    fetchA(),
    fetchB(),
  ])
  yield a
  yield b
}
```

---

## 十一、最佳实践

### 11.1 选择正确的工具

```ts
// ✅ 单次异步操作 → Promise
async function fetchUser(id: string): Promise<User> {
  return await db.users.findOne({ id })
}

// ✅ 流式数据 → Async Generator
async function* streamUserActivities(userId: string): AsyncGenerator<Activity> {
  let cursor = null
  while (true) {
    const { activities, nextCursor } = await db.activities.find({
      userId,
      cursor,
    })
    for (const activity of activities) {
      yield activity
    }
    if (!nextCursor) return
    cursor = nextCursor
  }
}
```

### 11.2 错误处理

```ts
async function* safeStream(): AsyncGenerator<Item> {
  try {
    while (true) {
      const item = await fetchItem()
      yield item
    }
  } catch (error) {
    // 记录错误
    logError(error)
    
    // 优雅退出
    return
  }
}

// 调用方处理错误
try {
  for await (const item of safeStream()) {
    process(item)
  }
} catch (error) {
  // stream 内部的 try/catch 已经处理过
  // 这里的 catch 只会捕获流本身的错误
}
```

### 11.3 资源清理

```ts
async function* streamFile(path: string): AsyncGenerator<Buffer> {
  const file = await fs.open(path, 'r')
  try {
    while (true) {
      const chunk = await file.read()
      if (chunk.bytesRead === 0) return
      yield chunk.buffer
    }
  } finally {
    // 重要：无论正常完成还是异常退出，都要清理
    await file.close()
  }
}
```

### 11.4 性能优化技巧

```ts
// ✅ 技巧 1：背压（Backpressure）
async function* withBackpressure(items: Item[]) {
  const concurrency = 10
  const queue: Promise<Item>[] = []
  
  for (const item of items) {
    if (queue.length >= concurrency) {
      // 等待任意一个完成
      yield await Promise.race(queue)
      // 清理已完成的
      queue.splice(queue.findIndex(p => isResolved(p)), 1)
    }
    queue.push(processItem(item))
  }
  
  // 处理剩余的
  for (const promise of queue) {
    yield await promise
  }
}

// ✅ 技巧 2：批量 yield
async function* batched(items: Item[], batchSize: number = 10) {
  let batch: Item[] = []
  
  for (const item of items) {
    batch.push(item)
    if (batch.length >= batchSize) {
      yield batch
      batch = []
    }
  }
  
  if (batch.length > 0) yield batch
}
```

---

## 十二、总结

### 12.1 一句话对比

| 类型 | 一句话 |
| --- | --- |
| **Promise** | 一次性的"未来值"，要么成功要么失败 |
| **Generator** | 可暂停的函数，可以产出多个值 |
| **Async Generator** | 异步 + 可暂停 + 多值，是 Promise 和 Generator 的超集 |

### 12.2 何时用什么

```
问题：是否需要多个值？
├── 否 → 用 Promise
└── 是
    ├── 这些值是同步的？
    │   ├── 是 → 用 Generator
    │   └── 否 → 用 Async Generator ✅
    └──
问题：是否需要流式输出？
├── 是 → 用 Async Generator ✅
└── 否 → 看是否需要异步
```

### 12.3 类型层次关系

```
Generator<T>
  ↓ extends
AsyncGenerator<T>     ←── Async Generator = Generator + Promise

Promise<T>
  ↓ 和
AsyncGenerator<T>     ←── 完全不同的两个抽象
```

### 12.4 Claude Code 的选择

Claude Code 大量使用 `async function*` 是**正确的选择**，因为：

1. **Agent 是流式的**：模型响应、工具执行都是持续的
2. **需要可中断**：用户可以随时 Ctrl+C 中断
3. **需要低延迟**：用户应该立即看到响应
4. **需要低内存**：不能缓存所有结果

理解这一点，对于阅读 Claude Code 源码至关重要！

### 12.5 给学习者的建议

学习 Async Generator 的最佳路径：

1. **先掌握 Promise**：理解异步基础
2. **再学 Generator**：理解可暂停函数
3. **最后学 Async Generator**：组合两者
4. **实践**：写一个简单的 SSE 客户端
5. **深入**：阅读 Claude Code 的 `query.ts` 和 `QueryEngine.ts`

### 12.6 一句话总结

> **异步生成器是 JavaScript 异步编程的终极抽象**，掌握它就掌握了现代 Node.js 编程的精髓。

**它让"流式 + 异步 + 可中断"这三件本来复杂的事情，变得简单自然。**