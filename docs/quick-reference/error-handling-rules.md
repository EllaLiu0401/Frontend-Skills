# Error Handling Rules

> 快速查找错误处理规则

## Async Errors

### ✅ DO: 总是捕获异步错误

```typescript
// ✅ Good: async/await with try-catch
async function fetchUserData(userId: string) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw new Error(`Unable to load user ${userId}`);
  }
}

// ✅ Good: Promise with .catch()
function fetchUserData(userId: string) {
  return fetch(`/api/users/${userId}`)
    .then(response => response.json())
    .catch(error => {
      console.error('Failed to fetch user:', error);
      throw new Error(`Unable to load user ${userId}`);
    });
}
```

### ❌ DON'T: 忽略错误

```typescript
// ❌ Bad: 没有错误处理
async function fetchUserData(userId: string) {
  const response = await fetch(`/api/users/${userId}`);
  return response.json(); // 如果失败怎么办？
}

// ❌ Bad: 空 catch
try {
  await fetchData();
} catch (error) {
  // 什么都不做
}
```

**Rule**: 所有异步操作必须有错误处理

---

## Error Boundaries

### ✅ DO: 使用 Error Boundary

```typescript
// ✅ Good: Error Boundary 组件
class ErrorBoundary extends React.Component<
  { children: ReactNode },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // 记录到错误追踪服务
    logErrorToService(error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// 使用
function App() {
  return (
    <ErrorBoundary>
      <UserProfile />
    </ErrorBoundary>
  );
}
```

### ✅ DO: 分层的 Error Boundaries

```typescript
// ✅ Good: 不同层级的边界
function App() {
  return (
    <ErrorBoundary fallback={<AppError />}>
      <Header />
      
      <ErrorBoundary fallback={<SidebarError />}>
        <Sidebar />
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<ContentError />}>
        <MainContent />
      </ErrorBoundary>
    </ErrorBoundary>
  );
}
```

**Rule**: 在关键组件边界使用 Error Boundary

---

## User Feedback

### ✅ DO: 提供清晰的错误反馈

```typescript
// ✅ Good: 具体的错误消息
function UserProfile({ userId }: Props) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .catch(err => {
        setError('Unable to load user profile. Please try again.');
        console.error(err);
      })
      .finally(() => setLoading(false));
  }, [userId]);
  
  if (loading) return <Spinner />;
  
  if (error) {
    return (
      <div className="error">
        <p>{error}</p>
        <button onClick={() => window.location.reload()}>
          Retry
        </button>
      </div>
    );
  }
  
  if (!user) return <div>User not found</div>;
  
  return <div>{user.name}</div>;
}
```

### ❌ DON'T: 静默失败

```typescript
// ❌ Bad: 错误时什么都不显示
function UserProfile({ userId }: Props) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .catch(() => {}); // 静默失败！
  }, [userId]);
  
  if (!user) return null; // 用户不知道发生了什么
  
  return <div>{user.name}</div>;
}
```

**Rule**: 错误必须对用户可见且可操作

---

## Error States

### ✅ DO: 区分不同的错误状态

```typescript
// ✅ Good: 详细的错误状态
type ErrorType = 'network' | 'not-found' | 'unauthorized' | 'server' | null;

function DataDisplay({ id }: Props) {
  const [data, setData] = useState(null);
  const [errorType, setErrorType] = useState<ErrorType>(null);
  
  useEffect(() => {
    fetchData(id)
      .then(setData)
      .catch(error => {
        if (error.status === 404) {
          setErrorType('not-found');
        } else if (error.status === 401) {
          setErrorType('unauthorized');
        } else if (error.status >= 500) {
          setErrorType('server');
        } else if (error.message === 'Network Error') {
          setErrorType('network');
        }
      });
  }, [id]);
  
  if (errorType === 'not-found') {
    return <NotFound />;
  }
  
  if (errorType === 'unauthorized') {
    return <LoginPrompt />;
  }
  
  if (errorType === 'network') {
    return <NetworkError onRetry={() => fetchData(id)} />;
  }
  
  if (errorType === 'server') {
    return <ServerError />;
  }
  
  return <div>{data}</div>;
}
```

### ❌ DON'T: 通用错误消息

