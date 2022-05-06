# Lambda Function Bind

## 表达形式

```cpp
[capture list](parameter list) -> return type { function body }
//捕获列表，是lambda所在函数中定义的局部变量列表，通常为空
```

#### 实例

```cpp
//打印 42
auto f = [] { return 42; }
//调用
cout<<f()<<endl;

//传入sort的谓词函数，作用：排序二级条件，更短
bool isShorter(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
//lambda形式
[](const string &s1, const string &s2)
{
    return s1.size()<s2.size();
}
```

## 捕获列表

lambda 可以将局部变量放入[]中来使用[]中的局部变量

#### 实例

```cpp
//调用lambda的函数中含有一个局部变量sz，如果不放入捕获列表，则无法使用
[sz](const string &a)
{
    return a.size()>sz;
}
```

### 值捕获

类似于参数传递，创建lambda时，创建对变量的拷贝，**不允许修改**

```cpp
void fcn1()
{
    size_t v1 = 42;//局部变量
    //对象f2包含对v1的拷贝
    auto f = [v1]
    { return v1; };
    v1 = 0;
    //j=42
    auto j = f();
}
```

### 引用捕获

创建lambda时，创建对变量的引用

```cpp
void fcn1()
{
    size_t v1 = 42;//局部变量
    //对象f2包含对v1的引用
    auto f = [&v1]
    { return v1; };
    v1 = 0;
    //j=0
    auto j = f();
}
```

## 隐式捕获

[&] 编译器会根据lambda代码自动进行引用捕获

[=] 自动进行值捕获

如果混用隐式捕获和现实捕获，第一个参数必须为隐式捕获

## 捕获列表

| []                  |                                 |
| ------------------- | ------------------------------- |
| [names]             |                                 |
| [&]                 |                                 |
| [=]                 |                                 |
| [&,identifier_list] | 引用捕获0个至多个identifier列表 |
| [=,identifier_list] | 值捕获                          |

## 可变lambda

值拷贝变量的修改不会改变原值，因此可以使用mutable改变捕获变量的值

默认lambda使用值捕获时，之后在lambda函数体内，使用被值捕获的变量时，该变量值将永远是其被捕获时，被lambda看到的值，一般这个值是无法改变的。

**如果加上mutable，则会使得该“值捕获变量”的值，可以在被捕获的值的基础上进行变化。**
且多次调用lambda函数对捕获变量值，所造成的改动会被累积。这是由于lambda函数其实也是一种类，然后下面这个初始化语句

```cpp
void fcn1()
{
    size_t v1 = 42;
    auto f = [v1]() mutable
    {
        v1 = 30;
        return v1;
    };
    v1 = 0;
    auto j = f();
    cout << j << endl;
}
```

## 指定返回类型

```cpp
[] (int i)->int {if(i<0)return -i;else return i;};
```

## bind

可以很容易的为find_if写一个比较函数，但find_if只接受一元谓词，即该函数只能有一个形参，如果有两个就很难实现。

```cpp
#include <functional>
auto newCallable = bind(callable, arg_list);
//当调用newCallable时，会调用callable，并传入arg_list中的参数
//占位符形式为 std::placeholders::_n
//表示新的函数对象中的参数的位置，当调用新的函数对象时，新函数对象会调用被调用函数，并且其参数会传递到被调用函数参数列表中持有与新函数对象中位置对应的占位符。
 void function(arg1,arg2,arg3,arg4,arg5)
 {
     
 }
auto g = bind(function,a,b,_2,c,_1);
//原函数有5个参数
//g函数有2个参数，分别为_2,_1代表着原函数的arg3和arg5，a,b,c变成了固定值成为了g()的一部分
//将function(arg1,arg2,arg3,arg4,arg5),映射为g(_1,_2)

bool check_size(const string &s,string::size_type sz)
{
    return s.size() >= sz;
}
int main(int, char**) {
    string s = "hello";
    bool a = check_size(s, 6);
    auto check6 = bind(check_size, std::placeholders::_1, 6);
    bool b = check6(s);
}
//可以将
```
### 绑定引用参数
默认下，非占位符的参数以拷贝的形势传入到bind中
```cpp
for_each(v.begin(),v.end(),
        [&os,c](const string &s){os<<s<<c;});
//输出v的每个元素
//同样可以用函数替代lambda
ostream &print(ostream &os,const string &s,char c)
{
    return os<<s<<c;
}
//如果使用bind完成呢
for_each(v.begin(),v.end(),bind(print,os,_1,' '));
//会出错
//因此如果希望传递给bind的一个对象而不拷贝他，就要使用ref
```

### ref

```cpp
for_each(v.begin(),v.end(),bind(print,ref(os),_1,' '));
```

