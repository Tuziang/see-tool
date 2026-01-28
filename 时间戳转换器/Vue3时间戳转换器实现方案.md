> 在线工具网址：[https://see-tool.com/timestamp-converter](https://see-tool.com/timestamp-converter)

> 工具截图：
> ![工具截图](工具截图.png)

## 一、核心功能设计

时间戳转换器包含三个主要模块:
1. **实时时间戳显示**: 自动刷新的当前时间戳(秒/毫秒)
2. **时间戳转日期**: 将Unix时间戳转换为可读日期格式
3. **日期转时间戳**: 将日期时间转换为Unix时间戳

## 二、实时时间戳显示实现

### 2.1 核心状态管理

```javascript
// 响应式数据
const autoRefresh = ref(true)           // 自动刷新开关
const currentSeconds = ref(0)           // 当前秒级时间戳
const currentMilliseconds = ref(0)      // 当前毫秒级时间戳

let refreshInterval = null              // 定时器引用
```

### 2.2 更新时间戳逻辑

```javascript
// 更新当前时间戳
const updateCurrentTimestamp = () => {
  if (!process.client) return           // SSR 保护
  const now = Date.now()                // 获取当前毫秒时间戳
  currentSeconds.value = Math.floor(now / 1000)  // 转换为秒
  currentMilliseconds.value = now
}
```

**关键点**:
1. **SSR 保护**: 使用 `process.client` 判断,避免服务端渲染错误
2. **Date.now()**: 返回毫秒级时间戳,性能优于 `new Date().getTime()`
3. **秒级转换**: 使用 `Math.floor()` 向下取整

### 2.3 自动刷新机制

```javascript
// 监听自动刷新开关
watch(autoRefresh, (val) => {
  if (!process.client) return

  if (val) {
    updateCurrentTimestamp()            // 立即更新一次
    refreshInterval = setInterval(updateCurrentTimestamp, 1000)  // 每秒更新
  } else {
    if (refreshInterval) {
      clearInterval(refreshInterval)    // 清除定时器
      refreshInterval = null
    }
  }
})
```

**关键点**:
1. **立即更新**: 开启时先执行一次,避免1秒延迟
2. **定时器管理**: 关闭时清除定时器,防止内存泄漏
3. **1秒间隔**: `setInterval(fn, 1000)` 实现秒级刷新

### 2.4 生命周期管理

```javascript
onMounted(() => {
  if (!process.client) return
  updateCurrentTimestamp()
  if (autoRefresh.value) {
    refreshInterval = setInterval(updateCurrentTimestamp, 1000)
  }
})

onUnmounted(() => {
  if (refreshInterval) {
    clearInterval(refreshInterval)      // 组件销毁时清理定时器
  }
})
```

**说明**:
- 组件挂载时初始化时间戳和定时器
- 组件卸载时必须清理定时器,防止内存泄漏

## 三、时间戳转日期实现

### 3.1 格式自动检测

```javascript
// 检测时间戳格式(秒 or 毫秒)
const detectTimestampFormat = (ts) => {
  const str = String(ts)
  return str.length >= 13 ? 'milliseconds' : 'seconds'
}
```

**判断依据**:
- **秒级时间戳**: 10位数字 (如: 1706425716)
- **毫秒级时间戳**: 13位数字 (如: 1706425716000)
- **临界点**: 13位作为分界线

### 3.2 核心转换逻辑

```javascript
const convertTimestampToDate = () => {
  if (!process.client) return
  if (!timestampInput.value.trim()) {
    safeMessage.warning(t('timestampConverter.notifications.enterTimestamp'))
    return
  }

  try {
    let ts = parseInt(timestampInput.value)

    // 自动检测或手动指定格式
    const format = tsInputFormat.value === 'auto'
      ? detectTimestampFormat(ts)
      : tsInputFormat.value

    // 统一转换为毫秒
    if (format === 'seconds') {
      ts = ts * 1000
    }

    const date = new Date(ts)

    // 验证日期有效性
    if (isNaN(date.getTime())) {
      safeMessage.error(t('timestampConverter.notifications.invalidTimestamp'))
      return
    }

    // ... 后续处理
  } catch (err) {
    safeMessage.error(t('timestampConverter.notifications.convertFailed'))
  }
}
```

**关键点**:
1. **输入验证**: 检查空值和有效性
2. **格式统一**: 统一转换为毫秒级时间戳
3. **有效性检查**: `isNaN(date.getTime())` 判断日期是否有效
4. **异常捕获**: try-catch 保护,防止程序崩溃

### 3.3 时区处理

```javascript
// 获取本地时区偏移
const getTimezoneOffset = () => {
  const offset = -date.getTimezoneOffset()  // 注意负号
  const hours = Math.floor(Math.abs(offset) / 60)
  const minutes = Math.abs(offset) % 60
  const sign = offset >= 0 ? '+' : '-'
  return `UTC${sign}${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}`
}
```

**说明**:
- `getTimezoneOffset()` 返回的是 UTC 与本地时间的分钟差
- 返回值为正表示本地时间落后于 UTC,需要取反
- 格式化为 `UTC+08:00` 形式

```javascript
// 获取指定时区的偏移
const getTimezoneOffsetForZone = (timezone) => {
  if (timezone === 'local') {
    return getTimezoneOffset()
  }

  try {
    const utcDate = new Date(date.toLocaleString('en-US', { timeZone: 'UTC' }))
    const tzDate = new Date(date.toLocaleString('en-US', { timeZone: timezone }))
    const offset = (tzDate - utcDate) / (1000 * 60)
    const hours = Math.floor(Math.abs(offset) / 60)
    const minutes = Math.abs(offset) % 60
    const sign = offset >= 0 ? '+' : '-'
    return `GMT${sign}${hours}`
  } catch (e) {
    return ''
  }
}
```

**关键技巧**:
- 使用 `toLocaleString()` 的 `timeZone` 参数转换时区
- 通过 UTC 和目标时区的时间差计算偏移量
- 异常捕获处理无效时区名称

### 3.4 日期格式化输出

```javascript
// 根据选择的时区格式化本地时间
let localTime = date.toLocaleString(
  locale.value === 'en' ? 'en-US' : 'zh-CN',
  { hour12: false }
)

if (tsOutputTimezone.value !== 'local') {
  try {
    localTime = date.toLocaleString(
      locale.value === 'en' ? 'en-US' : 'zh-CN',
      {
        timeZone: tsOutputTimezone.value === 'UTC' ? 'UTC' : tsOutputTimezone.value,
        hour12: false
      }
    )
  } catch (e) {
    // 时区无效时回退到本地时间
    localTime = date.toLocaleString(
      locale.value === 'en' ? 'en-US' : 'zh-CN',
      { hour12: false }
    )
  }
}
```

**格式化选项**:
- `hour12: false`: 使用24小时制
- `timeZone`: 指定时区(如 'Asia/Shanghai', 'UTC')
- 根据语言环境自动调整日期格式

### 3.5 年中第几天/第几周计算

```javascript
// 计算年中第几天
const getDayOfYear = (d) => {
  const start = new Date(d.getFullYear(), 0, 0)  // 去年12月31日
  const diff = d - start
  const oneDay = 1000 * 60 * 60 * 24
  return Math.floor(diff / oneDay)
}

// 计算年中第几周
const getWeekOfYear = (d) => {
  const start = new Date(d.getFullYear(), 0, 1)  // 今年1月1日
  const days = Math.floor((d - start) / (24 * 60 * 60 * 1000))
  return Math.ceil((days + start.getDay() + 1) / 7)
}
```

**算法说明**:
1. **年中第几天**: 当前日期 - 去年最后一天 = 天数差
2. **年中第几周**: (天数差 + 1月1日星期几 + 1) / 7 向上取整

### 3.6 相对时间计算

```javascript
// 相对时间(如: 3天前, 2小时后)
const getRelativeTime = (timestamp) => {
  if (!process.client) return ''

  const now = Date.now()
  const diff = now - timestamp
  const seconds = Math.abs(Math.floor(diff / 1000))
  const minutes = Math.floor(seconds / 60)
  const hours = Math.floor(minutes / 60)
  const days = Math.floor(hours / 24)

  const isAgo = diff > 0  // 是否是过去时间
  const units = tm('timestampConverter.timeUnits')

  let value, unit
  if (seconds < 60) {
    value = seconds
    unit = units.second
  } else if (minutes < 60) {
    value = minutes
    unit = units.minute
  } else if (hours < 24) {
    value = hours
    unit = units.hour
  } else {
    value = days
    unit = units.day
  }

  return isAgo
    ? t('timestampConverter.timeAgo', { value, unit })
    : t('timestampConverter.timeAfter', { value, unit })
}
```

**逻辑分析**:
1. **时间差计算**: 当前时间 - 目标时间
2. **单位选择**: 自动选择最合适的单位(秒/分/时/天)
3. **方向判断**: 正数为"前",负数为"后"
4. **国际化**: 使用 i18n 支持多语言

### 3.7 完整结果对象

```javascript
const weekdays = tm('timestampConverter.weekdays')
const timezoneLabel = tsOutputTimezone.value === 'local'
  ? `${t('timestampConverter.localTimezone')} (${getTimezoneOffset()})`
  : `${tsOutputTimezone.value} (${getTimezoneOffsetForZone(tsOutputTimezone.value)})`

tsToDateResult.value = {
  timezone: timezoneLabel,           // 时区信息
  local: localTime,                  // 本地时间
  utc: date.toUTCString(),          // UTC 时间
  iso: date.toISOString(),          // ISO 8601 格式
  relative: getRelativeTime(ts),    // 相对时间
  dayOfWeek: weekdays[date.getDay()],  // 星期几
  dayOfYear: getDayOfYear(date),    // 年中第几天
  weekOfYear: getWeekOfYear(date)   // 年中第几周
}
```

## 四、日期转时间戳实现

### 4.1 设置当前时间

```javascript
// 设置为当前时间
const setToNow = () => {
  if (!process.client) return
  const now = new Date()
  const year = now.getFullYear()
  const month = String(now.getMonth() + 1).padStart(2, '0')
  const day = String(now.getDate()).padStart(2, '0')
  const hours = String(now.getHours()).padStart(2, '0')
  const minutes = String(now.getMinutes()).padStart(2, '0')
  const seconds = String(now.getSeconds()).padStart(2, '0')
  dateTimeInput.value = `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
}
```

**格式化技巧**:
- `padStart(2, '0')`: 补齐两位数(如: 9 → 09)
- 月份需要 +1 (getMonth() 返回 0-11)
- 格式: `YYYY-MM-DD HH:mm:ss`

### 4.2 核心转换逻辑

```javascript
const convertDateToTimestamp = () => {
  if (!process.client) return

  if (!dateTimeInput.value) {
    safeMessage.warning(t('timestampConverter.notifications.selectDateTime'))
    return
  }

  try {
    const date = new Date(dateTimeInput.value)

    // 验证日期有效性
    if (isNaN(date.getTime())) {
      safeMessage.error(t('timestampConverter.notifications.invalidDateTime'))
      return
    }

    // 根据时区调整
    let finalDate = date

    if (dateInputTimezone.value === 'UTC') {
      // UTC 时区: 需要加上本地时区偏移
      finalDate = new Date(date.getTime() + date.getTimezoneOffset() * 60000)
    } else if (dateInputTimezone.value !== 'local') {
      // 其他时区: 计算时区差异
      const localDate = date
      const tzString = localDate.toLocaleString('en-US', {
        timeZone: dateInputTimezone.value
      })
      const tzDate = new Date(tzString)
      const offset = localDate.getTime() - tzDate.getTime()
      finalDate = new Date(localDate.getTime() - offset)
    }

    const ms = finalDate.getTime()
    const seconds = Math.floor(ms / 1000)

    dateToTsResult.value = {
      seconds,                    // 秒级时间戳
      milliseconds: ms,           // 毫秒级时间戳
      iso: finalDate.toISOString()  // ISO 8601 格式
    }

    safeMessage.success(t('timestampConverter.notifications.convertSuccess'))
  } catch (err) {
    safeMessage.error(t('timestampConverter.notifications.convertFailed'))
  }
}
```

**时区处理详解**:

1. **本地时区 (local)**:
   - 直接使用用户输入的日期时间
   - 不做任何调整

2. **UTC 时区**:
   - 用户输入的是 UTC 时间
   - 需要加上 `getTimezoneOffset()` 转换为本地时间戳
   - 例: 输入 "2024-01-01 00:00:00 UTC" → 北京时间 "2024-01-01 08:00:00"

3. **其他时区 (如 Asia/Tokyo)**:
   - 计算目标时区与本地时区的偏移量
   - 通过 `toLocaleString()` 转换时区
   - 调整时间戳以反映正确的时间

### 4.3 时区转换原理

```javascript
// 示例: 将 "2024-01-01 12:00:00" 从东京时区转换为时间戳

