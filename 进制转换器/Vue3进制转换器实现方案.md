# 进制转换器核心逻辑解析

本文解析本项目中进制转换器的核心 JavaScript 实现逻辑。

在线工具网址：[https://see-tool.com/base-converter](https://see-tool.com/base-converter)

工具截图：
![在这里插入图片描述](Snipaste_2026-01-23_19-03-32.png.png)

## 1. 核心设计思想：十进制中转站

为了实现二进制、八进制、十进制、十六进制的**实时同步转换**，我们采用了"十进制为中心"的策略。

无论用户修改哪个输入框，逻辑流程统一为：
1.  **输入解析**：将当前输入的进制转换为 **十进制** (Decimal)。
2.  **统分发**：将这个 **十进制** 数转换为 **所有其他进制** 并更新对应的输入框。

这种设计避免了 $N \times N$ 的复杂转换关系，将复杂度降低为 $O(N)$。

## 2. 关键代码实现

### 输入处理 (`onSyncInput`)

这是最核心的函数。它接收一个 `source` 参数（如 `'binary'`, `'hex'`），指明数据来源。

```javascript
const onSyncInput = (source) => {
  if (isUpdating) return // 1. 防止循环触发
  isUpdating = true

  let decimalNum = 0
  
  // 2. 解析为十进制
  // 使用 parseInt(string, radix)
  if (source === 'hex') {
     // ...清洗数据，移除 0x 前缀...
     decimalNum = parseInt(cleanValue, 16)
  } 
  // ...其他进制同理

  // 3. 分发到其他进制
  // 使用 number.toString(radix)
  if (source !== 'binary') binaryValue.value = formatValue(decimalNum.toString(2), 2)
  if (source !== 'octal')  octalValue.value = formatValue(decimalNum.toString(8), 8)
  // ...
  
  isUpdating = false
}
```

### 格式化展示 (`formatValue`)

为了提升体验，生成的数值需要经过格式化处理：
*   **前缀**：如二进制加 `0b`，十六进制加 `0x`。
*   **分组**：二进制每4位一组，十进制每3位一组（千分位），从右向左切割。

```javascript
const formatValue = (value, base) => {
  // ...前缀处理...
  
  // 数字分组逻辑（从右向左）
  if (digitGrouping.value && formatted.length > 4) {
    // 决定分组长度：十进制/八进制3位，二进制/十六进制4位
    const groupSize = (base === 10 || base === 8) ? 3 : 4
    
    // 利用循环切割字符串
    // ...
  }
  return formatted
}
```

## 3. 自定义进制 (2-36进制)

对于 "自定义转换" 模式，利用了 JS 原生 API 的强大能力：
*   **parseInt(str, radix)**: 支持 2-36 进制的解析。
*   **num.toString(radix)**: 支持 2-36 进制的输出。

这使得我们可以轻松实现如 "7进制转32进制" 这种非常规需求，只需两行核心代码：

```javascript
// 任意进制 -> 十进制
const decimal = parseInt(input, fromBase)
// 十进制 -> 目标进制
const result = decimal.toString(toBase)
```

## 总结

本工具的核心在于充分利用 JavaScript 原生的 `parseInt` 和 `toString` 方法，配合精巧的"十进制中转"架构，以极少的代码量实现了强大且健壮的多进制同步转换功能。