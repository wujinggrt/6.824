# Lab 2

#### GFS

为什么会拥有**三**个副本

#### Raft 笔记

[extended-raft-note](./extended-raft-note.md)

#### 2A

对于 election timeout，使用一个 goroutine 来按 ms 计时.

创建的时候，使用通道来阻塞？

#### 学习到 Go 的地方

在使用多线程的时候，指的是 goroutine。 调用的函数自身不能够结束，不然地址空间释放了，那些 goroutine 也就凉了。也就是，如果要调用，那么就需要这个函数不会借宿，除非所有的 goroutine 结束，这儿可以使用 WaitGroup。

例子：

下面这个，一下子就结束了，因为调用 goroutine 的 main 结束了，自然线程不能存在。

```Go
func main() {
	fmt.Println("Main")
	go func () {
		for {
			fmt.Println("Hello")
			time.Sleep(time.Second)
		}
	}()
}
```

正解：添加一个让 main 不会结束的条件，无论是 WaitGroup 还是写个死循环，不结束就 OK。

```Go
func main() {
	fmt.Println("Main")
	go func () {
		for {
			fmt.Println("Hello")
			time.Sleep(time.Second)
		}
	}()
	for {
	}
}
```