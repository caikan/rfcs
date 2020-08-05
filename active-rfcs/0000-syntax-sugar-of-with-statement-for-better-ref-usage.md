- Start Date: 2020-11-4
- Target Major Version: 2.x / 3.x
- Reference Issues: N/A
- Implementation PR: N/A

Syntax sugar of `with` statement for better `ref` usage

# Summary

本RFC提供一种API和编译方式，让人们可以用符合javascript的直觉去使用`ref`

# Basic example

New API:

```ts
import { reactive, toRefs } from 'vue'

function withRefs<ObjectLiteral>(obj: ObjectLiteral) : ObjectLiteral & {_refs: object} {
  const reactiveObj = reactive(obj)
  const _refs = toRefs(reactiveObj)
  const withObj = Object.create({ _refs })
  Object.keys(_refs).forEach(key => {
    Object.defineProperty(withObj, key, {
      get: () => reactiveObj[key],
      set: val => reactiveObj[key] = val,
    })
  })
  return withObj
}
```

Source:

```js
import { withRefs } from 'vue'
const MyComponent = {
  setup() {
    with(withRefs({
      count: 0
    }))

    const inc = () => count++

    return {..._refs, inc}
  }
}
```

Compiled output

```js
const MyComponent = {
  setup() {
    {
      const _refs = {
        count: ref(0)
      }
      const _refs0 = _refs_0

      const inc = () => _refs_0.count++

      return {..._refs, inc}
    }
  }
}
```

# Motivation

尽可能地简化ref的用法，以符合人们理解js的直觉。
最大限度地兼容js语法，让核心代码甚至可以在浏览器环境中无需修改直接运行。
不创造新的语义概念，以免增加理解成本和造成额外的混淆

# Detailed design

1. New API: `withRefs`
此API配合`with`语句使用，当传入一个对象字面量时，可以产生类似于变量声明的效果。

```ts
const { reactive, toRefs } = require('vue')

function withRefs<ObjectLiteral extends object>(obj: ObjectLiteral): ObjectLiteral & { _refs: ObjectLiteral } {
  const reactiveObj = reactive(obj)
  const _refs = toRefs(reactiveObj)
  const withObj = Object.create({ _refs })
  Object.keys(_refs).forEach(key => {
    Object.defineProperty(withObj, key, {
      get: () => reactiveObj[key],
      set: val => reactiveObj[key] = val,
    })
    withObj[`$${key}`] = _refs[key]
  })
  return withObj
}
```

2. Compiling for 'with' statements
理论上包含`with`语句的代码在非严格模式下是可以直接运行的。
但是为了避免`with`原有的各种缺陷，优化运行时效率，我们应在编译阶段对其进行优化处理。
本RFC的动机是为了取代Composition API烦琐的ref声明和使用方式，因此我们规定
`with`表达式中的`withRefs`API的参数，必须为对象字面量，否则应该抛出编译错误。
`with(withRefs)`语句在编译后产生一个语句块，和一个在此语句块作用域内有效的`_refs`对象，


```js
with(withRefs({
  refName: refValue
})) {
  // with block statements

}
```
Compiled Output
```js
{
  // every `with` block generates a `_refs` object 
  const _refs = {
    key: ref(value)
  }
  // A unique _refs alia named `_refs_${index}` is implicitly auto generated
  // for subsequent references.
  // It is accessible only in the compiled code
  
}
```

# Drawbacks

N/A

# Alternatives

N/A

# Adoption strategy

Fully backwards compatible.

# Unresolved questions

N/A