```typescript
// ❌ Bad: 所有错误都一样处理
function DataDisplay({ id }: Props) {
  const [error, setError] = useState(false);
  
  if (error) {
    return <div>Error occurred</div>; // 不够具体
  }
  
  return <div>{/* ... */}</div>;
}
```

**Rule**: 根据错误类型提供不同的用户体验

---

## Retry Logic

### ✅ DO: 提供重试机制

```typescript
// ✅ Good: 带重试的数据获取
function useDataWithRetry<T>(
  fetchFn: () => Promise<T>,
  maxRetries = 3
) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [retryCount, setRetryCount] = useState(0);
  
  const fetch = useCallback(async () => {
    try {
      const result = await fetchFn();
      setData(result);
      setError(null);
    } catch (err) {
      if (retryCount < maxRetries) {
        setRetryCount(count => count + 1);
        // 指数退避
        setTimeout(fetch, Math.pow(2, retryCount) * 1000);
      } else {
        setError(err as Error);
      }
    }
  }, [fetchFn, retryCount, maxRetries]);
  
  useEffect(() => {
    fetch();
  }, [fetch]);
  
  return { data, error, retry: fetch, retryCount };
}

// 使用
function UserProfile({ userId }: Props) {
  const { data, error, retry, retryCount } = useDataWithRetry(
    () => fetchUser(userId)
  );
  
  if (error) {
    return (
      <div>
        <p>Failed to load user (attempt {retryCount}/3)</p>
        <button onClick={retry}>Retry</button>
      </div>
    );
  }
  
  return <div>{data?.name}</div>;
}
```

**Rule**: 网络错误应该允许用户重试

---

## Input Validation

### ✅ DO: 验证用户输入

```typescript
// ✅ Good: 前端验证
function validateEmail(email: string): string | null {
  if (!email) return 'Email is required';
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return 'Email format is invalid';
  }
  return null;
}

function EmailForm() {
  const [email, setEmail] = useState('');
  const [error, setError] = useState<string | null>(null);
  
  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    
    // 验证
    const validationError = validateEmail(email);
    if (validationError) {
      setError(validationError);
      return;
    }
    
    // 提交
    try {
      await submitEmail(email);
      toast.success('Email submitted!');
    } catch (err) {
      setError('Failed to submit email');
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={e => {
          setEmail(e.target.value);
          setError(null); // 清除错误
        }}
      />
      {error && <span className="error">{error}</span>}
      <button type="submit">Submit</button>
    </form>
  );
}
```

### ❌ DON'T: 只依赖后端验证

```typescript
// ❌ Bad: 没有前端验证
function EmailForm() {
  const [email, setEmail] = useState('');
  
  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    
    // 直接提交，没有验证
    try {
      await submitEmail(email);
    } catch (err) {
      // 后端返回错误才知道
      alert('Invalid email');
    }
  };
  
  return (/* ... */);
}
```

**Rule**: 前端验证改善用户体验，后端验证保证安全

---

## Error Logging

### ✅ DO: 记录关键错误

```typescript
// ✅ Good: 结构化的错误日志
function logError(
  error: Error,
  context: {
    component?: string;
    userId?: string;
    action?: string;
    metadata?: Record<string, unknown>;
  }
) {
  // 生产环境发送到错误追踪服务
  if (process.env.NODE_ENV === 'production') {
    errorTrackingService.log({
      message: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString(),
      ...context
    });
  }
  
  // 开发环境输出到控制台
  console.error('Error:', {
    error,
    context,
    timestamp: new Date().toISOString()
  });
}

// 使用
try {
  await saveUserProfile(userId, data);
} catch (error) {
  logError(error as Error, {
    component: 'UserProfile',
    userId,
    action: 'saveProfile',
    metadata: { dataSize: JSON.stringify(data).length }
  });
  
  toast.error('Failed to save profile');
}
```

### ✅ DO: 记录错误边界错误

```typescript
// ✅ Good: Error Boundary 记录
componentDidCatch(error: Error, errorInfo: ErrorInfo) {
  logError(error, {
    component: 'ErrorBoundary',
    componentStack: errorInfo.componentStack,
    metadata: {
      boundary: this.props.boundaryName || 'unknown'
    }
  });
}
```

**Rule**: 生产环境的错误必须被记录和追踪

---

## Toast Notifications

### ✅ DO: 使用 Toast 通知用户

