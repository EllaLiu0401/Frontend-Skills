# TypeScript Rules

> 快速查找 TypeScript 编码规则

## Type Inference

### ✅ DO: 让类型推断工作

```typescript
// ✅ Good
const [count, setCount] = useState(0);
const items = ['a', 'b', 'c'];
const double = (x: number) => x * 2;

// ❌ Bad: 冗余注解
const [count, setCount] = useState<number>(0);
const items: string[] = ['a', 'b', 'c'];
const double = (x: number): number => x * 2;
```

### ❌ DON'T: 在这些情况下依赖推断

```typescript
// ❌ Wrong: 需要显式类型
const [user, setUser] = useState(null); // 推断为 null

// ✅ Correct
const [user, setUser] = useState<User | null>(null);

// ❌ Wrong: 空数组
const [items, setItems] = useState([]);

// ✅ Correct
const [items, setItems] = useState<Item[]>([]);
```

**When to add explicit types:**
- Union types (e.g., `User | null`)
- Empty arrays/objects
- Function parameters (always)
- Public API boundaries (exports)

**Source**: [Type Inference Best Practices](../typescript/type-inference-best-practices.md)

---

## Type Errors

### Common Fixes

#### Error: "Type 'X' is not assignable to type 'Y'"

```typescript
// 问题：类型不匹配
interface User {
  id: number;
  name: string;
}

const user = { id: 1 }; // ❌ Missing 'name'

// 解决方案 1: 添加缺失字段
const user: User = { id: 1, name: 'Alice' };

// 解决方案 2: 使用 Partial
const user: Partial<User> = { id: 1 };
```

#### Error: "Object is possibly 'null' or 'undefined'"

```typescript
// ❌ Problem
function greet(user: User | null) {
  return user.name; // Error!
}

// ✅ Solution 1: Optional chaining
function greet(user: User | null) {
  return user?.name;
}

// ✅ Solution 2: Guard clause
function greet(user: User | null) {
  if (!user) return undefined;
  return user.name;
}

// ✅ Solution 3: Nullish coalescing
function greet(user: User | null) {
  return user?.name ?? 'Guest';
}
```

#### Error: "Property 'X' does not exist on type 'Y'"

```typescript
// ❌ Problem
const data: any = fetchData();
console.log(data.user.name); // 运行时可能出错

// ✅ Solution: 定义正确的类型
interface ApiResponse {
  user: {
    name: string;
  };
}

const data: ApiResponse = fetchData();
console.log(data.user.name);
```

---

## Avoid `any`

### ❌ DON'T: 使用 `any`

```typescript
// ❌ Bad
function process(data: any) {
  return data.value;
}
```

### ✅ DO: 使用更具体的类型

```typescript
// ✅ Option 1: 具体类型
interface Data {
  value: string;
}
function process(data: Data) {
  return data.value;
}

// ✅ Option 2: 泛型
function process<T extends { value: string }>(data: T) {
  return data.value;
}

// ✅ Option 3: unknown (如果真的不知道类型)
function process(data: unknown) {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return (data as { value: string }).value;
  }
}
```

---

## Generics

### 何时使用泛型 | When to Use Generics

```typescript
// ✅ Good: 需要类型灵活性
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// ❌ Bad: 不需要泛型
function useState<T>(initial: T) {
  return initial; // 直接返回 initial，不需要泛型
}
```

### 何时不用泛型 | When NOT to Use Generics

```typescript
// ❌ Bad: 冗余泛型
useState<number>(0);
useState<string>('');

// ✅ Good: 推断
useState(0);
useState('');
```

**Rule**: 只在需要类型参数化时使用泛型

---

## Interface vs Type

### ✅ DO: 优先使用 Interface

```typescript
// ✅ Good: 用于对象形状
interface User {
  id: number;
  name: string;
}

// ✅ Good: 可以扩展
interface Admin extends User {
  permissions: string[];
}
```

### Use Type for:

