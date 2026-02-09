# React Rules

> 快速查找 React 编码规则

## useEffect Dependencies

### ✅ DO: 包含所有依赖

```typescript
// ✅ Good
useEffect(() => {
  fetchData(userId);
}, [userId]); // userId 被使用了，所以必须在依赖数组中

// ✅ Good
const handleClick = useCallback(() => {
  console.log(count);
}, [count]); // count 被使用了

useEffect(() => {
  element.addEventListener("click", handleClick);
  return () => element.removeEventListener("click", handleClick);
}, [handleClick]); // handleClick 被使用了
```

### ❌ DON'T: 遗漏依赖

```typescript
// ❌ Bad: userId 应该在依赖中
useEffect(() => {
  fetchData(userId);
}, []); // Bug! userId 变化时不会重新获取

// ❌ Bad: 禁用 lint 规则
useEffect(() => {
  fetchData(userId);
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []); // 不要这样做！
```

**Rule**: 依赖数组必须包含 effect 中使用的所有外部值

**Source**: [useEffect Dependency Array Pitfalls](../react/useeffect-dependency-array-pitfalls.md)

---

## Cleanup Effects

### ✅ DO: 清理副作用

```typescript
// ✅ Good: 清理定时器
useEffect(() => {
  const timer = setInterval(() => {
    console.log("tick");
  }, 1000);

  return () => clearInterval(timer);
}, []);

// ✅ Good: 清理事件监听
useEffect(() => {
  const handler = () => console.log("resize");
  window.addEventListener("resize", handler);

  return () => window.removeEventListener("resize", handler);
}, []);

// ✅ Good: 取消请求
useEffect(() => {
  const abortController = new AbortController();

  fetch("/api/data", { signal: abortController.signal }).then(handleData);

  return () => abortController.abort();
}, []);
```

### ❌ DON'T: 忘记清理

```typescript
// ❌ Bad: 定时器泄漏
useEffect(() => {
  setInterval(() => {
    console.log("tick");
  }, 1000);
  // 忘记清理！组件卸载后定时器还在运行
}, []);

// ❌ Bad: 事件监听泄漏
useEffect(() => {
  window.addEventListener("resize", handleResize);
  // 忘记清理！
}, []);
```

**Rule**: 所有订阅、定时器、事件监听都必须清理

---

## UI-Behavior Sync

### ✅ DO: 确保交互元素有功能

```typescript
// ✅ Good: 按钮有实际功能
<button onClick={handleExport}>
  Export Report
</button>

// ✅ Good: 功能未完成时隐藏
{exportEnabled && (
  <button onClick={handleExport}>
    Export Report
  </button>
)}

// ✅ Good: 禁用时有说明
<button disabled={!hasData}>
  Export Report
</button>
{!hasData && <p>Add data to enable export</p>}
```

### ❌ DON'T: 发布无功能的UI

```typescript
// ❌ Bad: 空处理函数
<button onClick={() => {}}>
  Export Report
</button>

// ❌ Bad: TODO
<button onClick={() => console.log('TODO')}>
  Export Report
</button>

// ❌ Bad: 死链接
<Link to="/settings">Settings</Link>
// 但 /settings 路由不存在
```

**Rule**: 用户能看到并点击的元素，必须有工作的功能

**Source**: [UI-Behavior Synchronization](../react/ui-behavior-synchronization.md)

---

## Performance Optimization

### useMemo

```typescript
// ✅ Good: 昂贵计算使用 useMemo
const sortedItems = useMemo(() => {
  return items.sort((a, b) => a.value - b.value);
}, [items]);

// ❌ Bad: 简单计算不需要
const doubled = useMemo(() => count * 2, [count]); // 过度优化

// ✅ Good: 复杂对象使用 useMemo
const config = useMemo(
  () => ({
    theme: "dark",
    size: "large",
    options: complexCalculation(),
  }),
  [
    /* deps */
  ],
);
```

**When to use useMemo:**

- 昂贵的计算（循环、排序、过滤大数组）
- 创建的对象会传给优化过的子组件
- 依赖数组变化不频繁

