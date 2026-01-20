# PR Review Checklist

> PR 审查完整检查清单

使用这个清单系统地审查每个 Pull Request，确保代码质量。

## 提交前自查 | Before Submitting PR

### 🔧 基础检查 | Basic Checks

- [ ] **Linter 通过**: 运行 `npm run lint` 或 `yarn lint`，无错误
- [ ] **类型检查通过**: 运行 `tsc --noEmit`，无类型错误
- [ ] **测试通过**: 所有单元测试和集成测试通过
- [ ] **构建成功**: 运行 `npm run build`，无构建错误
- [ ] **无调试代码**: 删除所有 `console.log`、`debugger`、`TODO` 注释
- [ ] **无注释代码**: 删除所有注释掉的代码块
- [ ] **Git diff 检查**: 查看 diff，确保没有意外的改动

### ✨ 功能检查 | Functionality

- [ ] **功能完整**: PR 描述的所有功能都已实现
- [ ] **交互可用**: 所有按钮、链接、表单都有工作的功能
- [ ] **边界情况**: 测试了空数据、错误状态、极限值
- [ ] **错误处理**: 所有异步操作都有错误处理
- [ ] **加载状态**: 异步操作显示加载指示器
- [ ] **用户反馈**: 操作成功/失败有明确提示
- [ ] **响应式**: 在不同屏幕尺寸下测试

### 📝 代码质量 | Code Quality

- [ ] **无未使用代码**: 删除未使用的 imports、变量、函数、组件
- [ ] **命名清晰**: 变量、函数、组件名字能表达意图
- [ ] **函数简短**: 函数 < 50 行，组件 < 100 行
- [ ] **无重复代码**: 提取了重复逻辑
- [ ] **注释适当**: 复杂逻辑有注释说明
- [ ] **类型安全**: 避免 `any`，正确处理 `null`/`undefined`
- [ ] **一致性**: 遵循项目代码风格

### ⚛️ React 特定 | React-Specific

- [ ] **useEffect 依赖**: 依赖数组完整且正确
- [ ] **清理副作用**: 清理了定时器、订阅、事件监听
- [ ] **避免不必要重渲染**: 正确使用 `useMemo`/`useCallback`/`React.memo`
- [ ] **状态管理**: 使用了合适的状态管理方式
- [ ] **组件拆分**: 大组件拆分成小组件
- [ ] **自定义 Hook**: 复用逻辑提取到 Hook

### 🎨 TypeScript 特定 | TypeScript-Specific

- [ ] **类型推断**: 让类型推断工作，避免冗余注解
- [ ] **避免 any**: 使用具体类型或 `unknown`
- [ ] **null 处理**: 正确处理可能为 null 的值
- [ ] **泛型合理**: 只在需要时使用泛型
- [ ] **接口定义**: Props 和数据结构有清晰的类型定义

### ⚡ 性能 | Performance

- [ ] **无性能回归**: 没有引入明显的性能问题
- [ ] **大列表优化**: 长列表使用虚拟化
- [ ] **图片优化**: 图片压缩、使用合适格式、懒加载
- [ ] **代码分割**: 适当使用动态导入
- [ ] **缓存策略**: API 数据有适当缓存

### 🧪 测试 | Testing

- [ ] **有测试覆盖**: 新功能有单元测试
- [ ] **测试通过**: 所有测试都通过
- [ ] **测试质量**: 测试了主要场景和边界情况
- [ ] **不依赖实现**: 测试行为，不是实现细节

### ♿ 可访问性 | Accessibility

- [ ] **键盘导航**: 可以用键盘操作所有交互元素
- [ ] **语义化 HTML**: 使用正确的 HTML 标签
- [ ] **ARIA 属性**: 需要时添加 ARIA 属性
- [ ] **颜色对比**: 文字和背景有足够对比度
- [ ] **屏幕阅读器**: 重要内容对屏幕阅读器友好

### 🔒 安全 | Security

- [ ] **输入验证**: 用户输入都经过验证
- [ ] **XSS 防护**: 避免直接插入用户内容到 HTML
- [ ] **敏感信息**: 没有提交密码、token 等敏感信息
- [ ] **依赖安全**: 没有引入有安全漏洞的依赖

---

## 审查他人 PR | Reviewing Others' PRs

### 第一步: 理解变更 | Understand Changes

- [ ] 阅读 PR 描述和相关 issue
- [ ] 理解要解决的问题
- [ ] 查看改动的文件和代码量
- [ ] 理解整体架构思路

### 第二步: 代码审查 | Code Review

#### 正确性 | Correctness
- [ ] 逻辑是否正确？
- [ ] 是否处理了边界情况？
- [ ] 是否有潜在的 bug？
- [ ] 错误处理是否充分？

#### 代码质量 | Code Quality
- [ ] 代码是否清晰易懂？
- [ ] 命名是否恰当？
- [ ] 是否有重复代码？
- [ ] 复杂度是否合理？

#### 架构设计 | Architecture
- [ ] 设计是否合理？
- [ ] 是否符合项目架构？
- [ ] 是否有更好的实现方式？
- [ ] 是否考虑了扩展性？

#### 性能影响 | Performance Impact
- [ ] 是否有性能问题？
- [ ] 是否有不必要的计算/渲染？
- [ ] 数据结构是否高效？
- [ ] 是否有潜在的内存泄漏？

