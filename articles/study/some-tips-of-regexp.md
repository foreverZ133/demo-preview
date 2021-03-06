# 正则小知识记录

基础的用法就不讲咯，下面的列表可以应付大多数场景。

**字面量**，匹配一个具体字符。比如 `/a/` 匹配字符 `"a"`，又比如  `\n` 匹配换行符，又比如 `\.` 匹配小数点。<br />
**字符组**，匹配一个字符，可以是多种可能之一，比如  `[0-9]`，表示匹配一个数字。也有  `\d`的简写形式。另外还有反义字符组，比如  `[^0-9]`和  `\D`等。<br />
**量词**，表示一个字符连续出现，比如  `a{1,3}`表示`"a"`字符连续出现 3 次。还有 `+` `?` `*` 等简写<br />
**锚点**，匹配一个位置，而不是字符。比如 `^` 匹配字符串的开头，还有 `\b` 和 和 `(?=a)`<br />
**分组**，用括号表示一个整体，比如  `(ab)+`，表示"ab"两个字符连续出现多次，也可以使用非捕获分组`(?:ab)+`。<br />
**分支**，多个子表达式多选一，比如  `abc|bcd`，表达式匹配 `"abc"` 或者 `"bcd"` 字符子串。<br />
**反向引用**，比如  `\2`，表示引用第 2 个分组。

### (?:) 匹配但不导出

众所周知，`()` 的匹配内容，在 `replace` `match` 和 `exce` 时会多导出括号中的内容，<br />而如果加上的不是 `()` 而是 `(?:)` 则表示匹配，但不导出。

```javascript
'ABP-123'.match(/(\w+)(-|_|add)(\d+)/); // ["ABP-123", "ABP", "-", "123"]
'ABP-123'.match(/(\w+)(?:-|_|add)(\d+)/); // ["ABP-123", "ABP", "123"]
```

### (?=) 匹配位置

用得少的时候经常将它们混淆，但实际功能很大不同，同时也是较难的用法。

```javascript
'123456'.replace(/(?=\d{3}$)/, ','); // "123,456"
'123456'.replace(/(?=(\d{3})+$)/g, ','); // ",123,456"
'123456'.replace(/\B(?=(\d{3})+$)/g, ','); // "123,456"
'12345.6789'.replace(/\B(?=(\d{3})+$)/g, ','); // "12345.6,789"
'12345.6789'.replace(/\B(?=(\d{3})+\.)/g, ','); // "12,345.6789"
// '12345.6789'.replace(/\B(?=(\d{3})+(\.|$))/g, ','); // "12,345.6,789"
```

第 6 条用了很久很久去尝试，结果无法用正则实现。<br />
其中 `$` 和 `\.` 结尾匹配结果上是包含关系，并非或者关系，所以最后匹配的还是 `$` 结尾的结果，<br />
那么为了实现我们要的效果，就必须是绝对或者的关系，即 `\.\d+` 和 非 `\.\d` 这两个，<br />
然而，后者（即下文匹配非它字符串）尚没有办法完成，所以只能靠多个正则或拆分字符串来匹配前者了。

至于 `(?<=)` `(?!=)` `(?!<)` 也是类似的，带 `<` 表示判断本字符前面的，带 `!` 表示判断不是某字符

### 匹配非它字符串

非它字符比较好弄，`[^a]` 就可以，但如果想匹配 `[^(ab)]` 的效果就很难了。<br />
经过了非常长久的尝试，恐怕真的是做不到。

只能将原字符串拆开或挑选所需的部分，以下是自己曾想到过的处理方式：

```javascript
'12345.6789'
  .split('.')
  .map((x, i) => (i === 1 ? x : x.replace(/\B(?=(\d{3})+$)/g, ',')))
  .join('.');

'12345.6789'.replace(/\d(?=\.)/, match => {
  return x.replace(/\B(?=(\d{3})+$)/g, ',');
});

const str = '1pon-120411-008'; // 120411-008 | ABP-123
const match = str.match(/\d+-\d+/) || str.match(/\w+-\d+/);

'120411add008'.split(/add|-|_/);
```

