---
title: '[Golang] Array v.s. Slice'
tags:
  - Golang
categories:
  - Programming
  - Golang
date: 2019-12-30 20:41:00
slug: golang-array-and-slice
---

## Array

陣列的長度固定，且為陣列類型的一部份。
（`[4]int` 以及 `[5]int` 為兩種獨立且不相容的不同type）


### 宣告及初始化array
```go

// var [len]Type，無初始化值，則初始值為zero value
var arr1 [2]int

// var [len]Type，並給初始值
var arr2 = [2]int{1, 2}

// var [...]Type，讓complier自行計算長度
var arr3 = [...]int{1, 2}

// := [len]Type，在func中使用簡短宣告符號，無給初始值
arr4 := [2]int{}

// := [len]Type，並給初始值
arr5 := [2]int{}

// := [...]Type，讓complier自行計算長度
arr6 := [...]int{1, 2}
```
<!--more-->
當賦值和傳遞array時，是複製整個陣列內容
```go
package main

import (
	"fmt"
)

func main() {
	arr1 := [2]string{"apple", "banana"}
	arr2 := [2]string{}

	arr2 = arr1

	fmt.Printf("arr1: %p, %v \n", &arr1, arr1)
	fmt.Printf("arr2: %p, %v \n", &arr2, arr2)

	arr3(arr1)
}

func arr3(x [2]string) {
	fmt.Printf("arr3: %p, %v \n", &x, x)
}
```
執行結果
```
arr1: 0x40a0e0, [apple banana] 
arr2: 0x40a0f0, [apple banana] 
arr3: 0x40a120, [apple banana] 
```

### Array 缺點
上面三個array的位址皆不相同，驗證傳遞時都是複製整個陣列  
如此，可發現使用array的兩個弊處
<table><tr><td bgcolor=AliceBlue>
1. array是固定長度, 使用起來很不彈性</p>
2. 傳遞時複製整個組數，array一大時，便會消耗大量記憶體
</td></tr></table>

{{% notice warning %}}
所以在大多數的Go code中，<font color=DarkBlue>**slice較常被使用**</font>
{{% /notice %}}


## Slice

slice為Go中的一種資料結構，定義在 `src/runtime/slice.go` 中：
src/runtime/slice.go
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

