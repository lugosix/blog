---
title: "golang 1.14.1 bug 导致线上服务卡死"
date: 2022-01-09T16:00:39+08:00
---
# 现象
12月29日晚突然服务报警，上游服务访问超时数量显著上升，初步排查访问某一容器的链接全部超时，摘流后恢复。

继续查看指标，发现摘流后 cpu 占用率仍居高不下，本地发起链接仍无法访问。
![](/blog/go1.14.1bug/1.jpg)
# 分析
## perf
cpu 占用率一直保持不变，首先猜测原因是某个线程在死循环，且此时其他线程停住了，这个还挺奇怪的。用 perf 抓下看看此时 cpu 在干嘛。执行  
```bash
perf record -p 进程号 -F 99 --call-graph dwarf  -- sleep 10
```  
对目标进程进行采样，随后  ```perf report ``` 得到  
![](/blog/go1.14.1bug/2.jpg)
感觉毫无头绪，用户代码部分全部指向 golang 运行时，而且看不到位于用户代码的调用源头，生成火焰图看下
![](/blog/go1.14.1bug/3.jpg)
依然收获不大，能活得到的信息是 runtime.osyield 和 runtime.morestack 占用了大部分时间，调用源头并没有奇迹般的出现。
## dlv
事发时并不在现场，是后续接触到这个问题，经了解同事此时已安装了 dlv，感觉是个好方法。于是执行  dlv attach 进程号

将 dlv 挂到目标进程上，成功后执行 goroutines -with running 获得到当前正在运行的 goroutine（以下简称 g）
![](/blog/go1.14.1bug/4.jpg)  
可以看到只有两个 g 在运行，148 那个是程序初始化的时候执行的信号注册函数。问题大概率就出在 180424466 上，切换到 180424466 上查看调用栈
![](/blog/go1.14.1bug/5.jpg)  
发现在分配内存时触发了 gc，对照源码能看到
![](/blog/go1.14.1bug/6.jpg)  
这个调用是要在系统栈上执行 stopTheWorldWithSema，但通过 dlv 查看的调用栈上就看不到接下来的链路了，大概率是因为切换栈导致的，通过单步调试的方式（过程略）确定了 frame 0 与 fame 1 之间缺少的调用链路为
```
 stopTheWorldWithSema → notetsleep → notetsleep_internal → futexsleep → futex
```
同样由单步调试发现  

![](/blog/go1.14.1bug/8.jpg)  
这个 g 卡在了这个循环里。回过头看下这个函数到底在干啥
![](/blog/go1.14.1bug/9.jpg)   
可以看到这个是 gc 进行 stw 的实现函数。go 1.14 开始引入了通过发信号实现的抢占式调度，按理说所有 p（gmp 模型中的 p） 应该都会在有限时间内被抢占然后停住，但实际上确实没有，一直卡在这个循环里，导致 gc 的 stw 阶段无法完成，也就不能正常对外提供服务。

直接原因找到了，但还不知道根本原因，即为什么有 p 停不下来。这块还出现了一个意外，程序被我搞挂了，现场没了，只剩一个提前 dump 出来的 core 文件，以下都基于对 core 文件的调试

先分析下 preemptall 函数，整个调用链路如下  
![](/blog/go1.14.1bug/10.jpg)  

![](/blog/go1.14.1bug/11.jpg)  

![](/blog/go1.14.1bug/12.jpg)  

![](/blog/go1.14.1bug/13.jpg)  


最终是通过 tgkill 系统调用把信号发到了某个线程，整体代码不多看起来都挺正常的。但通过 dlv 执行 goroutines 人工遍历一遍 g 后发现并没有 g 在处理信号，也就是说要么信号没发，要么信号没收到。到这感觉又排查不下去了，信号发没发因为不能单步调试了，无法简单的获得。排查信号发了没收到中间涉及到系统层面了，排查更加困难。只能先换个方向去考虑，既然是有 p 没停住，看看能不能找到这个 p，先看下 p 的结构
![](/blog/go1.14.1bug/14.jpg)  
可以看到有 p 当前的状态 status 和绑定的 m（gmp 模型中的那个 m），通过 status 可以获得当前 p 的状态。被 gc 停下来的 p 的 status 应该是 3。这样的话就看下能不能遍历一下所有 p 把那个 status 不等于 3 的 p 找出来。

