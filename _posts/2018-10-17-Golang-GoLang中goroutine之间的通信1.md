---
layout:     post
title:      GoLang中goroutine之间的通信
subtitle:   并发编程基础，探索go的并发奥义
date:       2018-10-17
author:     XuHAo
header-img: img/post-bg-BJJ.png
catalog: true
tags:
    - Golang
    - 并发
    - 多线程
---
# GoLang中goroutine之间的通信


**在go程序中，最被人所熟知的便是并发特性，一方面有goroutine这类二级线程，对这种不处于用户态的go程的支持，另一方面便是对并发编程的简便化，可以快捷稳定的写出支持并发的程序。**


- 先回顾进程or线程之间的通信方式

    inte-process communication(IPC)  
    其中Go支持的IPC方法有管道、信号和socket。篇(shui)幅(ping)有限，一张图引入回忆。
     ![ipc图解.jpg](https://upload-images.jianshu.io/upload_images/7344916-0371e5cb7fddc8a1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 并发和并行

  简单来讲 并发就是可同时发起执行的程序，并行就是可以在支持并行的硬件上执行的并发程序；换句话说，并发程序代表了所有可以实现并发行为的程序，这是一个比较宽泛的概念，并行程序也只是他的一个子集。 这个问题在知乎笔试中出现过，并行和并发不是一个概念。

复习过过去的基础知识后，进入主题。

>开发go程序，不管是系统性的k8s平台，还是基于传统的web开发，都常常使用goroutine来并发处理任务，有时候goroutine之间是相互独立的，但是也有时候goroutine之间是需要同步和通信的；另一种情况是父goroutine需要控制属于他的子goroutine。而goroutine的设计机制为，goroutine退出只能由本身进行控制，不同与传统的用户态协程，不允许从外部强制结束该goroutine，除非goroutine奔溃或者main函数结束。目前实现多个goroutine间的同步与通信大致有：

- 全局共享变量
- channel管道通信  ---CSP模型
- context包        ---在1.7版本后引入


---
全局共享变量：  
实现思路为： 
1. 申明一个全局变量。
2. 在这个全局变量作用域中，开启多个go程，多个go程实际上是共享这个全局变量。
3. 开启的多个go程实现循环，不断的监听这个全局变量的情况，若全局变量属性触发一定条件，跳出循环，按串行顺序执行到该go程结尾，go程生命周期结束。

代码示例：  

    package main
    import (
	    "fmt"
	    "time"
    )

    func main() {
	    running := true
	    f := func() {
		    for running { //控制的全局共享变量
			    fmt.Println("sub proc running...")
			    time.Sleep(1 * time.Second)
		    }
    		fmt.Println("sub proc exit")
    	}
    	go f()
    	go f()
    	go f()
    	time.Sleep(2 * time.Second)
    	running = false //全局共享变量改变
    	time.Sleep(3 * time.Second)
    	fmt.Println("main proc exit")
    }

- 优点：实现简单，不抽象，直白方便，一个变量即可简单控制子goroutine的进行。
- 缺点:   
   -  不能适应结构复杂的设计，功能有限，只能适用于子go程中读，外主程或父go程来写全局变量，若子go程中进行写，会出现数据同步问题，需要加锁解决，不加锁面对map这类线程不安全的结构会报错。
   -  还有不适合用于同级的子go程间的通信，全局变量传递的信息太少。
   -  还有就是主进程无法等待所有子goroutine退出，因为这种方式只能是单向通知，所以这种方法只适用于非常简单的逻辑且并发量不太大的场景。


---
channel通信

首先解释golang中的channel：channel是go中的核心部分之一，结构体简单概括就是一个ring队列+一个锁 有兴趣的同学可以去研究一下源码构建。在使用中可以将channel看做管道，通过channel迸发执行的go程之间就可以发送或者接受数据，从而对并发逻辑进行控制。
> go的channel的设计是建立在CSP(Communicating Sequential Process)，中文可以叫做通信顺序进程，是一种并发编程模型，由 Tony Hoare 于 1977 年提出。简单来说，CSP 模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。CSP 模型的关键是关注 channel，而不关注发送消息的实体。Go 语言实现了 CSP 部分理论，goroutine 对应 CSP 中并发执行的实体，channel 也就对应着 CSP 中的 channel。 也就是说，CSP 描述这样一种并发模型：多个Process 使用一个 Channel 进行通信, 这个 Channel 连结的 Process 通常是匿名的，消息传递通常是同步的（有别于 Actor Model）。

设计思路：
1. 创建一个sync包中WaitGroup实例 var wg sync.WaitGroup
2. 创建一个chan，负责控制go程退出
3. 在每一个go程被创建前，执行注册. wg.Add(1)
4. 创建go后，在go程中监听信号chan能否收到，使用select机制（和io多路复用相似）
5. runtime主程 直接关闭chan，也可以选择发送信号量。通知子go程结束循环，结束go程
6. go程调用 wg.Done()注销后再退出，所以在进行go程 使用defer，go程退出执行。
7. wg.Wait() 在注册的所有信息注销后才继续执行下一步。原理是实现一个for死循环，在注册的值消耗完毕后，跳出死循环。

代码解释：  

    package main
    import (
        "fmt"
        "os"
        "os/signal"
        "sync"
        "syscall"
        "time"
    )
    //执行的go程方法，传入的chan
    func consumer(stop <-chan bool) {
    	for {
    		select {                    //机制类似epoll
    		case <-stop:
    			fmt.Println("exit sub goroutine")
    			return
    		default:
    			fmt.Println("running...")
    			time.Sleep(500 * time.Millisecond)
    		}
    	}
    }
    func main() {
    		stop := make(chan bool)
            var wg sync.WaitGroup
            // Spawn example consumers
            for i := 0; i < 3; i++ {
                wg.Add(1)
                go func(stop <-chan bool) {
                    defer wg.Done()
                    consumer(stop)
                }(stop)
            }
            close(stop) //直接关闭chan，否则传递3次信号量
            fmt.Println("stopping all jobs!")
            wg.Wait()
            fmt.Println("OVER")
    }
>channel通信控制基于CSP模型，相比于传统的线程与锁并发模型，避免了大量的加锁解锁的性能消耗，而又比Actor模型更加灵活，使用Actor模型时，负责通讯的媒介与执行单元是紧耦合的–每个Actor都有一个信箱。而使用CSP模型，channel是第一对象，可以被独立地创建，写入和读出数据，更容易进行扩展。