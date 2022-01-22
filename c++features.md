### 四种cast

#### static_cast

​		可用于void*类型转换为任意类型；非const转换为const；多态时的上行转换，但是下行转换时是不安全的。

#### dynamic_cast

​		可用于动态类型的上行转换和下行转换。上行转换时不做任何的检测，可以直接转换成功；下行转换时会通过RTTI来检测是否能够转换成功，转换指针失败时返回nullptr，转换引用失败时，抛出bad_cast异常。同时，因为下行转换时需要用到RTTI，所以只有当基类中有虚函数时才能用dynamic_cast进行下行转换，否则编译会报错。RTTI相关的还有typeid运算符。

#### const_cast

​		可用于非const转换为const；const转换为非const。

#### reinterpret_cast

​		几乎什么都可以转，可能会出问题，所以尽量少用。

#### dynamic_pointer_cast

​		这个同dynamic_cast用法相同，不同点在于dynamic_pointer_cast作用于智能指针，对智能指针进行上行和下行转换。

