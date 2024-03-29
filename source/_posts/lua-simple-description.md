---
title: Lua 语言简述
date: 2019-05-23 10:28:52
tags: [lua]
---
## Lua 介绍
Lua 是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放。
其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。
### 环境安装（mac）
使用 homebrew ：
{% codeblock [homebrew 安装 lua 环境] %}
$ brew install lua
{% endcodeblock %}
安装完成后，打开terminal，执行如下命令，启动lua交互式编程
{% codeblock [启动 lua 交互式编程] %}
$ lua -i
Lua 5.3.5  Copyright (C) 1994-2018 Lua.org, PUC-Rio
>
{% endcodeblock %}
### 数据类型
Lua 中有 8 个基本类型分别为：nil、boolean、number、string、userdata、function、thread 和 table。

| 数据类型       | 描述           | 示例           |
| :-----------: |:--------------|:--------------|
| nil           | 只有值 nil 属于该类，表示一个无效值，在条件表达式中相当于 false | nil |
| boolean       | 逻辑值 | true/false |
| number        | 双精度类型浮点数(支持10进制科学计数法) | 4/4.3/0.2/2e+1/0.4e-1//898989e-06 |
| string        | 字符串，添加一对双引号/单引号表示 | "wildman"/'wildman'/"nil" |
| function      | 函数 | print()/type() |
| userdata      | 表示任意存储在变量的 C 数据结构 | struct/指针 |
| thread        | 线程 | 协同程序（coroutine）：拥有独立的栈、局部变量和指令指针 | 
| table         | 表 | {}/{"apple","orange"}/{["key1"]=value1,["key2"]=value2}
### 变量
{% codeblock [变量定义示例] %}
> method1 = {}             -- 全局变量（空表类型）
> function method1.joke()
>> a = 1                   -- 全局变量（赋值为1）
>> local b = 2             -- 局部变量（赋值为2）
>> end
> a = 10                   -- 全局变量（赋值为10）
> b = 11                   -- 全局变量（赋值为11）
> print(a)
10
> print(b)
11
> method1.joke()           -- 调用自定义function，句a,b变量重新赋值
> print(a)
1
> print(b)
11
{% endcodeblock %}
### 循环

| 循环类型       | 描述           |
| :-----------: |:--------------|
| while 循环 | 	在条件为 true 时，让程序重复地执行某些语句。执行语句前会先检查条件是否为 true |
| for 循环 | 重复执行指定语句，重复次数可在 for 语句中控制 |
| repeat...until | 重复执行循环，直到 指定的条件为真时为止（类似do while） |
| 循环嵌套 | 可以在循环内嵌套一个或多个循环语句（while do ... end;for ... do ... end;repeat ... until;） |

#### while 循环
{% codeblock [基本语法] %}
while(condition)
do
   statements
end
{% endcodeblock %}
statements(循环体语句) 可以是一条或多条语句，condition(条件) 可以是任意表达式，在 condition(条件) 为 true 时执行循环体语句。
{% codeblock [示例] %}
> my_method = {}
> function my_method.print2()
>> local times = 1
>> while(times<3)
>> do
>> print("haahah")
>> print("print"..times.."times")
>> times = times + 1
>> end
>> end
> my_method.print2()
haahah
print1times
haahah
print2times
{% endcodeblock %}
#### for 循环
{% codeblock [基本语法] %}
for var=exp1,exp2,exp3 do  
    <执行体>  
end 
{% endcodeblock %}
var 从 exp1 变化到 exp2，每次变化以 exp3 为步长递增 var，并执行一次 "执行体"。exp3 是可选的，如果不指定，默认为1。
{% codeblock [示例] %}
> my_method = {}
> function my_method.print2()
>> for i=1,2,1
>> do
>> print("haahah")
>> end
>> end
> my_method.print2()
haahah
haahah
{% endcodeblock %}
#### repeat...until
{% codeblock [基本语法] %}
repeat
   statements
