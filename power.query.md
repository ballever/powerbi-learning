# M语言

## 写在前面

这一章节的内容完全来自于官方文档，官方的中文文档不太好，我看得很费劲，所以直接看的英文文档，看完后优化了官方中文文档，有些地方加了一些自己的备注。建议有能力直接看官方的英文文档。  

## 简单介绍

### 概述

Microsoft Power Query包含了许多强大的功能,来增强“数据获取”上的体验。它的核心功能是筛选和合并，也称为“mash-up”（混聚），即可以从多种类型的数据源中，提取一个或多个数据合集，并将他们混合汇聚，转换成统一格式，可供后续流程使用的数据。整个数据的的mashup过程，都使用了Power Query 公式语言（通常也称为“M”）。 Power Query 在 Excel 和 Power BI 中嵌入 M 文档，以完成数据的自动化可重复mashup。  
  
本文将介绍 M 语言的规范。我们会通过以下渐进式的步骤逐一介绍每一个块内容：  

1. 词法结构（lexical structure）：  
2. 基础结构的基本概念:值,表达式,环境,变量,标识符,计算模型。  
3. 值（包括基元值和结构化值）的详细规范,这种规范也定义了语言的目标域。  
4. 类型：值具有类型，而类型本身也是一种特殊的值，他们构成了各种值的基本原理并且他们本身携带的元数据信息又可以指明结构化值的形态。  
5. 运算符: M语言中的运算符决定了可以形成哪种形式的表达式。
6. 错误: 运算法或者函数,在表达式的计算过程中，可能会出现错误。虽然错误不是值，但可以通过多种方法将错误映射回值来处理错误。
7. Let: let表达式可以引入辅助定义，以便在更小的步骤中生成复杂的表达式。
8. 函数: 另一种特殊值，为 M 提供丰富标准库的基础，并允许添加新抽象 。
9. 条件语句: 表达式支持条件计算。
10. 分区： 节提供简单的模块化机制 。 （Power Query 尚未使用节。）
11. 合并语法：  

在本章中会简单地介绍以下各个部分的概念以便帮助大家有一个直观的任何和了解，在之后的章节中会更加具体地介绍每块内容。  
如果你是一位经验丰富的程序语言使用者，那么你可以把M语言理解成一种纯粹、高阶、动态类型、部分惰性的函数式编程语言。“the formula language specified in this document is a mostly pure, higher-order, dynamically typed, partially lazy functional language.”  

### 表达式和值  

M语言的核心结构是表达式。通过一个可计算的表达式，来产生一个值。  
尽管许多值都可以按字面形式写成表达式，但值不是表达式。例如，表达式 1 的计算结果为值 1；表达式 1+1 的计算结果为值 2。这个区别非常微妙，但是很重要。表达式是计算的方法；值是计算的结果。  
下面的示例演示了 M 中可用的不同类型的值。通常，使用字面意思直接编写的值，他们可以作为表达式使用，表达式的计算结果就是它本身。 （请注意，// 指示注释的开头，注释延续至行尾。）  

* 原始值（primitive value）是一个单一的值，比如数字，逻辑值，文本或者null。null用来表明数据不存在  

````M
123                  // A number
true                 // A logical
"abc"                // A text
null                 // null value
````

* 列表（list）值是值的有序序列 。 M 支持无限列表，但如果写入的是文本，则列表具有固定长度。 花括号 { 和 } 表示列表的开头和结尾。  

````M
{123, true, "A"}     // list containing a number, a logical, and a text 
{1, 2, 3}            // list of three numbers
````

* 记录（record）是一组字段。字段是名称/值对，其中名称是在字段的记录中唯一的文本值。记录值的文本语法允许将名称写成不带引号的形式，这种形式也称为“标识符”(identifiers)。下面显示了一个记录，其中包含名为 A、B和 C 的三个字段，这些字段具有值 1、2 和 3。  

````M
[
      A = 1,  
      B = 2,  
      C = 3
]
````

* 表(table)是一组按列（按名称标识）和行组织的值。没有用于创建表的字面量语法，但有几个标准函数可用于从列表或记录创建表。  

````M
#table( {"A", "B"}, { {1, 2}, {3, 4} } )
````

这将创建一个形状如下所示的表：  

A | B
---|---
1 | 2
3 | 4
5 | 6

* 函数（function）是一个值，当带着参数进行调用时，将生成一个新值。函数编写的方法是在括号中列出函数的“参数”，后跟箭头符号 => 和定义函数的表达式。该表达式通常引用参数（按参数名称引用）。  

````M
(x, y) => (x + y) / 2
````

### 计算  

在电子表格中，计算的先后顺序是根据单元格中公式的依赖关系来决定的，而M语言的计算模型就是根据表格中的这种计算模型演化而来。  
如果你有excel表格的公式编辑经验，你可能会意识到左侧的公式在计算时会生成右侧的值：  
![value](./power-query/m-spec-formula-value.png)  
  