#### 测试覆盖 | Test Coverage
- [ ] 是否有足够的测试？
- [ ] 测试是否覆盖了主要场景？
- [ ] 测试是否有意义？
- [ ] 是否需要更多测试？

### 第三步: 实际运行 | Run the Code

- [ ] Checkout 到 PR 分支
- [ ] 安装依赖 (如果有新依赖)
- [ ] 运行项目
- [ ] 手动测试功能
- [ ] 测试边界情况
- [ ] 检查控制台是否有错误

### 第四步: 提供反馈 | Provide Feedback

#### 好的反馈示例 | Good Feedback

✅ **具体且建设性**:
```
这个函数有点长(80行)，建议拆分成:
1. validateInput()
2. processData()
3. saveResult()

这样更易测试和维护。
```

✅ **解释原因**:
```
建议使用 useCallback 包装 handleClick，
因为它传给了 React.memo 包装的子组件，
不然每次父组件渲染都会导致子组件重渲染。
```

✅ **提供替代方案**:
```
这里用 for 循环有点复杂，可以考虑:
const result = items
  .filter(item => item.active)
  .map(item => item.value);
```

❌ **避免模糊反馈**:
```
这段代码不好
命名有问题
改一下
```

#### 反馈分类 | Categorize Feedback

标注评论的重要性：

- 🔴 **必须修改** (blocking): 严重问题，必须修复才能合并
- 🟡 **建议改进** (suggestion): 有更好的做法，但不是必须
- 🔵 **疑问** (question): 不理解的地方，需要解释
- 💡 **知识分享** (nitpick): 分享知识，不要求改动
- ✅ **赞同** (praise): 好的实现，值得肯定

### 第五步: 批准或请求修改 | Approve or Request Changes

- [ ] **Approve**: 代码质量好，没有问题
- [ ] **Comment**: 有小问题或建议，但可以合并
- [ ] **Request Changes**: 有必须修复的问题

---

## 常见问题检查 | Common Issues to Look For

### TypeScript 问题

```typescript
// ❌ 冗余的泛型
useState<number>(0)

// ❌ 使用 any
const data: any = response

// ❌ 没处理 null
user.name.toUpperCase() // user 可能是 null

// ❌ 不必要的类型断言
const value = data as string
```

### React 问题

```typescript
// ❌ 缺少依赖
useEffect(() => {
  fetchData(userId);
}, []); // userId 应该在依赖中

// ❌ 没有清理
useEffect(() => {
  const timer = setInterval(() => {}, 1000);
  // 忘记清理
}, []);

// ❌ 渲染中创建函数
<Child onClick={() => {}} />

// ❌ 直接修改状态
items.push(newItem);
setItems(items);
```

### 代码质量问题

```typescript
// ❌ 未使用的导入
import { unused } from 'lib';

// ❌ 注释掉的代码
// const oldCode = () => {};

// ❌ console.log
console.log('debug');

// ❌ Magic number
setTimeout(fn, 5000); // 5000 是什么？

// ❌ 函数太长
function bigFunction() {
  // 200 行代码...
}
```

### 性能问题

```typescript
// ❌ 不必要的重渲染
function Parent() {
  return <Child data={{ value: 1 }} />; // 每次新对象
}

// ❌ 没优化的大列表
{items.map(item => <Item key={item.id} />)} // 1000+ items

// ❌ 昂贵的计算没缓存
const sorted = items.sort(...); // 每次渲染都排序
```

---

## PR 审查时间建议 | Time Guidelines

- **小 PR** (< 100 行): 10-15 分钟
- **中等 PR** (100-500 行): 30-45 分钟
- **大 PR** (> 500 行): 1-2 小时，或建议拆分

**提示**: 如果 PR 太大，建议作者拆分成多个小 PR。

---

## 审查工具 | Review Tools

### 浏览器工具
- React DevTools: 检查组件层级和性能
- Redux DevTools: 检查状态变化
- Network Tab: 检查 API 调用
- Console: 检查错误和警告

### 命令行工具
```bash
# 类型检查
npm run type-check

# Lint
npm run lint

# 测试
npm run test

# 构建
npm run build

# 检查 bundle 大小
npm run analyze
```

### GitHub/GitLab 功能
- 逐文件审查
- 批量评论
- 建议改动
- 标记已审查文件

---

## 快速检查清单 | Quick Checklist

复制这个到你的 PR 审查评论中：

```markdown
## Review Checklist

### 基础
- [ ] Linter 通过
- [ ] 类型检查通过
- [ ] 测试通过

### 功能
- [ ] 功能完整可用
- [ ] 错误处理完善
- [ ] 边界情况已测试

### 代码质量
- [ ] 无未使用代码
- [ ] 命名清晰
- [ ] 无重复逻辑

### React/TypeScript
- [ ] useEffect 依赖正确
- [ ] 类型推断合理
- [ ] 无性能问题

### 其他
- [ ] 已实际运行测试
- [ ] 无安全问题
- [ ] 文档已更新(如需要)
```

---

## Related

- [TypeScript Rules](./typescript-rules.md)
- [React Rules](./react-rules.md)
- [Code Quality Rules](./code-quality-rules.md)
- [PR-0161 Example](../best-practices/pr-0161-dashboard-foundation.md)
