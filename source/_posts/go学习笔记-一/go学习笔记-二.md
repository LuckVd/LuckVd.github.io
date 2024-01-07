---
title: go学习笔记(二)
date: 2024-01-07 16:39:41
categories: 
- 学习笔记
tags:
- 后端
- Go 
---

## goroutines and concurrency

使用关键字go func()可以启动一个协程。

```Go
package main

import (
    "fmt"
    "runtime"
)

func say(s string) {
    for i := 0; i < 5; i++ {
        runtime.Gosched()
        fmt.Println(s)
    }
}

func main() {
    go say("world") // create a new goroutine
    say("hello")    // current goroutine
}
```

对于这个例子，其中runtime.Gosched()是显示指定让出cpu，执行另一个协程。该例子执行结果为

```cmd
hello
world
hello
world
hello
world
hello
world
hello
```

如果注释掉runtime.Gosched()，那么结果如下。

```cmd
hello
hello
hello
hello
hello
```

这样的原因是主程序迅速执行完毕，导致协程未执行。

修改代码，在runtime.Gosched()前加一个输出，观察具体的执行过程：

```go
func say(s string) {
	for i := 0; i < 5; i++ {
		fmt.Println(1, s)
		runtime.Gosched()
		fmt.Println(2, "  "+s)
	}
}
```

结果为

```cmd
1 hello
1 world
2   hello
1 hello
2   world
1 world
2   hello
1 hello
2   world
1 world
2   hello
1 hello
2   world
1 world
2   hello
1 hello
2   world
1 world
2   hello
```

可以看到首先执行的是主程序，遇到runtime.Gosched()后执行协程，协程遇到runtime.Gosched()后继续执行主程序。主程序最后一次执行完后退出程序，所以协程少执行一次。



## channels

A `channel` is like two-way pipeline in Unix shells: use `channel` to send or receive data.

```Go
ci := make(chan int)
cs := make(chan string)
cf := make(chan interface{})

ch <- v    // send v to channel ch.
v := <-ch  // receive data from ch, and assign to v
```

channel的数据先进先出。如果设置了Buffered channels，也就是固定长度的channel。超出长度后会报错。

从channel获取数据的行为是阻塞的，如果channel内没有数据，读取的行为就会阻塞当前的协程。

- range:可以直接用range对channel进行遍历，

```Go
    c := make(chan int, 10)
    //write into c
    for i := range c {
        fmt.Println(i)
    }
```

- close:用close关闭channel。

- select: select 默认阻塞，当case中的channel存在数据时执行。 

  ```Go
  func fibonacci(c, quit chan int) {
      x, y := 1, 1
      for {
          select {
          case c <- x:
              x, y = y, x+y
          case <-quit:
              fmt.Println("quit")
              return
          }
      }
  }
  
  func main() {
      c := make(chan int)
      quit := make(chan int)
      go func() {
          for i := 0; i < 10; i++ {
              fmt.Println(<-c)
          }
          quit <- 0
      }()
      fibonacci(c, quit)
  }
  //在这个例子中，如果把fibonacci(c, quit)提前，则会发生死锁。
  
  // func main() {
  //     c := make(chan int)
  //     quit := make(chan int)
  //     fibonacci(c, quit)
  //     go func() {
  //         for i := 0; i < 10; i++ {
  //             fmt.Println(<-c)
  //         }
  //         quit <- 0
  //     }()
  // }
  
  ```

- 为了防止一直阻塞，可以设置超时：

  ```go
  case <-time.After(5 * time.Second):
                  println("timeout")
                  o <- true
                  break
  ```

  

The package `runtime` has some functions for dealing with goroutines.

- `runtime.Goexit()`

  Exits the current goroutine, but defered functions will be executed as usual.

- `runtime.Gosched()`

  Lets the scheduler execute other goroutines and comes back at some point.

- `runtime.NumCPU() int`

  Returns the number of CPU cores

- `runtime.NumGoroutine() int`

  Returns the number of goroutines

- `runtime.GOMAXPROCS(n int) int`