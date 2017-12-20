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

    npm install nb-expr-eval

基本用法
-------------------------------------

    var Parser = require('nb-expr-eval').Parser;

    var parser = new Parser();
    var expr = parser.parse('2 * x + 1');
    console.log(expr.evaluate({ x: 3 })); // 7

    // or
    Parser.evaluate('6 * x', { x: 7 }) // 42

使用文档
-------------------------------------

### 解析器 ###

核心，包含 `parse` 方法及一些静态方法用于解析表达式。

#### Parser()

构造一个新的 `Parser` 实例。

构造器可以接受一个可选的 `options` 参数。

下面的例子会创建一个不允许比较，也不允许逻辑运算符，但允许 `in` 运算符的解析器：

    var parser = new Parser({
      operators: {
        // These default to true, but are included to be explicit
        add: true,
        concatenate: true,
        conditional: true,
        divide: true,
        factorial: true,
        multiply: true,
        power: true,
        remainder: true,
        subtract: true,

        // Disable and, or, not, <, ==, !=, etc.
        logical: false,
        comparison: false,

        // The in operator is disabled by default in the current version
        'in': true
      }
    });

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

    js> expr = Parser.parse("2 ^ x");
    (2^x)
    js> expr.evaluate({ x: 3 });
    8

#### substitute(variable: string, expression: Expression | string | number)

使用 `expression` 来替换原有 `expression` 中的 `variable`。

    js> expr = Parser.parse("2 * x + 1");
    ((2*x)+1)
    js> expr.substitute("x", "4 * x");
    ((2*(4*x))+1)
    js> expr2.evaluate({ x: 3 });
    25

#### simplify(variables: object)

使用 `variables` 来替换表达式中的变量，从而简化表达式。函数不会被替换和简化。

实际上simplify只是简单的把变量替换了一下，然后将常数直接计算出结果。
所以 `((2*(4*x))+1)` 是没法直接简化的，除非替换 x 。但是 `2*4*x+1` 是可以简化的，
因为它等价于`(((2*4)*x)+1)`, 所以 `(2*4)` 会被简化为 "8", 生成结果 `((8*x)+1)`.

    js> expr = Parser.parse("x * (y * atan(1))").simplify({ y: 4 });
    (x*3.141592653589793)
    js> expr.evaluate({ x: 2 });
    6.283185307179586

#### variables(options?: object)

列出当前表达式中尚未被简化替换的变量。.

    js> expr = Parser.parse("x * (y * atan(1))");
    (x*(y*atan(1)))
    js> expr.variables();
    x,y
    js> expr.simplify({ y: 4 }).variables();
    x

 `variables` 默认只返回最顶层的变量，例如 `Parser.parse(x.y.z).variables()` 返回 `['x']`。
 如果希望获得细节，可以使用 `{ withMembers: true }`，随后 `Parser.parse(x.y.z).variables({ withMembers: true })` 
 将会返回`['x.y.z']`.

#### symbols(options?: object)

获取尚未简化的变量，以及所有函数

    js> expr = Parser.parse("min(x, y, z)");
    (min(x, y, z))
    js> expr.variables();
    min,x,y,z
    js> expr.simplify({ y: 4, z: 5 }).variables();
    min,x

与 `variables` 相似, `symbols` 接受可选参数 `{ withMembers: true }` 来显示对象成员.

#### toString()

将表达式转化为字符串 `toString()` 将自动添加括号。.

#### toJSFunction(parameters: array | string, variables?: object)

把 `Expression` 转换为js函数。 `parameters` 是参数名列表（Array），或者参数名字符串（逗号分隔的列表）

如果提供了可选参数 `variables` 表达式将会被简化。

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

### 表达式语法 ###

基本上和js语法差不多，除了部分运算符为数学表达式外。例如： `^` 运算符代表指数函数，而不是异或。

#### 运算符优先级（从上到下依次递减）

