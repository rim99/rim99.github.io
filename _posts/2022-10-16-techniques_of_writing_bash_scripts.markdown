---
layout: post
title: "Shell脚本编写技巧"
date: 2022-10-16 09:33:25 +0800
categories: 原创
---

能够高效率地编写Shell脚本应该是SRE/Infra工程师应该具备的基本素养。最近在写Shell脚本的时候踩了不少坑，总结一下。

## 追踪脚本的执行栈

简单的脚本通过echo命令输出log就可以看到我们关注的信息。但对于复杂的脚本，echo命令输出也不够清晰。`bash`命令提供了一个参数`-x`能够在执行脚本的同时，给出具体的执行栈和参数等情况。从而，更好地帮助我们调试脚本。

例如，我们有脚本：

```bash
➜ ✗ cat demo.sh 
function g() {
    return 0
}

function f() {
    echo using arg: $1
    g $1
}

f hello
```

用`-x`参数执行一下

```bash
➜  blog git:(master) ✗ bash -x demo.sh
+ f hello
+ echo using arg: hello
using arg: hello
+ g hello
+ return 0
```

## 异常处理

Shell脚本并没有类似于Java的Excpetion这样的机制来捕获异常栈。只能通过函数返回码来判断函数执行是否成功。

```bash
➜  cat demo.sh 
function f() {
    return 0
}

function g() {
    return 1
}

function execute() {
    $1
    if [[ $? -eq 0 ]]; then
        echo execute function $1 success
    else
        echo fail to execute function $1
    fi
}

execute f
execute g
```

执行一下看看

```bash
➜  bash -x demo.sh
+ execute f
+ f
+ return 0
+ [[ 0 -eq 0 ]]
+ echo execute function f success
execute function f success
+ execute g
+ g
+ return 1
+ [[ 1 -eq 0 ]]
+ echo fail to execute function g
fail to execute function g
```

## 用echo返回函数的文本内容

Shell函数必须返回数字。正常情况返回0，出错的时候用不同的数字代表不同的状态。

但如果我们需要返回文本呢？看例子：

```bash
➜  cat demo.sh 
function f() {
    echo using arg: $1
}

output=$(f hello)

echo output: [$output]
```

执行一下看看：

```bash
➜  bash -x demo.sh
++ f hello
++ echo using arg: hello
+ output='using arg: hello'
+ echo output: '[using' arg: 'hello]'
output: [using arg: hello]
```

## 使用local定义函数内部的变量

这一条可以避免函数内部的执行导致全局变量的变更。后者有时会导致难以定位的bug。

```bash
➜  cat demo.sh 
function f() {
    a_global_var="changed by f"
} 

a_global_var="init"
f
echo $a_global_var

➜  bash -x demo.sh
+ a_global_var=init
+ f
+ a_global_var='changed by f'
+ echo changed by f
changed by f
```

如果使用local的话，

```bash
➜  cat demo.sh
function f() {
    local a_global_var="changed by f"
    echo inside the f function, the var is [$a_global_var]
} 

a_global_var="init"
f
echo $a_global_var

➜  bash -x demo.sh
+ a_global_var=init
+ f
+ local 'a_global_var=changed by f'
+ echo inside the f function, the var is '[changed' by 'f]'
inside the f function, the var is [changed by f]
+ echo init
init
```

## 将标准错误重定向到标准输出

很多命令在执行的时候会产生log，但这些内容并不在标准输出中，而是在标准错误中。

如果脚本的运行环境不能够很好的打印出标准错误的log，那就需要把他们重定向到标准输出中。

举个例子，我们有一个`demo.sh`脚本在调用`help.sh`脚本。其中`help.sh`有一部分log输出在标准错误中。

```bash
➜  cat help.sh 
echo generated error >&2
echo hello
➜  cat demo.sh 
output=$(bash help.sh)
echo output: [$output]
```

执行一下`demo.sh`看看：

```bash
➜  bash -x demo.sh
++ bash help.sh
generated error
+ output=hello
+ echo output: '[hello]'
output: [hello]
```

我们再修改一下`demo.sh`，把标准错误里的内容重定向到标准输出里来：

```bash
➜  blog git:(master) ✗ cat demo.sh 
output=$(bash help.sh 2>&1)
echo output: [$output]
```

执行一下`demo.sh`看看：

```bash
➜  bash -x demo.sh
++ bash help.sh
+ output='generated error
hello'
+ echo output: '[generated' error 'hello]'
output: [generated error hello]
```

## 注意'与"的区别

当引号内的内容是恒定字符串的时候，`'`与`"`并没有太大的区别。但是如果引号内的字符串需要插值的时候，`'`会失效。

举个例子：

```bash
➜  cat demo.sh
a="world"
b="hello,$a"
c='hello,$a'

echo $b
echo $c
```

执行一下看看：

```bash
➜ bash -x demo.sh
+ a=world
+ b=hello,world
+ c='hello,$a'
+ echo hello,world
hello,world
+ echo 'hello,$a'
hello,$a
```
