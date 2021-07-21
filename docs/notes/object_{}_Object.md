
 typescript 中 Object、{} 和 object 的区别
====

时不时看到 React Props 类型声明中出现 `Object`、`{}` 和 `object`，但不是很清楚之间的差别，中文谷歌了一下，发现多篇文章中关于它们的介绍都与自己的验证结果或多或少有出入，所以简单总结一下。

## 结论

1. `{}` 完全等效于 `Object`。

2. 当 typescript 编译器选项（`compilerOptions`） `strictNullChecks` 为 `false` 时，`null` 和 `undefined` 都可以赋值给 `Object` 和 `object`，否则都将发生编译错误。

3. `Object`：表示除 `null` 和 `undefined` 外的所有**值**，包含了原始类型和非原始类型。

4. `object`：表示非原始类型。即除 `number`，`string`，`boolean`，`symbol`，`null`，`undefined` 之外的所有类型。

5. `Object` 和 `object` 却不能够在它上面任意的使用属性和方法，即便它真的有（如 `obj.toFixed()`👇），仅可以使用**所有对象**都存在的属性和方法（如 `constructor`、`toString`、`hasOwnProperty` 等）。

## 验证

typescript 编译选项中 `strictNullChecks` 设置 `true`，从上至下依次执行（比如 `let obj: {}; obj = { prop: 0 }; obj.prop; ...`），结果如下：

| 操作\类型 | `{}` | `Object` | `object` | `{[key: string]: any }`|
| -- | ---- | ---- | ---- | ---- |
| `obj = { prop: 0 }` | ✓ | ✓ | ✓ | ✓ |
| `obj.prop`          | ⨯ | ⨯ | ⨯ | ✓ |
| `obj.constructor`   | ✓ | ✓ | ✓ | ✓ |
| `obj = []`          | ✓ | ✓ | ✓ | ✓ |
| `obj.toString()`    | ✓ | ✓ | ✓ | ✓ |
| `obj = 1`           | ✓ | ✓ | ⨯ | ⨯ |
| `obj.toFixed()`     | ⨯ | ⨯ | ⨯ | ✓ |
| `obj = 'string'`    | ✓ | ✓ | ⨯ | ⨯ |
| `obj = false`       | ✓ | ✓ | ⨯ | ⨯ |
| `obj = null`        | ⨯ | ⨯ | ⨯ | ⨯ |
| `obj = undefined`   | ⨯ | ⨯ | ⨯ | ⨯ |

## 参考
- [Difference between 'object' ,{} and Object in TypeScript](https://stackoverflow.com/questions/49464634/difference-between-object-and-object-in-typescript)



