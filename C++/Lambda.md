# Lambda

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

