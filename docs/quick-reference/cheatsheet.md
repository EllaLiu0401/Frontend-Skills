# Frontend Coding Cheatsheet

> 一页纸速查手册 | One-page quick reference

打印或保存这个页面，在写代码时随时参考。

---

## TypeScript 精华 | TypeScript Essentials

### 类型推断优先 | Inference First
```typescript
// ✅ 推断
useState(0)
useState<User | null>(null)  // Union 需要显式

// ❌ 冗余
useState<number>(0)
```

### 避免 any | Avoid any
```typescript
// ✅ 用 unknown
function handle(data: unknown) {
  if (typeof data === 'string') { }
}

// ❌ 不要用 any
function handle(data: any) { }
```

### Null 处理 | Null Handling
```typescript
// ✅ Optional chaining
user?.profile?.name

// ✅ Nullish coalescing
const name = user?.name ?? 'Guest'
```

---

## React 精华 | React Essentials

### useEffect 依赖 | useEffect Dependencies
```typescript
// ✅ 完整依赖
useEffect(() => {
  fetch(url);
}, [url]);

// ✅ 清理
useEffect(() => {
  const timer = setInterval(() => {}, 1000);
  return () => clearInterval(timer);
}, []);
```

### 性能优化 | Performance
```typescript
// useMemo: 昂贵计算
const sorted = useMemo(() => 
  items.sort((a, b) => a - b),
  [items]
);

// useCallback: 传给子组件
const onClick = useCallback(() => {
  doSomething(value);
}, [value]);

// React.memo: 纯组件
const Child = React.memo(({ data }) => <div>{data}</div>);
```

### 状态更新 | State Updates
```typescript
// ✅ 不可变更新
setItems([...items, newItem])
setUser({ ...user, name: 'New' })

// ❌ 不要直接修改
items.push(newItem)
user.name = 'New'
```

---

## 代码质量 | Code Quality

### 命名规范 | Naming
```typescript
// Variables/Functions: camelCase
const userName = 'Alice'
function getUserData() {}

// Components/Classes: PascalCase
function UserProfile() {}
class ApiService {}

// Constants: UPPER_SNAKE_CASE
const MAX_RETRY = 3
const API_URL = 'https://...'

// Boolean: is/has/should/can
const isLoading = true
const hasError = false
```

### 函数原则 | Function Rules
```typescript
// ✅ 简短 (< 50 行)
// ✅ 单一职责
// ✅ 提前返回
function process(user: User | null) {
  if (!user) return null
  if (!user.active) return null
  return user.name
}

// ❌ 嵌套太深
function process(user: User | null) {
  if (user) {
    if (user.active) {
      return user.name
    }
  }
}
```

### 删除冗余 | Remove Redundant
```typescript
// ❌ 未使用的代码
import { unused } from 'lib'
const unusedVar = 'hello'

// ❌ 注释掉的代码
// const oldCode = () => {}

// ❌ console.log
console.log('debug')

// ✅ 全部删除
```

---

## 错误处理 | Error Handling

### 异步错误 | Async Errors
```typescript
// ✅ try-catch
try {
  const data = await fetchData()
} catch (error) {
  console.error('Fetch failed:', error)
  showErrorToast()
}

// ✅ .catch()
fetchData()
  .then(handleData)
  .catch(handleError)

// ✅ React Error Boundary
<ErrorBoundary>
  <App />
</ErrorBoundary>
```

### 用户反馈 | User Feedback
```typescript
// ✅ 加载状态
{isLoading && <Spinner />}

// ✅ 错误状态
{error && <ErrorMessage error={error} />}

// ✅ 成功反馈
toast.success('Saved successfully!')
```

---

## 常见错误 | Common Mistakes

### ❌ 空的事件处理
```typescript
<button onClick={() => {}}>Click</button>
// 应该: 实现功能或隐藏按钮
```

### ❌ 缺少依赖
```typescript
useEffect(() => {
  fetchData(userId)
}, []) // 应该: [userId]
```

### ❌ 忘记清理
```typescript
useEffect(() => {
  const timer = setInterval(() => {}, 1000)
  // 应该: return () => clearInterval(timer)
}, [])
```

### ❌ 渲染中创建对象
```typescript
<Child config={{ theme: 'dark' }} />
// 应该: useMemo
```

### ❌ 过度注解类型
```typescript
const [count, setCount] = useState<number>(0)
// 应该: useState(0)
```

---

## PR 检查清单 | PR Checklist

### 提交前 | Before Submit
- [ ] Lint 通过
- [ ] 类型检查通过
- [ ] 测试通过
- [ ] 删除 console.log
- [ ] 删除未使用代码
- [ ] 实际运行并测试

### 功能 | Functionality
- [ ] 交互元素有功能
- [ ] 错误处理完善
- [ ] 加载状态显示
- [ ] 边界情况测试

### 代码 | Code
- [ ] 命名清晰
- [ ] 函数简短
- [ ] 无重复代码
- [ ] 注释适当

### React | React
- [ ] useEffect 依赖完整
- [ ] 副作用已清理
- [ ] 性能优化合理

---

## 决策树 | Decision Trees

### 何时使用 useMemo?
```
计算是否昂贵? (排序、过滤、复杂计算)
  └─ 是 → 使用 useMemo
  └─ 否 → 依赖变化是否频繁?
      └─ 否 → 使用 useMemo
      └─ 是 → 不需要
```

### 何时使用 useCallback?
```
函数是否传给子组件?
  └─ 是 → 子组件是否 memo?
      └─ 是 → 使用 useCallback
      └─ 否 → 不需要
  └─ 否 → 是否作为 effect 依赖?
      └─ 是 → 使用 useCallback
      └─ 否 → 不需要
```

### 何时显式类型注解?
```
TypeScript 能推断吗?
  └─ 能 → 不需要注解
  └─ 不能 → 是以下情况之一吗?
      - Union types (T | null)
      - 空数组/对象
      - 函数参数
      - Public API
      └─ 是 → 需要注解
```

---

## 快速链接 | Quick Links

### 详细规则 | Detailed Rules
- [TypeScript Rules](./typescript-rules.md)
- [React Rules](./react-rules.md)
- [Code Quality Rules](./code-quality-rules.md)
- [PR Review Checklist](./pr-review-checklist.md)

### 学习资源 | Learning
- [Type Inference](../typescript/type-inference-best-practices.md)
- [useEffect Pitfalls](../react/useeffect-dependency-array-pitfalls.md)
- [UI-Behavior Sync](../react/ui-behavior-synchronization.md)
- [PR Examples](../best-practices/)

---

## 记住这些 | Remember These

1. **类型推断优先** - Let TypeScript infer
2. **完整依赖数组** - Complete useEffect deps
3. **UI 必须有行为** - No empty handlers
4. **删除未使用代码** - Remove unused code
5. **清理副作用** - Cleanup effects
6. **处理所有错误** - Handle all errors
7. **函数要简短** - Keep functions short
8. **命名要清晰** - Name things clearly
9. **避免重复** - DRY principle
10. **提前返回** - Early returns

---

**💡 Pro Tip**: 把这个页面设为浏览器首页或打印出来贴在显示器旁边！