// 步骤1: 创建本地时间对象
const localDate = new Date('2024-01-01 12:00:00')  // 假设本地是北京时间

// 步骤2: 转换为东京时区的字符串
const tzString = localDate.toLocaleString('en-US', { timeZone: 'Asia/Tokyo' })
// 结果: "1/1/2024, 1:00:00 PM" (东京比北京快1小时)

// 步骤3: 将字符串解析为日期对象
const tzDate = new Date(tzString)

// 步骤4: 计算偏移量
const offset = localDate.getTime() - tzDate.getTime()
// offset = -3600000 (负1小时的毫秒数)

// 步骤5: 应用偏移量
const finalDate = new Date(localDate.getTime() - offset)
```

**核心思想**:
- 通过两次转换计算时区差异
- 利用偏移量调整时间戳
- 确保时间戳代表的是正确的绝对时间

## 五、Date 对象核心 API 总结

### 6.1 创建日期对象

```javascript
// 当前时间
new Date()                          // 当前日期时间
Date.now()                          // 当前时间戳(毫秒)

// 从时间戳创建
new Date(1706425716000)             // 毫秒时间戳
new Date(1706425716 * 1000)         // 秒时间戳需要 * 1000

// 从字符串创建
new Date('2024-01-28')              // ISO 格式
new Date('2024-01-28 12:00:00')     // 日期时间
new Date('Jan 28, 2024')            // 英文格式