在M语言中,表达式中的各个部分都可以通过名字来引用表达式的其他部分，整个计算过程将会根据被引用的表达式的计算结果自动决定计算顺序。  
  
我们可以使用一个记录来生成一个与上述电子表格示例等效的表达式。 初始化字段的值时，可以通过使用字段名称引用记录中的其他字段，如下所示：  

````M
[  
    A1 = A2 * 2,  
    A2 = A3 + 1,  
    A3 = 1  
]
````  

上述表达式等效于以下表达式（因为两者都计算出相等的值）：  

````M
[  
    A1 = 4,  
    A2 = 2,  
    A3 = 1  
]
````  

记录可以包含在其他记录中，也可以嵌套在其他记录中 。 我们可以使用“查找运算符” ([]) 按名称访问记录的字段。 例如，以下记录具有一个名为 Sales 的字段（包含一个记录）和一个名为 Total 的字段（用于访问 Sales 记录的 FirstHalf 和 SecondHalf 字段）：  

```M
[  
    Sales = [ FirstHalf = 1000, SecondHalf = 1100 ],
    Total = Sales[FirstHalf] + Sales[SecondHalf]
]
```
  
计算后，上述表达式等效于以下表达式：  

```M
[  
    Sales = [ FirstHalf = 1000, SecondHalf = 1100 ],
    Total = 2100
]
```  

!> 记录是一组key/value组合,我们通过记录的名称和key来找到对应的值  

记录也可以包含在列表中。 我们可以使用“位置索引运算符” ({}) 按其数字索引访问列表中的项目。 从列表的开头开始，使用从零开始的索引来引用列表中的值。 例如，索引 0 和 1 用于引用下面列表中的第一和第二项：  

```M
[
    Sales =  
        {  
            [  
                Year = 2007,  
                FirstHalf = 1000,  
                SecondHalf = 1100,
                Total = FirstHalf + SecondHalf // 2100
            ],
            [  
                Year = 2008,  
                FirstHalf = 1200,  
                SecondHalf = 1300,
                Total = FirstHalf + SecondHalf // 2500
            ]  
        },
    TotalSales = Sales{0}[Total] + Sales{1}[Total] // 4600
]
```

!> {}表示列表,要访问列表中的元素使用索引,列表的索引编号是从0开始的,即访问列表的第一个元素,是{0}  

列表和记录中的成员的表达式（以及后面引入的 let 表达式）使用“延迟计算(lazy evaluation)”进行计算，这意味着它们只会根据需要进行计算。所有其他表达式都使用“迫切计算(eager evaluation)”进行计算，这意味着如果在计算过程中遇到它们，则将立即对其进行计算。考虑这一点的一种好方法是记住计算列表或记录表达式将返回一个列表或记录值，该值本身会记住在请求时（查找或索引运算符）需如何计算其列表项或记录字段。  

!> let包裹的内容,以及列表和记录表中表达式,都是延迟计算,在没被调用之前,只是记录了表达式的内容,不会去计算表达式的值,只有在被其他"迫切计算"表达式调用时才会使用表达式去计算值。  
比如let包括的内容只有在使用in的时候会调用计算let里的内容。  

### 函数  

在 M 中，函数的做用是将一组输入值映射到单个输出值。 函数的编写方法是，首先命名所需的一组输入值（函数的参数），然后在“转到”(=>) 符号后面提供表达式,该表达式使用这些输入值（函数的主体）来计算函数的结果。 例如：  

```M
(x) => x + 1                    // function that adds one to a value
(x, y) =>  x + y                // function that adds two values
```

函数是一个值，就像数字或文本值一样。 以下示例演示一个函数，展示了其作为字段Add的一个值,而后被其他几个字段调用或者执行。调用函数时，将指定一组值，这些值会在逻辑上替换函数正文表达式中所需的输入值。  

```M
[
    Add = (x, y) => x + y,
    OnePlusOne = Add(1, 1),     // 2
    OnePlusTwo = Add(1, 2)      // 3
]
```

### 库

M包含了一组已经定义好的标准库，可以在表达式中直接使用，简称为库。这些定义由一组固定名称的值组成。库里这些固定名称的值可以在表达式中直接使用，无需在表达式中明确声明。比如：  

```M
Number.E                        // Euler's number e (2.7182...)
Text.PositionOf("Hello", "ll")  // 2
```

### 运算符  

M包含了一组运算符，可以在表达式中使用。将运算符运用于操作对象即形成符号表达式。比如，在表达式 “1 + 2”中，数字“1”和“2”是操作对象，“+”是相加运算法。  
运算符的含义可以根据操作对象的类型而变化。 例如，加号运算符可用于数字以外的值类型：  

