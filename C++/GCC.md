# GCC

```bash
gcc -std=c++14 -march=native -pipe -O2 -Wall -Wextra -fopenmp -fPIC -fext-numeric-literals
```

1. **-std=c++14** 指定版本号
2. **-march=** 选择目标体系结构 native 为本机
3. **-pipe** 使用管道加速编译，使用管道使中间文件不生成，直接传出给下一个流程
4. **-O2** 优化等级
   1. -O0无任何优化
   2. -O1做少许优化，对代码分支，常量及表达式进行优化，减少编译时间
   3. -O2在O1的基础上，进行寄存器和指令级的优化，编译时会占用更多内存和编译时间
   4. -O3普通函数内联，针对循环进行优化
   5. -Os减少代码大小
5. **-Wall** 编译后显示所有警告
6. **-Wextra** 启用未由-Wall启用的额外警告
7. **-fopenmp** 启用OpenMP库（并行计算库）
8. **-fPIC** 编译动态库时，为了生成内存地址无关的代码
9. **-fext-numeric-literals**接受虚数、定点或机器定义的字面数字后缀作为GNU扩展。