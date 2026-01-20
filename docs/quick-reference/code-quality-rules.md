# Code Quality Rules

> 快速查找代码质量规则

## Unused Code

### ✅ DO: 删除未使用的代码

```typescript
// ❌ Bad: 未使用的 import
import { useState, useEffect, useCallback } from 'react';
import { sortBy } from 'lodash';
// 只用了 useState

function Component() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}

// ✅ Good: 只导入需要的
import { useState } from 'react';

function Component() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}
```

```typescript
// ❌ Bad: 定义了但从不使用
export const UnusedComponent = () => {
  return <div>Never rendered</div>;
};

const unusedVariable = 'hello';
const unusedFunction = () => {};

// ✅ Good: 删除或使用
// 完全删除，或者实际使用它们
```

**Rule**: 没有使用的代码必须删除

**Source**: [PR-0161](../best-practices/pr-0161-dashboard-foundation.md)

---

## Naming

### ✅ DO: 清晰的命名

```typescript
// ✅ Good: 描述性命名
const isUserLoggedIn = checkAuthStatus();
const filteredActiveUsers = users.filter(u => u.isActive);
function calculateTotalPrice(items: Item[]): number { }

// ❌ Bad: 模糊命名
const flag = checkAuthStatus();
const arr = users.filter(u => u.isActive);
function calc(items: Item[]): number { }
```

### Naming Conventions

```typescript
// Variables & Functions: camelCase
const userName = 'Alice';
function getUserById(id: string) { }

// Components & Classes: PascalCase
function UserProfile() { }
class ApiService { }

// Constants: UPPER_SNAKE_CASE
const MAX_RETRY_COUNT = 3;
const API_BASE_URL = 'https://api.example.com';

// Private: prefix with _
class Service {
  private _cache = new Map();
  private _fetchData() { }
}

// Boolean: prefix with is/has/should/can
const isLoading = true;
const hasError = false;
const shouldRender = true;
const canEdit = false;
```

**Rule**: 名字要能表达意图，不需要注释就能理解

---

## Single Responsibility

### ✅ DO: 一个函数做一件事

```typescript
// ❌ Bad: 做太多事情
function processUserData(user: User) {
  // 验证
  if (!user.email) throw new Error('No email');
  
  // 转换
  const normalized = user.email.toLowerCase();
  
  // 保存
  database.save({ ...user, email: normalized });
  
  // 发送邮件
  emailService.sendWelcome(user.email);
  
  // 记录日志
  logger.info('User processed', user.id);
}

// ✅ Good: 拆分职责
function validateUser(user: User) {
  if (!user.email) throw new Error('No email');
}

function normalizeEmail(email: string): string {
  return email.toLowerCase();
}

function saveUser(user: User) {
  database.save(user);
}

function sendWelcomeEmail(email: string) {
  emailService.sendWelcome(email);
}

function processUser(user: User) {
  validateUser(user);
  const normalizedUser = { ...user, email: normalizeEmail(user.email) };
  saveUser(normalizedUser);
  sendWelcomeEmail(normalizedUser.email);
  logger.info('User processed', user.id);
}
```

**Rule**: 函数应该只有一个改变的理由

---

## Function Size

### ✅ DO: 保持函数简短

```typescript
// ❌ Bad: 函数太长
function processOrder(order: Order) {
  // 100 行代码...
  // 太多逻辑、太多嵌套
}

// ✅ Good: 拆分成小函数
function validateOrder(order: Order) { }
function calculateTotal(order: Order): number { }
function applyDiscount(total: number, code: string): number { }
function createInvoice(order: Order, total: number): Invoice { }

function processOrder(order: Order) {
  validateOrder(order);
  const total = calculateTotal(order);
  const finalTotal = applyDiscount(total, order.discountCode);
  return createInvoice(order, finalTotal);
}
```

**Rule**: 函数 < 50 行，理想情况 < 20 行

---

## Early Returns