```M
1 + 2                   // numeric addition: 3
#time(12,23,0) + #duration(0,0,2,0) // time arithmetic: #time(12,25,0)
```

另一个根据操作对象不同而代表不同含义的运算符示例是连接运算符 (&)：  

```M
"A" & "BC"              // text concatenation: "ABC"
{1} & {2, 3}            // list concatenation: {1, 2, 3}
[ a = 1 ] & [ b = 2 ]   // record merge: [ a = 1, b = 2 ]
```

请注意，运算符不一定支持某些值的连接。 例如： 

```M
1 + "2"  // error: adding number and text is not supported
```

### 元数据

我们可以将一个值与另一个值进行关联,把这种关联信息存储起来,存储这种关联信息的数据就称为元数据。元数据会被表现为一条记录，称为元数据记录。元数据记录中的字段可以用来存储一条值的元数据。  
每一个值都有元数据记录，如果一条数据没有指定元数据记录，那么它的元数据记录就是空。  
元数据记录提供了一种非介入式的方法将附加信息关联到任何值上面。将元数据记录与值相关联不会更改该值或其行为。  
使用语法 x meta y 表示将元数据记录值 y 与现有的值 x 相关联。例如，以下将带有 Rating 和 Tags 字段的元数据记录与文本值 "Mozart" 相关联：  

```M
"Mozart" meta [ Rating = 5, Tags = {"Classical"} ]
```

对于已经包含非空元数据记录的值，应用 meta 的结果是计算现有和新的元数据记录的记录合并的结果。 例如，下面两个表达式是等价的：  

```M
("Mozart" meta [ Rating = 5 ]) meta [ Tags = {"Classical"} ] 
"Mozart" meta ([ Rating = 5 ] & [ Tags = {"Classical"} ])
```

可以使用 Value.Metadata 函数访问一个给定值的元数据记录。 在下面的示例中，ComposerRating 字段中的表达式访问 Composer 字段中值的元数据记录，然后访问元数据记录的 Rating 字段。  

```M
[ 
    Composer = "Mozart" meta [ Rating = 5, Tags = {"Classical"} ], 
    ComposerRating = Value.Metadata(Composer)[Rating] // 5
]
```

### Let表达式  

目前为止展示的很多示例,都是一个结果表达式中包含了所有文本表达式（一行里面有一串的表达式组合）。 “let”表达式允许一组值进行计算、分配名称，然后在“in”后面的后续表达式中使用 。 例如，在我们的销售数据示例中，可以执行以下操作：  

```M
let 
    Sales2007 =  
        [  
            Year = 2007,  
            FirstHalf = 1000,  
            SecondHalf = 1100, 
            Total = FirstHalf + SecondHalf // 2100 
        ], 
    Sales2008 =  
        [  
            Year = 2008,  
            FirstHalf = 1200,  
            SecondHalf = 1300, 
            Total = FirstHalf + SecondHalf // 2500 
        ] 
  in Sales2007[Total] + Sales2008[Total] // 4600
```

上述表达式的结果是一个数字值 (4600)，该值是根据绑定到名称 Sales2007 和 Sales2008 的值计算得出的。

### If表达式

if 表达式根据两个表达式的逻辑结果进行选择。 例如：

```M
if 2 > 1 then
    2 + 2
else  
    1 + 1
```

如果逻辑表达式 (2 > 1) 为 true，则选择第一个表达式 (2 + 2)；如果为 false，则选择第二个表达式 (1 + 1)。 将对选定的表达式（在本例中为 2 + 2）进行计算，并成为 if 表达式 (4) 的结果。  

### 错误  

一个错误表示一个表达式的过程无法产生值。  
错误是由运算符和函数遇到错误情况，或使用了错误表达式导致的。可以使用 try 表达式来处理错误。引发某一错误时，会指向引起这个错误的值，此值可用于指示错误发生的原因。  

```M
let Sales = 
    [ 
        Revenue = 2000, 
        Units = 1000, 
        UnitPrice = if Units = 0 then error "No Units"
                    else Revenue / Units 
    ],
    UnitPrice = try Number.ToText(Sales[UnitPrice])
in "Unit Price: " &
    (if UnitPrice[HasError] then UnitPrice[Error][Message]
    else UnitPrice[Value])
```

上面的示例访问 Sales[UnitPrice] 字段,对值进行格式化并产生结果：  

```M
"Unit Price: 2"
```

如果 Units 字段为零，UnitPrice 字段会引发错误，而 try 表达式则会处理此错误。 结果值将为：  

```M
"No Units"
```

try 表达式将正确的值和错误转换为一个记录值，这条记录可以值指示 try 表达式是否处理了错误，以及在处理错误时是否返回相对应的正确处理结果和错误处理结果。 例如，请考虑以下引发错误，然后立即进行处理的表达式：  