### useCallback

```typescript
// ✅ Good: 传给子组件的回调
const handleClick = useCallback(() => {
  doSomething(value);
}, [value]);

<OptimizedChild onClick={handleClick} />

// ❌ Bad: 不传给子组件，不需要
const handleClick = useCallback(() => {
  console.log('click');
}, []); // 没必要

// 只在本组件用
<button onClick={handleClick}>Click</button>
```

**When to use useCallback:**

- 传递给使用 `React.memo` 的子组件
- 作为 useEffect 的依赖
- 传递给第三方库（性能敏感）

### React.memo

```typescript
// ✅ Good: 昂贵渲染的纯组件
const ExpensiveComponent = React.memo(({ data }) => {
  // 复杂渲染逻辑
  return <div>{/* ... */}</div>;
});

// ❌ Bad: 简单组件不需要
const SimpleText = React.memo(({ text }) => {
  return <span>{text}</span>;
}); // 过度优化
```

**When to use React.memo:**

- 组件渲染成本高
- 相同 props 频繁重渲染
- 组件在列表中

---

## Component Structure

### ✅ DO: 保持组件简短

```typescript
// ✅ Good: 单一职责
function UserProfile({ userId }: Props) {
  const user = useUser(userId);

  if (!user) return <Loading />;

  return (
    <div>
      <UserAvatar user={user} />
      <UserInfo user={user} />
      <UserActions user={user} />
    </div>
  );
}

// ❌ Bad: 组件太大
function Dashboard() {
  // 200 行代码...
  // 太多状态、太多逻辑
  return (
    <div>
      {/* 100 行 JSX... */}
    </div>
  );
}
```

**Rule**: 组件 < 100 行，拆分更小的组件

### ✅ DO: 提取自定义 Hook

```typescript
// ✅ Good: 逻辑提取到 hook
function useUserData(userId: string) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .finally(() => setLoading(false));
  }, [userId]);

  return { user, loading };
}

function UserProfile({ userId }: Props) {
  const { user, loading } = useUserData(userId);
  // 组件只关注渲染
}
```

**When to extract hook:**

- 相同逻辑在多个组件使用
- 组件内逻辑超过 20 行
- 状态管理逻辑复杂

---

## State Management

### ✅ DO: Pass values directly to avoid race conditions

```typescript
// ❌ Bad: Uses stale state
const [date, setDate] = useState(new Date());

const handlePreset = () => {
  const newDate = getPresetDate();
  setDate(newDate); // Async!
  fetchData(); // Uses OLD date!
};

// ✅ Good: Pass new value directly
const handlePreset = () => {
  const newDate = getPresetDate();
  setDate(newDate);
  fetchData(newDate); // Uses NEW date
};

// ✅ Good: Accept override parameters
const fetchData = (overrideDate?: Date) => {
  const finalDate = overrideDate ?? date;
  api.get({ date: finalDate });
};
```

**Rule**: React state updates are async - never rely on state immediately after setting it

**Source**: [Async State Race Conditions](../state-management/async-state-race-conditions.md)

### ✅ DO: 选择合适的状态类型

```typescript
// ✅ Good: 本地状态
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// ✅ Good: 状态提升
function Parent() {
  const [selectedId, setSelectedId] = useState(null);
  return (
    <>
      <List selectedId={selectedId} onSelect={setSelectedId} />
      <Detail id={selectedId} />
    </>
  );
}

// ✅ Good: Context for theme/auth
const ThemeContext = createContext();
```

### ❌ DON'T: 过度使用 Context

```typescript
// ❌ Bad: 所有状态都用 Context
const AppContext = createContext();
// { user, theme, data, loading, error, ... } // 太多了！

// ✅ Good: 拆分 Context
const UserContext = createContext();
const ThemeContext = createContext();
const DataContext = createContext();
```

**Rule**: 优先本地状态 > 状态提升 > Context > 状态管理库

---

## Props Management

### ✅ DO: 解构 Props

