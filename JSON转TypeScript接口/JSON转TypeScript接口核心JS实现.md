# JSON转TypeScript接口核心JS实现

这篇只讲核心 JS 逻辑：一段 JSON 是如何一步步变成 TypeScript 接口代码的。

> 在线工具网址：[https://see-tool.com/json-to-typescript](https://see-tool.com/json-to-typescript)  
> 工具截图：  
> ![工具截图](工具截图.png)

## 1）状态与入口函数

先看最核心的状态和入口：

```js
const jsonInput = ref('')
const outputData = ref('')
const interfaceName = ref('RootObject')
const useType = ref(false)
const optionalProps = ref(true)
const errorMessage = ref('')

const convert = () => {
  const input = jsonInput.value.trim()
  if (!input) {
    outputData.value = ''
    errorMessage.value = ''
    return
  }

  try {
    const parsed = JSON.parse(input)
    if (parsed === null || (typeof parsed !== 'object' && !Array.isArray(parsed))) {
      errorMessage.value = '根节点必须是对象或数组'
      outputData.value = ''
      return
    }
    const rootName = interfaceName.value.trim() || 'RootObject'
    outputData.value = jsonToTypeScript(parsed, rootName)
    errorMessage.value = ''
  } catch (error) {
    errorMessage.value = `JSON 解析失败：${error.message}`
    outputData.value = ''
  }
}
```

这里做了三件事：输入清洗、JSON 解析、调用生成器。只要这一步跑通，工具就能稳定输出。

## 2）类型名生成：先合法，再去重

JSON 字段名常常不规范，比如有空格、短横线、数字开头，所以类型名要先标准化：

```js
const toPascalCase = (value, fallback = 'Item') => {
  const cleaned = String(value || '').replace(/[^A-Za-z0-9_$\s]/g, ' ').trim()
  if (!cleaned) return fallback
  const parts = cleaned.split(/\s+/).filter(Boolean)
  const combined = parts.map(part => part.charAt(0).toUpperCase() + part.slice(1)).join('')
  return /^[A-Za-z_$]/.test(combined) ? combined : fallback
}

const normalizeInterfaceName = (value, fallback = 'RootObject') => {
  if (!value) return fallback
  return toPascalCase(value, fallback) || fallback
}
```

嵌套对象会不断生成新接口名，所以还要处理重名：

```js
const interfaceNames = new Set()
const processedNames = new Set()

const generateInterfaceName = (base, suffix = '') => {
  const baseNormalized = normalizeInterfaceName(base, 'RootObject')
  const suffixNormalized = suffix ? toPascalCase(suffix) : ''
  const name = `${baseNormalized}${suffixNormalized}`

  if (!interfaceNames.has(name) && !processedNames.has(name)) {
    interfaceNames.add(name)
    return name
  }

  let counter = 2
  while (interfaceNames.has(`${name}${counter}`) || processedNames.has(`${name}${counter}`)) {
    counter += 1
  }
  const finalName = `${name}${counter}`
  interfaceNames.add(finalName)
  return finalName
}
```

## 3）类型推断：递归处理对象和数组

真正的核心在 `inferType`：

```js
const inferType = (value, key, parentName) => {
  if (value === null) return 'null'

  if (Array.isArray(value)) {
    if (value.length === 0) return 'any[]'
    const elementTypes = new Set(value.map(item => inferType(item, 'Item', parentName)))
    const types = Array.from(elementTypes)
    const inner = types.length === 1 ? types[0] : `(${types.join(' | ')})`
    return `${inner}[]`
  }

  if (typeof value === 'object') {
    const nestedName = generateInterfaceName(parentName, key)
    processObject(value, nestedName)
    return nestedName
  }

  if (typeof value === 'string') return 'string'
  if (typeof value === 'number') return 'number'
  if (typeof value === 'boolean') return 'boolean'
  return 'any'
}
```

比如 `[{ id: 1 }, { id: "2" }]` 会得到 `(RootItem | RootItem2)[]` 或联合类型形式，保证类型信息不丢。

## 4）对象转声明文本

推断完类型后，要拼成最终代码：

```js
const formatPropertyKey = key => {
  const value = String(key)
  const identifier = /^[A-Za-z_$][A-Za-z0-9_$]*$/
  return identifier.test(value) ? value : JSON.stringify(value)
}

const processObject = (obj, name) => {
  if (processedNames.has(name)) return
  processedNames.add(name)

  const optionalMark = optionalProps.value ? '?' : ''
  const lines = []
  lines.push(useType.value ? `export type ${name} = {` : `export interface ${name} {`)

  Object.entries(obj).forEach(([key, value]) => {
    const tsType = inferType(value, key, name)
    const propertyKey = formatPropertyKey(key)
    lines.push(`  ${propertyKey}${optionalMark}: ${tsType};`)
  })

  lines.push('}')
  interfaces.push(lines.join('\n'))
}
```

如果根节点本身是数组，再补一行根类型：

```js
if (Array.isArray(json)) {
  const arrayType = inferType(json, 'Item', baseName)
  const rootLine = `export type ${baseName} = ${arrayType};`
  return interfaces.length ? `${interfaces.join('\n\n')}\n\n${rootLine}` : rootLine
}
```

## 5）实时转换触发

输入实时更新，但不希望每敲一个字都立即解析，所以用了防抖：

```js
let debounceTimer = null
const scheduleConvert = () => {
  if (debounceTimer) clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    convert()
  }, 400)
}

watch(jsonInput, scheduleConvert)
watch(interfaceName, scheduleConvert)
watch([useType, optionalProps], convert)
```

最终效果就是：输入 JSON、改根接口名、切换 `interface/type`、切换可选属性，结果区都会自动刷新。

## 6）完整思路总结

这套实现本质是三层：

1. **输入层**：解析 JSON、处理错误
2. **推断层**：递归判断数据类型、生成嵌套接口名
3. **输出层**：拼接 TypeScript 声明文本

把这三层拆开后，代码可读性会很高，读者也能很容易定位：哪里在“解析”、哪里在“推断”、哪里在“输出”。