![](https://imgur.com/X3yxgmp.png)

各field代表如下:
<table><tr><td bgcolor=AliceBlue>
1. pointer：指向底層陣列開始的位置 </p>
2. len：lenth, 為slice的實際長度 </p>
3. cap：capacity, 為當前分給slice的容量大小    
</td></tr></table>

### 宣告slice

slice宣告方式有兩種，一種是直接像array一樣宣告, 只差不須指定大小：
```go
a := []int{1, 2, 3, 4, 5}
fmt.Println(len(a))  //5
fmt.Println(cap(b))  //5
```
另一種是使用內建函數make  
`func make([]T, len, cap) []T`  
make可以事先指定底層陣列長度以及欲使用的容量大小
```go {linenostart=4}
b := make([]int, 5, 5) //b == [0, 0, 0, 0, 0]
fmt.Println(len(b))  //5
fmt.Println(cap(b))  //5
```
使用make時，<font color=DarkBlue>capacity參數可以被省略</font>，而此時的cap==len

```go {linenostart=7}
c := make([]int, 5)  //c == [0, 0, 0, 0, 0]
fmt.Println(len(c))  //5
fmt.Println(cap(c))  //5
```

由此得知，slice 本身是一個<span class="dotunderletter">引用型別</span>，底層會有個指標指向一個array。上面宣告的三個slice  a , b , c 都可以用下圖表示

![](https://imgur.com/vsjGEip.png)


### Slice的各種使用
#### re-slicing 重新切片
使用冒號間隔上下兩參數`[:]` 可以擷取slice特定範圍的值，  
`[low-index:high-index]`  
表示取出從`low-index`開始的值到到`high-index減1`的值，high-index的索引代表<span class="dotunderletter">及於但不包含</span>(up to but not include)
兩個索引值都可以被忽略，忽略時，上邊界默認值為0, 下邊界默認值為len(x)

```go
x := []int{1, 2, 3, 4, 5}
y := x[:]  // y == [1 2 3 4 5]
y = x[:2]  // y == [1 2]
y = x[2:]  // y == [3 4 5]
y[2] = 0   // y == [3 4 0] ; x == [1 2 3 4 0]
```
須特別留意的是
{{% notice warning %}}
使用`[:]`重新切片後，底層的陣列不變，意即y.ptr == x.ptr + 8 (位移了兩位)。 當修改了`y[2]`的值變是修改了`x[4]`
{{% /notice %}}

若是不想要兩個slice共用同一個array，則要使用下一個內建函數copy

#### copy 拷貝
使用copy可以複製slice中的array到另一個新的slice的新的array
`copy(dist-slice, src-slice)`

若len(src)小於len(dist)，則會覆蓋目的端的前len(src)個數值：
```go
x := []int{1, 2, 3, 4, 5}
y := []int{6, 7, 8}
copy(x, y) // x == [6 7 8 4 5] 把y複製進x
y[2] = 0  // y == [6 7 0] ; x == [6 7 8 4 5] 改了y的值，x不受影響，因為底層array不同
```
若len(src)大於len(dist)，則會複製來源端與len(dist)等長的值：
```go
x := []int{1, 2, 3, 4, 5}
y := []int{6, 7, 8}
copy(y, x) // y == [1 2 3] 把x複製進y
```
另外也可以結合`:`指定要複製的位置：
```go
x := []int{1, 2, 3, 4, 5}
y := []int{6, 7, 8}
copy(y, x[2:]) // y == [3 4 5]，把x從index 2開始的值複製進y
```

#### Append 擴增
append用來在slice中添加資料，以下為用法：

```go

x := []int{1, 2, 3, 4, 5}

x = append(x, 6)  //append一個數給自己

y := append(x, 7, 8)  //用別的slice append多個數再給自己
 
c = append(b, a…) //拿另一個slice append另一個slice再給自己
```

</br>

{{% notice warning %}}
問題來了，使用append函數時，底層的array會是一樣的嗎？
答案是<font color=Red size= 5>**不一定**</font>
{{% /notice %}}
</p>

#### append底層array的共用與分裂

```go
x := []int{}  //len(x) == cap(x) == 0
y := append(x, 1) //len(y) == 1, cap(y) == 2
x = append(x, 2)
```

請問此時的`y[0]`是多少? `y[0] == 1`
這個例子下 x & y 分別會指向不同的array，因為在第二行，len(y) != cap(x) 的當下，y底層的array就已經另發配新的位址給新的array了

如果x換成以下宣告:

```go
x := make([]int, 0, 3)  //len(x) == 0, cap(x) == 3
y := append(x, 1) //len(y) == len(x) == 1 < cap(x) == 3
x = append(x, 2) //len(y) == len(x) == 2 < cap(x) == 3
```
請問此時的`y[0]`是多少？ `y[0] == 2`

`y[0]`會改變的原因是在一開始宣告時記憶體就已經分配cap(x) == 3的空間給底層array，此時使用append加入一個數到array，因為len(y) == len(x) == 1 < cap(x) =3，空間足夠，所以 **x及y的指標會指向同一個位址！** 而後再append一個數到array，也會因為空間足夠，所以兩個slices仍在同一個位址。

再衍生另一個問題－
若原先的最後一行 `x = append(x, 2)` 改成 `x = append(x, 2, 3, 4, 5)`，`y[0]`是多少？

```go
x := make([]int, 0, 3)
y := append(x, 1)
x = append(x, 2, 3, 4, 5)
```
此時的 `y[0] == 1`

因為x插入了4個數值，len(x) = 4 > cap(x) = 3，x使用的array便會換一個新的位址，並擴大capacity成len(x)的兩倍，意即cap(x) = 8
而原先的y保持原來的位址以及數值，len(y) = 1 < cap(y) = 3

由此可知，當在使用append時，如果len >= len時，會開另一塊更大記憶體的array，再把原先的array複製進去。  
擴大原則：當容量(cap)小於1024時，會擴大兩倍的容量；當大於1024後，擴大 1.25倍。


## 結論

{{% notice note %}}
1. 使用slice比起使用固定長度的array來得有彈性
2. slice本身為一個引用型別，底層有指標指向一個array
3. 重新切片指改變指標位址，底層陣列不變
4. copy為另外建一個新的底層array
2. 使用slice時建議事先使用make並規劃好容量cap，這樣在使用append時可以避免反覆重新分配記憶體
{{% /notice %}}




## Reference 
- [https://blog.golang.org/go-slices-usage-and-internals](https://blog.golang.org/go-slices-usage-and-internals)
- [https://studygolang.com/articles/6557](https://studygolang.com/articles/6557)
- [https://halfrost.com/go_slice/](https://halfrost.com/go_slice/)

2019的最後一篇文章，祝大家新年快樂 :sparkles:
希望明年一切都好，都能在想要的道路上開心的努力著！:muscle: