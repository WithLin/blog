---
title: "Go遇到的几个坑"
date: 2019-04-20T09:36:40+08:00
draft: false
author: "WithLin"
tags: ["go"]
categories: ["go"]
toc: true
comment: true
autoCollapseToc: false
---


## for-range的坑

[for-range代码](https://play.golang.org/p/jDEC9cE0Rv0)
```go
package main

import "fmt"

func main() {
    slice := []int{6,7,8}
    myMap := make(map[int]*int)

    for index, value := range slice {
        myMap[index] = &value
    }
    fmt.Println("=====new map=====")
    printMap(myMap)
}

func printMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}


//打印
=====new map=====
map[1]=8
map[2]=8
map[0]=8

```
> 由于以上打印的都是8这个值，也就是最后一个元素的指针地址。因为for-range创建了每个元素的副本，而不是直接返回元素的引用，
其实就是用了一个第三方的变量把value的值存起来的，导致每次遍历都是用的该第三方变量的地址。并不是返回slice里面元素的本身。


#### 针对上面的坑我们改造一下

```go
package main

import "fmt"

func main() {
    slice := []int{6,7,8}
    myMap := make(map[int]*int)

    for index, value := range slice {
        v := value
        myMap[index] = v
    }
    fmt.Println("=====new map=====")
    printMap(myMap)
}

func printMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}


//打印
=====new map=====
map[2]=8
map[0]=6
map[1]=7


// 或者通过一下方式 变通一下
package main

import "fmt"

func main() {
    slice := []int{6,7,8}
    myMap := make(map[int]*int)

    for index, _ := range slice {
        myMap[index] = &slice[index]
    }
    fmt.Println("=====new map=====")
    printMap(myMap)
}

func printMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}


=====new map=====
map[0]=6
map[1]=7
map[2]=8

```

#### 针对上面问题我们再扩展一下

```go
package main

import "fmt"

type Person struct {
	name string
}

func main() {
	person := make([]Person, 3)

	for _, p := range person {
		p.name = "withlin"
	}

	for _, p := range person {
		fmt.Println(p)
	}

}


//打印 
{}
{}
{}
//从这个例子更容易看出for range 遍历的时候对元素是复制出来给第三方的元素，而不是本身，优化 类似以上即可。


```


## 字符串判断


#### string()转换判断，涉及[#issue](https://github.com/golang/go/issues/26386)

```go

package main

import "fmt"

func main() {

	if string(55304) == string(55302) {
		fmt.Println("55304 equal 55302")
	} else {
		fmt.Println("55304 not equal 55302")
	}
}

//打印
55304 equal 55302

//我们再来看一个例子

package main

import "fmt"

func main() {

	if string(1) == string(2) {
		fmt.Println("1 equal 2")
	} else {
		fmt.Println("1 not equal 2")
	}
}

//打印
1 not equal 2


```
>针对上面的情况，因为string()内部尝试转换成UTF-8编码，如果转换失败，那么它会返回一个错误(utf8.RuneError),如果两个都转换失败也就是都
返回了utf8.RuneError这个错误，那么他们都是相等的。导致判达不到自己期待的值。那么我们应该怎么解决呢? 应该用strconv包去解决



## Unit问题


#### 报constant -1 overflows uint64错
```go

package main

import "fmt"

func main() {
    
	fmt.Println(uint64(-1))
}

//prog.go:7:20: constant -1 overflows uint64


//更改下上面的写法，编译不报错，并且运行成功
package main

import "fmt"

func main() {
    var i int = -1
	fmt.Println(uint64(i))
}


//打印 18446744073709551615


//再更改下写法
package main

import "fmt"

func main() {
    const i int = -1
	fmt.Println(uint64(i))
}

//编译时也报错 prog.go:7:20: constant -1 overflows uint64

```
>针对上面的问题，我们知道常量或者常数都不能这么写,[constants](https://blog.golang.org/constants)这篇文章有讲到，相关[issue](https://github.com/golang/go/issues/6923)

