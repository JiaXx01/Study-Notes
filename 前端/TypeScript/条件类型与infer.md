# 条件类型

在前几章已经了解过了`条件`类型，类似于`三元表达式`，条件类型使用`extends`判断类型的兼容性而非判断类型的全等性

`条件类型`通常与`范型`一起使用`type LiteralType<T> = T extends string ? 'string' : 'other';`

使用条件类型对复杂类型进行比较

```ts
type Fun = (...args: any[]) => any

type FunctionConditionType<T extends Fun> = T extends (...args: any[]) => string
  ? 'A string return func!'
  : 'A non-string return func!'

//  "A string return func!"
type StringResult = FunctionConditionType<() => string>
// 'A non-string return func!';
type NonStringResult1 = FunctionConditionType<() => boolean>
// 'A non-string return func!';
type NonStringResult2 = FunctionConditionType<() => number>
```

通过`FunctionConditionType`判断两个函数类型是否具有兼容性

`条件类型`与`范型约束`都使用了`extends`关键字，但是他们的作用是不同的

- 条件类型中的`extends`用于类型兼容性的判断，就像`if else`
- 范型约束要求传入符合结构的类型参数，相当于作`参数校验`

# infer

`infer`关键字，用于在条件类型中提取类型的某一部分信息，比如说提取函数返回值类型，**`infer`只能在条件类型中使用**

```ts
type Fun = (...args: any[]) => any

type FunctionReturnType<T extends Fun> = T extends (...args: any[]) => infer R ? R : never
```

上面代码表达了，当传入的函数类型满足`(...args: any[]) => infer R`时，返回`infer R`位置的类型，否则返回`never`，这里的`infer R`就是对传入函数类型的返回值类型的推断，`R`就是待推断类型。在进行类型比较时，可以将`infer R`看作`any`

`infer`同样可以用于其他类型结构，比如数组

```ts
type Swap<T extends any[]> = T extends [infer A, infer B] ? [B, A] : T

type SwapResult1 = Swap<[1, 2]> // 符合元组结构，首尾元素替换[2, 1]
type SwapResult2 = Swap<[1, 2, 3]> // 不符合结构，没有发生替换，仍是 [1, 2, 3]
```

可以使用`rest`操作符，处理任意长度的数组

```ts
// 提取首尾成员的类型
type ExtractStartAndEndType<T extends any[]> = T extends [infer Start, ...any[], infer End]
  ? [Start, End]
  : T

// 调换首位成员类型
type SwapStartAndEdnType<T extends any[]> = T extends [infer Start, ...infer Ohter, infer End]
  ? [End, ...Other, Start]
  : T
```

`infer`用于对象

```
// 提取对象属性类型
type PropType<T, K extends keyof T> = T extends { [Key in K]: infer R } ? R : never

// 反转键名与键值
type ReverseKeyValue<T extends Record<string, unknown>> = T extends Record<infer Key, infer Value> ? Record<Value & string, Key> : never

type ReverseKeyValueResult1 = ReverseKeyValue<{ key: "value" }>; // { value: "key" }
```

在反转键名与键值时，使用了`& string`来确保反转后的键值复制字符串类型

```ts
type ReverseKeyValue<T extends Record<string, string>> =
  T extends Record<infer Key, infer Value> ? Record<Value, Key> : never
// 报错；"Type 'Value' does not satisfy the constraint 'string | number | symbol'"
```

如果没有使用`& string`，即使在`范型约束`中约束了键值类型为`string`，还是会出现类型错误。因为范型`Value`是从键值类型推导出来的，`Record<infer Key, infer Value>`这样子使用`infer`推导会导致类型信息丢失，不满足索引签名类型只允许`string | number | symbol`的要求

`infer`用于`Promise`

```ts
type PromiseValue<T> = T extends Promise<infer V> ? V : T

type PromiseValueResult1 = PromiseValue<Promise<number>> // number
type PromiseValueResult2 = PromiseValue<number> // number，但并没有发生提取
```

`infer`用于深层嵌套，采用递归处理任意嵌套深度

```ts
// 单层提取
type PromiseValue<T> = T extends Promise<infer V> ? V : T
type PromiseValueResult = PromiseValue<Promise<Promise<boolean>>> // Promise<boolean>

// 递归提取
type PromiseValue2<T> = T extends Promise<infer V> ? PromiseValue2<V> : T
type PromiseValue2Result = PromiseValue2<Promise<Promise<boolean>>> // Promise<boolean>
```

# 分布式条件类型

`分布式条件类型`也可以称为`条件类型的分布式特性`

```ts
type Condition<T> = T extends 1 | 2 | 3 ? T : never

type Res1 = Condition<1 | 2 | 3 | 4 | 5> // 1 | 2 | 3

type Res2 = 1 | 2 | 3 | 4 | 5 extends 1 | 2 | 3 ? 1 | 2 | 3 | 4 | 5 : never // never
```

上面的`Res1`和`Res2`看起来类型判的逻辑是相似的，只不过一个是以范型参数传给`Condition`另一个是直接进行判断，但是返回的结果却是完全不同。`Res1`更是得到了`1 | 2 | 3`这个奇怪的结果

**上面两个类型的差异：是否通过范型参数传入**

```ts
type Naked<T> = T extends string ? 'yes' : 'no'

type Wrapped<T> = [T] extends [string] ? 'yes' : 'no'

type Res1 = Naked<string | number> // 'yes' | 'no'

type Res2 = Wrapped<string | number> // 'no'
```

上面的`Res2`的类型是字符串字面量`'no'`，因为传入的类型是`string | number`，元组`[T]`接收的类型可能为`number`，所以不兼容`[boolean]`，返回`'no'`；但是`Res1`返回的还是联合类型

**上面两个类型的差异：范型参数是否被包裹**

**分布式条件类型生效的条件**

- 类型参数是联合类型
- 类型参数通过范型传入
- 条件类型中的范型参数不能被包裹

分布式条件类型的效果

- 官方解释：对于属于裸类型参数的检查类型，条件类型会在实例化时期自动分发到联合类型上
- 将传入给条件类型的联合类型拆开，每个分支单独进行条件类型判断，将最后的结果合并起来

官方解释中的`裸类型参数`，就是指范型参数是否完全裸露，没有被包裹。除了使用数组包裹还可以使用其他方式

```ts
export type NoDistribute<T> = T & {} // 包裹范型参数 T

type Wrapped<T> = NoDistribute<T> extends boolean ? 'yes' : 'no'

type Res1 = Wrapped<number | boolean> // 'no'
type Res2 = Wrapped<true | false> // 'yes'
type Res3 = Wrapped<true | false | 599> // 'no'
```

通过裸露范型参数来让分布式条件类型生效；但是有时候，需要通过包裹范型参数来禁用分布式特性。比如说判断联合类型的兼容性

```ts
type Naked<T, U> = T extends U ? 'yes' : 'no'
type Res = Naked<1 | 2, 1 | 2 | 3> // 'yes'
type Res2 = Naked<1 | 2, 1> // 'yes' | 'no'

type CompareUnion<T, U> = [T] extends [U] ? 'yes' : 'no'
type CompareRes1 = CompareUnion<1 | 2, 1 | 2 | 3> // 'yes'
type CompareRes2 = CompareUnion<1 | 2, 1> // 'no'
```

通过`Res2`可以发现，如果没有包裹范型参数，就无法正确判断联合类型的兼容性