```M
try error "negative unit count"
```

上面的表达式计算结果为以下嵌套的记录值。这也解释了之前单价示例中的 [HasError]、[Error] 和 [Message] 字段为什么可以直接使用。  

```M
[ 
    HasError = true, 
    Error = 
        [ 
            Reason = "Expression.Error", 
            Message = "negative unit count", 
            Detail = null 
        ] 
]
```

!> 当使用try时会自动创建一条记录,你可以直接访问try表达式中的字段。单价示例中try Number.ToText(Sales[UnitPrice])的结果是  
```M
//WHEN uNITS = 1000
[ 
    HasError = false,
    Value = 2
]

//WHEN uNITS = 0
[ 
    HasError = true, 
    Error = 
        [ 
            Reason = "Expression.Error", 
            Message = "No Units", 
            Detail = null 
        ] 
]
```

处理错误常见的方式是使用默认值替换错误。 try 表达式可以与一个可选的 otherwise 子句一起使用，从而以紧凑的形式实现：  

```M
try error "negative unit count" otherwise 42
// 42 当错误发生时,返回42
// if true then "value" else 42  
```

## 词法结构(Lexical Structure)  

### 文档  

M 文档是 Unicode 字符的有序序列。 M 允许在 M 文档的不同部分使用不同类别的 Unicode 字符。 有关 Unicode 字符类的信息，请参阅 Unicode 标准，版本 3.0 中的第 4.5 节。  
文档要么由一个表达式组成，要么由组织成节的多组定义构成 。 第 10 章对节进行了详细说明。 从概念上讲，以下步骤用于从文档中读取表达式：  

* 文档根据其字符编码方案被解码为一个 Unicode 字符序列。
* 执行词法分析，从而将 Unicode 字符流转换为令牌流。 本节余下的小节将介绍词法分析。
* 执行词法分析，从而将令牌流转换为可计算的形式。 后续部分将介绍此过程。

### 语法约定  

词法和句法用语法产生式表示。每种语法产生式将非终端符号以及非终端符号的可能扩展定义为非终端符号或者终端符号序列。在语法生产式中，非终端符号_non-terminal symbols_显示为斜体，终端符号显示为固定长度字体。  

文法产生式的第一行是定义的非终端符号的名称，后跟冒号。 每一个后续缩进行都包含一个非终端符的可能扩展，该非终端符由一系列非终端符或终端符符号组成。 例如，产生式：  

_if-expression_:  

　　if _if-condition_ then _true-expression_ else _false-expression_  

定义一个 if-expression 由令牌 if 后跟 if-condition，令牌 then 后跟 true-expression 以及令牌 else 后跟 false-expression 组成 。  

当非终端符号有多个可能的扩展时，不同的扩展在单独的行中列出。 例如，产生式：  

_variable-list_：  
　　_variable_  
　　_variable-list_ , _variable_  

定义一个 variable-list，它由一个变量组成，也可以由另一个variable-list后跟variable 组成。 换言之，这个定义是递归的，它指定变量列表由一个或多个（用逗号分隔的）变量组成。  

下标后缀“opt”用于指示可选符号。 产生式：  

_field-specification_:  

　　optional<sub>opt</sub> field-name = field-type  
为以下的简写：  

_field-specification_:  

　　field-name = field-type  
　　optional field-name = field-type  

并定义了 field-specification中以终端符号optional开头的可选参数，后跟 field-name、终端符号 = 和 field-type。  

替代项通常在单独的行中列出，但在有许多替代项的情况下，可以在单独的一行里列出所有替代项，并在这一行前面使用“one of”。 这是对在单独的行中列出每个替代项的简化。 例如，产生式：  

_decimal-digit_: one of  
　　0 1 2 3 4 5 6 7 8 9  

为以下的简写：  
_decimal-digit_:  

　　0  
　　1  
　　2  
　　3  
　　4  
　　5  
　　6  
　　7  
　　8  
　　9  

### 词法分析  

在词法级别，M 文档由一系列空白区域、注释和令牌元素组成 。 以下各节将介绍这些产生式。 在语法中，只有令牌元素是有意义的。  

lexical-unit:  
　　lexical-elements<sub>opt</sub>  
lexical-elements:  
　　lexical-element  
　　lexical-element  
　　lexical-elements  
lexical-element:  
　　whitespace  
　　token comment  

### 空白区域

空格用于分隔 M 文档中的注释和令牌。 空格包括空格字符（它是 Unicode 类 Zs 的一部分），以及水平和垂直制表符、换页符和换行符序列。 换行字符序列包括回车符、换行符、后跟换行符的回车符、下一行和段落分隔符。  

