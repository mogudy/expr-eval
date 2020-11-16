JavaScript Expression Evaluator
===============================

This is a modified version of https://github.com/silentmatt/expr-eval
这是基于Matthew Crumley的expr-eval修改而成。

Description
-------------------------------------

处理数学表达式和有限的字符串、日期

支持的运算符、函数见后表

安装
-------------------------------------

    npm install @nbxx/nb-expr-eval

基本用法
-------------------------------------
```js
    var Parser = require('@nbxx/nb-expr-eval').Parser;

    var parser = new Parser();
    var expr = parser.parse('2 * x + 1');
    console.log(expr.evaluate({ x: 3 })); // 7

    // or
    Parser.evaluate('6 * x', { x: 7 }) // 42
```
使用文档
-------------------------------------

* [Parser](#parser)
    - [Parser()](#parser-1)
    - [parse(expression: string)](#parseexpression-string)
    - [Parser.parse(expression: string)](#parserparseexpression-string)
    - [Parser.evaluate(expression: string, variables?: object)](#parserevaluateexpression-string-variables-object)
* [Expression](#expression)
    - [evaluate(variables?: object)](#evaluatevariables-object)
    - [substitute(variable: string, expression: Expression | string | number)](#substitutevariable-string-expression-expression--string--number)
    - [simplify(variables: object)](#simplifyvariables-object)
    - [variables(options?: object)](#variablesoptions-object)
    - [symbols(options?: object)](#symbolsoptions-object)
    - [toString()](#tostring)
    - [toJSFunction(parameters: array | string, variables?: object)](#tojsfunctionparameters-array--string-variables-object)
* [Expression Syntax](#expression-syntax)
    - [Operator Precedence](#operator-precedence)
    - [Unary operators](#unary-operators)
    - [Array literals](#array-literals)
    - [Pre-defined functions](#pre-defined-functions)
    - [Custom JavaScript functions](#custom-javascript-functions)
    - [Constants](#constants)
### 解析器 ###

核心，包含 `parse` 方法及一些静态方法用于解析表达式。

#### Parser()

构造一个新的 `Parser` 实例。

构造器可以接受一个可选的 `options` 参数。

下面的例子会创建一个不允许比较，也不允许逻辑运算符，但允许 `in` 运算符的解析器：

```js
    var parser = new Parser({
      operators: {
        // 默认开启，也可以手动关闭
        add: true,
        concatenate: true,
        conditional: true,
        divide: true,
        factorial: true,
        multiply: true,
        power: true,
        remainder: true,
        subtract: true,

        // 关闭逻辑运算，包含： and, or, not, <, ==, !=, 等等
        logical: false,
        comparison: false,

        // 关闭 in 以及 = 操作符
        'in': false,
        assignment: false
      }
    });
```
#### parse(expression: string)

把表达式解析为 `Expression` 对象

#### Parser.parse(expression: string)

 `new Parser().parse(expression)` 的静态等价方法

#### Parser.evaluate(expression: string, variables?: object)

使用 `variables` 对象中的变量和方法来解析、计算表达式

`Parser.evaluate(expr, vars)` 等价于 `Parser.parse(expr).evaluate(vars).`

### Expression ###

`Parser.parse(str)` 返回一个 `Expression` 对象， `Expression` 对象与 js 函数很相似，实际上也可以被直接转换为函数。

#### evaluate(variables?: object)

使用 `variables` 中的变量来计算 `expression` 对象, 如果有变量未被解析，则抛出异常。
```js
    js> expr = Parser.parse("2 ^ x");
    (2^x)
    js> expr.evaluate({ x: 3 });
    8
```
#### substitute(variable: string, expression: Expression | string | number)

使用 `expression` 来替换原有 `expression` 中的 `variable`。
```js
    js> expr = Parser.parse("2 * x + 1");
    ((2*x)+1)
    js> expr.substitute("x", "4 * x");
    ((2*(4*x))+1)
    js> expr2.evaluate({ x: 3 });
    25
```
#### simplify(variables: object)

使用 `variables` 来替换表达式中的变量，从而简化表达式。函数不会被替换和简化。

实际上simplify只是简单的把变量替换了一下，然后将常数直接计算出结果。
所以 `((2*(4*x))+1)` 是没法直接简化的，除非替换 x 。但是 `2*4*x+1` 是可以简化的，
因为它等价于`(((2*4)*x)+1)`, 所以 `(2*4)` 会被简化为 "8", 生成结果 `((8*x)+1)`.
```js
    js> expr = Parser.parse("x * (y * atan(1))").simplify({ y: 4 });
    (x*3.141592653589793)
    js> expr.evaluate({ x: 2 });
    6.283185307179586
```
#### variables(options?: object)

列出当前表达式中尚未被简化替换的变量。.
```js
    js> expr = Parser.parse("x * (y * atan(1))");
    (x*(y*atan(1)))
    js> expr.variables();
    x,y
    js> expr.simplify({ y: 4 }).variables();
    x
```
 `variables` 默认只返回最顶层的变量，例如 `Parser.parse(x.y.z).variables()` 返回 `['x']`。
 如果希望获得细节，可以使用 `{ withMembers: true }`，随后 `Parser.parse(x.y.z).variables({ withMembers: true })`
 将会返回`['x.y.z']`.

#### symbols(options?: object)

获取尚未简化的变量，以及所有函数
```js
    js> expr = Parser.parse("min(x, y, z)");
    (min(x, y, z))
    js> expr.symbols();
    min,x,y,z
    js> expr.simplify({ y: 4, z: 5 }).symbols();
    min,x
```
与 `variables` 相似, `symbols` 接受可选参数 `{ withMembers: true }` 来显示对象成员.

#### toString()

将表达式转化为字符串 `toString()` 将自动添加括号。.

#### toJSFunction(parameters: array | string, variables?: object)

把 `Expression` 转换为js函数。 `parameters` 是参数名列表（Array），或者参数名字符串（逗号分隔的列表）

如果提供了可选参数 `variables` 表达式将会被简化。
```js
    js> expr = Parser.parse("x + y + z");
    ((x + y) + z)
    js> f = expr.toJSFunction("x,y,z");
    [Function] // function (x, y, z) { return x + y + z; };
    js> f(1, 2, 3)
    6
    js> f = expr.toJSFunction("y,z", { x: 100 });
    [Function] // function (y, z) { return 100 + y + z; };
    js> f(2, 3)
    105
```
### 表达式语法 ###

基本上和js语法差不多，除了部分运算符为数学表达式外。例如： `^` 运算符代表指数函数，而不是异或。

#### 运算符优先级（从上到下依次递减）

运算符                 | 相依性 | 说明
:----------------------- | :------------ | :----------
(...)                    | None          | 分组
f(), x.y, a[i]           | Left          | 方法调用、成员变量访问、数组下标访问
!                        | Left          | 阶乘
^                        | Right         | 乘方
+, -, not, sqrt, etc.    | Right         | 一元前置运算符 (列表见后文)
\*, /, %                 | Left          | 乘以、除以、取余
+, -, \|\|               | Left          | 加、减、列表合并
==, !=, >=, <=, >, <, in | Left          | 等于、不等于、大于等于、小于等于、大于、小于、"in" 作为中间运算符，代表左侧操作数在右侧操作数之内
and                      | Left          | 逻辑 AND
or                       | Left          | 逻辑 OR
x ? y : z                | Right         | 三元运算符 (if x then y else z)
=                        | Right         | 变量赋值
;                        | Left          | 表达式分隔符

"in" 和 = 两个运算符默认关闭，需要显式开启才能使用
```js
    const parser = new Parser({
      operators: {
        'in': true,
        'assignment': true
      }
    });
    // 现在可以使用 'x in array' 和 'y = 2*x' 两种表达式了
```
#### 一元运算符
解析器包含一些内置函数，实际上会作为一元运算符来使用。他们和预定义函数的区别是他们只接受一个参数，
而且不需要括号包围起来。包含括号的一元运算符优先级和函数一致，不包含括号的一元运算符优先级按运算符优先级来计算（低于 '^'）。
例如，`sin(x)^2` 等价于 `(sin x)^2`, 而 `sin x^2` 等价于 `sin(x^2)`。
 `+` 和 `-` 两种一元运算符是例外，因为不存在对应的函数，所以优先级始终按运算符优先级来计算。

运算符 | 说明
:------- | :----------
-x       | 负数
+x       | 一元加号。可以将一个操作数转为数字类型
x!       | 对于正整数是阶乘，对于非正整数是 gamma(x + 1) 。
abs x    | x 的绝对值
acos x   | 反余弦 x  (in radians)
acosh x  | 反双曲余弦 x (in radians)
asin x   | 反正弦 x (in radians)
asinh x  | 反双曲正弦 x (in radians)
atan x   | 反正切 x (in radians)
atanh x  | 反双曲正切 x (in radians)
cbrt x   | 开三次方 x
ceil x   | 向上取整 x， 大于等于 x 的最小整数
cos x    | 余弦 x (x is in radians)
cosh x   | 双曲余弦 x (x is in radians)
exp x    | 以 e 为底数的指数/反对数函数，等价于 e^x
expm1 x  | e^x - 1
floor x  | 向下取整 x，小于等于 x 的最大的整数
length x | x 的字符串长度
ln x     | x 的自然对数
log x    | x 的自然对数
log10 x  | x 的对数 （底数为 10 的对数）
log2 x   | x 的对数 （底数为 2 的对数）
log1p x  | (1+x) 的自然对数
not x    | 逻辑 NOT
round x  | 舍入取整，使用四舍五入法
sign x   | x 的正负。-1 代表负数、0 代表 0、1 代表 正数
sin x    | 正弦 x (x is in radians)
sinh x   | 双曲正弦 x (x is in radians)
sqrt x   | 平方根，如果 x 为负数， 结果为 NAN
tan x    | 正切 x (x is in radians)
tanh x   | 双曲正切 x (x is in radians)
trunc x  | 直接取整，舍去小数部分。x 为正数时向下取整， x 为负数时向上取整

#### 预定义函数

除了运算符外，还有一些预定义的函数。预定义函数不会被 simplify 处理


函数     | 说明
:------------ | :----------
random(n)     | 获取 [0,n) 之间的随机数，如果 n = 0，或者未提供 n ,则默认使用 1 代替 n
fac(n)        | n! (factorial of n: "n * (n-1) * (n-2) * … * 2 * 1") Deprecated. Use the ! operator instead.
min(a,b,…)    | 列表中最小的数字
max(a,b,…)    | 列表中最大的数字
hypot(a,b)    | 勾股定理。斜边长, a, b 分别指直角三角形的两个直角边，结果是直角三角形斜边长.
pyt(a, b)     | 等价于 hypot.
pow(x, y)     | 等价于 x^y
atan2(y, x)   | atan( x/y ). i.e. 坐标系中点 (0, 0) 到点 (x, y) 连线与x轴之间夹角. ( result is in radians)
roundTo(x, n) | 将 x 四舍五入 n 位小数.
map(f, a)     | 列表映射。遍历列表 a 的每个元素 x，并计算 f(x) 作为新列表的元素值
fold(f, y, a) | 列表汇聚。遍历列表 a 的每个元素 x、序号 i，给定一个初始值，计算 f(y, x, i) 作为新列表的元素。最后计算新列表的和。
filter(f, a)  | 列表筛选。遍历列表 a 的每个元素 x、序号 i，只保留 f(x, i) 为 true 的元素
indexOf(x, a) | 返回列表/字符串中第一个匹配的元素的下标，没有找到则返回 -1
join(sep, a)  | 使用 sep 拼接 列表 a 中的全部字符串元素
if(c, a, b)   | 三元运算符 c ? a : b 的 function 形式。但总会计算全部分支，如果对性能有要求应该直接使用三元运算符。
substr(str, start, length)  | 获取字符串 str 中从 start 开始持续 length 个字符，如果不填 length 则获取从 start 开始到字符串结尾

#### 列表处理

列表能直接使用 [] 定义，例如：
    [ 1, 2, 3, 2+2, 10/2, 3! ]

#### 自定义函数

可以使用 `name(params) = expression` 的格式来自定义函数。函数支持多参数

Examples:
```js
    square(x) = x*x
    add(a, b) = a + b
    factorial(x) = x < 2 ? 1 : x * factorial(x - 1)
```

#### 自定义 js 函数

parser 有一个属性 functions ，记录了所有的函数。如果想要在全局的角度自定义函数，只需要添加即可
```js
    const parser = new Parser();

    // Add a new function
    parser.functions.customAddFunction = function (arg1, arg2) {
      return arg1 + arg2;
    };

    // Remove the factorial function
    delete parser.functions.fac;

    parser.evaluate('customAddFunction(2, 4) == 6'); // true
    //parser.evaluate('fac(3)'); // This will fail
```
#### 预定义常量

以下是预定义的常量:

常量     | 说明
:----------- | :----------
E            | 等价于 js 的 `Math.E`
PI           | 等价于 js 的 `Math.PI`
true         | 逻辑 `true`
false        | 逻辑 `false`

与函数类似，预定义常量也保存在 parser 属性中。可以随意在 `parser.consts` 中添加自定义的常量。比如：
```js
    const parser = new Parser();
    parser.consts.R = 1.234;

    console.log(parser.parse('A+B/R').toString());  // ((A + B) / 1.234)
```
To disable the pre-defined constants, you can replace or delete `parser.consts`:
```js
    const parser = new Parser();
    parser.consts = {};
```

### Tests ###

1. `cd` to the project directory
2. Install development dependencies: `npm install`
3. Run the tests: `npm test`
