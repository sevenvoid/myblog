---
title: Golang(一) 包和文件
date: 2017/05/05
categories:
- Go
tags:
- Go
---
从今天开始会不遗余力的学习Golang语言，这里记录一下Golang相关的一些知识，当做学习笔记。

## 安装
Golang语言的安装直接参看[官方文档](https://golang.org/)，即可完成安装，一般需要自己设置一下$GOPATH环境变量，来指向你自己的工作目录即可，并新建三个子目录src、pkg、bin：

+ src目录存放了Golang的源码文件，以.go为文件后缀扩展名
+ pkg目录存放了编译后的arch文件，以.a为文件后缀扩展名
+ bin目录存放了编译后的可执行文件，可以直接./xxx（Linux下）执行

## 包的管理
Golang语言是基于包来管理整个项目结构的，在新建一个.go的源文件时就会声明该文件所属的包：

```Go
package main
```
包系统设计的目的都是为了简化大型程序的设计和维护工作，通过将一组相关的特性放进一个独立
的单元以便于理解和更新，在每个单元更新的同时保持和程序中其它单元的相对独立性。

每个包还通过控制包内名字的可见性和是否导出来实现封装特性。Go语言规定，包内的成员如果首字母是大写的，则可以被其他包直接访问，称为导出；而如果首字母是小写，则只允许同一包下的所有文件才可以访问。通过限制包成员的可见性并隐藏包API 的具体实现，将允许包的维护者在不影响外部包用户的前提下调整包的内部实现。

通过限制包内变量的可见性，还可以强制用户通过某些特定函数来访问和更新内部变量，这样可以保证内部变量的一致性和并发时的互斥约束。

Go语言的闪电般的编译速度主要得益于三个语言特性：

+ 所有导入的包必须在每个文件的开头显式声明，这样的话编译器就没有必要读取和分析整个源文件来判断包的依赖关系。
+ 禁止包的环状依赖，因为没有循环依赖，包的依赖关系形成一个有向无环图，每个包可以被独立编译，而且很可能是被并发编译。
+ 编译后包的目标文件不仅仅记录包本身的导出信息，目标文件同时还记录了包的依赖关系。因此，在编译一个包的时候，编译器只需要读取每个直接导入包的目标文件，而不需要遍历所有依赖的的文件。

<!--more-->

### 包声明
包声明语句的主要目的是确定当前包被其它包导入时默认的标识符(也称为包名)。按照惯例，一个包的名字和包的导入路径的最后一个字段相同，因此即使两个包的导入路径不同，它们依然可能有一个相同的包名。声明包时，也可以添加相应的包注释，如果包注释很大，通常会放到一个独立的doc.go文件中。

```
package mymath //声明一个mymath包

import "mymath"     //导入两个不同导入路径下同名的包
import "sevenvoid.com/gobasic/mymath"
```

导入路径是相当于$GOPATH下的包路径，比如在我的工作目录下有有如下的目录结构，总共有两个包main和mymath包：

```
cd $GOPATH
./src
----sevenvoid.com/
------gobasic/
--------mymath/
----------math.go
--------main.go
```

则如果包的导入路径为sevenvoid.com/gobasic，导入的包可以为main包，或者导入路径为sevenvoid.com/gobasic/mymath，导入的包可以为mymath包，通常来说一个目录下的所有源码文件均属于同一个包。

### 构建包
包的编译与安装均可以看作是构建包，有两条相关的命令：

+ go build ：构建指定的包和它依赖的包，然后丢弃除了最后的可执行文件之外所有的中间编译结果。
+ go install：它会保存每个包的编译成果，而不是将它们都丢弃。被编译的包会被保存到pkg目录下，可执行程序被保存到bin目录。

执行命令时需要注意有如下几种方式：

```
//1、任何目录下使用包的导入路径
cd anywhere
go build sevenvoid.com/gobasic     //将会在当前目录下生成可执行文件
go install sevenvoid.com/gobasic   //将会在bin目录下生成可执行文件，pkg目录下生成包文件

//2、$GOPATH下只能使用./的相对路径
cd $GOPATH
go build ./src/sevenvoid.com/gobasic
go install ./src/sevenvoid.com/gobasic

//3、进入项目的根路径，执行相应命令后，会构建所有的包
cd $GOPATH/src/sevenvoid.com/gobasic
go build
go install

//4、如果处于中间目录将会构建报错
cd $GOPATH/src/sevenvoid.com/
go install gobasic/
......can not find package gobasic....
```

### 包的导入
可以在一个源文件中包含零到多个导入包声明语句。每个导入声明可以单独指定一个导入路径，也可以通过圆括号同时导入多个导入路径。

```
import "fmt"
import "os"

import (
    "fmt"
    "os"
)
```

导入包是还可以匿名导入，或者给某个长路径的导入路径定义一个简短的新名字：

```
import (
    format "fmt" //设置别名
    _ "os"  //匿名导入
)
 ```
 
匿名导入的目的只是想利用导入包而产生的副作用:它会计算包级变量的初始化表达式和执行导入包的init初始化函数。
### 包的初始化
包的初始化首先是解决包级变量的依赖顺序，然后安照包级变量声明出现的顺序依次初始化:

```
var a = b + c // a 第三个初始化, 为 3
var b = f() // b 第二个初始化, 为 2, 通过调用 f (依赖c)
var c = 1 // c 第一个初始化, 为 1
func f() int { return c + 1 }
```

如果包中含有多个.go源文件，它们将按照发给编译器的顺序进行初始化，Go语言的构建工具首先会 将.go文件根据文件名排序，然后依次调用编译器编译。

对于在包级别声明的变量，如果有初始化表达式则用表达式初始化，还有一些没有初始化表达式的，例如某些表格数据初始化并不是一个简单的赋值过程。在这种情况下，我们可以用一个特殊的init初始化 函数来简化初始化工作。

每个文件都可以包含多个init初始化函数，在每个文件中的init初始化函数，在程序开始执行时按照它们声明的顺序被自动调用。

每个包在解决依赖的前提下，以导入声明的顺序初始化，每个包只会被初始化一次。因此，如果一个p包导入了q包，那么在p包初始化的时候可以认为q包必然已经初始化过了。
 
## 命令行工具
Go环境安装成功后，官方同时也提供了一系列的命令行工具，具体的使用方式可以查看官方文档，或者使用go --help查看

```
go --help
$ go ...
     build            compile packages and dependencies
     clean            remove object files
     doc              show documentation for package or symbol
     env              print Go environment information
     fmt              run gofmt on package sources
     get              download and install packages and dependencies
     install          compile and install packages and dependencies
     list             list packages
     run              compile and run Go program
     test             test packages
     version          print Go version
     vet              run go tool vet on packages
 Use "go help [command]" for more information about a command.
```


