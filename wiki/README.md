# 用自己编译自己：一个 C 语言子集编译器

> 先有编译器，还是先有源码？——这个项目或许能给你一个答案。

这是一个用 C 语言写成的 **C 语言子集编译器**，灵感来源于 [c4](https://github.com/rswier/c4)。它足够小，可以编译自身；足够完整，足以让你理解一个编译器从字符到执行的全过程。

---

## 特性

- 支持 `int`、`char`、指针类型
- 支持函数定义与调用、递归
- 支持 `if` / `else`、`while`、`return`、`enum`
- 支持完整的一元与二元运算符（含位运算、逻辑运算、比较运算）
- 支持数组访问与类型转换
- 内置简易虚拟机，直接解释执行字节码
- **可以编译自身（自举）**

---

## 架构总览

```
源代码（.c 文件）
       │
       ▼
┌──────────────────┐
│   词法分析器      │  next()
│                  │  将字符流切割为 token 序列
│  支持的 token：   │
│  标识符 / 数字   │
│  字符串 / 关键字  │
│  运算符 / 标点    │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│        语法分析器             │  递归下降解析
│                              │
│  program()                   │  入口，循环解析全局声明
│  ├─ global_declaration()     │  全局变量 / 函数 / enum
│  │   ├─ enum_declaration()   │  枚举常量
│  │   └─ function_declaration()
│  │       ├─ function_parameter()  形参列表
│  │       └─ function_body()       函数体
│  │           └─ statement()       语句
│  │               └─ expression()  表达式（按优先级递归）
│  └─ ...
│                              │
│  同步生成字节码，写入 text 段  │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────┐
│    虚拟机         │  eval()
│                  │  解释执行 text 段中的字节码
│  寄存器：         │
│  pc / bp / sp / ax
│                  │
│  内存段：         │
│  text / data / stack
└──────────────────┘
```

---

## 虚拟机指令集

### 1. 存取指令
| 指令 | 说明 |
|------|------|
| `IMM` | 将立即数加载到 ax |
| `LEA` | 将局部变量地址加载到 ax |
| `LC`  | 从地址（ax）读取 char |
| `LI`  | 从地址（ax）读取 int |
| `SC`  | 将 ax 写入栈顶地址（char） |
| `SI`  | 将 ax 写入栈顶地址（int） |
| `PUSH`| 将 ax 压入栈 |

### 2. 运算指令
| 指令 | 说明 |
|------|------|
| `ADD` / `SUB` / `MUL` / `DIV` / `MOD` | 算术运算 |
| `OR` / `XOR` / `AND` | 位运算 |
| `SHL` / `SHR` | 移位运算 |
| `EQ` / `NE` / `LT` / `LE` / `GT` / `GE` | 比较运算 |

### 3. 控制流指令
| 指令 | 说明 |
|------|------|
| `JMP` | 无条件跳转 |
| `JZ`  | ax == 0 时跳转 |
| `JNZ` | ax != 0 时跳转 |
| `CALL`| 调用函数（保存返回地址） |
| `ENT` | 创建栈帧（函数入口） |
| `ADJ` | 清理参数（调整 sp） |
| `LEV` | 恢复栈帧并返回 |

### 4. 系统调用
| 指令 | 对应函数 |
|------|----------|
| `OPEN` | `open()` |
| `CLOS` | `close()` |
| `READ` | `read()` |
| `PRTF` | `printf()` |
| `MALC` | `malloc()` |
| `FREE` | `free()` |
| `MSET` | `memset()` |
| `MCMP` | `memcmp()` |
| `EXIT` | `exit()` |

---

## 符号表结构

词法分析阶段，所有标识符被统一存入符号表（一个平铺的整数数组）：

```
每个标识符占 IdSize 个槽位：
[ Token | Hash | Name | Type | Class | Value | BType | BClass | BValue ]
```

其中 `BType` / `BClass` / `BValue` 用于在函数内部临时保存被遮蔽的全局变量信息，函数返回后恢复。

---

## 编译与运行

> ⚠️ 编译器运行在 **32 位模式**下，需要使用 `-m32` 标志。macOS 用户注意：Xcode 12 起已移除 32 位支持，建议在 Linux 或 Docker 环境下运行。

```bash
# 编译本编译器（注意 -m32）
gcc -m32 cc.c -o cc

# 或者直接用 Makefile
make compile

# 用它编译一个 C 源文件
./cc hello.c

# 开启 debug 模式（打印每条字节码指令）
./cc -d hello.c
```

### 运行测试

```bash
# 运行内置测试套件
make test
# 等价于：./cc test.c
```

---

## 自举：它如何编译自己

这个编译器最有趣的地方在于——它可以编译自身：

```bash
# 一步完成：编译自身并运行测试
make bootstrap
# 等价于：./cc cc.c test.c

# 或者手动分步：
# 第一步：用系统 gcc 编译得到第一个二进制
gcc -m32 cc.c -o cc

# 第二步：用 cc 编译自身源码
./cc cc.c
```

---

## 已知问题

`test.c` 中标注了三个已知失败的测试用例，均与逻辑运算符对**负数**的处理有关：

```c
test(1, c || c);   // failed — c 为负数时 || 结果不正确
test(TRUE, b && c); // failed — && 对负数操作数处理有误
test(TRUE, c && c); // failed
```

根本原因：VM 的 `JNZ`（不等于零则跳转）指令在处理负数时行为不符合 C 标准——C 标准规定任何非零值均为真，但当前实现可能仅检查符号位。

这正是「先有鸡还是先有蛋」的工程答案：第一个编译器总是由别的工具引导而来，此后它便可以自我延续。

---

## 参考资料

**项目**
- [rswier / c4 — C in four functions](https://github.com/rswier/c4)
- [lotabout / 手把手教你构建 C 语言编译器](https://github.com/lotabout/write-a-C-interpreter)
- [archeryue / cpc](https://github.com/archeryue/cpc)
- [tch0 / JustAToyCCompiler](https://github.com/tch0/JustAToyCCompiler)

**文章**
- [有哪些关于 c4 - C in four functions 编译器的文章？（知乎）](https://www.zhihu.com/question/28249756/answer/84307453)

**书籍**
- Niklaus Wirth — *Compiler Construction*
- [计算机系统要素：从零开始构建现代计算机（nand2tetris）](https://github.com/woai3c/nand2tetris)