```typescript
// ✅ Unions
type Status = 'idle' | 'loading' | 'success' | 'error';

// ✅ Intersections
type UserWithTimestamp = User & { createdAt: Date };

// ✅ Mapped types
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

**Rule**: Interface for objects, Type for everything else

---

## Null & Undefined

### ✅ DO: 明确处理 null/undefined

```typescript
// ✅ Good: 明确的类型
function getUser(id: string): User | null {
  const user = database.find(id);
  return user ?? null;
}

// ✅ Good: 使用时检查
const user = getUser('123');
if (user) {
  console.log(user.name);
}
```

### ❌ DON'T: 混淆 null 和 undefined

```typescript
// ❌ Bad: 不一致
function getUser(id?: string): User | undefined | null {
  // 混乱！
}

// ✅ Good: 选一个
function getUser(id?: string): User | null {
  // 清晰
}
```

**Rule**: 在代码库中统一使用 `null` 或 `undefined`，不要混用

---

## Type Guards

### 创建类型守卫 | Creating Type Guards

```typescript
// ✅ Good: 使用类型谓词
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// 使用
function process(data: unknown) {
  if (isUser(data)) {
    console.log(data.name); // TypeScript 知道这是 User
  }
}
```

### 内置类型守卫 | Built-in Type Guards

```typescript
// typeof
if (typeof value === 'string') {
  value.toUpperCase();
}

// instanceof
if (error instanceof Error) {
  console.log(error.message);
}

// in operator
if ('name' in user) {
  console.log(user.name);
}

// Array.isArray
if (Array.isArray(value)) {
  value.map(x => x);
}
```

---

## Common Mistakes

### ❌ 错误 1: 过度注解

```typescript
// ❌ Bad
const user: User = {
  id: 1,
  name: 'Alice',
};
const users: User[] = [user];
const count: number = users.length;

// ✅ Good
const user = {
  id: 1,
  name: 'Alice',
} as User; // 或者不加类型，让推断工作
const users = [user];
const count = users.length;
```

### ❌ 错误 2: 使用 any 逃避问题

```typescript
// ❌ Bad
const data = JSON.parse(response) as any;

// ✅ Good
interface ApiResponse {
  // 定义实际结构
}
const data = JSON.parse(response) as ApiResponse;
```

### ❌ 错误 3: 忽略 null/undefined

```typescript
// ❌ Bad
function process(user: User) {
  // 假设 user 总是存在
  return user.name.toUpperCase();
}

// ✅ Good
function process(user: User | null) {
  return user?.name.toUpperCase() ?? 'Unknown';
}
```

---

## Utility Types

### 常用工具类型 | Common Utility Types

```typescript
// Partial - 所有属性可选
type PartialUser = Partial<User>;
// { id?: number; name?: string; }

// Required - 所有属性必需
type RequiredUser = Required<PartialUser>;

// Pick - 选择部分属性
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit - 排除部分属性
type UserWithoutId = Omit<User, 'id'>;

// Record - 创建对象类型
type UserMap = Record<string, User>;
// { [key: string]: User }

// ReturnType - 获取函数返回类型
type Result = ReturnType<typeof fetchUser>;
```

---

## Configuration

### tsconfig.json 必备配置

```json
{
  "compilerOptions": {
    "strict": true,                  // ✅ 必须
    "noImplicitAny": true,          // ✅ 必须
    "strictNullChecks": true,       // ✅ 必须
    "strictFunctionTypes": true,    // ✅ 必须
    "noUnusedLocals": true,         // ✅ 推荐
    "noUnusedParameters": true,     // ✅ 推荐
    "noImplicitReturns": true,      // ✅ 推荐
    "noFallthroughCasesInSwitch": true  // ✅ 推荐
  }
}
```

---

## Quick Checklist

写 TypeScript 代码时检查：

- [ ] 是否让类型推断工作？
- [ ] 是否避免了 `any`？
- [ ] 是否处理了 `null`/`undefined`？
- [ ] 泛型参数是否必要？
- [ ] 类型守卫是否正确？
- [ ] 是否使用了合适的工具类型？
- [ ] 函数参数是否有类型？
- [ ] 导出的 API 是否有明确类型？

---

## Related

- [Type Inference Best Practices](../typescript/type-inference-best-practices.md) - 详细说明
- [PR-0161](../best-practices/pr-0161-dashboard-foundation.md) - 实际案例