还是通过刚才的 stopTheWorldWithSema 源码发现有一个 allp 变量，看起来有所有的 p 的信息。经搜索 go 运行时有一些全局变量保存了所有的 p 和 m 等信息，如下
![](/blog/go1.14.1bug/15.jpg)  
于是在 dlv 内原地执行 p allp，将这个切片全部打印出来，然后挨个查看 status，如下（忽略无用的 p）
![](/blog/go1.14.1bug/16.jpg)  
果然找到了唯一一个 status 不等于 3 的 p，就是它没停下来。p 找到了，下一步自然就是要看下这个 p 绑定的 m 现在到底执行啥，在 dlv 内执行
```
p *(*runtime.m)(824636343168)
```
打印出这个 p 绑定的这个 m 结构体的具体内容，先看下 m 的结构定义
![](/blog/go1.14.1bug/17.jpg)  
可以看到 curg 表示了当前 m 正在执行的 g，然后看一下刚才输出的 m 的结果（省略部分无关内容）
![](/blog/go1.14.1bug/18.jpg)  
curg 竟然是 nil，都没 g 了这个 m 怎么还在执行，是在执行什么。不过此时回到刚才的 runtime.preemptone 函数，发现它会先检查当前 g.m.curg 是否为空，为空就不发信号了，于是上面那个问题就解决了，是信号一直没有发出去，导致的死循环。接下来就要排查是什么时机会把 curg 置为 nil，但经过全文搜索也没发现 curg 直接赋值 nil 的地方，单独搜索 curg 结果又实在是太多了，遍历一遍很困难，先放一下。得再次换个思路排查了，通过逐个观察 m 结构的字段，发现有一个 procid 字段，用来在 go 内标识线程号的，输出如下  
![](/blog/go1.14.1bug/20.jpg)  

在 dlv 内执行 
```
threads
``` 
后输出的信息如下  
![](/blog/go1.14.1bug/21.jpg)  
thread 后边跟着的数字正好就是这个 procid，（恰好这个 osyield 显的有些格格不入）于是执行 thread 1365 切换到这个 m 上，然后查看调用栈如下  
![](/blog/go1.14.1bug/22.jpg)  
可以看到此时 m 正在执行这段代码，但这段代码也很奇怪，这个栈底是 morestack，更早的调用源头也没了，通过 dlv 的 goroutines 查看也看不到在执行这个的 g。感觉又一次查不下去了，只好再次换思路，先看看这个调用链在干嘛，在跟踪链路的过程中发现 frame 4 的 runtime.goschedImpl 函数内一个操作  
![](/blog/go1.14.1bug/23.jpg)  
runtime.dropg 源码如下  
![](/blog/go1.14.1bug/24.jpg)  
原来把  curg 置为 nil 是在这里！通过函数调用封装了一层，难怪之前搜不到。此时这个调用链的怀疑度被大大增加，巧合太多可能就不是巧合了。但因为现在没法单步调试，当前 core 文件这个部分也已经执行完了，没有办法回溯到之前这个 curg 的值，还是先放一放，回头看一下调用源头有没有获得他的局部变量，先继续往下看链路 

查看 runtime.adjusttimers 部分源码如下  
![](/blog/go1.14.1bug/25.jpg)  
对照调用栈可以看到这个 m 就是执行到 osyield() 了，随后第二个红箭头把索引减了1，而这个循环每次执行会把索引加1，也就是之后的循环如果 s 的值不变，他就会一直进入 timerModifying 分支无限循环。隐约觉得问题可能就在这，如果另一个正在修改这个 timer 的 g 还没把状态从 timerModifying 修改为别的就被暂停或者抢占，那可能这块就会有很长时间的循环，最坏的情况是双方在互相等待对方，导致循环永远无法结束。接下来顺着这个思路排查，回过头开始仔细分析下这个 thread 1365 调用栈。

首先看 morestack ，但大致看了下都是汇编的实现，汇编不是很熟，先放一边继续看 newstack，代码如下
![](/blog/go1.14.1bug/26.jpg)  
可以看到第一行就获取了一下当前的 g，执行 p thisg 输出看一下
![](/blog/go1.14.1bug/27.jpg)  

![](/blog/go1.14.1bug/28.jpg)  
goid 竟然是 0，很奇怪。没什么头绪继续往下看，看到一个 gp 的赋值（第二个红箭头，这个 gp 就是上面要找的那个被置为 nil 的 curg 为 nil 之前的值），取的是当前 g 对应的 m 的 curg，感觉也很奇怪，当前在执行的 g 不就应该是 curg 吗，执行 p gp 看一下
![](/blog/go1.14.1bug/29.jpg)  
此时出现了一个之前分析没有出现过的 g 它的 id 为 10 ，突然感觉答案应该要来了，切换到 g 10 并查看调用栈
![](/blog/go1.14.1bug/30.jpg)  
 看名字估计是用户层代码在调整 timer，省略中间对照源码分析调用栈的过程，直接看 modtimer 代码
