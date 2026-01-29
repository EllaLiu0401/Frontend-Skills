# Quick Reference - Coding Rules

> **快速查找编码规则 | Quick rule lookup for daily coding**

这是一个按照问题类型组织的规则速查手册。在写代码或review PR时，快速找到你需要的规则。

This is a quick reference organized by problem types. Find the rules you need when coding or reviewing PRs.

## 📋 目录 | Table of Contents

### 🚀 快速开始 | Quick Start

- **[Cheatsheet](./cheatsheet.md)** - 一页纸速查手册，打印或保存！
- **[i18n Checklist](./i18n-checklist.md)** - Internationalization quick reference and audit commands
- **[Authentication Checklist](./authentication-checklist.md)** - Security checklist for authentication implementation

### 📚 完整规则 | Complete Rules

- [Common Mistakes Checklist](#common-mistakes-checklist) - 常见错误检查清单
- [By Problem Type](#by-problem-type) - 按问题类型查找
- [By Technology](#by-technology) - 按技术栈查找
- [Code Review Checklist](#code-review-checklist) - PR审查清单

---

## Common Mistakes Checklist

在提交PR前，检查这些常见问题：

### ⚡ 性能问题 | Performance

- [ ] 是否有不必要的重渲染？
- [ ] 是否正确使用了 `useMemo` / `useCallback`？
- [ ] 大列表是否使用了虚拟化？
- [ ] 是否有内存泄漏（未清理的订阅、定时器）？

**详见**: [Performance Rules](./performance-rules.md)

### 🔧 TypeScript 问题 | TypeScript

- [ ] 是否过度使用显式类型注解？（应该让类型推断工作）
- [ ] 是否正确处理了 `null` / `undefined`？
- [ ] 泛型参数是否必要？
- [ ] 是否使用了 `any`（应该避免）？

**详见**: [TypeScript Rules](./typescript-rules.md)

### ⚛️ React 问题 | React

- [ ] `useEffect` 依赖数组是否正确？
- [ ] 所有交互元素是否有实际功能？
- [ ] 是否有未使用的组件或props？
- [ ] 组件是否过大（应该拆分）？

**详见**: [React Rules](./react-rules.md)

### 🎨 代码质量 | Code Quality

- [ ] 是否有未使用的代码（imports、变量、函数）？
- [ ] 命名是否清晰且符合规范？
- [ ] 是否有重复代码可以提取？
- [ ] 是否添加了必要的注释？

**详见**: [Code Quality Rules](./code-quality-rules.md)

### 🌍 国际化 | Internationalization

- [ ] 是否有硬编码的用户可见文本？
- [ ] aria-label 是否使用了 i18n？
- [ ] 时间/日期格式是否国际化？
- [ ] 是否更新了所有语言文件？

**详见**: [i18n Checklist](./i18n-checklist.md)

### 🐛 错误处理 | Error Handling

- [ ] 异步操作是否有错误处理？
- [ ] 是否有用户友好的错误提示？
- [ ] 是否有错误边界（Error Boundary）？
- [ ] 是否记录了关键错误？

**详见**: [Error Handling Rules](./error-handling-rules.md)

### 🔐 安全与认证 | Security & Authentication

- [ ] 是否有服务端认证检查？
- [ ] 是否依赖客户端认证（不安全）？
- [ ] 认证重定向是否正确？
- [ ] 是否测试了未认证访问情况？

**详见**: [Authentication Checklist](./authentication-checklist.md)

---

## By Problem Type

### 🚨 我遇到这些问题... | When I see...

#### "组件一直重新渲染" | Component keeps re-rendering

→ [React Performance Rules](./react-rules.md#performance-optimization)

- Check `useMemo` / `useCallback` usage
- Check if creating new objects/functions in render
- Check parent component re-renders

#### "TypeScript 报类型错误" | TypeScript type errors

→ [TypeScript Rules](./typescript-rules.md#type-errors)

- Check if inference can handle it
- Check null/undefined handling
- Check union types

#### "useEffect 行为不符合预期" | useEffect not working as expected

→ [React Rules](./react-rules.md#useeffect-rules)

- Check dependency array
- Check cleanup function
- Check execution timing

#### "点击按钮没反应" | Button click doesn't work

→ [React Rules](./react-rules.md#ui-behavior-sync)

- Check event handler exists
- Check event handler implementation
- Check if disabled correctly

#### "代码太复杂难以维护" | Code is too complex

→ [Code Quality Rules](./code-quality-rules.md#complexity)

- Check function size
- Check nesting levels
- Check abstraction opportunities

#### "API 调用失败但没提示" | API fails without feedback

→ [Error Handling Rules](./error-handling-rules.md#api-errors)

- Check error boundaries
- Check error state handling
- Check user feedback

---

## By Technology

### TypeScript

**[Complete TypeScript Rules →](./typescript-rules.md)**

Quick rules:

1. 让类型推断工作，避免冗余注解
2. 优先使用 `interface` 定义对象类型
3. 使用 `strict: true` 配置
4. 避免使用 `any`，使用 `unknown` 代替

**Source**: [Type Inference Best Practices](../typescript/type-inference-best-practices.md)

### React

**[Complete React Rules →](./react-rules.md)**

Quick rules:

1. `useEffect` 依赖数组必须完整
2. 交互元素必须有工作的功能
3. 清理副作用（订阅、定时器）
4. 避免在渲染中创建新对象/函数

**Source**: [UI-Behavior Synchronization](../react/ui-behavior-synchronization.md), [useEffect Dependency Array Pitfalls](../react/useeffect-dependency-array-pitfalls.md)

### Performance

**[Complete Performance Rules →](./performance-rules.md)**

Quick rules:

1. 使用 React DevTools Profiler 测量
2. `useMemo` 用于昂贵计算
3. `useCallback` 用于传递给子组件的函数
4. 虚拟化长列表

### Code Quality

**[Complete Code Quality Rules →](./code-quality-rules.md)**

Quick rules:

1. 删除未使用的代码
2. 函数保持简短（< 50 行）
3. 单一职责原则
4. 清晰命名胜过注释

---

## Code Review Checklist

### Before Submitting PR

#### 基础检查 | Basic

- [ ] Linter 无错误
- [ ] 所有测试通过
- [ ] 无 console.log / debugger
- [ ] 无注释掉的代码

#### 功能检查 | Functionality

- [ ] 所有交互元素都能工作
- [ ] 错误情况有正确处理
- [ ] 加载状态有显示
- [ ] 边界情况已测试

#### 代码质量 | Code Quality

- [ ] 无未使用的 imports/variables
- [ ] 无重复代码
- [ ] 命名清晰
- [ ] 复杂逻辑有注释

#### 性能 | Performance

- [ ] 无不必要的重渲染
- [ ] 大数据集使用虚拟化
- [ ] 图片已优化
- [ ] 懒加载适当使用

**Complete checklist**: [PR Review Checklist](./pr-review-checklist.md)

---

## Quick Tips by Scenario

### 写新组件时 | When Creating New Component

```
1. 定义 Props interface
2. 实现基本渲染
3. 添加交互逻辑
4. 添加错误处理
5. 优化性能（如需要）
6. 添加测试
```

### 重构代码时 | When Refactoring

```
1. 确保有测试覆盖
2. 小步骤重构
3. 每步后运行测试
4. 提交前检查 diff
5. 确保功能不变
```

### 修复 Bug 时 | When Fixing Bugs

```
1. 重现问题
2. 找到根本原因
3. 写测试验证 bug
4. 修复问题
5. 确保测试通过
6. 考虑相似问题
```

### Review PR 时 | When Reviewing PR

```
1. 理解改动目的
2. 检查功能正确性
3. 检查代码质量
4. 检查性能影响
5. 检查测试覆盖
6. 提供建设性反馈
```

---

## 最常用的规则 | Most Important Rules

### Top 10 Rules to Remember

1. **类型推断优先** | Type Inference First
   - 让 TypeScript 推断类型，避免冗余注解
   - [详细说明](./typescript-rules.md#type-inference)

2. **完整的依赖数组** | Complete Dependency Arrays
   - `useEffect` 依赖必须完整
   - [详细说明](./react-rules.md#useeffect-dependencies)

3. **UI必须有行为** | UI Must Have Behavior
   - 不要发布没有功能的按钮/链接
   - [详细说明](./react-rules.md#ui-behavior-sync)

4. **删除未使用代码** | Remove Unused Code
   - 没用到的代码要删掉或连接起来
   - [详细说明](./code-quality-rules.md#unused-code)

5. **清理副作用** | Cleanup Side Effects
   - `useEffect` 返回清理函数
   - [详细说明](./react-rules.md#cleanup-effects)

6. **错误必须处理** | Handle All Errors
   - 异步操作必须有 try-catch 或 .catch()
   - [详细说明](./error-handling-rules.md#async-errors)

7. **避免渲染中计算** | Avoid Expensive Renders
   - 使用 `useMemo` 缓存昂贵计算
   - [详细说明](./react-rules.md#memoization)

8. **命名要清晰** | Clear Naming
   - 名字要能表达意图
   - [详细说明](./code-quality-rules.md#naming)

9. **单一职责** | Single Responsibility
   - 函数/组件只做一件事
   - [详细说明](./code-quality-rules.md#single-responsibility)

10. **提前返回** | Early Returns
    - 用 early return 减少嵌套
    - [详细说明](./code-quality-rules.md#early-returns)

---

## 如何使用这个参考手册 | How to Use This Guide

### 场景 1: 写代码前

浏览相关技术的规则页面，记住关键原则

### 场景 2: 遇到问题

使用 [By Problem Type](#by-problem-type) 快速找到解决方案

### 场景 3: PR Review

使用 [Code Review Checklist](#code-review-checklist) 系统检查

### 场景 4: 学习提升

阅读完整的规则文档，理解背后的原理

---

## Related Resources

- [Best Practices](../best-practices/) - 详细的最佳实践文档
- [PR Notes](../best-practices/README.md) - PR review 学习笔记
- [Contributing Guide](../../CONTRIBUTING.md) - 如何添加新内容

---

**Pro Tip**: 把这个页面加入浏览器书签，随时查阅！ 📑
