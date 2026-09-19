# Class Behavior

以下结果均基于仓库当前的 `src/api/attributes.ts` 实跑，未修改实现或现有测试。

## 1. 换行分隔后读取并追加同名类

构造与调用：

```ts
import * as cheerio from './src/index.ts';

const $ = cheerio.load('<div class="a\nb"></div>');
const el = $('div');

el.attr('class');
el.hasClass('b');
el.addClass('b');
el.attr('class');
```

逐次读到的值：

- 追加前的 `class`：`"a\nb"`
- `el.hasClass('b')`：`true`
- 追加后的 `class`：`"a\nb b"`
- 追加后的字符串长度：`5`
- 追加后的字符形态：`a␊b␠b`，其中 `␊` 是换行符，`␠` 是空格

所以，`hasClass` 认为第二个类 `b` 已经存在，但 `addClass('b')` 仍然会再写入一个 `b`。

## 2. 向 `apple` 追加带前导空格的 `" a b"`

构造与调用：

```ts
const $ = cheerio.load('<div class="apple"></div>');
const el = $('div');

el.addClass(' a b');
el.attr('class');
```

读到的值：

- 追加后的 `class`：`"apple  a b"`
- JSON 表示：`"apple  a b"`
- 字符串长度：`10`
- 空格总数：`3`
- 字符形态：`apple␠␠a␠b`

空格分布如下：

- `apple` 和 `a` 之间：`2` 个连续空格
- `a` 和 `b` 之间：`1` 个空格
- 开头：`0` 个空格
- 结尾：`0` 个空格

多出的一个空格来自输入开头。`" a b".split(/\s+/)` 会得到 `['', 'a', 'b']`，实现没有过滤第一个空字符串；这个空字符串会额外追加一个空格，最后的 `trim()` 只去掉整体首尾空格，不会去掉中间的连续空格。

## 3. 已存在类 `x` 时再次追加 `x`

构造与调用：

```ts
const $ = cheerio.load('<div class="x"></div>');
const el = $('div');

el.addClass('x');
el.attr('class');
```

读到的值：

- 追加后的 `class`：`"x"`
- 字符串长度：`1`
- 空格总数：`0`

## 4. 两处规则分别是什么

`hasClass` 的规则在 `src/api/attributes.ts:917` 到 `src/api/attributes.ts:920`：

```ts
(idx === 0 || rspace.test(clazz[idx - 1])) &&
(end === clazz.length || rspace.test(clazz[end]))
```

模块顶部的 `rspace` 在 `src/api/attributes.ts:18` 定义为 `/\s+/`。因此，候选类名前一个字符或后一个字符只要是任意 JavaScript 空白字符，就满足边界条件。换行符 `\n` 能通过 `/\s+/` 测试，所以 `"a\nb"` 中的 `b` 被 `hasClass('b')` 判定为存在。

`addClass` 的重复判断在 `src/api/attributes.ts:979` 到 `src/api/attributes.ts:987`：

```ts
let setClass = ` ${className} `;

for (const cn of classNames) {
  const appendClass = `${cn} `;
  if (!setClass.includes(` ${appendClass}`)) setClass += appendClass;
}

setAttr(el, 'class', setClass.trim());
```

这里没有重新按 `rspace` 拆分旧的 `class`，也没有用 `/\s+/` 判断旧值中的类边界；它只是在旧值前后各补一个 ASCII 空格，然后查找精确子串 `` ` ${cn} ` ``。

问题 3 不会重复写入，是因为旧值先变成 `" x "`，而候选 `x` 查找的精确子串正是 `" x "`，已经存在，所以不追加。

问题 1 会重复写入，是因为旧值先变成 `" a\nb "`，候选 `b` 查找的精确子串是 `" b "`。现有 `b` 前面是换行符 `\n`，不是 ASCII 空格，因此 `" b "` 找不到；随后实现追加 `"b "`，最终 `trim()` 得到 `"a\nb b"`。
