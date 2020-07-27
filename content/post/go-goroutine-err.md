---
title: "Go下的gorutine处理错误的方式"
date: 2019-04-08T15:34:56+08:00
lastmod: 2019-04-08T15:34:56+08:00
draft: false
author: "WithLin"
tags: [ "go"]
categories: ["go"]
toc: true
comment: true
autoCollapseToc: false
---

## goroutine里的错误怎么传递？


###### 假设doSomeThing函数在跑一些任务，我的最初想法是通过chan err把错误回调出来

``` golang
func doSomeThing(some string,sh chan string,chanErr chan error){

    err := Do()
    if err != nil {
        chanErr <- err
        return
    }

    for {
        select{
            case <- sh:
            err :=doAnOtherSomeTing()
            if err !=  nil{
                chanErr <- err
                return
            }
        }
        default:
        chanErr <- nil
    }

}


func main(){
    chanErr := make(chan error)
    go doSomeThing(some,sh,chanErr)
    fmt.Println(<-chanErr)
}

```

###### 针对以上的其实还有更骚的操作利用context的方式去做。这里用的golang的[Sync.errgroup](https://github.com/golang/sync/blob/e225da77a7e68af35c70ccbf71af2b83e6acac3c/errgroup/errgroup.go)

```golang

var g errgroup.Group 
var urls =  []string{
    "http://www.baidu.com",
    "http://www.golang.com",
    "http://www.163.com",
}

for _,url := range urls{
    url := url
    g.Go(func() error{
        resp,err := http.Get(url)
        if err == nil {
            resp.Body.Close()
        }
        return err
    })
}

if err := g.Wait(); err == nil {
    fmt.Println("Successfully fetched all URLs.")
}



```

