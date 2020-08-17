---
title: "Golang 使用 reflect 包实现未知结构体的泛实例化"
date: "2020-05-25"
categories: ["学习笔记"]
---

开发中可能会遇到这种场景：同一个抽象方法，可能会接收不同的结构体参数，在方法内需要实例化这样一个结构体并进行赋值处理。虽然通过合理的代码结构安排与搭配上对接口的统一，可以实现这种抽象化的操作（球球快出泛型吧）。但有时候实际业务并没有很大，对于统一实现接口的需求并不高，都实现接口会显得代码很冗余，因此可以通过以下方式简单实现对于未知结构体的泛实例化。

如果我们对传入的结构体需要赋值的参数都固定好了类型，那么可以直接使用 interface 的 map 来做待赋值数据的存储，直接使用 reflect 包搞定。

``` go
// 反射传入的结构体并实例化一个实例，根据结构体字段在CSV行中找对应数据，找到的就设值
func instantiate(unknownSt interface{}, jsonData map[string]interface{}) interface{} {
	// 获取结构体的类型反射
	sType := reflect.TypeOf(unknownSt).Elem()
	// 根据类型实例化一个新结构体
	retSt := reflect.New(sType).Elem()
	// 遍历每一个结构体类型成员
	for i := 0; i < sType.NumField(); i++ {
		// 成员类型
		f := sType.Field(i)
		// 新结构体的对应成员
		fv := retSt.Field(i)
		// 查找数据Map，赋值
		if v, ok := jsonData[f.Tag.Get("json")]; ok && v != nil {
			fv.Set(reflect.ValueOf(v))
		}
	}
	return retSt
}

type tSt struct {
	A string `json:"a"`
	B int    `json:"b"`
}

func main() {
	a := "aa"
	b := 6
	jd := make(map[string]interface{}, 0)
	jd["a"] = a
	jd["b"] = b
	ret := instantiate(&tSt{}, jd)
	fmt.Println(ret)
}
```

如果我们是直接从外部数据（比如文本、字符串）中解析值并赋值到结构体，也可以根据需要手动判断类型，比如如下代码：

``` go
func instantiate(unknownSt interface{}, jsonData map[string]string) interface{} {
	// 获取结构体的类型反射
	sType := reflect.TypeOf(unknownSt).Elem()
	// 根据类型实例化一个新结构体
	retSt := reflect.New(sType).Elem()
	// 遍历每一个结构体类型成员
	for i := 0; i < sType.NumField(); i++ {
		// 成员类型
		f := sType.Field(i)
		// 新结构体的对应成员
		fv := retSt.Field(i)
		// 查找数据Map，根据结构体成员类型对值进行转化赋值
		if v, ok := jsonData[f.Tag.Get("json")]; ok && v != "" {
			switch fv.Type().String() {
			case "string":
				fv.SetString(v)
			case "int", "int64":
				ival, err := strconv.ParseInt(v, 10, 0)
				if err != nil {
					log.Println(err)
				}
				fv.SetInt(ival)
			case "float64":
				ival, err := strconv.ParseFloat(v, 0)
				if err != nil {
					log.Println(err)
				}
				fv.SetFloat(ival)
			default:
			}
		}
	}
	return retSt
}
```

