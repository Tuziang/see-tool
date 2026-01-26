> 在线工具网址：[https://see-tool.com/date-calculator](https://see-tool.com/date-calculator)

> 工具截图：
> ![工具截图](工具截图.png)

## 一、核心功能设计

日期计算器包含四个独立模块:
1. **日期间隔计算**: 计算两个日期之间的天数、周数、月数、年数
2. **日期加减计算**: 在基准日期上加减指定时间单位
3. **年龄计算**: 精确计算年龄(年/月/日)
4. **工作日计算**: 统计工作日、周末天数

## 二、日期间隔计算实现

### 2.1 核心计算逻辑

```javascript
const dateDiff = computed(() => {
  if (!startDate.value || !endDate.value) {
    return { days: 0, weeks: 0, months: 0, years: 0 }
  }

  const start = new Date(startDate.value)
  const end = new Date(endDate.value)

  // 确保开始日期小于结束日期(自动排序)
  const [earlierDate, laterDate] = start <= end ? [start, end] : [end, start]

  let diffTime = laterDate.getTime() - earlierDate.getTime()

  // 如果包含结束日期,增加一天
  if (includeEndDate.value) {
    diffTime += 24 * 60 * 60 * 1000
  }

  const diffDays = Math.floor(diffTime / (1000 * 60 * 60 * 24))

  // 计算精确的月数差异
  let months = (laterDate.getFullYear() - earlierDate.getFullYear()) * 12
  months += laterDate.getMonth() - earlierDate.getMonth()

  // 如果日期不足一个月,减去一个月
  if (laterDate.getDate() < earlierDate.getDate()) {
    months--
  }

  // 计算年数
  const years = Math.floor(months / 12)

  return {
    days: diffDays,
    weeks: Math.floor(diffDays / 7),
    months: Math.max(0, months),
    years: Math.max(0, years)
  }
})
```

**关键点**:
1. **自动排序**: 无论用户输入顺序,自动识别较早和较晚的日期
2. **包含结束日期**: 可选项,影响天数计算(+1天)
3. **精确月数**: 考虑日期不足一个月的情况
4. **负数保护**: 使用 `Math.max(0, value)` 防止负数

### 2.2 辅助工具函数

```javascript
// 交换开始和结束日期
const swapDates = () => {
  const temp = startDate.value
  startDate.value = endDate.value
  endDate.value = temp
}

// 设置结束日期为今天
const setToday = (type) => {
  if (!process.client) return
  const today = new Date().toISOString().split('T')[0]
  if (type === 'diff') {
    endDate.value = today
  }
}
```

## 三、日期加减计算实现

### 3.1 核心计算逻辑

```javascript
const calculatedDate = computed(() => {
  if (!baseDate.value) {
    return ''
  }

  if (!amount.value || amount.value === 0) {
    return baseDate.value
  }

  const base = new Date(baseDate.value)
  // 根据操作类型确定正负
  const value = operation.value === 'add' ? parseInt(amount.value) : -parseInt(amount.value)

  switch (unit.value) {
    case 'days':
      base.setDate(base.getDate() + value)
      break
    case 'weeks':
      base.setDate(base.getDate() + (value * 7))
      break
    case 'months':
      base.setMonth(base.getMonth() + value)
      break
    case 'years':
      base.setFullYear(base.getFullYear() + value)
      break
  }

  return base.toISOString().split('T')[0]
})
```

**关键点**:
1. **操作符处理**: 减法通过负数实现,统一使用加法逻辑
2. **原生 Date API**: 利用 `setDate`/`setMonth`/`setFullYear` 自动处理溢出
3. **格式化输出**: `toISOString().split('T')[0]` 获取 YYYY-MM-DD 格式

### 3.2 获取星期几

```javascript
const getWeekday = (dateStr) => {
  if (!dateStr) return ''
  const weekdays = tm('dateCalculator.weekdays')
  if (!weekdays || !Array.isArray(weekdays)) return ''
  const date = new Date(dateStr)
  return weekdays[date.getDay()] || ''
}
```

**说明**:
- `getDay()` 返回 0-6,其中 0 代表周日
- 从国际化配置中读取星期名称数组

## 四、年龄计算实现

### 4.1 精确年龄计算

```javascript
const age = computed(() => {
  if (!birthDate.value || !ageCalculateDate.value) {
    return { years: 0, months: 0, days: 0, totalDays: 0 }
  }

  const birth = new Date(birthDate.value)
  const calculate = new Date(ageCalculateDate.value)

  // 如果出生日期晚于计算日期,返回0
  if (birth > calculate) {
    return { years: 0, months: 0, days: 0, totalDays: 0 }
  }

  // 计算精确年龄
  let years = calculate.getFullYear() - birth.getFullYear()
  let months = calculate.getMonth() - birth.getMonth()
  let days = calculate.getDate() - birth.getDate()

  // 调整天数
  if (days < 0) {
    months--
    // 获取上个月的天数
    const lastMonth = new Date(calculate.getFullYear(), calculate.getMonth(), 0)
    days += lastMonth.getDate()
  }

  // 调整月数
  if (months < 0) {
    years--
    months += 12
  }

  // 计算总天数
  const totalDays = Math.floor((calculate.getTime() - birth.getTime()) / (1000 * 60 * 60 * 24))

  return {
    years: Math.max(0, years),
    months: Math.max(0, months),
    days: Math.max(0, days),
    totalDays: Math.max(0, totalDays)
  }
})
```

**关键点**:
1. **逐级调整**: 先调整天数,再调整月数,最后得到年数
2. **借位逻辑**: 天数不足时从月份借位,月份不足时从年份借位
3. **上月天数**: 使用 `new Date(year, month, 0)` 获取上月最后一天
4. **总天数**: 单独计算,用于显示"已活xx天"

### 4.2 派生数据计算

```javascript
// 模板中使用
Math.floor(age.totalDays / 30.44)  // 总月数(平均每月30.44天)
Math.floor(age.totalDays / 7)      // 总周数
age.totalDays                      // 总天数
```

## 五、工作日计算实现

### 5.1 核心计算逻辑

```javascript
const workDays = computed(() => {
  if (!workStartDate.value || !workEndDate.value) {
    return { total: 0, weekdays: 0, weekends: 0 }
  }

  const start = new Date(workStartDate.value)
  const end = new Date(workEndDate.value)

  // 确保开始日期不大于结束日期
  if (start > end) {
    return { total: 0, weekdays: 0, weekends: 0 }
  }

  let weekdays = 0
  let weekends = 0
  const current = new Date(start)

  // 包含开始和结束日期
  while (current <= end) {
    const dayOfWeek = current.getDay()
    if (dayOfWeek === 0 || dayOfWeek === 6) { // 周日=0, 周六=6
      weekends++
    } else {
      weekdays++
    }
    current.setDate(current.getDate() + 1)
  }

  return {
    total: weekdays + weekends,
    weekdays: excludeWeekends.value ? weekdays : weekdays + weekends,
    weekends
  }
})
```

**关键点**:
1. **逐日遍历**: 从开始日期循环到结束日期,逐日判断
2. **星期判断**: `getDay()` 返回 0(周日) 或 6(周六) 为周末
3. **可选排除**: 根据 `excludeWeekends` 决定是否排除周末
4. **包含边界**: 包含开始和结束日期

## 六、状态管理

### 6.1 响应式状态定义

```javascript
// Tab 切换
const activeTab = ref('difference')

// 日期间隔计算
const startDate = ref('')
const endDate = ref('')
const includeEndDate = ref(false)

// 日期加减计算
const baseDate = ref('')
const operation = ref('add')      // 'add' | 'subtract'
const amount = ref(0)
const unit = ref('days')          // 'days' | 'weeks' | 'months' | 'years'

// 工作日计算
const workStartDate = ref('')
const workEndDate = ref('')
const excludeWeekends = ref(true)

// 年龄计算
const birthDate = ref('')
const ageCalculateDate = ref('')
```

### 6.2 初始化默认值

```javascript
onMounted(() => {
  if (!process.client) return
  const today = new Date().toISOString().split('T')[0]
  startDate.value = today
  endDate.value = today
  baseDate.value = today
  workStartDate.value = today
  workEndDate.value = today
  birthDate.value = ''  // 不设置默认出生日期
  ageCalculateDate.value = today
})
```

**说明**:
- 使用 `process.client` 判断避免 SSR 问题
- 出生日期不设默认值,避免误导用户

## 七、日期处理技巧

### 7.1 Date 对象的自动溢出处理

```javascript
// JavaScript 的 Date 会自动处理溢出
const date = new Date('2024-01-31')
date.setMonth(date.getMonth() + 1)  // 自动变为 2024-03-02(2月没有31日)
```

### 7.2 获取上月最后一天

```javascript
// 将日期设为0,会自动回退到上月最后一天
const lastDayOfLastMonth = new Date(year, month, 0)
```

### 7.3 日期格式化

```javascript
// ISO 格式转 YYYY-MM-DD
const dateStr = new Date().toISOString().split('T')[0]
```

## 八、核心算法总结

```
日期间隔计算:
  时间戳相减 → 转换为天数
  年月日逐级计算 → 处理借位

日期加减计算:
  原生 Date API → 自动处理溢出

年龄计算:
  年月日分别相减 → 逐级调整借位

工作日计算:
  逐日遍历 → 判断星期几 → 统计分类
```

**核心原则**:
1. **利用原生 API**: Date 对象的自动溢出处理
2. **边界处理**: 防止负数、空值、非法日期
3. **精确计算**: 考虑月份天数差异、闰年等特殊情况
4. **用户友好**: 自动排序、可选配置、实时计算