空格：  
　　带有 Unicode 类 Zs 的任何字符  
　　水平制表符字符 (U+0009)  
　　垂直制表符 (U+000B)  
　　换页符 (U+000C)  
　　后跟换行符 (U+000A) 的回车符 (U+000D)  
　　new-line-character  
new-line-character：  
　　回车符 (U+000D)  
　　换行符 (U+000A)  
　　换行符 (U+0085)  
　　行分隔符 (U+2028)  
　　段落分隔符 (U+2029)  

为了与添加 end-of-file 标记的源代码编辑工具兼容，并使文档能够被看作正确终止的行序列，将按顺序对 M 文档应用以下转换：  

* 如果文档的最后一个字符是 Control-Z 字符 (U+001A)，则删除此字符。  
* 如果文档非空并且文档的最后一个字符不是回车符 (U+000D)、换行符 (U+000A)、行分隔符 (U+2028) 或段落分隔符 (U+2029)，则在文档末尾添加回车符 (U+000D)。  

### 注释  

支持两种形式的注释：单行注释和分隔注释。 单行注释以字符 // 开头，并扩展到//所在行的末尾。 分隔注释以字符 /\* 开头，以字符 \*/ 结尾。分隔注释可能跨多行。  

comment:  
　　single-line-comment  
　　delimited-comment  
single-line-comment:  
　　// single-line-comment-characters<sub>opt</sub>  
single-line-comment-characters:  
　　single-line-comment-character single-line-comment-characters<sub>opt</sub>  
single-line-comment-character:  
　　Any Unicode character except a new-line-character  
delimited-comment:  
　　/\* delimited-comment-text<sub>opt</sub> asterisks \*/  
delimited-comment-text:  
　　delimited-comment-section delimited-comment-text<sub>opt</sub>  
delimited-comment-section:  
　　/  
　　asterisks<sub>opt</sub> not-slash-or-asterisk  
asterisks:  
　　\* asterisks<sub>opt</sub>  
not-slash-or-asterisk:  
　　Any Unicode character except \* or /  

注释不能嵌套。在当行注释//中使用/\* 和 \*/没有什么特殊意思，在分割注释/\* 和 \*/中使用单行注释//，//也没有特殊意思  
在注释内的文本文字不会被处理。  

### 令牌

令牌是标识符、关键字、文字、运算符或标点符号。 空白和注释用于分隔标记，但不会将其视为令牌。
token:  
　　identifier  
　　keyword  
　　literal  
　　operator-or-punctuator  

#### 字符转义序列  

M 文本值可以包含任意 Unicode 字符。 然而，文字文本仅限于图形字符，需要对非图形字符使用转义序列。 例如，要在文字文本中包含回车符、换行符或制表符，可以分别使用 #(cr)、#(lf) 和 #(tab) 转义序列。  

```M
"a#(lf)b"

/* 返回
a
b
*/
```

若要在文字文本中嵌入转义序列开始字符 #(，# 本身需要进行转义：  

```M
"#(#)("
//返回 #(
```

单个转义序列中可以包含多个转义码，用逗号分隔；因此，以下两个序列是等效的：  

```M
#(cr,lf) 
#(cr)#(lf)
```

下面介绍了转移序列的机制  

character-escape-sequence:  
　　#( escape-sequence-list )  
escape-sequence-list：  
　　single-escape-sequence  
　　single-escape-sequence , escape-sequence-list  
single-escape-sequence：  
　　long-unicode-escape-sequence  
　　short-unicode-escape-sequence  
　　control-character-escape-sequence  
　　escape-escape  
long-unicode-escape-sequence：  
　　hex-digit hex-digit hex-digit hex-digit hex-digit hex-digit hex-digit hex-digit  
short-unicode-escape-sequence：  
　　hex-digit hex-digit hex-digit hex-digit  
control-character-escape-sequence：  
　　control-character  
control-character：  
　　cr  
　　lf  
　　tab  
escape-escape:  
　　#  

#### 文本  

文字文本是值的源代码展示形式  

literal:  
　　logical-literal  
　　number-literal  
　　text-literal  
　　null-literal  
　　verbatim-literal  

##### null文本  

null 文本用于写入 null 值。 null 值表示不存在的值。  

null-literal:  
　　null  

##### 逻辑文本  

逻辑文本用于写入值 true 和 false，并生成逻辑值。

logical-literal:  
　　true  
　　false  

##### 数字文本

数字文字用于写入数字值并生成数值。  

number-literal:  
      decimal-number-literal  
      hexadecimal-number-literal  
decimal-number-literal:  
      decimal-digits . decimal-digits exponent-part<sub>opt</sub>  
      . decimal-digits exponent-part<sub>opt</sub>  
      decimal-digits exponent-part<sub>opt</sub>  
decimal-digits:  
      decimal-digit decimal-digits<sub>opt</sub>  
decimal-digit: one of  
      0 1 2 3 4 5 6 7 8 9  
exponent-part:  
      e signopt decimal-digits  
      E signopt decimal-digits  
