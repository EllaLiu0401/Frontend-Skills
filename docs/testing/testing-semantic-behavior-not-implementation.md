# 测试语义行为而非实现细节

## 核心原则

**测试应该关注组件的行为和输出，而不是实现细节。**

当测试耦合了实现细节（如CSS类名、内部状态、私有方法）时，重构会破坏测试，即使功能正常。这违背了测试的目的。

---

## 问题：测试CSS类名

### ❌ 不好的做法

```typescript
it('progress bars have correct DaisyUI classes', () => {
  const { container } = render(<ProgressCard data={mockData} />);
  
  const progressBars = container.querySelectorAll('progress');
  
  progressBars.forEach((progressBar) => {
    expect(progressBar).toHaveClass('progress');
    expect(progressBar).toHaveClass('progress-primary');
    expect(progressBar).toHaveClass('w-full');
  });
});
```

**为什么不好？**

1. **耦合CSS框架**：如果换UI库（DaisyUI → Tailwind UI → MUI），测试失败
2. **耦合类名**：`.progress-primary` 改成 `.progress-blue`，测试失败
3. **没有测试真正的功能**：CSS类名存在 ≠ 进度条正确显示
4. **维护成本高**：每次UI重构都要改测试

---

## 解决方案：测试语义行为

### ✅ 好的做法

```typescript
it('progress bars have correct value attributes', () => {
  const { container } = render(<ProgressCard data={mockData} />);
  
  const progressBars = container.querySelectorAll('progress');
  
  // 测试HTML语义属性
  expect(progressBars[0]).toHaveAttribute('value', '75');
  expect(progressBars[0]).toHaveAttribute('max', '100');
  
  expect(progressBars[1]).toHaveAttribute('value', '50');
  expect(progressBars[1]).toHaveAttribute('max', '100');
});

it('progress bars have proper accessibility labels', () => {
  const { container } = render(<ProgressCard data={mockData} />);
  
  const progressBars = container.querySelectorAll('progress');
  
  // 测试可访问性
  expect(progressBars[0]).toHaveAttribute('aria-label', 'Task completion: 75%');
  expect(progressBars[1]).toHaveAttribute('aria-label', 'Upload progress: 50%');
});
```

**为什么好？**

1. **解耦实现**：换UI库不影响测试
2. **测试真正的功能**：value/max 是进度条的核心属性
3. **保证可访问性**：aria-label 确保屏幕阅读器能用
4. **稳定性高**：语义属性不会随UI重构改变

---

## 测试层级：由低到高

### 1. 实现细节（❌ 避免）

- CSS类名
- 内部状态变量名
- 私有方法
- 组件内部结构

**例子：**
```typescript
// ❌ 测试内部状态
expect(wrapper.state('isLoading')).toBe(true);

// ❌ 测试CSS类名
expect(button).toHaveClass('btn-primary');
```

### 2. DOM结构（⚠️ 谨慎使用）

- HTML标签
- DOM层级
- 元素数量

**例子：**
```typescript
// ⚠️ 可以，但脆弱
expect(container.querySelector('button')).toBeInTheDocument();

// ✅ 更好：通过角色查询
expect(screen.getByRole('button', { name: 'Submit' })).toBeInTheDocument();
```

### 3. 语义属性（✅ 推荐）

- HTML语义属性（value, max, type, checked）
- ARIA属性（aria-label, aria-describedby, role）
- data-testid（作为最后手段）

**例子：**
```typescript
// ✅ 测试表单值
expect(input).toHaveValue('user@example.com');

// ✅ 测试可访问性
expect(button).toHaveAttribute('aria-pressed', 'true');
```

### 4. 用户行为（✅ 最佳）

- 用户可见的文本
- 交互结果
- 屏幕阅读器体验

**例子：**
```typescript
// ✅ 测试用户可见内容
expect(screen.getByText('Welcome back, John')).toBeInTheDocument();

// ✅ 测试交互
await user.click(screen.getByRole('button', { name: 'Submit' }));
expect(screen.getByText('Success!')).toBeInTheDocument();
```

---

## 实战案例：重构进度条测试

### 场景

一个显示数据分析结果的卡片组件，包含多个进度条。

### Before: 测试CSS类名 ❌

```typescript
import { render } from '@testing-library/react';
import { DataCard } from './DataCard';

describe('DataCard', () => {
  it('renders progress bars with correct styles', () => {
    const { container } = render(
      <DataCard 
        items={[
          { label: 'CPU Usage', value: 75 },
          { label: 'Memory', value: 50 }
        ]} 
      />
    );
    
    const progressBars = container.querySelectorAll('.progress');
    
    // ❌ 测试CSS类名
    progressBars.forEach(bar => {
      expect(bar).toHaveClass('progress');
      expect(bar).toHaveClass('progress-striped');
      expect(bar).toHaveClass('w-full');
    });
  });
});
```

**问题：**
- 如果设计师要求改用不同的进度条样式，测试失败
- 如果换CSS框架，测试失败
- 没有验证进度条的实际值

### After: 测试语义行为 ✅

```typescript
import { render, screen } from '@testing-library/react';
import { DataCard } from './DataCard';

describe('DataCard', () => {
  it('displays progress values correctly', () => {
    const { container } = render(
      <DataCard 
        items={[
          { label: 'CPU Usage', value: 75 },
          { label: 'Memory', value: 50 }
        ]} 
      />
    );
    
    const progressBars = container.querySelectorAll('progress');
    
    // ✅ 测试HTML语义属性
    expect(progressBars[0]).toHaveAttribute('value', '75');
    expect(progressBars[0]).toHaveAttribute('max', '100');
    
    expect(progressBars[1]).toHaveAttribute('value', '50');
    expect(progressBars[1]).toHaveAttribute('max', '100');
  });
  
  it('provides accessible labels for screen readers', () => {
    const { container } = render(
      <DataCard 
        items={[
          { label: 'CPU Usage', value: 75 },
          { label: 'Memory', value: 50 }
        ]} 
      />
    );
    
    const progressBars = container.querySelectorAll('progress');
    
    // ✅ 测试可访问性
    expect(progressBars[0]).toHaveAttribute('aria-label', 'CPU Usage: 75%');
    expect(progressBars[1]).toHaveAttribute('aria-label', 'Memory: 50%');
  });
  
  it('displays readable labels for sighted users', () => {
    render(
      <DataCard 
        items={[
          { label: 'CPU Usage', value: 75 },
          { label: 'Memory', value: 50 }
        ]} 
      />
    );
    
    // ✅ 测试用户可见内容
    expect(screen.getByText('CPU Usage')).toBeInTheDocument();
    expect(screen.getByText('75%')).toBeInTheDocument();
    expect(screen.getByText('Memory')).toBeInTheDocument();
    expect(screen.getByText('50%')).toBeInTheDocument();
  });
});
```

**优势：**
- ✅ UI重构不破坏测试
- ✅ 保证功能正确性（值正确）
- ✅ 保证可访问性（屏幕阅读器可用）
- ✅ 测试用户实际看到的内容

---

## React Testing Library 查询优先级

按照推荐顺序使用：

### 1. **可访问性查询**（最优先）

```typescript
// ✅ 通过角色
screen.getByRole('button', { name: 'Submit' });
screen.getByRole('textbox', { name: 'Email' });

// ✅ 通过label
screen.getByLabelText('Email address');

// ✅ 通过placeholder
screen.getByPlaceholderText('Enter your email');

// ✅ 通过文本
screen.getByText('Welcome back');
```

### 2. **语义查询**

```typescript
// ⚠️ 可以，但不如角色查询好
screen.getByDisplayValue('current value');
screen.getByAltText('profile picture');
screen.getByTitle('Close');
```

### 3. **Test IDs**（最后手段）

```typescript
// ⚠️ 仅当没有更好选择时使用
screen.getByTestId('custom-element');
```

### 4. **避免使用**

```typescript
// ❌ 不要用
container.querySelector('.my-class');
container.querySelector('#my-id');
wrapper.find('.component');
```

---

## 可访问性测试清单

测试组件时，确保验证：

### ✅ ARIA属性

```typescript
// 按钮状态
expect(button).toHaveAttribute('aria-pressed', 'true');

// 加载状态
expect(spinner).toHaveAttribute('aria-busy', 'true');

// 描述性标签
expect(input).toHaveAttribute('aria-describedby', 'error-message');

// 标签关联
expect(progressBar).toHaveAttribute('aria-label', 'Upload progress: 75%');
```

### ✅ 语义HTML

```typescript
// 使用正确的HTML元素
expect(screen.getByRole('button')).toBeInTheDocument(); // <button>
expect(screen.getByRole('navigation')).toBeInTheDocument(); // <nav>
expect(screen.getByRole('main')).toBeInTheDocument(); // <main>
```

### ✅ 键盘导航

```typescript
import userEvent from '@testing-library/user-event';

it('can be navigated with keyboard', async () => {
  const user = userEvent.setup();
  render(<Dialog />);
  
  // Tab to button
  await user.tab();
  expect(screen.getByRole('button', { name: 'Close' })).toHaveFocus();
  
  // Press Enter
  await user.keyboard('{Enter}');
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
});
```

### ✅ 焦点管理

```typescript
it('manages focus correctly', () => {
  const { rerender } = render(<Modal isOpen={false} />);
  
  // 打开时焦点进入
  rerender(<Modal isOpen={true} />);
  expect(screen.getByRole('dialog')).toHaveFocus();
  
  // 关闭时焦点返回
  rerender(<Modal isOpen={false} />);
  expect(document.body).toHaveFocus();
});
```

---

## 重构时的测试策略

### 步骤1：识别实现细节

审查现有测试，找出：
- 测试CSS类名的
- 测试内部状态的
- 测试私有方法的
- 使用 `.querySelector()` 的

### 步骤2：转换为行为测试

对每个实现细节测试，问：

**"用户能感知这个吗？"**

- ✅ **能**：转换为用户行为测试
- ❌ **不能**：删除测试

**示例：**

```typescript
// ❌ 用户不关心类名
expect(button).toHaveClass('btn-primary');

// ✅ 转换：用户关心按钮是否可用
expect(screen.getByRole('button', { name: 'Submit' })).toBeEnabled();
```

### 步骤3：添加可访问性测试

对每个交互元素，添加：

```typescript
// 标签
expect(input).toHaveAttribute('aria-label', 'Search');

// 状态
expect(button).toHaveAttribute('aria-pressed', 'false');

// 关系
expect(input).toHaveAttribute('aria-describedby', 'error-msg');
```

---

## 常见反模式

### 1. 测试Mock而非真实行为

```typescript
// ❌ 不好
it('calls onClick', () => {
  const onClick = jest.fn();
  render(<Button onClick={onClick} />);
  
  fireEvent.click(screen.getByRole('button'));
  expect(onClick).toHaveBeenCalled();
});

// ✅ 更好：测试结果
it('submits the form', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);
  
  await user.type(screen.getByLabelText('Email'), 'test@example.com');
  await user.click(screen.getByRole('button', { name: 'Login' }));
  
  expect(screen.getByText('Welcome back!')).toBeInTheDocument();
});
```

### 2. 测试快照而非功能

```typescript
// ❌ 不好
it('renders correctly', () => {
  const { container } = render(<Card />);
  expect(container).toMatchSnapshot();
});

// ✅ 更好：测试关键内容
it('displays user information', () => {
  render(<Card user={{ name: 'John', email: 'john@example.com' }} />);
  
  expect(screen.getByText('John')).toBeInTheDocument();
  expect(screen.getByText('john@example.com')).toBeInTheDocument();
});
```

### 3. 过度依赖测试ID

```typescript
// ❌ 不好
<div data-testid="progress-bar" />
screen.getByTestId('progress-bar');

// ✅ 更好：使用语义元素
<progress aria-label="Upload progress: 50%" value={50} max={100} />
screen.getByRole('progressbar');
```

---

## 总结

### 记住这个原则

> **测试应该像用户一样使用你的应用。**

### 检查清单

- ✅ 测试用户可见的文本和内容
- ✅ 测试用户交互和结果
- ✅ 测试可访问性（ARIA, 键盘导航）
- ✅ 测试语义HTML属性
- ❌ 不测试CSS类名
- ❌ 不测试内部状态
- ❌ 不测试实现细节

### 问自己

1. **如果我重构实现但保持功能不变，测试会失败吗？**
   - 如果会 → 测试耦合了实现细节
   
2. **这个测试保护的是用户关心的行为吗？**
   - 如果不是 → 删除或重写测试

3. **这个功能对视障用户可用吗？**
   - 如果不确定 → 添加可访问性测试

---

## 延伸阅读

- [Testing Library Guiding Principles](https://testing-library.com/docs/guiding-principles/)
- [Common mistakes with React Testing Library](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)

---

**最后更新：** 2026-01-21
