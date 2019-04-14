#### part 1

首先生成 nReduce 个文件，然后写入它。

inFile = reduceName(...)

考虑如何读取文件，从 io.Reader 中读取。string 可以直接用 []byte 赋值吗? Ok, it works. s := string(byte_slice).

```Go
ioutil.ReadFile(inFile)
```

map: 首先通过 mapF 处理过，得到 []KeyValue，然后在把这个结果切片按照哈希，分配到 R 个文件中去。

reduce: 读取出所有 map 产生的第 Ri 个文件，然后排序，在调用 reduceF 来把相同的 Key 都聚集为一个结果。最后保存起来。

自己写的 doMap 没问题，reduce 那儿需要注意下。

。。。为什么代码放一晚上，起来之后就 work 了。

总结下，就是关于 io 的问题。然后把他们都用 ihash 映射到各个中间文件去。

#### part 3

The master calls schedule() twice during a MapReduce job, once for the Map phase, and once for the Reduce phase. 

scedule 会被调用两次。map 的时候，reduce 的时候，都会等待 channel 传送数据过来。

There will usually be more tasks than worker threads, so schedule() **must give each worker a sequence of tasks**, one at a time.

**任务会比线程多**，所以会有作业等待 worker 来完成的情况，也就是 registerChan 的接收方的阻塞。

schedule() should wait until all tasks have completed, and then return.

使用 sync.WaitGroup 来保证。这个功能也保证了所有的 Map 执行完之后才开始 reduce。

从 registerChan 中读取 worker

nTasks 在 map 阶段是输入文件的数量，在 reduce 阶段是中间文件的数量（多少个 map 就需要读取多少）。总之就是需要读取的文件的数量。

注释代码前半部分最后两行：

```Go
// ... registerChan will yield all
// existing registered workers (if any) and new ones as they register.
```

读取的时候，也需要写入 chan. 否则 worker 就会只工作一次，而 worker 的数量是**小于** nTask 的数量的。处理就是让 worker 工作完了之后，重新去 registerChan 里面准备好处理下一个 job.

写入 chan 的时候，为什么要使用 goroutine 多线程来让 worker 重新准备？因为 channel 在接收方准备好的之前会阻塞。所以使用 goroutine。可以看下面的 channel 的用法。


```Go
// 这儿不用 goroutine 会阻塞
go func () {
    registerChan <- worker
}()
```

#### part 4

the master should re-assign the task given to the failed worker to another worker.

发生错误的时候，re-assign，那么需要一个 for 循环，并且把这个 worker 重新提交，也就是重新写入 registerChan，如果成功，那就 break 循环，不用重新提交。

#### part 5

同样的词语可能会出现在相同的文件里面，所以这里需要借助 map 来完成去掉相同的文件名。

在失败的时候，产生的中间文件会影响测试，所以需要删除。也就是使用 rm 删除写中间文件，然后在重新测试。

```bash
$ rm mrtmp*
$ rm diff.out
```

在使用 map 的时候忘记赋值 1 了，导致不能够筛选出不同的文件，所以一大堆重复的文件出现。

## channel 的用法
---

#### 一般情况，不带缓冲

发送和接收操作在**另一端准备好之前**都会阻塞。


发送操作（协程或者函数中的），在**接收者准备好之前**是阻塞的。

比如这会产生死锁：

```Go
package main

import "fmt"

func main() {
	ch := make(chan int)
	ch <- 1 // 接受者没有准备好, 到这儿阻塞
	fmt.Println(<-ch) // 上一句阻塞了，所以一直等待，这句一直没能执行
}
```

修改之后，它能够运行。

```Go
package main

import "fmt"

func main() {
	ch := make(chan int)
	go func () {
		ch <- 1
	}()
	fmt.Println(<-ch)
}
```

#### 带缓冲

仅当信道的缓冲区填满后，向其发送数据时才会阻塞。当缓冲区为空时，接受方会阻塞。

下面代码不会阻塞。

```Go
package main

import "fmt"

func main() {
	ch := make(chan int, 2)
	ch <- 1
	ch <- 3
	fmt.Println(<-ch)
	fmt.Println(<-ch)
}
```