```typescript
// ✅ Good: 不同类型的通知
import { toast } from 'react-hot-toast';

// 成功
async function saveData(data: Data) {
  try {
    await api.save(data);
    toast.success('Data saved successfully!');
  } catch (error) {
    toast.error('Failed to save data');
  }
}

// 加载
async function fetchData() {
  const promise = api.fetch();
  
  toast.promise(promise, {
    loading: 'Loading data...',
    success: 'Data loaded!',
    error: 'Failed to load data'
  });
  
  return promise;
}

// 警告
function validateForm() {
  if (!email) {
    toast.error('Email is required', {
      icon: '⚠️'
    });
    return false;
  }
  return true;
}
```

**Rule**: 使用 toast 提供即时反馈

---

## Network Errors

### ✅ DO: 区分网络错误类型

```typescript
// ✅ Good: 详细的错误处理
async function apiRequest<T>(url: string): Promise<T> {
  try {
    const response = await fetch(url);
    
    // HTTP 错误
    if (!response.ok) {
      if (response.status === 401) {
        throw new Error('UNAUTHORIZED');
      }
      if (response.status === 404) {
        throw new Error('NOT_FOUND');
      }
      if (response.status >= 500) {
        throw new Error('SERVER_ERROR');
      }
      throw new Error('REQUEST_FAILED');
    }
    
    return response.json();
  } catch (error) {
    // 网络错误
    if (error instanceof TypeError && error.message === 'Failed to fetch') {
      throw new Error('NETWORK_ERROR');
    }
    
    throw error;
  }
}

// 使用
try {
  const data = await apiRequest('/api/user');
} catch (error) {
  switch (error.message) {
    case 'NETWORK_ERROR':
      toast.error('No internet connection');
      break;
    case 'UNAUTHORIZED':
      redirectToLogin();
      break;
    case 'NOT_FOUND':
      show404Page();
      break;
    case 'SERVER_ERROR':
      toast.error('Server error. Please try again later.');
      break;
    default:
      toast.error('Request failed');
  }
}
```

**Rule**: 网络错误需要区分并提供不同的处理

---

## Form Validation

### ✅ DO: 实时验证反馈

```typescript
// ✅ Good: 实时验证
function useFormValidation<T extends Record<string, string>>(
  initialValues: T,
  validators: Record<keyof T, (value: string) => string | null>
) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Record<keyof T, string | null>>({} as any);
  const [touched, setTouched] = useState<Record<keyof T, boolean>>({} as any);
  
  const handleChange = (field: keyof T) => (value: string) => {
    setValues(prev => ({ ...prev, [field]: value }));
    
    // 只在字段被触碰后验证
    if (touched[field]) {
      const error = validators[field](value);
      setErrors(prev => ({ ...prev, [field]: error }));
    }
  };
  
  const handleBlur = (field: keyof T) => () => {
    setTouched(prev => ({ ...prev, [field]: true }));
    const error = validators[field](values[field]);
    setErrors(prev => ({ ...prev, [field]: error }));
  };
  
  const validate = () => {
    const newErrors = {} as Record<keyof T, string | null>;
    let isValid = true;
    
    for (const field in validators) {
      const error = validators[field](values[field]);
      newErrors[field] = error;
      if (error) isValid = false;
    }
    
    setErrors(newErrors);
    setTouched(
      Object.keys(validators).reduce((acc, key) => {
        acc[key as keyof T] = true;
        return acc;
      }, {} as Record<keyof T, boolean>)
    );
    
    return isValid;
  };
  
  return { values, errors, touched, handleChange, handleBlur, validate };
}
```

**Rule**: 表单验证应该实时且清晰

---

## Quick Checklist

错误处理检查：

- [ ] 所有异步操作有 try-catch 或 .catch()
- [ ] 使用了 Error Boundary
- [ ] 错误有清晰的用户反馈
- [ ] 区分了不同的错误类型
- [ ] 网络错误可以重试
- [ ] 用户输入有验证
- [ ] 关键错误被记录
- [ ] Toast 通知用于即时反馈
- [ ] 表单验证实时反馈

---

## Related

- [React Rules](./react-rules.md)
- [Code Quality Rules](./code-quality-rules.md)
- [Error Handling Docs](../error-handling/)