![](/blog/go1.14.1bug/31.jpg)  
 可以看到最终是停在了 addInitializedTimer 函数内，而对这个 timer 的状态修改是在之后，这就跟之前的猜测对上了，打印之前的 timer 的地址和当前进入 modtimer 函数内的 timer 的地址做比较，果然是一致的！大概率确定就是这个原因了，g 10 在修改 timer 的过程中被一个无名 g 0 抢占了，然后这个 g 0 还需要等 g 10 把 timer 修改完，死锁发生了。

但现在还剩最关键的一个问题，当前这个 goid 为 0 的 g 到底是哪来的，为什么栈底只到 morestack 看不到之前了，也有没有正常创建 g 一定会在栈底存在的 runtime.goexit 函数。只能回去硬啃一下 morestack 的实现  
![](/blog/go1.14.1bug/32.jpg)  
这个是 go 用的 plan9 汇编，通过 dlv 看下真正的 x86 汇编
![](/blog/go1.14.1bug/33.jpg)  
通过注释和阅读源码可以发现，在执行 newstack 前，morestack 通过修改 bx 和 sp 寄存器（用来标识栈底和栈顶）把执行栈替换为了当前 m 所有的 g0 的执行栈（专门在调度时用的栈），所以 dlv 内才看不到 morestack 之前的调用。现在就差一个最直接的证据表明是在 g 10 内触发了这个 morestack 调用。还是从这个源码发现，他会把这个函数的 caller 的 caller 的上下文（pc, sp 等）保存到 m 的 morebuf 内，所以通过查看源码链路（过程略），执行 frame 6，然后 p morebuf 得到
![](/blog/go1.14.1bug/34.jpg)  
此时这个 pc 就应该是 g 10 的调用栈的栈顶那个帧，即 frame 0（morestack 外层还有个相关函数调用，略过），验证下执行 addr2line -Cfi -e (二进制文件) 0x43dea2 得到
![](/blog/go1.14.1bug/35.jpg)  
完全一致，到此就可以确认这个问题的根本原因，见下一节
# 根因
在 g 10 执行修改当前 p 的 timer 的链路中，准备执行 runtime.wakeNetPoller 时触发了 g 的抢占，此时 timer 的修改没有完成，转而接下来去执行 morestack （这个是协作式抢占，复用了 newstack 去实现），在 morestack 内通过汇编把执行栈切换为了当前 m 的 g0 上，接下来的 morestack 链路中会重新执行 go 的调度函数 runtime.schedule() ，而在这个函数中需要等待当前 p 的 timer 都修改完毕，然而此时整个程序处于 gc 的 stw 阶段，其他所有 p 已经全部暂停，也就导致 g 10 无法继续执行，从而形成了死锁，进而导致程序无法正常对外服务。

直观的来看问题出在 timer 的更新上，不应该允许 timer 未修改完就被打断或者 curg 为 nil 也应该发信号。查看新版本 timer 相关的代码，确实已经做了修改（但并不是前面那两种方式），跟踪 commit 可以看到 https://github.com/golang/go/commit/355f53f0a0a5d79032068d4914d7aea3435084ec 这个提交，把 wakeNetPoller 的调用放到了 timer 状态更新之后，确实可以修复这个问题。从 commit 注释中也可以看到是为了解决 https://github.com/golang/go/issues/38023 ，这个 issue 的现象和当前问题完全一致，也再次验证了排查的正确性
# 复现
必要条件
- golang 1.14.1
- 使用了 go 提供的 timer（包括第三方库内的使用）

满足这些也不一定立马就会出现问题，还需要运行时的代码按照一定顺序执行，所以问题产生有一定概率。不过随着运行时间变长，这个问题的出现是确定的。

```go
package main
 
import (
    "fmt"
    "runtime"
    "sync"
    "time"
)
 
func main() {
    runtime.GOMAXPROCS(2)
 
    wg := sync.WaitGroup{}
    wg.Add(3)
 
    m := 10000
    go func() {
        defer wg.Done()
        cnt := 0
        for {
            runtime.GC()
            if cnt == m {
                fmt.Println("gc done")
                cnt = 0
            }
            cnt++
        }
    }()
 
    cnt := 0
    t := time.NewTimer(1*time.Microsecond)
    go func() {
        defer wg.Done()
        for {
            <-t.C
            t.Reset(1*time.Microsecond)
            if cnt == m {
                fmt.Println("t.C")
                cnt = 0
            }
            cnt++
        }
    }()
 
    cnt1 := 0
    t1 := time.NewTimer(2*time.Microsecond)
    go func() {
        defer wg.Done()
        for {
            <-t1.C
            t1.Reset(2*time.Microsecond)
            if cnt1 == m {
                fmt.Println("t1.C")
                cnt1 = 0
            }
            cnt1++
        }
    }()
 
    wg.Wait()
 
    return
}
```
感谢 cf 和 yzf 两位同事后续提供的复现示例代码