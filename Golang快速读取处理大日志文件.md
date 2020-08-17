---
title: "Golang 快速读取处理大日志文件"
date: "2020-07-11"
categories: ["学习笔记"]
---

本文翻译并改进于 [此文](https://medium.com/swlh/processing-16gb-file-in-seconds-go-lang-3982c235dfa2)

* * *

当今世界的计算机系统每天会产生海量的日志，但是这些日志并不适合存入到数据库中。因为这些数据都是不可变的，并且只用于数据分析和故常排除。因此大部分情况下这些日志都会被存放在磁盘的文件中。往往这些日志文件都会增长到 GB 级别，在需要对日志和打点进行数据分析时，如何快速高效的读取海量日志文件便成为了很影响工作效率的一件事。本文介绍一种使用 Golang 通过异步 IO 快速读取大日志文件的方法。

相关的工具代码在 [Github](https://github.com/linvon/coreader) 可以找到

### Start

首先我们来打开文件。我们使用 Go 标准库的 os.File 来处理所有文件的 IO。

``` go 
f, err := os.Open(fileName)
if err != nil {
    fmt.Println("cannot able to read the file", err)
    return
}
defer file.Close()  //Do not forget to close the file
```

打开文件之后，我们有两种方式来处理文件：

- 一行一行的读取文件。这种方式减轻了内存的压力，但是会在文件 IO 上会费更多的时间
- 一次性将整个文件读入内存。这种方式会消耗掉很多内存但极大地提高了运行效率

鉴于我们的日志文件很大，可能会有几个 GB 甚至十几 GB，我们无法将整个文件读入到内存中，但是第一种方法同样不太符合我们的需求，因为我们期望能在秒级内处理整个日志文件。
好在我们还有第三种方法，我们可以平衡两种方法，将整个文件按块读取到内存中，在 Go 中使用 bufio.NewReader() 就可以做到这一点。

``` go
r := bufio.NewReader(f)
for {
buf := make([]byte,4*1024) //the chunk size
n, err := r.Read(buf) //loading chunk into buffer
buf = buf[:n]
if n == 0 {
   
     if err != nil {
       fmt.Println(err)
       break
     }
     if err == io.EOF {
       break
     }
     return err
  }
}
```

每当我们读入了一个块，我们就启动一个 Goroutine 来并发的处理文件块，这样上面的代码会变成：

``` go
//sync pools to reuse the memory and decrease the preassure on //Garbage Collector
linesPool := sync.Pool{New: func() interface{} {
    lines := make([]byte, 500*1024)
    return lines
}}
stringPool := sync.Pool{New: func() interface{} {
    lines := ""
    return lines
}}
slicePool := sync.Pool{New: func() interface{} {
    lines := make([]string, 100)
    return lines
}}
r := bufio.NewReader(f)
var wg sync.WaitGroup //wait group to keep track off all threads
for {

    buf := linesPool.Get().([]byte)
    n, err := r.Read(buf)
    buf = buf[:n]
    if n == 0 {
        if err != nil {
            fmt.Println(err)
            break
        }
        if err == io.EOF {
            break
        }
        return err
    }
    nextUntillNewline, err := r.ReadBytes('\n') //read entire line

    if err != io.EOF {
        buf = append(buf, nextUntillNewline...)
    }

    wg.Add(1)
    go func() {

        //process each chunk concurrently
        //start -> log start time, end -> log end time

        ProcessChunk(buf, &linesPool, &stringPool, &slicePool, start, end)
        wg.Done()

    }()
}
wg.Wait()
```

上面的代码在效率上有两个优化点：

- **sync.Pool** 是一个高效的实例池，可以用于减轻 GC 的压力。我们可以对不同的切片复用内存分配，这大大减少了内存上的压力。
- 使用了 **Go Routines** 来并发处理文件块，这样可以进行异步 IO，提高了 CPU 的利用率，加快了整个文件的处理速度

最后我们来实现处理函数

``` go
func ProcessChunk(chunk []byte, linesPool *sync.Pool, stringPool *sync.Pool, slicePool *sync.Pool, start time.Time, end time.Time) {
	//another wait group to process every chunk further                             
	var wg2 sync.WaitGroup
	logs := stringPool.Get().(string)
	logs = string(chunk)
	linesPool.Put(chunk) //put back the chunk in pool
	//split the string by "\n", so that we have slice of logs
	logsSlice := strings.Split(logs, "\n")
	stringPool.Put(logs) //put back the string pool
	chunkSize := 100 //process the bunch of 100 logs in thread
	n := len(logsSlice)
	noOfThread := n / chunkSize
	if n%chunkSize != 0 { //check for overflow 
		noOfThread++
	}
	length := len(logsSlice)
	//traverse the chunk
	for i := 0; i < length; i += chunkSize {

		wg2.Add(1)
		//process each chunk in saperate chunk
		go func(s int, e int) {
			for i:= s; i<e;i++{
				text := logsSlice[i]
				if len(text) == 0 {
					continue
				}
				
				processLine(text)
			}
			textSlice = nil
			wg2.Done()

		}(i*chunkSize, int(math.Min(float64((i+1)*chunkSize), float64(len(logsSlice)))))
		//passing the indexes for processing
	}
	wg2.Wait() //wait for a chunk to finish
	logsSlice = nil
}

```

上面的代码处理 16GB 的文件耗时大概在 25S 左右


### 多元化兼容处理

在实际的生产环境中，我们肯定不会任由日志文件疯狂增长下去，一般情况下都会将日志文件进行切割和压缩。日志文件往往都会按照一定名称格式来存储，并且归档为 .gz 类型的文件。因此我在原本的代码基础上增加了一些定制内容，包括

- 指定文件名称格式化，支持多文件处理。（多文件处理采用了串行方式，实测并行文件处理在异步 IO 上优化不大，且会占用更多的内存）
- 支持 gz 文件读取
- 行处理函数可定制，自由定制输出格式