### 回溯

当向后匹配时，发现没匹配上，就会向前回溯，<br />
可以看作是 `for` 找下一个字符，结果没找到，就又来一个 `for` 向前找，如此反复。<br />
这也是为什么有的正则很卡性能的一个点。

举个例子，用 `/=._"/` 去匹配 `"=abc"ef"`，在 ._ 会一路匹配到最后，<br />
结果没字符了，又还有个 `"` 后引号没找到，就又从最后向前找，直到找到等于号或者后引号。

为了解决这个性能问题，就是要让 `._` 不要去匹配那么多，<br />
比如改为 `[^"]_` 或 `(.(?!="))\*` 都是可行的。

### 回溯形式

#### 贪婪量词

尽可能地多匹配，没匹配就向前寻找

#### 惰性量词

尽可能地少匹配，没匹配就从头寻找

```javascript
const regex = /(\d{1,3})(\d{1,3})/; // 贪婪量词
const regex2 = /(\d{1,3}?)(\d{1,3})/; // 惰性量词
'12345'.match(regex); // ["12345", "123", "45"]
'12345'.match(regex2); // ["1234", "1", "234"]
```

regex2 是先根据 `\d{1,3}?` 匹配到了 `"123"` 并且标记自己有惰性量词，<br />
再 `\d{1,3}` 匹配到 `"45"` 时就没了，又知道前面有惰性量词，因此回溯，<br />
又回到了 `\d{1,3}?`，这回我只匹配 `"1"` 试试，然后 `\d{1,3}` 匹配到 `"234"` 还有余，不错不错，尽可能少了。

非常好记，当有惰性量词时，匹配数量取那个最少的就对了。

但也可见，惰性量词并不意味着就是性能的提升。<br />上例中的 `'=abc"ef'.match(/=.*?"/)`  虽然简约，却不见得是种好的方案。

#### 分支结构

```javascript
'candy'.match(/can|candy/); // ["can"]
'candy'.match(/candy|can/); // ["candy"]
```

其实这里我并没有很理解它的运行模式，算是留了个坑吧。

```javascript
'1pon-120411-008'.match(/(\d+-\d+)|(\w+-\d+)/); // ["1pon-120411"]
```

### 家庭作业

```css
/**
 * 引入字体图标
 */
@font-family {
  font-family: 'xx';
  src: url() format('woff'), url();
}
.icon {
  font-family: 'xx';
  font-size: 10px;
}
/* --- 响应式 ---- */
@media (min-width: 992px) {
  .icon {
    font-size: 14px;
  }
}
```

将上述文本转为如下的 json 格式数据：

```js
[
  {
    "key": "@font-family",
    "attrs": {
      "font-family": "xx",
      "src": "url() format('woff'), url()"
    }
  },
  {
    "key": ".icon",
    "attrs": {
      "font-family": "xx",
      "font-size": "10px"
    }
  },
  {
    "key": "@media (min-width: 992px)",
    "child": [
      {
        "key": ".icon",
        "attrs": {
          "font-size":"14px"
        }
      }
    ]
  }
]
// 参考答案：https://github.com/forever-z-133/others/issues/135
```

<a name="YuDn9"></a>

### 附录

[https://jex.im/regulex/#!flags=&re=%5E(a%7Cb)\*%3F%24](<https://jex.im/regulex/#!flags=&re=%5E(a%7Cb)*%3F%24>)<br />[https://juejin.im/post/5965943ff265da6c30653879](https://juejin.im/post/5965943ff265da6c30653879)

<a name="a4huU"></a>

### 正则题库

[https://alf.nu/RegexGolf](https://alf.nu/RegexGolf)<br />[https://regexone.com/lesson/introduction_abcs](https://regexone.com/lesson/introduction_abcs)<br />[https://regexcrossword.com/](https://regexcrossword.com/)<br />[http://callumacrae.github.io/regex-tuesday/](http://callumacrae.github.io/regex-tuesday/)
