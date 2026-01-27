# Dashboard Metrics 组件设计模式

## 概述

Dashboard指标展示是数据密集型应用的核心功能。本文档总结设计高质量metrics组件的最佳实践。

---

## 组件架构

### 层级结构

```
MetricsSection (容器)
  ├── StatCard (基础卡片) x4
  └── TopItemsCard (复杂卡片)
       ├── 列表数据
       ├── 进度条
       └── 操作按钮
```

### 责任分离

- **容器组件**：布局、数据传递
- **卡片组件**：单个指标展示
- **工具函数**：格式化、计算

---

## TypeScript 类型设计

### 核心原则

1. **显式定义数据结构**
2. **使用 readonly 保证不可变性**
3. **提供清晰的JSDoc注释**

### 示例：Metrics类型定义

```typescript
/**
 * Trend indicator for metrics
 */
export interface Trend {
  /** Percentage value (e.g., 12 means 12%) */
  value: number;
  /** true = increase, false = decrease */
  isPositive: boolean;
  /** Optional label (e.g., "vs last month") */
  label?: string;
}

/**
 * Top item in a ranked list
 */
export interface TopItem {
  /** Unique identifier */
  id: string;
  /** Display name */
  name: string;
  /** Numeric count */
  count: number;
  /** Percentage (0-100) */
  percentage: number;
  /** Optional description */
  description?: string;
}

/**
 * Dashboard metrics data
 */
export interface DashboardMetrics {
  totalCount: {
    value: number;
    trend: Trend;
  };
  avgDuration: {
    /** Duration in minutes */
    value: number;
    /** Formatted string (e.g., "2:34") */
    formatted: string;
    trend: Trend;
  };
  /** Top 5 items */
  topItems: TopItem[];
}
```

**要点：**
- ✅ JSDoc为每个字段提供说明
- ✅ 注明单位（minutes, percentage）
- ✅ 说明格式（"2:34"）
- ✅ 嵌套对象有明确结构

---

## 组件设计

### 1. Props接口设计

```typescript
interface MetricsSectionProps {
  readonly metrics: DashboardMetrics;
  readonly isLoading?: boolean;
  readonly onRefresh?: () => void;
}
```

**最佳实践：**
- ✅ 使用 `readonly` 防止意外修改
- ✅ 可选props用 `?` 标记
- ✅ 回调函数明确命名（`onRefresh` 而非 `refresh`）

### 2. 容器组件 - MetricsSection

```typescript
import { Grid, Flex } from '@your-ui-lib';
import type { DashboardMetrics } from '@/types';

interface MetricsSectionProps {
  readonly metrics: DashboardMetrics;
}

export function MetricsSection({ metrics }: MetricsSectionProps) {
  return (
    <Flex direction="column" gap="md">
      {/* 网格布局 - 4列 */}
      <Grid cols={{ default: 1, sm: 2, lg: 4 }} gap="md">
        <StatCard
          title="Total Count"
          value={formatNumber(metrics.totalCount.value)}
          trend={metrics.totalCount.trend}
        />
        
        <StatCard
          title="Avg Duration"
          value={metrics.avgDuration.formatted}
          trend={metrics.avgDuration.trend}
        />
        
        {/* ... 更多卡片 */}
      </Grid>

      {/* 全宽复杂卡片 */}
      <TopItemsCard items={metrics.topItems} />
    </Flex>
  );
}
```

**设计要点：**
- ✅ 响应式布局（1列 → 2列 → 4列）
- ✅ 一致的间距（`gap="md"`）
- ✅ 数据格式化在容器层完成
- ✅ 复杂组件独立提取

### 3. 基础卡片 - StatCard

```typescript
interface StatCardProps {
  readonly title: string;
  readonly value: string | number;
  readonly trend?: Trend;
  readonly icon?: React.ReactNode;
  readonly actions?: React.ReactNode;
}

export function StatCard({ 
  title, 
  value, 
  trend,
  icon,
  actions 
}: StatCardProps) {
  return (
    <Card>
      <Flex align="center" justify="between">
        {icon && <Icon>{icon}</Icon>}
        <Heading level={5}>{title}</Heading>
      </Flex>
      
      <Text size="2xl" weight="bold">
        {value}
      </Text>
      
      {trend && (
        <Flex align="center" gap="xs">
          <TrendIcon isPositive={trend.isPositive} />
          <Text color={trend.isPositive ? 'success' : 'error'}>
            {trend.value}%
          </Text>
          {trend.label && (
            <Text color="secondary">{trend.label}</Text>
          )}
        </Flex>
      )}
      
      {actions && <Box mt="md">{actions}</Box>}
    </Card>
  );
}
```