// 从参数创建
new Date(2024, 0, 28)               // 年, 月(0-11), 日
new Date(2024, 0, 28, 12, 0, 0)     // 年, 月, 日, 时, 分, 秒
```

### 6.2 获取日期信息

```javascript
const date = new Date()

// 获取年月日
date.getFullYear()      // 年份 (2024)
date.getMonth()         // 月份 (0-11, 0=1月)
date.getDate()          // 日期 (1-31)
date.getDay()           // 星期 (0-6, 0=周日)

// 获取时分秒
date.getHours()         // 小时 (0-23)
date.getMinutes()       // 分钟 (0-59)
date.getSeconds()       // 秒 (0-59)
date.getMilliseconds()  // 毫秒 (0-999)

// 获取时间戳
date.getTime()          // 毫秒时间戳
date.valueOf()          // 同 getTime()

// 时区相关
date.getTimezoneOffset()  // 本地时区与 UTC 的分钟差
```

### 6.3 设置日期信息

```javascript
const date = new Date()

// 设置年月日
date.setFullYear(2024)
date.setMonth(0)        // 0-11
date.setDate(28)

// 设置时分秒
date.setHours(12)
date.setMinutes(30)
date.setSeconds(45)
date.setMilliseconds(500)

// 设置时间戳
date.setTime(1706425716000)
```

### 6.4 格式化输出

```javascript
const date = new Date()

// 标准格式
date.toString()         // "Sun Jan 28 2024 12:00:00 GMT+0800 (中国标准时间)"
date.toDateString()     // "Sun Jan 28 2024"
date.toTimeString()     // "12:00:00 GMT+0800 (中国标准时间)"

// ISO 格式
date.toISOString()      // "2024-01-28T04:00:00.000Z"
date.toJSON()           // 同 toISOString()

// UTC 格式
date.toUTCString()      // "Sun, 28 Jan 2024 04:00:00 GMT"

// 本地化格式
date.toLocaleString()           // "2024/1/28 12:00:00"
date.toLocaleDateString()       // "2024/1/28"
date.toLocaleTimeString()       // "12:00:00"

// 自定义本地化
date.toLocaleString('zh-CN', {
  year: 'numeric',
  month: '2-digit',
  day: '2-digit',
  hour: '2-digit',
  minute: '2-digit',
  second: '2-digit',
  hour12: false,
  timeZone: 'Asia/Shanghai'
})
```
