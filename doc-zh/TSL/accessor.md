### 设计理由
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

有效存储结构化类型的数据并支持高效优雅的操纵是一项挑战。

我们可以直接使用大多数面向对象语言支持的对象去建模用户数据，如C++或者C#。它提供了一种方便直观的数据操纵方式。我们可以通过它的接口去操纵一个对象，例如：`int Id = cell.Id`或者`cell.Links[0] = 1000001`，这个里面的`cell`是这个类型的一个对象。这种方式尽管简单优化，但是有严重的不足之处。

* 首先，在内存中保存对象的存储开销非常高。
* 其次，面向对象语言通常不是为了运行时处理大量对象而设计的。
* 最后，加载和存储数据需要大量时间，因为序列化/反序列化很耗时，尤其是数据庞大的时候。

此外，我们可以将`value`作为blob并通过指针访问数据，将数据以blob形式存储可以最小化内存开销。它还可以提高数据操纵的性能，因为数据操纵不涉及数据反序列化。但是，系统不知道数据的结构。我们可以真正在blob里操纵数据之前，我们需要知道准确的内存布局，也就是说，我们需要通过指针和地址偏移量去访问blob中的数据元素，这使得编程变得困难而且易于出错。

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

这是因为像上述例子中`Links`这样的数据字段是引用类型。这种结构的数据字段不是平滑地分布在内存中的，我们无法用一个结构化指针去操纵一块平滑的内存。

我们可以将`value`组件看作blob并通过一个高层语言声明和访问数据。在第三种方式里，我们通过一个声明式语言（比如SQL）定义并操作数据。但是，声明式语言通常表达能力有限或者缺乏有效的实现，而这两者对于GE来说都是尤其重要的。

GE将用户数据存储为blobs而不是运行时对象，这样存储开销达到最小化。与此同时，GE允许我们像在C#或者Java里的那样以面向对象的方式去访问数据。例如，在GE里面，即使我们是在blob里操作数据，我们可以做如下的事情：

``` C#
CellA cell = new CellA();
int Id = cell.Id;
cell.Links[0] = 1000001;
```

换句话说，我们仍然可以一种优雅、面向对象的方式在blob上操作。GE是通过`cell accessor`的机制实现它的。具体来说，我们使用TSL脚本声明数据模式。GE编译脚本然后为定义在TSL脚本里的cell构造器生成cell accessors。然后，如果数据是C#运行时对象，我们可以通过cell accessors访问blob数据，但事实上是cell accessors映射了在cell构造器里声明的字段与它准确的内存位置。所有数据访问操作将被正确地映射到准确内存位置，而不需要任何内存复制开销。

让我们用一个例子来演示cell accessor如何工作。在使用cell accessor之前，我们必须先指定它的cell结构，这个是通过TSL完成。对于前文里的例子，我们在TSL里面定义如下的数据结构：

```
cell struct CellA
{
    int id;
    List<long> Links;
}
```

注意，`CellA`不是C#里的`struct`定义，尽管看上去相似。这份代码片段将被TSL编译器编译为`CellA_Accessor`。编译器生成的代码用于将`CellA`字段上操作翻译为底层blob的内存操纵行为。在编译之后，我们使用面向对象数据访问接口去访问blob里的数据，如下所示。

浏览器不支持SVG。

除了直观的数据操纵接口之外，cell accessors也可以提供线程安全的数据操纵保证。GE被设计用于在多线程环境运行，其中大量cell之间以非常复杂的方式交互。为了使应用程序开发人员的生活更轻松，GE通过cell accessor提供了线程安全的cell操纵接口。

### 使用
在使用accessor的时候，必须应用一些使用规则。

#### 不能缓存Accessor
Accessor就像一个数据“指针”工作，它不能被缓存给将来使用，因为accessor指向的内存块可能被其他accessor操作移动。例如，我们有一个`MyCell`定义如下：

```
cell struct MyCell
{
    List<string> list;
}
```

``` C#
TrinityConfig.CurrentRunningMode = RunningMode.Embedded;
Global.LocalStorage.SaveMyCell(0, new List<string> {"aaa", "bbb", "ccc", "ddd"});
using (var cell = Global.LocalStorage.UseMyCell(0))
{
    Console.WriteLine("Example of non-cached accessors:");
    IEnumerable<StringAccessor> enumerable_accessor_collection = cell.list.Where(element => element.Length >= 3);
    foreach(var accessor in enumerable_accessor_collection)
    {
        Console.WriteLine(accessor);    
    }
    Console.WriteLine("Example of cached accessors:");
    List<StringAccessor> cached_accessor_list = cell.list.Where(element => element.Length >= 3).ToList(); // Note the ToList() at the end
    foreach (var accessor in cached_accessor_list)
    {
        Console.WriteLine(accessor);
    }
}
```

上面显示的代码片段将输出：

```
Example of non-cached accessors:
aaa
bbb
ccc
ddd
Example of cached accessors:
ddd
ddd
ddd
ddd
```

对于例子中的非缓存的accessor，accessor的值在被使用的时候才进行计算。对于例子中缓存的accessor，accessor通过`cell.list.Where(element => element.Length >= 3).ToList()`返回的列表实际上是指向同一个accessor的引用，并且这个accessor指向的是`cell.list`最后一个元素。

#### cell accessor和message accessor在使用后必须被dispose
cell accessor是`disposible`对象，在使用后必须被dispose。在C#里面，disposable对象可以使用`using`构造体去清理：

``` C#
long cellId = 314;
using (var myAccessor = Global.LocalStorage.UseMyCell(cellId))
{
    // Do something with myAccessor
}
```
> cell accessor或message accessor必须被正确地dispose。如果一个accessor在使用后没有dispose会发生非托管资源泄露。

TSL协议的*request/response* reader和writer被称作*Message Accessors*。它们也是disposible对象，也必须在使用后被正确dispose。

``` C#
using (var request = new MyRequestWriter(...))
{
  using (var response = Global.CloudStorage.MyProtocolToMyServer(0, request))
  {
    // Do something with the response
  }
}
```

#### cell accessor不能以嵌套方式使用
每个cell accessor都有关联的自旋锁（spin-lock）。嵌套使用cell accessor可能导致死锁。下面的代码片段里显示的cell accessor被不正确使用了:

``` C#
using (var accessorA = Global.LocalStorage.UseMyCellA(cellIdA))
{
  using(var accessorB = Global.LocalStorage.UseMyCellB(cellIdB))
  {
    // Nested cell accessors may cause deadlocks
  }
}
```

> cell accessor不应该在另一个accessor里嵌入使用。两个或以上的嵌套cell accessor可能导致死锁。

注意，一个cell accessor以及一个或多个request/response reader(s)/writer(s)可以嵌套方式使用。

