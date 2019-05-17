# 语法

## 变量类型

* bool、string

* （u）int、（u）int8、（u）int16、（u）int32、（u）int64、uintptr

* byte、rune

* float32、float64、complex64,complex128

  golang只存在强制类型装换

## 定义变量

1、使用var 关键字

```shell
var a,b,c,bool
var s1,s2 string = "hello","word"
```

* 可以放在函数内，也可以直接放在包内
* 可以使用var () 集中定义

2、使用``:=``定义

```shell
a,b,c,d:=true,false,5,"hello"
```

* 只能在函数内使用

## 常量

* 关键词：const 

## 条件语句

### if

```shell

```

### switch

## 循环

### for

## 函数

* 返回值类型写在最后面
* 可返回多个值
* 函数作为参数
* 没有默认参数，可选参数、函数重载

## 指针

* GoLang 只有值传递一种
* 通过指针来达成引用传递的效果

## 数组

* 数组是值类型  
* go语言一般不直接使用数组

```go
var arr1 [5]int
arr2 :=[3]int{1,3,5}
arr3 :=[...]int{2,4,6,8,10} 
var grid [4][5]bool

//遍历数组
for i,v := range arr3{
    fmt.Println(i,v)
}
```

## Slice(切片)

* slice可以向后扩展，不可以向前扩展
* 添加元素时如果超越cap,系统会重新分配更大的底层数组
* 由于值传递，必须接收append的返回值

```go
####CREATE#########
s1 := []int{2,4,6,8}
//创建一个长度为16的切片
s2 :=  make([]int,16)
//创建一个长度为10，cap为32的切片
s3 := make([]int,10.32)
#####copy###########
copy(s2,s1)
#####delete#########
//没有删除方法，利用跳过方法删除
s2 = append(s2[:3],s2[4:]...)
```

## MAP

* map[K]V   ——简单map

*  map[K1]map[K2]V ——复合map

* make(map[string]int)——创建

* m[key]——获取

* value,ok:= m[key]来判断是否存在key

  1、map的key

  map使用哈希表，必须可以比较相等

  除了slice,map,function的内建类型都可以作为key

  Struct类型不包括上述字段，也可以作为key

```go
m :=map[string]string {
    "name":"ccmouse",
    "course":"golang",
}
// empty map
m2 := make(map[string]int)
// nil
m3 := map[string]int
//取值
name ：= m["name"]
name,ok := m["name"]
if name,ok:=m["name"];ok{
    fmt....
}else{
    .....
}
//删除
delete(m,"name")
//遍历
for k,v := range m {
    fmt.Println(k,v)
}

```

## 结构体和方法

* go语言仅支持封装，不支持继承和多态
* go语言没有class，只有struct 
* 使用.来访问成员  

```shell
# 定义
type [name] struct{}
示例：
type treeNode struct{
    value int
    left,right *treeNode
}
# 使用
var root treeNode
root = treeNode{value:3}
root = new(treeNode)

#### 接收者
func (node treeNode) print(){
    fmt.print(node.value)
}
```

## 封装

* 名字一般使用CameCase
* 首字母大写：public
* 首字母小写：private

public和private针对的是包

## 包

* 一个目录一个包
* main包包含可执行入口
* 为结构定义的方法必须放在同一个包内
* 可以是不同的文件

扩展包

* 组合
* 别名

## GOPATH

* go get 命令
* gopm 来获取无法下载的包
* go build来编译
* go install 产生pkg文件和可执行文件
* go run 直接编译运行

## 接口

* 接口由使用者定义
* 只要实现定义的方法，就认定实现了接口
* 接口可以组合

### 常用系统接口

Stringer ——相当于 toString

Reader——

Writer——

## 函数式编程