**设计要点：**
- ✅ 灵活的value类型（string | number）
- ✅ 可选的trend、icon、actions
- ✅ 语义化颜色（success/error）
- ✅ 清晰的视觉层级

### 4. 复杂卡片 - TopItemsCard

```typescript
interface TopItemsCardProps {
  readonly items: readonly TopItem[];
  readonly onViewAll?: () => void;
}

export function TopItemsCard({ items, onViewAll }: TopItemsCardProps) {
  const isEmpty = items.length === 0;
  
  const content = isEmpty ? (
    <EmptyState message="No data available" />
  ) : (
    <Flex direction="column" gap="md">
      {/* 列表 */}
      <Flex direction="column" gap="sm">
        {items.map((item, index) => {
          // 前3项高亮
          const color = index < 3 ? 'primary' : 'secondary';
          
          return (
            <Flex key={item.id} direction="column" gap="xs">
              {/* 标题和百分比 */}
              <Flex align="center" justify="between">
                <Text variant="body">{item.name}</Text>
                <Badge size="sm" color={color}>
                  {item.percentage}%
                </Badge>
              </Flex>
              
              {/* 进度条 */}
              <Progress
                value={item.percentage}
                max={100}
                color={color}
                aria-label={`${item.name}: ${item.percentage}%`}
              />
            </Flex>
          );
        })}
      </Flex>
      
      {/* 操作按钮 */}
      {onViewAll && (
        <Button 
          variant="outline" 
          size="sm" 
          onClick={onViewAll}
          className="w-full"
        >
          View All
        </Button>
      )}
    </Flex>
  );
  
  return (
    <StatCard 
      title="Top Items" 
      value={items.length}
      actions={content}
    />
  );
}
```

**设计要点：**
- ✅ 空状态处理
- ✅ 前N项视觉差异化
- ✅ 可访问性（aria-label）
- ✅ 可选的操作回调

---

## 数字格式化

### 需求

大数字需要人类可读的格式：
- `3500` → `"3.5K"`
- `1500000` → `"1.5M"`
- `500` → `"500"`

### 实现

```typescript
/**
 * Format large numbers with K/M suffixes
 *
 * Examples: 3500 → "3.5K", 1500000 → "1.5M"
 *
 * @param value - Number to format
 * @returns Formatted string with appropriate suffix
 */
export function formatNumber(value: number): string {
  if (value >= 1_000_000) {
    return `${(value / 1_000_000).toFixed(1)}M`;
  }
  if (value >= 1_000) {
    return `${(value / 1_000).toFixed(1)}K`;
  }
  return value.toString();
}
```

### 测试

```typescript
import { describe, it, expect } from 'vitest';
import { formatNumber } from './formatNumber';

describe('formatNumber', () => {
  it('formats numbers under 1000 as-is', () => {
    expect(formatNumber(500)).toBe('500');
    expect(formatNumber(0)).toBe('0');
    expect(formatNumber(999)).toBe('999');
  });

  it('formats thousands with K suffix', () => {
    expect(formatNumber(3500)).toBe('3.5K');
    expect(formatNumber(1000)).toBe('1.0K');
    expect(formatNumber(12345)).toBe('12.3K');
  });

  it('formats millions with M suffix', () => {
    expect(formatNumber(1_500_000)).toBe('1.5M');
    expect(formatNumber(2_000_000)).toBe('2.0M');
  });

  it('handles edge cases', () => {
    expect(formatNumber(1_000_000)).toBe('1.0M');
    expect(formatNumber(999_999)).toBe('1000.0K');
  });
});
```

**要点：**
- ✅ 清晰的JSDoc说明
- ✅ 边界条件测试
- ✅ 使用数字分隔符（`1_000_000`）提升可读性

---

## 时间格式化

### 需求

将分钟数格式化为 `h:mm` 格式：
- `60` → `"1:00"`
- `154` → `"2:34"`
- `5` → `"0:05"`

### 实现

```typescript
/**
 * Format duration in minutes to "h:mm" format
 *
 * @param minutes - Duration in minutes
 * @returns Formatted string (e.g., "2:34")
 */
export function formatDuration(minutes: number): string {
  const hours = Math.floor(minutes / 60);
  const mins = minutes % 60;
  return `${hours}:${mins.toString().padStart(2, '0')}`;
}
```

### 测试

```typescript
describe('formatDuration', () => {
  it('formats whole hours', () => {
    expect(formatDuration(60)).toBe('1:00');
    expect(formatDuration(120)).toBe('2:00');
  });

  it('formats hours with minutes', () => {
    expect(formatDuration(154)).toBe('2:34');
    expect(formatDuration(65)).toBe('1:05');
  });

  it('formats minutes under an hour', () => {
    expect(formatDuration(45)).toBe('0:45');
    expect(formatDuration(5)).toBe('0:05');
  });

  it('handles zero', () => {
    expect(formatDuration(0)).toBe('0:00');
  });
});
```

---

## 国际化（i18n）

### 结构化翻译键

```json
{
  "dashboard": {
    "metrics": {
      "totalCount": "Total Count",
      "avgDuration": "Avg Duration",
      "topItems": "Top Items",
      "viewAll": "View All",
      "noData": "No data available",
      "vsLastMonth": "vs last month"
    }
  }
}
```

### 使用

```typescript
import { useTranslations } from 'next-intl'; // 或 react-i18next

export function MetricsSection({ metrics }: MetricsSectionProps) {
  const t = useTranslations('dashboard.metrics');
  
  return (
    <StatCard
      title={t('totalCount')}
      value={formatNumber(metrics.totalCount.value)}
      trend={{
        ...metrics.totalCount.trend,
        label: t('vsLastMonth')
      }}
    />
  );
}
```

**要点：**
- ✅ 嵌套键提供上下文
- ✅ 动态值（trend.label）支持翻译
- ✅ 复用通用词汇（noData）

---

## 响应式设计

### Grid布局

```typescript
<Grid 
  cols={{ 
    default: 1,   // 移动端：1列
    sm: 2,        // 平板：2列
    lg: 4         // 桌面：4列
  }} 
  gap="md"
>
  {/* 卡片 */}
</Grid>
```

### 条件渲染

```typescript
export function MetricsSection({ metrics }: MetricsSectionProps) {
  const isMobile = useMediaQuery('(max-width: 640px)');
  
  return (
    <Flex direction="column" gap="md">
      {/* 桌面：4列网格 */}
      {!isMobile && (
        <Grid cols={4} gap="md">
          {/* 所有卡片 */}
        </Grid>
      )}
      
      {/* 移动端：垂直堆叠 */}
      {isMobile && (
        <Flex direction="column" gap="sm">
          {/* 卡片 */}
        </Flex>
      )}
    </Flex>
  );
}
```

---

## Mock数据设计

### 原则

1. **类型安全**：Mock数据必须符合接口
2. **真实性**：数据反映实际使用场景
3. **可复用**：用于开发和测试

### 示例

```typescript
import type { DashboardMetrics } from '@/types';

/**
 * Mock dashboard metrics for development and testing
 */
export const mockMetrics: DashboardMetrics = {
  totalCount: {
    value: 3500,
    trend: {
      value: 21.3,
      isPositive: true,
      label: 'vs last month',
    },
  },
  avgDuration: {
    value: 154, // minutes
    formatted: '2:34',
    trend: {
      value: 6.9,
      isPositive: true,
      label: 'vs last month',
    },
  },
  topItems: [
    {
      id: '1',
      name: 'Product Inquiry',
      count: 84,
      percentage: 34,
      description: 'Questions about product features',
    },
    {
      id: '2',
      name: 'Support Request',
      count: 62,
      percentage: 25,
      description: 'Technical help',
    },
    // ... more items
  ],
};
```

### 开发中使用

```typescript
export default function DashboardPage() {
  // 开发环境用mock，生产环境用API
  const metrics = process.env.NODE_ENV === 'development' 
    ? mockMetrics 
    : useMetricsData();
  
  return (
    <div>
      {process.env.NODE_ENV === 'development' && (
        <Alert type="info">
          Development Preview - Using Mock Data
        </Alert>
      )}
      <MetricsSection metrics={metrics} />
    </div>
  );
}
```

---

## 可访问性

### 进度条

```typescript
<progress
  value={75}
  max={100}
  aria-label="CPU Usage: 75%"  // ✅ 描述性标签
/>
```

### 趋势指示器

```typescript
<Flex align="center" gap="xs">
  {/* ✅ 视觉图标 */}
  <TrendIcon isPositive={trend.isPositive} />
  
  {/* ✅ 屏幕阅读器文本 */}
  <span className="sr-only">
    {trend.isPositive ? 'increased' : 'decreased'} by
  </span>
  
  <Text color={trend.isPositive ? 'success' : 'error'}>
    {trend.value}%
  </Text>
</Flex>
```

### 空状态

```typescript
function EmptyState({ message }: { message: string }) {
  return (
    <Flex 
      direction="column" 
      align="center" 
      justify="center"
      role="status"  // ✅ ARIA角色
      aria-live="polite"  // ✅ 动态内容通知
    >
      <Icon name="inbox" aria-hidden="true" />
      <Text>{message}</Text>
    </Flex>
  );
}
```

---

## 性能优化

### 1. 避免不必要的重渲染

```typescript
import { memo } from 'react';

export const StatCard = memo(function StatCard({ 
  title, 
  value, 
  trend 
}: StatCardProps) {
  // ...
});
```

### 2. 数字格式化缓存

```typescript
import { useMemo } from 'react';

export function MetricsSection({ metrics }: MetricsSectionProps) {
  // 缓存格式化结果
  const formattedCount = useMemo(
    () => formatNumber(metrics.totalCount.value),
    [metrics.totalCount.value]
  );
  
  return <StatCard title="Total" value={formattedCount} />;
}
```

### 3. 虚拟化长列表

```typescript
import { useVirtualizer } from '@tanstack/react-virtual';

export function LargeItemsList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });
  
  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: virtualItem.start,
              height: virtualItem.size,
            }}
          >
            <ItemRow item={items[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 错误处理

### 加载状态

```typescript
export function MetricsSection({ 
  metrics, 
  isLoading 
}: MetricsSectionProps) {
  if (isLoading) {
    return (
      <Grid cols={4} gap="md">
        {Array.from({ length: 4 }).map((_, i) => (
          <SkeletonCard key={i} />
        ))}
      </Grid>
    );
  }
  
  return (
    <Grid cols={4} gap="md">
      {/* 实际内容 */}
    </Grid>
  );
}
```

### 错误边界

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error }: { error: Error }) {
  return (
    <Alert type="error">
      <Text>Failed to load metrics: {error.message}</Text>
      <Button onClick={() => window.location.reload()}>
        Retry
      </Button>
    </Alert>
  );
}

export default function DashboardPage() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <MetricsSection metrics={metrics} />
    </ErrorBoundary>
  );
}
```

---

## 测试策略

### 单元测试

```typescript
describe('MetricsSection', () => {
  it('displays all metric cards', () => {
    render(<MetricsSection metrics={mockMetrics} />);
    
    expect(screen.getByText('Total Count')).toBeInTheDocument();
    expect(screen.getByText('3.5K')).toBeInTheDocument();
  });
  
  it('formats duration correctly', () => {
    render(<MetricsSection metrics={mockMetrics} />);
    
    expect(screen.getByText('2:34')).toBeInTheDocument();
  });
  
  it('shows trend indicators', () => {
    render(<MetricsSection metrics={mockMetrics} />);
    
    // 测试趋势值
    expect(screen.getByText('21.3%')).toBeInTheDocument();
    // 测试趋势标签
    expect(screen.getByText('vs last month')).toBeInTheDocument();
  });
});
```

### 集成测试

```typescript
describe('Dashboard Integration', () => {
  it('updates when data changes', async () => {
    const { rerender } = render(
      <MetricsSection metrics={mockMetrics} />
    );
    
    expect(screen.getByText('3.5K')).toBeInTheDocument();
    
    // 更新数据
    const updatedMetrics = {
      ...mockMetrics,
      totalCount: { value: 5000, trend: mockMetrics.totalCount.trend }
    };
    
    rerender(<MetricsSection metrics={updatedMetrics} />);
    
    expect(screen.getByText('5.0K')).toBeInTheDocument();
  });
});
```

---

## 总结

### 设计检查清单

#### 类型安全
- ✅ 所有props有TypeScript接口
- ✅ 使用 `readonly` 防止修改
- ✅ JSDoc注释说明单位和格式

#### 组件设计
- ✅ 职责单一（容器/卡片/工具函数）
- ✅ Props接口清晰最小化
- ✅ 支持响应式布局

#### 用户体验
- ✅ 数字格式化（3.5K, 1.5M）
- ✅ 时间格式化（2:34）
- ✅ 趋势指示器（颜色+图标）
- ✅ 空状态处理

#### 可访问性
- ✅ 进度条有 aria-label
- ✅ 趋势有屏幕阅读器文本
- ✅ 空状态有 role 和 aria-live

#### 性能
- ✅ Memo避免重渲染
- ✅ useMemo缓存计算
- ✅ 长列表虚拟化

#### 可测试性
- ✅ Mock数据类型安全
- ✅ 测试语义行为而非实现
- ✅ 单元+集成测试覆盖

---

**最后更新：** 2026-01-21
