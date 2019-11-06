<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [深度探索c++对象模型](#深度探索c对象模型)
	- [1.关于对象](#1关于对象)
		- [1.1c++对象模式](#11c对象模式)
		- [1.3对象的差异](#13对象的差异)
	- [3.Data语意学](#3data语意学)
		- [3.3Data Member的存取](#33data-member的存取)
		- [3.4继承与Data Member](#34继承与data-member)
	- [4.Function语意学](#4function语意学)

<!-- /TOC -->
# 深度探索c++对象模型
记录该书的笔记，参照译者给出的建议，先观看1,3,4章，然后再挑选感兴趣的章节阅读。
## 1.关于对象
* 封装后的布局成本
c++在布局以及存取时间上的主要额外负担是由virtual引起的，包括
  1. virtual function机制，用来实现有效率的执行期绑定(runtime binding)
  2. virtual base class，用来实现多次出现在继承体系中的base class只有一个单一而被共享的实例

### 1.1c++对象模式

* 简单对象模式
object储存指向member和functionmember的指针。
* 表格驱动对象模型
将所有和members有关的信息抽出来，放在一个data member table和一个member function table，class object只存放这两个table的指针。
这个模型没有应用于c++编译器中，但member function table称为支持virtual functions的一个有效方案。
* c++对象模型
此模型将非静态成员变量配置于每一个class object内，静态成员变量被存放在个别的class object之外。静态以及非静态成员函数都存放在个别class object之外。
虚函数通过以下两个步骤支持：
  * 每个class产生一堆指向virtual functions的指针，放在表格(虚函数表，virtual table/vtbl)中。
  * 每个class object被安插一个指针，指向相关的virtual table，这个指针被称为vptr。每个class关联的type_info object经由virtual table指出，通常在第一个。  

### 1.3对象的差异
只有通过pointer或reference的间接的间接处理，才能支持多态性质。
多态只存在于一个个的public class中。nonpublic的派生以及void*指针，也可以说是多态，但是没有被语言明确的支持，需要显示转换才行。

一般而言，一个表示一个class object由以下几方面构成：
* nonstatic data members的总和大小
* 加上任何犹豫alignment的需求而填补上的空间(可能在members之间，也可能在集合边界，在32位机上，通常为4bytes，以使bus运输量达到最高效率)
* 为了实现virtual而产生的额外负担

转换(cast)其实是一种编译器指令。大部分情况下它并不改变一个指针所含的真正地址，只是影响其“被指出内存的大小和内容”的解释方式。

## 3.Data语意学
### 3.3Data Member的存取
* static data members
  每一个static data member都只有一个实例，存放在  data segment中。每次取时，都会转换为对改唯一extern实例的直接操作。例如:
  ```
  origin.chunkSize = 250 to Point3D::chunkSize = 250
  pt->chunkSize = 250 to Point3D::chunkSize = 250
  ```
  即使是复杂继承而来的类，也一样如此。
  如果对静态成员变量取地址，会得到一个指向其数据类型的指针，这是因为static member并不在class中。
* nonstatic data members
此时，需要把class object的起始地址加上data member的偏移地址找到。
当偏移位置在编译期可知时，派生类和基类以及结构体成员的存取效率是相同的。当为虚拟继承时，存取速度会稍慢。
### 3.4继承与Data Member
一般而言，具体继承并不会增加空间或存取时间上的额外负担。
将一个class分解成多层，可能会因为每层的alignment而膨胀，消耗额外的空间。
* 对于多态，会带来空间和存取时间上的额外负担：
  * 将产生一个virtual talble，用来存放声明的每一个virtual function的地址。大小一般为被声明的virtual functions的个数，再加上一两个slots，用来支持运行时类型识别。
  * 每个class中导入一个vptr，提供执行期的链接，使每个object能够找到对应的virtual table。
  * 加强constructor，使其能够设定vptr。
  * 加强destructor，抹消vptr。

* 多重继承下，由于只有第一个base class是和当前class起始地址一致，所以只有第一个base class和单继承情况相同，在其他base class来说，都需要修改地址。
* 虚拟继承
  要实现虚继承，需要将多个class的虚基类折叠成一个由derived class维护的object。
  一般实现方式如下：class内如果含有一个或多个虚基类对象，那么将其分割为两部分，一个不变区域和一个共享区域。不变区域不管如何演化，都可以直接存取。共享区域会随着每次派生操作而产生变化，因此只能间接存取，各种编译器实现技术之间的差异就在于间接存取的方式不同。以下是三种主流策略：
  * cfront在derived class object中安插一些指针，每个指针指向一个virtual base class，要存取继承来的虚基类成员，可以通过指针间接的完成。
  这种模型有两个缺点，每个对象必须为每个virtual base class背负一个额外的指针。并且虚拟继承链的加长，会导致间接存取层次的增加。
  * 间接存取可以通过取得所有原始virtual base class指针来解决。
  * 对于负担不一致的问题，MS编译器通过virtual base class table来实现，将真正的指针放在table中，class只需要一个指针即可。
  * 对于负担不一致问题，另一种方案是在virtual function table中放置虚基类的offset，这样同样可以将虚基类公用一个实体。
## 4.Function语意学
