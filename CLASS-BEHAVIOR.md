# Class Behavior

以下结果均按当前仓库的 `src/index.ts` 实跑，读取方式为 Cheerio 的 `hasClass()`
和 `attr('class')`。

## 一、换行分隔后再追加同名类

构造元素时，让 `class` 属性中的两个类名之间包含一个实际换行符：

```js
const $ = load('<div class="first\nsecond"></div>');
const div = $('div');

div.hasClass('second');
div.addClass('second');
div.attr('class');
```

实跑结果：

- 初始 `div.attr('class')` 是 `"first\nsecond"`，其中 `\n` 是换行符 `U+000A`。
- `div.hasClass('second')` 是 `true`。
- 追加后 `div.attr('class')` 的实际字符串是：

```text
first
second second
```

- 该结果按 JS/JSON 字符串显示为 `"first\nsecond second"`。
- 结果长度是 `19`；原有换行符仍在，新增部分是一个空格和第二个 `second`。

## 二、现有 `apple`，追加带前导空格的 ` a b`

构造和调用：

```js
const $ = load('<div class="apple"></div>');
const div = $('div');

div.addClass(' a b');
div.attr('class');
```

实跑结果：

```text
apple  a b
```

- 精确值是 `"apple  a b"`。
- 总共有 `3` 个空格。
- `apple` 和 `a` 之间有 `2` 个空格。
- `a` 和 `b` 之间有 `1` 个空格。
- 属性开头有 `0` 个空格，末尾有 `0` 个空格。
- 结果长度是 `10`。

原因是 `' a b'.split(/\s+/)` 得到
`['', 'a', 'b']`。实现会把开头的空字符串也当成一个待追加项，并以一个空格表示；最后的
`trim()` 只移除属性最外侧的空白，不会移除中间多出的空格。

## 三、现有 `x`，再追加 `x`

构造和调用：

```js
const $ = load('<div class="x"></div>');
const div = $('div');

div.addClass('x');
div.attr('class');
```

实跑结果：

- `div.attr('class')` 是 `"x"`。
- 结果长度是 `1`。
- 同一个类名没有写第二遍。

## 四、两处判断规则和差异原因

### `hasClass()` 的存在性判断

`hasClass()` 位于 `src/api/attributes.ts:905`。它在 `src/api/attributes.ts:917`
到 `src/api/attributes.ts:920` 要求目标类名前后满足以下任一条件：

- 位于属性开头或结尾；或
- 相邻字符能通过 `rspace` 匹配。

`rspace` 在 `src/api/attributes.ts:18` 定义为
`/\s+/`。因此空格、换行符、制表符等空白字符都会被当成类名边界。第一个问题里的
`"first\nsecond"` 中，`second` 前面是换行符，所以 `hasClass('second')` 返回
`true`。

### `addClass()` 的重复判断

`addClass()` 位于 `src/api/attributes.ts:948`。关键规则是：

- `src/api/attributes.ts:967` 用 `value.split(rspace)` 拆分传入的新类名。
- `src/api/attributes.ts:979`
  把现有属性格式化为字面空格包裹的字符串：`` ` ${className} ` ``。
- `src/api/attributes.ts:984` 用 `setClass.includes(` ${appendClass}`)`
  判断重复，等价于查找字面字符串 `" 类名 "`。
- `src/api/attributes.ts:987` 最后调用 `trim()`，只去掉结果首尾的空白。

这里的重复判断使用的是字面空格 `U+0020`，不是 `/\s+/`。第一个问题里，现有
`second` 前面是换行符；包装后的字符串中没有字面片段 `" second "`，所以
`addClass('second')` 认为它不存在并再次追加。第三个问题里，现有 `x` 被包装成
`" x "`，字面片段 `" x "` 能被找到，所以不会追加第二遍。

第二个问题的额外空格也来自这套字面拼接：前导空白拆分产生的空类名被追加成一个空格，随后
`a` 和 `b` 各自再按 `"类名 "` 拼接，最终形成 `"apple  a b"`。