### ✅ DO: 使用提前返回

```typescript
// ❌ Bad: 嵌套太深
function processUser(user: User | null) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        if (user.email) {
          return sendEmail(user.email);
        }
      }
    }
  }
  return null;
}

// ✅ Good: 提前返回
function processUser(user: User | null) {
  if (!user) return null;
  if (!user.isActive) return null;
  if (!user.hasPermission) return null;
  if (!user.email) return null;
  
  return sendEmail(user.email);
}
```

**Rule**: 优先处理边界情况，减少嵌套

---

## DRY (Don't Repeat Yourself)

### ✅ DO: 提取重复代码

```typescript
// ❌ Bad: 重复逻辑
function formatUserName(user: User) {
  return user.firstName + ' ' + user.lastName;
}

function displayUser(user: User) {
  const name = user.firstName + ' ' + user.lastName;
  console.log(name);
}

function saveUser(user: User) {
  const fullName = user.firstName + ' ' + user.lastName;
  database.save({ ...user, fullName });
}

// ✅ Good: 提取到一个函数
function getFullName(user: User): string {
  return `${user.firstName} ${user.lastName}`;
}

function formatUserName(user: User) {
  return getFullName(user);
}

function displayUser(user: User) {
  console.log(getFullName(user));
}

function saveUser(user: User) {
  database.save({ ...user, fullName: getFullName(user) });
}
```

**Rule**: 同样的逻辑不要写两次

---

## Magic Numbers

### ✅ DO: 使用命名常量

```typescript
// ❌ Bad: Magic numbers
function retryRequest() {
  for (let i = 0; i < 3; i++) { // 3 是什么？
    setTimeout(() => {
      fetch(url);
    }, 1000 * i); // 1000 是什么？
  }
}

if (user.age > 18) { // 18 是什么意思？
  allowAccess();
}

// ✅ Good: 命名常量
const MAX_RETRY_ATTEMPTS = 3;
const RETRY_DELAY_MS = 1000;
const LEGAL_AGE = 18;

function retryRequest() {
  for (let i = 0; i < MAX_RETRY_ATTEMPTS; i++) {
    setTimeout(() => {
      fetch(url);
    }, RETRY_DELAY_MS * i);
  }
}

if (user.age >= LEGAL_AGE) {
  allowAccess();
}
```

**Rule**: 给数字和字符串常量起名字

---

## Comments

### ✅ DO: 解释 "为什么"，不是 "是什么"

```typescript
// ❌ Bad: 解释代码在做什么（显而易见）
// 检查用户是否登录
if (user.isLoggedIn) {
  // 显示欢迎消息
  showWelcomeMessage();
}

// ✅ Good: 解释为什么这样做
// 需要延迟显示，因为动画需要 300ms 完成
setTimeout(showWelcomeMessage, 300);

// 使用 WeakMap 而不是 Map 避免内存泄漏
// 当组件卸载时，引用会自动清理
const cache = new WeakMap();
```

### ✅ DO: 复杂逻辑加注释

```typescript
// ✅ Good: 复杂算法需要注释
function calculateDiscount(items: Item[]): number {
  // 应用分层折扣策略:
  // - 前 10 件: 无折扣
  // - 11-50 件: 10% 折扣
  // - 51+ 件: 20% 折扣
  // 业务需求: TICKET-1234
  
  let discount = 0;
  const count = items.length;
  
  if (count > 50) {
    discount = 0.2;
  } else if (count > 10) {
    discount = 0.1;
  }
  
  return discount;
}
```

### ❌ DON'T: 注释掉的代码

```typescript
// ❌ Bad: 注释掉的代码
function processData(data: Data) {
  // const oldWay = data.map(x => x * 2);
  // return oldWay.filter(x => x > 10);
  
  return data
    .map(x => x * 2)
    .filter(x => x > 10);
}

// ✅ Good: 删除注释掉的代码
function processData(data: Data) {
  return data
    .map(x => x * 2)
    .filter(x => x > 10);
}
```