```typescript
// ✅ Good
function UserCard({ name, email, avatar }: Props) {
  return <div>{name}</div>;
}

// ❌ Bad
function UserCard(props) {
  return <div>{props.name}</div>;
}
```

### ✅ DO: 使用默认值

```typescript
// ✅ Good
function Button({
  variant = 'primary',
  size = 'medium',
  children
}: Props) {
  return <button className={`btn-${variant}-${size}`}>{children}</button>;
}
```

### ❌ DON'T: Props Drilling

```typescript
// ❌ Bad: Props 传递太多层
<App>
  <Layout user={user}>
    <Sidebar user={user}>
      <Menu user={user}>
        <UserItem user={user} />
      </Menu>
    </Sidebar>
  </Layout>
</App>

// ✅ Good: 使用 Context
const UserContext = createContext();

<App>
  <UserContext.Provider value={user}>
    <Layout>
      <Sidebar>
        <Menu>
          <UserItem /> {/* 内部使用 useContext(UserContext) */}
        </Menu>
      </Sidebar>
    </Layout>
  </UserContext.Provider>
</App>
```

---

## Error Handling

### ✅ DO: 使用 Error Boundary

```typescript
// ✅ Good
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    logError(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }
    return this.props.children;
  }
}

// 使用
<ErrorBoundary>
  <App />
</ErrorBoundary>
```

### ✅ DO: 处理异步错误

```typescript
// ✅ Good
function UserProfile({ userId }: Props) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .catch(setError);
  }, [userId]);

  if (error) return <ErrorMessage error={error} />;
  if (!user) return <Loading />;
  return <div>{user.name}</div>;
}
```

---

## Common Mistakes

### ❌ 错误 1: 在渲染中创建函数/对象

```typescript
// ❌ Bad: 每次渲染创建新函数
function Parent() {
  return <Child onUpdate={() => console.log('update')} />;
  // 每次渲染 Child 都会收到新的 onUpdate
}

// ✅ Good
function Parent() {
  const handleUpdate = useCallback(() => {
    console.log('update');
  }, []);
  return <Child onUpdate={handleUpdate} />;
}

// ❌ Bad: 每次渲染创建新对象
function Parent() {
  return <Child config={{ theme: 'dark' }} />;
}

// ✅ Good
function Parent() {
  const config = useMemo(() => ({ theme: 'dark' }), []);
  return <Child config={config} />;
}
```

### ❌ 错误 2: 直接修改状态

```typescript
// ❌ Bad
const [items, setItems] = useState([]);
items.push(newItem); // 不要直接修改！
setItems(items);

// ✅ Good
setItems([...items, newItem]);

// ❌ Bad
const [user, setUser] = useState({ name: "Alice" });
user.name = "Bob"; // 不要直接修改！
setUser(user);

// ✅ Good
setUser({ ...user, name: "Bob" });
```

### ❌ 错误 3: 过度使用 useEffect

```typescript
// ❌ Bad: 不需要 effect
function Component({ value }: Props) {
  const [doubled, setDoubled] = useState(value * 2);

  useEffect(() => {
    setDoubled(value * 2);
  }, [value]);

  return <div>{doubled}</div>;
}

// ✅ Good: 直接计算
function Component({ value }: Props) {
  const doubled = value * 2;
  return <div>{doubled}</div>;
}
```

---

## Quick Checklist

写 React 代码时检查：

- [ ] 是否避免了 async state 竞态条件？
- [ ] useEffect 依赖数组是否完整？
- [ ] 是否清理了副作用？
- [ ] 交互元素是否有功能？
- [ ] 是否避免了不必要的重渲染？
- [ ] 组件是否足够简短？
- [ ] 是否正确使用了 useMemo/useCallback？
- [ ] 是否有错误处理？
- [ ] Props 是否解构了？

---

## Related

- [Async State Race Conditions](../state-management/async-state-race-conditions.md)
- [useEffect Dependency Array Pitfalls](../react/useeffect-dependency-array-pitfalls.md)
- [UI-Behavior Synchronization](../react/ui-behavior-synchronization.md)
- [PR-0161](../best-practices/pr-0161-dashboard-foundation.md)
