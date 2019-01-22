# 设计理由
GE有一个名为Memory Cloud（内存云）的分布式内存存储基础架构。这个memory cloud由一些memory trunks组成，集群中的每台机器都承载256个memory trunks。我们将一台机器的本地内存空间分为多个memory trunks的原因在于两点：

（1）可以不需要任何锁的开销实现trunk层面的并行；

（2）Memory trunk使用hash机制去做内存寻址。由于hash冲突的可能性较高，单个大hash表的性能是不太好的。

GE的memory cloud提供key-value访问接口。Key是64位的全局唯一标识符，value是任意长度的blob。由于memory cloud分布在多台机器上，我们不能使用它的物理内存地址去找到一个key-value对。为了根据一个给定key定位到它的value，我们首先识别存储key-value对的机器，然后在该机器上的一个memory trunk中找到这个key-vaule对。

对于不同的GE应用，key-value中的value部件有不同的数据结构或数据模式。我们使用`cell`术语来表示一个value部件。例如，我们考虑具有以下内容的cell：

（1）一个32位整数类型的Id；
（2）一组64位整数类型的Links。

也就是说，我们想要一个可变长度列表（带有Id）作为value组件。在C#里面，这样的结构可以被定义成下面这样：

```C#
struct CellA
{
  int Id;
  List<long> Links;
}
```

有效存储结构化类型的数据并支持高效优雅的操作是一项挑战。

我们可以直接使用大多数面向对象语言支持的对象去建模用户数据，如C++或者C#。它提供了一种方便直观的数据操作方式。我们可以通过它的接口去操作一个对象，例如：`int Id = cell.Id`或者`cell.Links[0] = 1000001`，这个里面的`cell`是这个类型的一个对象。这种方式尽管简单优化，但是有严重的不足之处。

* 首先，在内存中保存对象的存储开销非常高。
* 其次，面向对象语言通常不是为了运行时处理大量对象而设计的。
* 最后，加载和存储数据需要大量时间，因为序列化/反序列化很耗时，尤其是数据庞大的时候。

此外，我们可以将`value`作为blob并通过指针访问数据，将数据以blob形式存储可以最小化内存开销。它还可以提高数据操作的性能，因为数据操作不涉及数据反序列化。但是，系统不知道数据的结构。我们可以真正在blob里操作数据之前，我们需要知道准确的内存布局，也就是说，我们需要通过指针和地址偏移量去访问blob中的数据元素，这使得编程变得困难而且易于出错。

对于上面描述的数据结构（cellA，译者注），在blob里面，我们需要知道字段`Id`存在偏移量为0的位置，`Links`第一个元素是在偏移量为8的位置。为了拿到字段`Id`的值并将值设到`Links[0]`，需要写如下的代码：

```C++
byte* ptr = GetBlobPtr(123); // 拿到memory的指针
int Id = *(int*)ptr; // 拿到Id的值
ptr += sizeof(int); 
int listLen = *(int*)ptr;
ptr+=sizeof(int);
*(long*)ptr = 1000001; // Links[0]设置为1000001
```

注意，我们不能简单地通过像C#这样的编程语言提供的内置关键词`struct`将blob转换为定义的结构。也就是说下面这样的代码是不行的：

``` C#
// C# code snippet
struct
{
  int Id;
  List<long> Links;
}
....

struct * cell_p = (struct*) GetBlobPtr();
int id = *cell_p.Id; 
*cell_p.Links[0] = 100001;
```

这是因为像上述例子中`Links`这样的数据字段是引用类型。这种结构的数据字段不是平滑地分布在内存中的，我们无法用一个结构化指针去操作一块平滑的内存。
