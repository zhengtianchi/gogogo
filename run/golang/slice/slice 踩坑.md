# slice 踩坑



案例一(slice传参)：

```Go
//情况一
// 情况一：输出[1,2,3]
func main() {
	slice := make([]int,0,4)
	slice = append(slice,1,2,3)
	TestSlice(slice)
	fmt.Println(slice)
}

func TestSlice(slice []int)  {
	slice = append(slice,4)
}

//情况二
func main() {
	slice := make([]int,0,4)
	slice = append(slice,1,2,3)
	TestSlice(slice)
	fmt.Println(slice)
}

func TestSlice(slice []int)  {
	slice = append(slice,4)
	slice[0] = 10
}


//情况三
func main() {
	slice := make([]int,0,3)
	slice = append(slice,1,2,3)
	TestSlice(slice)
	fmt.Println(slice)
}


func TestSlice(slice []int)  {
	slice = append(slice,4)
	slice[0] = 10
}
```

情况一：输出[1,2,3]

情况二：输出[10,2,3]

情况三：输出[1,2,3]

这里需要明确两个点：

1、golang中只有值传递

2、golang中slice是一个struct，结构如下：

![image-20211105144537337.png](https://img-blog.csdnimg.cn/img_convert/f8535a60ab4fbff5eea0156f9a4eaf10.webp?x-oss-process=image/format,png)

 

### 情况一和情况二：

外部的slice 传参到TestSlice，这里发生了复制，结构如下：

![image-20211105144606073.png](https://img-blog.csdnimg.cn/img_convert/f8e9bbb9f08f006e4713789ea4efca87.webp?x-oss-process=image/format,png)

由于slice中持有的是数组的指针，所以这里两个slice指向的是同一个数组。所以改变同一个数组会影响到两个slice。

但是由于打印slice是受len控制的，所以这里情况一就会打印[1,2,3]。但是情况二就会打印[10,1,2]。

通过强行修改len，可以打印出1，2，3，4

```Go
func main() {
	slice := make([]int,0,4)
	slice = append(slice,1,2,3)
	TestSlice(slice)
	(*reflect.SliceHeader)(unsafe.Pointer(&slice)).Len = 4 //强制修改slice长度
	fmt.Println(slice)
}


func TestSlice(slice []int)  {
	slice = append(slice,4)
}
```

### 情况三：

情况三跟一、二的区别在于，情况三的初始容量是3，并且随后放入了1，2，3三个元素。所以在传参前slice就已经满了。

然后在函数里发生了append，导致数组发生了扩容。

扩容逻辑：

1、根据策略申请一个更大的数组空间(slice容量的扩容规则：当原slice的cap小于1024时，新slice的cap变为原来的2倍；原slice的cap大于1024时，新slice变为原来的1.25倍)

2、copy 旧数组中的数据到新数组

3、添加新增的数据

4、将数组的指针复制给slice

扩容后，结构如下：

![image-20211105144857810.png](https://img-blog.csdnimg.cn/img_convert/9bad8ced02d66d815cd2556c502fd78b.webp?x-oss-process=image/format,png)

所以函数里改变数组对原始的slice没有任何改变

可以通过下列方式看出slice底层的数组地址变化，可以发现前两个输出值一样，第三个输出不一样。证明指向的数组产生了变化。

```Go
func main() {
	slice := make([]int,0,3)
	slice = append(slice,1,2,3)
	fmt.Println(unsafe.Pointer(&slice[0]))
	TestSlice(slice)
	fmt.Println(slice)
}


func TestSlice(slice []int)  {
	fmt.Println(unsafe.Pointer(&slice[0]))
	slice = append(slice,4)
	slice[0] = 10
	fmt.Println(unsafe.Pointer(&slice[0]))
}
```

## 案例二(slice append)：

```Go
//情况一
func main() {
   slice1 := make([]int, 0, 4)
   slice1 = append(slice1, 1, 2, 3)
   slice2 := append(slice1, 4)
   slice2[0] = 10
   fmt.Println(slice1)
   fmt.Println(slice2)
}

//情况二
func main() {
	slice1 := make([]int, 0, 4)
	slice1 = append(slice1, 1, 2, 3)
	slice2 := append(slice1, 4,5)
	slice2[0] = 10

	fmt.Println(slice1)
	fmt.Println(slice2)
}
```

情况一：输出[10,2,3] [10,2,3,4]

情况二：输出[1,2,3] [10,2,3,4,5]

原理是类似的，append过程如果没有发生扩容，那么两个slice就指向同一个数组，如果发生扩容就会分别指向不同的数组。

## 案例三(切片)：

```Go
//情况一

func main() {
   slice1 := make([]int, 0, 4)
   slice1 = append(slice1, 1, 2, 3)
   slice2 := slice1[:len(slice1)-1]
   slice2[0] = 10

   fmt.Println(slice1)
   fmt.Println(slice2)
}


//情况二
func main() {
  slice1 := make([]int, 0, 4)
  slice1 = append(slice1, 1, 2, 3)
	slice2 := slice1[:len(slice1)-1]
	slice2 = append(slice2,11,12,13,14,15)

	slice2[0] = 10
	fmt.Println(slice1)
	fmt.Println(slice2)
}
```

情况一：输出[10,2,3] [10,2]

情况二：输出[1 2 3] [10 2 11 12 13]

原理是类似的，append过程如果没有发生扩容，那么两个slice就指向同一个数组，如果发生扩容就会分别指向不同的数组。

## 深拷贝

可以看出，golang的slice操作默认都是浅拷贝。触发发生扩容才会让两个slice指向不同的数组。在实际业务中，很多场景是需要深拷贝的，这个时候可以使用copy函数

```scss
copy(newSlice,oldSlice)
```