until( condition )
{% endcodeblock %}
循环条件判断语句（condition）在循环体末尾部分，所以在条件进行判断前循环体都会执行一次。
如果条件判断语句（condition）为 false，循环会重新开始执行，直到条件判断语句（condition）为 true 才会停止执行。
{% codeblock [示例] %}
> my_method = {}
> function my_method.print2()
>> local flag = 1
>>repeat
>>   print("haahah")
>>   flag = flag + 1
>>until(flag > 2)
>> end
> my_method.print2()
haahah
haahah
{% endcodeblock %}
#### 循环嵌套
lua编程语言支持同类型循环嵌套以及不同类型循环嵌套（常规性支持），详情略。

### 流程控制

| 语句       | 描述           |
| :-----------: |:--------------|
| if 语句 | if语句 由一个布尔表达式作为条件哦安段，其后紧跟其他语句组成 |
| if...else语句 | if语句 可以与 else语句 搭配使用，在 if条件表达式为false时执行else语句代码 |
| if嵌套语句 | 可以在 if 或 else if 中使用一个或多个 if 或 else if 语句 |

{% codeblock [if 基本语法] %}
if(布尔表达式)
then
   --[ 在布尔表达式为 true 时执行的语句 --]
end
{% endcodeblock %}

{% codeblock [if...else 基本语法] %}
if(布尔表达式)
then
   --[ 布尔表达式为 true 时执行该语句块 --]
else
   --[ 布尔表达式为 false 时执行该语句块 --]
end
{% endcodeblock %}

{% codeblock [if 嵌套 基本语法] %}
if( 布尔表达式 1)
then
   --[ 布尔表达式 1 为 true 时执行该语句块 --]
   if(布尔表达式 2)
   then
      --[ 布尔表达式 2 为 true 时执行该语句块 --]
   end
end
{% endcodeblock %}

## 基本函数库
常用函数：type,print,pcall,xpcall
### type(v)
功能：返回参数的类型名："nil","number","string","boolean","table","function","thread","userdata"

{% codeblock [函数 type 示例] %}
> type(hah)                 -- hah变量未定义，返回"nil"类型
nil
> a=1
> type(a)                   -- a变量赋值为数字1，返回"number"类型
number
> a="str"
> type(a)                   -- a变量赋值为字符串str，返回"string"类型
string
> a=false
> type(a)                   -- a变量赋值为false，返回"boolean"类型
boolean
> a={}
> type(a)                   -- a变量赋值为表，返回"table"类型
table
> 
{% endcodeblock %}

### print(...)
功能：简单的以tostring方式格式化输出参数内容
{% codeblock [函数 print 示例] %}
> a="str"
> print(a)
str
> b="bstr"
> print(a,b)
str	bstr
> 
{% endcodeblock %}

### pcall(f,arg1,...)
功能：在保护模式下调用函数（即发生的错误将不会发射给调用者）
当调用函数成功，第一个返回值为true，二、三或更多为函数正常返回值
当调用函数失败，第一个返回值为false，第二个为错误msg
{% codeblock [函数 pcall 示例] %}
> function method1(arg1,arg2) do
>> local res = arg1 + arg2
>> return res
>> end
>> end
> pcall(method1,1,2)
true	3
> pcall(method1)
false	stdin:2: attempt to perform arithmetic on a nil value (local 'arg1')
> a,b=pcall(method1)
> print(a)
false
> print(b)
stdin:2: attempt to perform arithmetic on a nil value (local 'arg1')
> 
{% endcodeblock %}

### xpcall(f,err)
功能：与pcall类似（但似乎不能传参数？），但可指定一个新的错误处理函数句柄
当调用函数成功，同pcall
当调用函数失败，将返回false以及err返回的结果
{% codeblock [函数 pcall 示例] %}
> function method1(arg1,arg2) do
>> local res = arg1 + arg2
>> return res
>> end
>> end
> function myerr() do
>> return "there is err"
>> end
>> end
> xpcall(method1,myerr)
false	there is err
{% endcodeblock %}

更多函数调用请查看 [基本函数库](https://www.kancloud.cn/digest/luanote/119926)。
## 推荐链接
[菜鸟教程](https://www.runoob.com/lua/lua-decision-making.html)——快速了解基本概念与设计
[Lua学习笔记](https://www.kancloud.cn/digest/luanote/119923)——详细了解lua语言特点以及库