sign: one of  
      + -  
hexadecimal-number-literal:  
      0x hex-digits  
      0X hex-digits  
hex-digits:  
      hex-digit hex-digits<sub>opt</sub>  
hex-digit: one of  
      0 1 2 3 4 5 6 7 8 9 A B C D E F a b c d e f  

请注意，如果数字文本中包含小数点，则它后面必须至少有一个数字。 例如，1.3 是数字文本，但 1. 和 1.e3 不是。  

##### 文字文本

文本文字用于写入 Unicode 字符序列并生成文本值。  
text-literal:  
      " text-literal-characters<sub>opt</sub> "  
text-literal-characters：  
      text-literal-character text-literal-characters<sub>opt</sub>  
text-literal-character：  
      single-text-character  
      character-escape-sequence  
      double-quote-escape-sequence  
single-text-character：  
      除后跟 ( (U+0028) 的 " (U+0022) 或 # (U+0023) 外的任何字符  
double-quote-escape-sequence:  
      "" (U+0022, U+0022)  

若要在文本值中包含引号，请重复使用引号，如下所示：  

```M
"The ""quoted"" text" 
// The "quoted" text
```  

可使用 character-escape-sequence 产生式在文本值中写入字符，而无需在文档中将它们直接编码为 Unicode 字符。 例如，回车和换行可以用文本值写入：  

```M
"Hello world#(cr,lf)A"  
/*
Hello world
A
*/
```

#### 逐字文本

逐字文本用于存储用户作为代码输入但无法正确分析为代码的 Unicode 字符序列。 在运行时，它会生成一个错误值。  (不明白)

verbatim-literal:  
      #!" text-literal-characters<sub>opt</sub> "  

#### 标识符  

标识符是用于引用值的名称。 标识符可以是常规标识符，也可以是带引号的标识符。  

identifier:  
　　regular-identifier  
　　quoted-identifier  
regular-identifier:  
　　available-identifier  
　　available-identifier dot-character regular-identifier  
available-identifier:  
　　A keyword-or-identifier that is not a keyword  
keyword-or-identifier：  
　　identifier-start-character identifier-part-characters<sub>opt</sub>  
identifier-start-character：  
　　letter-character  
　　underscore-character  
identifier-part-characters：  
　　identifier-part-character identifier-part-characters<sub>opt</sub>  
identifier-part-character：  
　　letter-character  
　　decimal-digit-character  
　　underscore-character  
　　connecting-character  
　　combining-character  
　　formatting-character  
dot-character：  
　　. (U+002E)  
underscore-character:  
　　_ (U+005F)  
letter-character:  
　　Lu、Ll、Lt、Lm、Lo 或 Nl 类的 Unicode 字符  
combining-character:  
　　Mn 或 Mc 类的 Unicode 字符  
decimal-digit-character:  
　　Nd 类的 Unicode 字符  
connecting-character:  
　　Pc 类的 Unicode 字符  
formatting-character:  
　　Cf 类的 Unicode 字符 
带引号的标识符可用于允许零个或多个 Unicode 字符的任何序列用作标识符，包括关键字、空格、注释、运算符和标点符号。   
quoted-identifier:  
　　#" text-literal-characters<sub>opt</sub> "  
注意，转义序列和用于转义引号的双引号可以在带引号的标识符中使用，就像在 text-literal 中一样 。  
以下示例对包含空格字符的名称使用标识符引号：  

```M
[ 
    #"1998 Sales" = 1000, 
    #"1999 Sales" = 1100, 
    #"Total Sales" = #"1998 Sales" + #"1999 Sales"
]
```

以下示例使用标识符引号将 + 运算符包含在标识符中：  

```M
[ 
    #"A + B" = A + B, 
    A = 1, 
    B = 2 
]
```

##### 通用标识符  

在 M 中有两个地方不会应为包含空格或关键字或数字文字的标识符引起的歧义。 这两个地方分别是记录中的字段名称，以及在字段访问运算符 ([ ]) 中，M 允许这样的标识符，而不必使用带引号的标识符。　　

```M
[ 
    Data = [ Base Line = 100, Rate = 1.8 ], 
    Progression = Data[Base Line] * Data[Rate]
]
```  

用于命名和访问字段的标识符称为通用标识符，定义如下：  
generalized-identifier：  
　　generalized-identifier-part  
　　generalized-identifier 仅用空格分隔 (U+0020)  
generalized-identifier-part:  
　　generalized-identifier-segment  
　　decimal-digit-character generalized-identifier-segment  
generalized-identifier-segment:  
　　keyword-or-identifier  
　　keyword-or-identifier dot-character keyword-or-identifier  

#### 关键字  

_关键字_是保留的类似标识符的字符序列，不能用作标识符，除非使用[标识引用机制](/power.query?id=标识符)或允许使用[通用标识符](power.query?id=%e9%80%9a%e7%94%a8%e6%a0%87%e8%af%86%e7%ac%a6)。  
keyword: one of  
　　and as each else error false if in is let meta not null or otherwise  
　　section shared then true try type #binary #date #datetime  
　　#datetimezone #duration #infinity #nan #sections #shared #table #time  

#### 运算符和标点符号  

有多种运算符和标点符号。 表达式中使用运算符来描述涉及一个或多个操作对象的操作。 例如，表达式 a + b 使用 + 运算符添加两个操作对象 a 和 b。 标点符号用于分组和分隔。  
operator-or-punctuator: one of  
　　, ; = < <= > >= <> + - * / & ( ) [ ] { } @ ! ? => .. ...　　

## 基本概念  

### 值(value)  

单个数据称为_值(value)_。 广义上讲，有两个常规类别的值：基元值和结构化值。前者是值的最基本形式(atomic)，后者由基元值和其他结构化值构成。 例如，值  

```M
1 
true
3.14159 
"abc"
```

是基元，因为它们不由其他值构成。 但是，值  

```M
{1, 2, 3} 
[ A = {1}, B = {2}, C = {3} ]
```

是使用基元值进行构造的，在这条记录中，是使用其他结构化值构造的。  

### 表达式(expression)  

_表达式_是用于构造值的公式。 表达式可以使用多种语法结构形成。 下面是一些表达式示例。 每一行都是一个单独的表达式。

```M
"Hello World"             // a text value 
123                       // a number 
1 + 2                     // sum of two numbers 
{1, 2, 3}                 // a list of three numbers 
[ x = 1, y = 2 + 3 ]      // a record containing two fields: x and y 
(x, y) => x + y           // a function that computes a sum 
if 2 > 1 then 2 else 1    // a conditional expression 
let x = 1 + 1  in x * 2   // a let expression 
error "A"                 // error with message "A"
```

如上所示，最简单的表达式形式，文字本身就是值。  
更复杂的表达式由其他表达式（称为 sub-expressions）组成。 例如：  

```M
1 + 2
```

上面的例子实际上又3个表达式组成，文字_1_和_2_是表达式_1+2_的子表达式。  
在表达式中执行由句法结构定义好的算法，称为_计算_表达式。每种类型的表达式都具有其计算规则。 例如，文字表达式（如 1）将生成一个常数值，而表达式 a + b 将通过计算其他两个表达式（a 和 b）来获取生成的值，并根据一组规则将它们相加。  

### 环境和变量



## 值(value)  

## 类型

## 运算符  

## Let  

### Let表达式(expression)

一个let表达式可以用来获取变量的中间计算结果的值。  

let-expression:  
　　let variable-list in expression  
variable-list:  
　　variable  
　　variable , variable-list  
variable:  
　　variable-name = expression  
variable-name:  
　　identifier  

下面的示例显示要计算的中间结果，这些结果存储在变量 x、y 和 z 中，以供在后续计算 x + y + z 中使用：  

```M
let     x = 1 + 1,
        y = 2 + 2,     
        z = y + 1 
in
        x + y + z
```  

此表达式的结果为：  

```M
11  // (1 + 1) + (2 + 2) + (2 + 2 + 1)
```  

在计算 let-expression 中的表达式时，存在以下情况：  

* 变量列表中的表达式定义了一个新的作用域，其中包含来自 variable-list 产生式的标识符，并且在计算 variable-list 产生式中的表达式时必须存在 。 variable-list 中的表达式可能相互引用。
* 在计算 let-expression 中的表达式之前，必须先计算 variable-list 中的表达式。
* 除非使用了 variable-list 中的表达式，否则不能对其进行计算。
* 传播在计算 let-expression 中的表达式期间引发的错误。
let 表达式可以看作是隐式记录表达式的语法糖。下面的表达式与上面的表达式等效：  

```M
[     x = 1 + 1,
      y = 2 + 2,
      z = y + 1,
      result = x + y + z 
][result]
```

## 条件语句  

## 函数

## 错误处理

M语言表达式的计算，只会产生两种输出结果：  

* 生成单个值。
* 生成错误信息，来表明表达式在计算的过程中无法产生一个值。一个错误包含了一个记录，这条记录里的内容可提供更详细的信息来表明是什么导致计算无法完成。

可以在表达式内部产生错误，也可以在表达式内部直接处理错误。

### 引发错误

引发错误语法：  
error-raising-expression:  
　　error expression

文本值可作为错误表达式的简写形式：  

```M
error "Hello, world" // error with message "Hello, world"
```

完整的错误值是一条记录，可以用Error.Record来构造这条记录：  

```M
error Error.Record("FileNotFound", "File my.txt not found","my.txt")
```

上面的表达式等价于：  

```M
error [ 
    Reason = "FileNotFound", 
    Message = "File my.txt not found", 
    Detail = "my.txt" 
]
```

!> 直接使用文本的时候，实际上是记录了error的message为那个文本  

引发一个错误，将导致当前在计算的表达式停止，此时与这个错误关联的表达式都会展开，直到：  

* 错误信息传播到了记录中的一个字段、分区中的一个成员、let内的一个变量，这里统称为条目，错误信息会被一起保存在那个条目里，然后扩散。对该条目的任何后续访问都将导致引发相同的错误。 记录、节或 let 表达式的其他条目不一定会受到影响（除非它们访问之前标记为有错误的条目）。  
* 已达到顶级表达式。 在这种情况下，计算顶级表达式的结果是一个错误而不是一个值。  
* 已达到 try 表达式。 在这种情况下，将捕获错误并以值的形式返回。  

### 处理错误  

处理错误的语法：  
error-handling-expression:  
　　try protected-expression otherwise-clauseopt  
protected-expression:  
　　expression  
otherwise-clause:  
　　otherwise default-expression  
default-expression:  
　　expression  

在不使用_otherwise-clause_情况下，有可能出现以下几种情况：  

* 如果_protected-expression_的计算结果不会引发错误，并且产生了值x，那么_error-handling-expression_产生的结果是一条如下形式的记录：  

```M
[ HasErrors = false, Value = x ]
```

* 如果_protected-expression_的计算结果引发了错误，并且产生了错误值e，那么_error-handling-expression_产生的结果是一条如下形式的记录：  

```M
[ HasErrors = true, Error = e ]
```

在使用_otherwise-clause_情况下，有可能出现以下几种情况：  

* 必须在_otherwise-clause_之前计算_protected-expression_。  
* 当且仅当计算 protectedexpression 引发错误时，才必须计算_otherwise-clause_。
* 如果在计算 protectedexpression 引发错误，_otherwise-clause_计算出来的结果就是最终的_error-handling-expression_的结果  
* 在计算_otherwise-clause_的时所产生的错误会扩散  

下面的示例演示了在没有引发错误的情况下的 error-handling-expression：  

```M
let
    x = try "A"
in
    if x[HasError] then x[Error] else x[Value] 
// "A"
```

下面的示例演示了引发错误，然后对其进行处理：  

```M
let
    x = try error "A" 
in
    if x[HasError] then x[Error] else x[Value] 
// [ Reason = "Expression.Error", Message = "A", Detail = null ]
```

try表达式处理产生的错误结果可由otherwise来替代：  

```M
try error "A" otherwise 1 
// 1
```

如果 otherwise 子句也引发错误，那么整个 try 表达式也会引发错误：  

```M
try error "A" otherwise error "B" 
// error with message "B"
```

### 初始化记录和let时的错误

下面的示例显示了一个记录初始值设定项，其中字段 A 引发错误，并且由两个其他字段 B 和 C 访问。 字段 B 不处理 A 引发的错误，但是 C 会处理此错误。 最终字段 D 不访问 A，因此不受 A 中的错误影响。  

```M
[ 
    A = error "A", 
    B = A + 1,
    C = let x =
            try A in
                if not x[HasError] then x[Value]
                else x[Error], 
    D = 1 + 1 
]
```

计算以上表达式的结果是：

```M
[ 
    A = // error with message "A" 
    B = // error with message "A" 
    C = "A", 
    D = 2 
]
```
M 中的错误处理应在接近错误原因的位置执行，以处理惰性字段初始化以及延迟闭包计算的影响。下面这个例子展示了尝试使用try进行错误处理时，最终并没有得到想要的错误处理结果。  

```M
let
    f = (x) => [ a = error "bad", b = x ],
    g = try f(42) otherwise 123
in 
    g[a]  // error "bad"
```

在此示例中，定义 g 用于处理调用 f 时引发的错误。 但是，错误发生在一个由程序初始化的字段中，那么只有在这个初始化字段被调用时，才会引发错误。因此函数f产生的记录_[ a = error "bad", b = 42 ]_，直接传递给了try，由于a没有被调用，因此try的结果是么有error，直接返回了这条记录。  
当最终访问a，即g[a]时，才引发了错误。  

### 不执行的错误  

当编写表达式的时候，作者可能会希望忽略表达式的部分内容不去执行，但是仍然希望表达式的其他部分可以正常执行。一种方法是当执行进入这部分不想被执行的部分时，引发一个错误。比如：  

```M
(x, y) =>
     if x > y then
         x - y
     else
         error Error.Record("Expression.Error", 
            "Not Implemented")
```

省略号（...）可用作error的快捷方式。  

not-implemented-expression:
　　...  

例如，下面的示例等效于前面的示例：  

```M
(x, y) => if x > y then x - y else ...
```  

## 分区  

## 合并语法
