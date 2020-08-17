---
title: "Go 函数中使用append修改传入的slice为何无效？"
date: "2020-04-11"
categories: ["学习笔记"]
---


我们都知道在 Go 中明确表示没有传引用，所有的函数参数都是传值。而像 map、chan、slice 这些引用类型，虽然是传值，但在函数内亦可以修改其值，达到传引用的效果

但在执行如下代码时，会发现跟预想的结果不一样：

``` go
func Change(slice []int) {
    slice = append(slice, 1)
}
func main() {
    slice := []int{}
    Change(slice)
    fmt.Println(slice)
}
```

执行后会发现，slice 仍旧是空的切片，其原因是：
**如果切片的当前大小不足以附加新值，那么切片需要动态增长，从而更改了基础数组。如果没有返回此新切片，那么追加的更改是不生效的。**

其解决办法有两种：

- 直接 return 返回新切片

``` go
func Change(slice []int) []int {
    slice = append(slice, 1)
    return slice
}
func main() {
    slice := []int{}
    slice = Change(slice)
    fmt.Println(slice)
}
```

- 使用切片指针作为参数

``` go
func Change(slice *[]int) {
    *slice = append(*slice, 1)
}
func main() {
    slice := []int{}
    Change(&slice)
    fmt.Println(slice)
}
```