记录effective c++中的一些知识点，会比较随意。
* 以const,enum,inline替换define
使用const/enum替换define的常量，在调试中带有符号信息，并且不会出现多份。
形似函数的宏，可以考虑用inline方式，避免宏定义容易出现的一些错误
* 在使用变量前初始化
需要注意，在进入构造函数前，会调用各个类成员的构造进行初始化，因此，采用构造函数内赋值的方式进行，会进行两次赋值，效率更低。
* 若不想使用编译器生成的函数，就该明确拒绝
例如，不想被拷贝，那么就需要禁止产生默认的拷贝构造。可以用delte关键字，或者声明为private来实现。
* 多态基类的析构声明为virtual
如果基类的析构不是virtual的，那么用基类指针指向派生类析构的时候，通常会导致派生类对象没有被销毁掉。
* 不在构造和析构中调用virtual函数
因为构造和析构会在构造对象前调用基类的，因此不会下降至derived class。
* 以对象管理资源
也被称为资源获取即初始化(Resource Acquisition Is Initialization, RAII)
* 资源管理类中小心copy行为
因为RAII对象资源的复制行为决定了其对象的复制行为，常见的为：抑制copying，引用计数。
* 在资源管理类中提供对原始资源的访问
* 若所有参数都需要类型转换，采用non-member函数
如果this指针所指的隐喻参数也需要隐式类型转换，那么这个函数必须为non-member
* 尽量少做转型动作
旧式风格： (T) expression T(expression)
c++新式风格：const/dynamic/reinterpret/static_cast
dynamic_cast非常慢，应该尽量避免，同时宁可使用新式的转型，也不用旧式的，因为前者容易区分，且功能也有分类。
* 将文件间的编译依存关系降至最低
采用pimpl方式，尽可能使用引用、指针实现，并尽量以声明代替定义。
* 绝对不要重新定义继承来的non-virtual函数
* 了解typename的双重意义
在template声明式中，typename和class意义相同。但是，当template中嵌套了从属的名称，会假定不是个类型，此时可以用typename告知这个是类型。
* 不要忽视编译器的警告