**Rule**: 代码应该自解释，只在必要时注释

---

## Error Messages

### ✅ DO: 清晰的错误消息

```typescript
// ❌ Bad: 模糊的错误
throw new Error('Invalid input');
throw new Error('Error');
throw new Error('Failed');

// ✅ Good: 具体的错误信息
throw new Error('Email format is invalid: missing @ symbol');
throw new Error('User ID must be a positive integer, got: ' + userId);
throw new Error('Failed to fetch user data: network timeout after 5s');
```

### ✅ DO: 包含上下文

```typescript
// ✅ Good
try {
  await processOrder(orderId);
} catch (error) {
  throw new Error(
    `Failed to process order ${orderId}: ${error.message}`
  );
}
```

**Rule**: 错误消息要帮助调试，包含相关信息

---

## File Organization

### ✅ DO: 逻辑分组

```typescript
// ✅ Good: 分组导入
// React
import { useState, useEffect } from 'react';

// Third-party
import { format } from 'date-fns';
import axios from 'axios';

// Internal
import { Button } from '@/components/Button';
import { useAuth } from '@/hooks/useAuth';
import { formatCurrency } from '@/utils/format';

// Types
import type { User, Order } from '@/types';

// Constants
const MAX_ITEMS = 100;
const DEFAULT_PAGE_SIZE = 20;

// Component
export function OrderList() {
  // ...
}
```

### ✅ DO: 一个文件一个组件

```typescript
// ❌ Bad: 多个组件在一个文件
// components.tsx
export function Button() { }
export function Input() { }
export function Form() { }

// ✅ Good: 每个组件一个文件
// Button.tsx
export function Button() { }

// Input.tsx
export function Input() { }

// Form.tsx
export function Form() { }
```

**Rule**: 文件结构清晰，便于查找和维护

---

## Complexity

### ✅ DO: 减少圈复杂度

```typescript
// ❌ Bad: 高复杂度
function getStatus(user: User) {
  if (user.isActive) {
    if (user.isPremium) {
      if (user.hasSubscription) {
        return 'premium-active';
      } else {
        return 'premium-inactive';
      }
    } else {
      if (user.isTrial) {
        return 'trial';
      } else {
        return 'basic';
      }
    }
  } else {
    return 'inactive';
  }
}

// ✅ Good: 降低复杂度
function getStatus(user: User) {
  if (!user.isActive) return 'inactive';
  if (user.isPremium) {
    return user.hasSubscription ? 'premium-active' : 'premium-inactive';
  }
  return user.isTrial ? 'trial' : 'basic';
}

// ✅ Better: 使用查找表
const STATUS_MAP = {
  'active-premium-subscribed': 'premium-active',
  'active-premium-not-subscribed': 'premium-inactive',
  'active-not-premium-trial': 'trial',
  'active-not-premium-not-trial': 'basic',
  'not-active': 'inactive'
};

function getStatus(user: User) {
  const key = [
    user.isActive ? 'active' : 'not-active',
    user.isPremium ? 'premium' : 'not-premium',
    user.hasSubscription ? 'subscribed' : 'not-subscribed'
  ].join('-');
  
  return STATUS_MAP[key] || 'unknown';
}
```

**Rule**: 圈复杂度 < 10，理想 < 5

---

## Quick Checklist

写代码时检查：

- [ ] 是否删除了未使用的代码？
- [ ] 命名是否清晰？
- [ ] 函数是否足够简短？
- [ ] 是否使用了提前返回？
- [ ] 是否有重复代码？
- [ ] Magic numbers 是否命名了？
- [ ] 注释是否必要且清晰？
- [ ] 错误消息是否具体？
- [ ] 文件组织是否合理？
- [ ] 复杂度是否可接受？

---

## Related

- [PR-0161](../best-practices/pr-0161-dashboard-foundation.md)
- [Contributing Guide](../../CONTRIBUTING.md)