Operator                 | Associativity | Description
:----------------------- | :------------ | :----------
(...)                    | None          | Grouping
f(), x.y                 | Left          | Function call, property access
!                        | Left          | Factorial
^                        | Right         | Exponentiation
+, -, not, sqrt, etc.    | Right         | Unary prefix operators (see below for the full list)
\*, /, %                 | Left          | Multiplication, division, remainder
+, -, \|\|               | Left          | Addition, subtraction, concatenation
==, !=, >=, <=, >, <, in | Left          | Equals, not equals, etc. "in" means "is the left operand included in the right array operand?" (disabled by default)
and                      | Left          | Logical AND
or                       | Left          | Logical OR
x ? y : z                | Right         | Ternary conditional (if x then y else z)

 `in` 运算符在当前版本被默认禁用。如果需要使用，先构造一个 `Parser` 实例并将 `operators.in` 设置为 `true`。
  例如：

    var parser = new Parser({
      operators: {
        'in': true
      }
    });
    // Now parser supports 'x in array' expressions

#### 一元运算符

解析器包含一些内置函数，实际上会作为一元运算符来使用。他们和预定义函数的区别是他们只接受一个参数，
而且不需要括号包围起来。包含括号的一元运算符优先级最高，不包含括号的一元运算符优先级仅次于 '^'。
例如，`sin(x)^2` 等价于 `(sin x)^2`, 而 `sin x^2` 等价于 `sin(x^2)`。

 `+` 和 `-` 两种一元运算符是例外，因为不存在对应的函数，所以优先级永远最高。.

运算符 | 说明
:------- | :----------
-x       | 负数
+x       | 将操作数转化为数字类型。
x!       | 对于正整数是阶乘，对于非正整数是 gamma(x + 1) 。
abs x    |  x 的绝对值
acos x   | 反余弦，x 等于弧度
acosh x  | 反双曲余弦，x 等于弧度
asin x   | 反正弦，x 等于弧度
asinh x  | 反双曲正弦，x 等于弧度
atan x   | 反正切，x 等于弧度
atanh x  | 反双曲正切，x 等于弧度
ceil x   | 向上取整
cos x    | 余弦，x 等于弧度
cosh x   | 双曲余弦，x 等于弧度
exp x    | 指数函数，等价于 e^x 
floor x  | 向下取整
length x |  x 的字符串长度
ln x     |  x 的自然对数
log x    |  x 的自然对数
log10 x  |  x 的常用对数
not x    | 逻辑否
round x  | 舍入取整，使用四舍五入法
sin x    | 正弦，x 等于弧度
sinh x   | 双曲正弦，x 等于弧度
sqrt x   | 平方根，如果 x 为负数， 结果为 NAN
tan x    | 正切，x 等于弧度
tanh x   | 双曲正切，x 等于弧度
trunc x  | 直接取整，舍去小数部分。x 为正数时向下取整， x 为负数时向上取整

#### 预定义函数

除了运算符外，还有一些预定义的函数。预定义函数不会被 simplify 处理

函数     | 说明
:----------- | :----------
random(n)    | 获取 [0,n) 之间的随机数，如果 n = 0，或者未提供 n ,则默认使用 1 代替 n
fac(n)       | 等价于 n! (factorial of n: "n * (n-1) * (n-2) * … * 2 * 1") Deprecated. Use the ! operator instead.
min(a,b,…)   | 列表中最小的数字
max(a,b,…)   | 列表中最大的数字
hypot(a,b)   | 斜边长, a, b 分别指直角三角形的两个直角边，结果是直角三角形斜边长.
pyt(a, b)    | 等价于 hypot
pow(x, y)    | 等价于 x^y
atan2(y, x)  | atan( x/y ). i.e. 坐标系中点 (0, 0) 到点 (x, y) 连线与x轴之间夹角.
if(c, a, b)  | 三元表达式 c ? a : b 的 function 形式
roundTo(x, n)  | 将 x 四舍五入 n 位小数.
month(date)  | 获取 date 的月份 
year(date)  | 获取 date 的年份
now(format)  | 获取现在的时间并以 format 格式输出，不填 format 则以ISO 8601格式输出
substr(str, start, length)  | 获取字符串 str 中从 start 开始持续 length 个字符，如果不填 length 则获取从 start 开始到字符串结尾
datediff(date1, date2, unit)  | 返回 date1 到 date2 之间的时间差， unit 可选范围：years, months, weeks, days, hours, minutes, seconds.

### Tests ###

1. `cd` to the project directory
2. Install development dependencies: `npm install`
3. Run the tests: `npm test`
