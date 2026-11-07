# TC 系统级编程语言 — 实施路线图

> 目标：完全自研的 TC 系统级编程语言，对标 C++ 并追求超越 C/C++ 的运行时性能。
> 全链路自研：前端 / 中端优化 / x64 后端 / 目标文件格式 / 链接器 / 运行时 / 标准库 / 安装包。
> 约束：直接生成 Windows x64 机器码；**不转译 C**；**不链接 C 运行时(CRT)**；**不静态导入 kernel32.dll**。
> 验收：用 TC 语言编写一个可运行的视频播放器。

---

## 0. 总体架构

```
                     ┌──────────────────────────────────────────────┐
   .tc 源码  ───────▶ │  Stage-0 引导编译器  tc.exe (C++ 宿主)        │
                     │  Lexer → Parser → Sema → HIR → x64 → PE64    │
                     └──────────────────────────────────────────────┘
                                      │ 自举
                                      ▼
                     ┌──────────────────────────────────────────────┐
   .tc 源码  ───────▶ │  Stage-1 编译器  tcc.tc  (TC 语言自研)        │
                     │  由 Stage-0 编译产出，从此不再依赖 C++         │
                     └──────────────────────────────────────────────┘
                                      │
                                      ▼
                     ┌──────────────────────────────────────────────┐
                     │  TC 运行时 (tcrt) + 标准库 (tclib)            │
                     │  PE 装载器 / 堆 / 线程 / 文件 / 图形 / 解码    │
                     │  全部 TC 语言编写，NT 系统调用直连            │
                     └──────────────────────────────────────────────┘
```

**Stage-0 的定位**：仅作为一次性引导工具（bootstrapper）。它存在的原因纯粹是先有鸡还是先有蛋问题——
在 TC 语言能被编译之前，必须有一个用别的语言写的编译器。Stage-0 完成后即冻结，
后续所有编译器功能演进都在 `tcc.tc`（Stage-1，TC 语言）中进行。

---

## 1. 构建与使用

```powershell
# 一键构建引导编译器 + 编译运行全部示例 + 校验退出码
powershell -ExecutionPolicy Bypass -File .\build.ps1

# 只构建编译器
powershell -ExecutionPolicy Bypass -File .\build.ps1 -SkipTests

# 直接用编译器
build\tc.exe examples\02_hello\hello.tc -o build\hello.exe
build\hello.exe

# 查看编译中间结果
build\tc.exe examples\01_fib\fib.tc --dump-ast --dump-code --stats
```

也可用 Visual Studio 直接打开 `tc.sln` 构建（x64 配置）。

自举验证（三阶段自编译 + 不动点校验，要求 `tcc3 == tcc4`）：

```powershell
tools\selfhost.cmd
```

Stage-1 编译器（TC 语言编写）由 Stage-0 一次产出后即可独立使用，不再依赖 C++ 宿主：

```powershell
build\tc.exe tcc\tc_main.tc -o build\tcc1.exe        # Stage-0 产出 Stage-1
build\tcc1.exe examples\10_runtime\rt_test.tc -o build\rt.exe
build\rt.exe                                          # 退出码 31
```

验证产物零依赖：

```powershell
dumpbin /imports build\hello.exe     # 应输出空导入表
dumpbin /headers build\hello.exe     # 查看 PE32+ / AMD64 头
```

---

## 2. 里程碑

### M0 — 引导编译器垂直切片 ✅ 已完成
- [x] 工程骨架、路线图、语言规格、架构文档
- [x] 词法分析器（嵌套块注释、全部数字字面量形式、完整运算符集）
- [x] 递归下降语法分析器 + AST（优先级爬升，`as` 位于一元与二元之间）
- [x] 语义分析：符号表、作用域、结构体定点布局、类型检查、`let` 不可变性
- [x] x64 机器码生成（自带编码器：ModRM/SIB/REX/位移/立即数）
- [x] PE32+ 生成器（**导入表恒为空**，无 `.reloc`，固定基址）
- [x] 编译器内建 `syscall` / `read_gs` → 直发 NT 内核
- [x] 零依赖验证样例：PEB 遍历 → 内存中解析 ntdll 导出 → 读系统调用号 → 输出

**验收结果（已在本机实测通过）**：

```
$ build.cmd
[2/3] 编译 Stage-0 引导编译器
      build\tc.exe  (308.5 KB)

[3/3] 编译并运行示例
      00_exit   退出码 42     OK   (1024 字节)
      01_fib    退出码 216    OK   (3072 字节)
      02_hello  退出码 0      OK   (7168 字节)
                → 输出: Hello from TC - native x64, zero imports, zero CRT.

Import table check (dumpbin /imports)
      00_exit.exe  - empty import table
      01_fib.exe   - empty import table
      02_hello.exe - empty import table
```

`02_hello.exe` 在**导入表为空**（不链接 CRT、不导入 kernel32.dll / ntdll.dll）的前提下，
通过 PEB 遍历定位 ntdll、内存解析导出目录、读取当前系统的调用号，
再以 `syscall` 指令直入内核完成 stdout 写入。

**M0 已知边界**（后续里程碑补齐）：
- 表达式中间值全部走栈，尚未做寄存器分配（M2）
- 不支持内联汇编块、enum/union、顶层常量、浮点、泛型（M1）
- 单文件编译，尚无模块系统（M1）
- 尚无 `.pdata` 异常展开信息（M4）

### M1 — 语言完备（前端）✅ 已完成
- [x] `enum`：显式基础类型、显式 / 自动取值、编译期内联（零开销）
- [x] `union`：成员偏移全为 0，大小与对齐按最大成员
- [x] 函数指针：`fn(A, B) -> R` 类型、取址、赋值、经表间接调用
- [x] `f32` / `f64`：SSE 标量指令、IEEE-754 比较（含 NaN 语义）、双向转换
- [x] 全局量：`const`（编译期折叠、不占存储）/ `let`（.data）/ `var`（.bss）
- [x] 编译期常量求值：数组长度、枚举值、全局初值、常量引用内联
- [x] `switch` / `case` / `default` / `fallthrough`（默认跳出，标签重复检测）
- [x] 数组字面量与逐元素初始化
- [x] `import` 多文件编译单元（include-once、相对路径 + `-I` 搜索）
- [x] 诊断系统：稳定错误码、`file:line:col`、源码片段与插入符
- [x] 结构体自引用 / 相互引用的定点布局（与声明顺序无关）

**验收结果（已在本机实测通过）**：

| 示例 | 覆盖内容 | 期望退出码 | 实测 |
|---|---|---|---|
| `00_exit` | 最小可执行文件 | 42 | ✅ 42 |
| `01_fib` | 结构体 / 递归 / 循环 / 指针 / 类型转换 | 216 | ✅ 216 |
| `02_hello` | 零依赖 NT 系统调用输出 | 0 | ✅ 0 |
| `03_enum` | 枚举 / switch / fallthrough | 118 | ✅ 118 |
| `04_float` | f32 / f64 / IEEE-754 语义 | 127 | ✅ 127 |
| `05_globals` | 全局量 / 联合体 / 函数指针 | 511 | ✅ 511 |
| `06_module` | 多文件编译单元（3 个源文件） | 7 | ✅ 7 |

全部产物的导入表经 `dumpbin /imports` 独立确认为空。

### M1.5 — 切片 / 泛型 / 聚合按值传递 ✅ 已完成

- [x] **切片 `[]T`**：内存布局 `{ 指针 @0, 长度 @8 }`，内建伪成员 `.ptr` / `.len`
      · 数组切片 `arr[lo..hi]` · 指针构造 `slice(ptr, len)` · 下标读写
- [x] **泛型单态化**：`fn max<T>(a: T, b: T) -> T`、`struct Pair<T>`
      · 实参推断（含 `*T`、`[]T`、`Pair<T>` 的结构化统合）· turbofish `f::<i32>(...)`
      · AST 级深拷贝 + 类型替换 · 按需惰性实例化 · 实例名改写后作为独立符号（幂等）
- [x] **聚合按值传递**：结构体 / 数组 / 切片统一 ABI
      · 聚合返回值 → 调用者提供目的缓冲区（隐藏首参）
      · 聚合参数 → 复制到临时区后按引用传递，保证副本语义
      · 整体赋值按字节展开复制
- [x] **栈帧临时区**：聚合值落地空间在代码生成过程中分配，序言留补丁位回填
- [x] 数组 / 切片类型驻留，使类型相等性退化为指针比较

**验收结果（已在本机实测通过）**：

| 示例 | 覆盖内容 | 期望退出码 | 实测 |
|---|---|---|---|
| `07_slice` | 切片构造 / 下标 / 传参 / 多视图 | 255 | ✅ 255 |
| `08_generic` | 泛型函数 / 泛型结构体 / 推断 / turbofish | 255 | ✅ 255 |
| `09_agg` | 结构体按值传参返回 / 副本语义 | 63 | ✅ 63 |

`08_generic` 声明 6 个函数，实际生成 **13 个**（7 个为单态化实例）。

**M1.5 已知边界**（后续补齐）：
- 真正的命名空间隔离与包管理 —— M1.6
- 全局量的指针初值（需要在 `.data` 中生成重定位）—— M1.6
- 泛型约束 `T: Bound`、泛型枚举 —— M1.6
- 聚合参数目前一律经内存副本传递，小聚合（≤16 字节）的寄存器传递优化在 M2

### M2 — 代码质量（中端）⏳ 进行中
- [ ] 线性 IR + 虚拟寄存器（SSA 形式：每个临时值只被赋值一次）
- [ ] 活跃区间分析 → 线性扫描寄存器分配 + 溢出处理
- [ ] 常量折叠 / 副本传播 / 死代码消除
- [ ] SIMD 向量化（AVX2 → AVX-512 分档）

### M2 — 代码质量（中端）
- SSA 化 IR、数据流分析
- 优化：内联、常量传播、死代码消除、循环不变量外提、公共子表达式消除
- 寄存器分配：图着色 / 线性扫描
- SIMD 向量化（AVX2 → AVX-512 分档）

### M3 — 高性能机制（超越 C/C++ 的关键）
- `region` 区域内存管理，替代 malloc/free 全局堆
- 编译期生命周期 + 无 GC 的确定性析构
- 零成本抽象：切片/迭代器完全内联展开
- `parallel for` 编译期自动多线程划分
- 别名分析驱动的激进重排

### M4 — 运行时 tcrt ⏳ 进行中（约 60%）

**已完成并实测通过**（`10_runtime`、`12_fileio` 回归）：
- `tcrt/sys/nt.tc`：PEB 遍历 → 动态解析 ntdll 导出 → 提取系统调用号，**零导入表**
- 标准句柄：从 `PEB->ProcessParameters` 读真实句柄（`-11` 伪句柄对内核无效）
- `tcrt/core/mem.tc`：区块递增分配器（对齐保证、跨区块大块分配）
- `tcrt/core/io.tc`：控制台输出、UTF-8↔UTF-16、整数格式化、数字输出
- `tcrt/core/io.tc` 文件 I/O：`rtFileOpen` / `rtFileCreate` / `rtFileRead` /
  `rtFileWrite` / `rtFileSize` / `rtReadWholeFile`
  · 相对路径 → 当前工作目录解析 → `\??\` 绝对对象路径
  · 路径规范化（折叠 `.` / `..`）
- `tcrt/rt.tc`：运行时聚合入口 `rtInit()`

**本阶段修复的三个编译器缺陷**（均有回归覆盖）：
- `||` 短路结果取反
- PE 头部大小写死 512 字节（≥4 个节时首节数据落入头部）
- `syscall` 返回值未符号扩展（内核把 NTSTATUS 放 EAX，负数状态码在 i64 里
  变成大正数，导致 `st < 0` 失效）—— `syscall` 现在返回符号扩展的 i32

**待办**：
- 自研 PE 装载器（可加载 TC 编译出的 DLL/插件）
- 堆分配器：TLSF / 段式大小类，多线程无锁
- 线程/纤程、同步原语、TLS
- 结构化异常与栈回溯（自研，不用 SEH 元数据）
- 大栈帧（> 4 KiB）暂不支持：以「缓冲区别用全局量」规避，逐页栈探测留待 M2

### M5 — 自举 ✅ 已完成（不动点收敛）

**自举已达成**：TC 语言编写的编译器（`tcc/`）能够编译自身，且产物收敛为不动点。

验证方法（`tools/selfhost.cmd`，一键复现）：

| 阶段 | 编译器 | 输入 | 产出 |
|---|---|---|---|
| 0 | `build\tc.exe`（Stage-0，C++ 宿主） | `tcc/tc_main.tc` | `tcc1.exe` |
| 1 | `tcc1.exe`（TC 编写） | `tcc/tc_main.tc` | `tcc2.exe` |
| 2 | `tcc2.exe` | `tcc/tc_main.tc` | `tcc3.exe` |
| 3 | `tcc3.exe` | `tcc/tc_main.tc` | `tcc4.exe` |

**不动点判据**：`tcc3` 与 `tcc4` 逐字节一致（SHA256 相同）。
`tcc1` 与 `tcc2` 允许不同 —— Stage-0（C++ 宿主）与 Stage-1（TC 编写，栈式求值）
的代码生成策略本就不同；从 TC 编译器自产的第二代起收敛，即证明编译器已自洽。

实测结果（本机通过，`tools\selfhost.cmd`）：

```
[1/5] Stage-0 编译 Stage-1 源码 -> build\tcc1.exe  (390144 字节, 15 个文件合并)
[2/5] tcc1 编译自身             -> build\tcc2.exe
[3/5] tcc2 编译自身             -> build\tcc3.exe
[4/5] tcc3 编译自身             -> build\tcc4.exe
[5/5] 不动点校验                -> tcc3 == tcc4 (SHA256 41a0f50b…)
```

**分段能力**（早期里程碑，已由示例回归覆盖）：
- [x] 词法器 `tcc/lexer.tc`：关键字/类型名经 Interner 按 id 匹配；嵌套块注释、
      数字分支、字符串与字符转义、完整运算符集
- [x] 语法分析器 `tcc/parser.tc`：递归下降 + 优先级爬升
- [x] 语义分析 `tcc/sema.tc`：类型系统 / 符号表 / 栈帧 / 类型检查
- [x] 代码生成 `tcc/x64.tc`：自研 x64 编码器（REX / ModRM / SIB）
- [x] PE 生成 `tcc/pe.tc`：PE32+ / 导入表恒空 / 重定位回填

**Stage-1 已覆盖的语言能力**（自举 + 12 个兼容示例实测通过）：
- 声明与语句：`module` / `import`（递归展开 + include-once）、`fn`、`let`/`var`、
  `if`/`else`、`while`、**`for … in lo..hi`**、**`switch` / `case` / `default` /
  `fallthrough`**、`return`、`break`/`continue`、块作用域
- 类型：内建标量、指针 `*T`、**切片 `[]T`（`.len` / `.ptr` 伪成员 + 下标）**、
  定长数组 `[N]T`、结构体、联合体、枚举、**函数指针 `fn(A) -> R`（取址 + 经指针调用）**、
  泛型实例 `Vec<T>`
- 表达式：字面量（字符 `'x'`、**`1.5f`（f32）**、数组 `[a, b, …]`）、一元/二元/
  赋值、**复合赋值 `+=` 等**、调用、成员 `.`/`->`、下标、**切片 `arr[lo..hi]`**、
  `as` 转换（**常量折叠按目标宽度截断**）、`sizeof`、`E::M` 枚举成员引用、
  内建 `syscall` / `read_gs` / `slice` / `sqrt`
- 顶层量：`const`（**含浮点**）、全局量 `let`/`var`（数据段 + RIP 相对寻址）、`pub` 修饰
- 泛型：函数与结构体单态化（turbofish 显式实参）
- 聚合 ABI：sret 返回值、聚合参数按副本引用传递、整体按字节复制
- 指针算术按元素大小缩放（指针 ± 整数、指针 − 指针、下标）

**Stage-1 回归**（`tools\s1regress.cmd`：`build\tcc1.exe` 编译并运行，退出码校验）：

| 示例 | 覆盖 | 期望 | 实测 |
|---|---|---|---|
| 00_exit | 最小可执行文件 | 42 | ✅ |
| 01_fib | for / 递归 / 指针 | 216 | ✅ |
| 03_enum | switch / fallthrough | 118 | ✅ |
| 04_float | 浮点 const / f32 后缀 | 127 | ✅ |
| 05_globals | 全局量 / union / 函数指针表 / 数组字面量 | 511 | ✅ |
| 06_module | 多文件 import | 7 | ✅ |
| 07_slice | 切片语法 / `.len` / 下标 | 255 | ✅ |
| 09_agg | 聚合按值 ABI | 63 | ✅ |
| 10_runtime | 运行时（syscall / 堆 / 句柄） | 31 | ✅ |
| 11_logic | 短路逻辑 | 18 | ✅ |
| 12_fileio | 文件 I/O | 31 | ✅ |
| 13_stdlib | 泛型标准库（Vec / Str / Map） | 16383 | ✅ |
| 27_stdlib2 | 标准库扩展 fs / time / fmt | 4095 | ✅ |

**Stage-1 尚未实现**：
- 泛型实参推断（`maxOf(3, 7)`；当前泛型调用须显式 turbofish `f::<i32>(…)`）
- 诊断的 `file:line:col` + 源码片段（当前输出错误消息，不含位置）
- `NtQuerySystemTime`（调用号提取正确，但本机返回 `STATUS_INVALID_PARAMETER_1`；
  时间库已改用 KUSER_SHARED_DATA 用户态读取路径）

**本阶段修复的 Stage-1 编译器缺陷**（均曾导致自举失败）：
1. 驱动未接入多文件 import 展开（`loadSourceRec` 从未被调用），且路径未规范化
   → include-once 失效、`..` 无法被 NT 对象管理器接受
2. import 扫描按 `"import"` 前缀判定，把 `import_: u64;` 这类字段声明整行吞掉
   → 改为「`import` 之后必须紧跟空白或引号」
3. `genFunc` 复用同一输出缓冲，而导出的 `FuncCode.bytes` 只是浅拷贝
   → 后一个函数覆盖前一个的机器码，**任何函数调用都会跳进被踩坏的代码**
4. `genCall` 用「发射序号」索引原始函数表（有泛型模板时错位）
   → ABI 判断错位；改为独立发射顺序表 `emitFuncs`
5. 全局量数据段偏移在 PE 阶段才分配，而代码生成已读走该偏移（当时为 0）
   → 所有全局量访问都指向数据段起点；改为在 Sema 阶段分配偏移
6. 成员 / 下标赋值未被认作左值（`e->field = x`、`v[i] = x` 全部报错）
   → 改为递归左值判定
7. 泛型模板（`Vec<T>` / `MapU64<V>`）被当作具体类型参与布局
   → 报「未声明的类型 T / V」；模板只登记进声明表，不建类型对象、不参与布局

**本阶段修复的 Stage-0 编译器缺陷**（均有回归覆盖）：
- 结构体布局循环用范围 for 遍历 `mod_->structs`，而 `layoutStruct` 解析泛型字段
  （如 `Vec<Param>`）时会把新实例 `push_back` 进同一向量，导致迭代器失效崩溃。
  改为下标循环（每次重新索引）后，自引用结构体（`Type` 的 `elem: *Type`）与
  泛型字段（`Vec<*Symbol>`）组合不再崩溃。

**下一步**：冻结 Stage-0（`tc/`）；此后编译器功能演进只在 `tcc/`（TC 语言）中进行。

### M5.5 — 语言增强：全局数组字面量初始化 + 元组返回 ✅ 已完成

为支撑 H.264/MP3 数据表与多值返回，新增两项语言特性（Stage-0 `tc/` 与 Stage-1 `tcc/`
**同步实现**，并重新走通自举不动点）：

**特性 1：全局数组字面量初始化**
- 语法：`var gTab: [N]T = [字面量, ...];`
- `GlobalVar` 新增 `initData: Vec<u8>`；Sema 新增 `foldArrayInit`：遍历 `ArrayLit`
  元素，按元素类型小端字节序折叠为字节序列（整数经 `truncInt` 截断、浮点走
  `f32BitsOf`/位模式），元素个数超长即报错
- PE 数据段写入优先消费 `initData`（覆盖原先仅标量 `initBits` 的路径），尾零区
  仍由加载器清零（`.bss` 语义）
- 验证：`var gTab: [4]i32 = [1,2,3,4]` 与 `var gTab2: [3]u8 = [10,20,30]` 编译运行，
  逐元素核对通过（退出码 0）

**特性 2：元组返回**
- 语法：类型 `(T1, T2)`、字面量 `(a, b)`、字段访问 `.0` / `.1`、`fn f() -> (i32, i32)`
- 元组类型 = **匿名结构体**（`parseTupleType` 生成字段名 `"0"`/`"1"` 的匿名 `StructDecl`），
  复用既有 sret 返回 / 聚合按值 ABI；`.0`/`.1` 走 `Member` 数字字段（`numName` 驻留）
- 元组字面量 `TupleLit` 的类型由**上下文**（`return` 目标类型 / `VarDecl` 声明类型）
  回填，对齐目标元组的字段类型与偏移；元素值按字段大小截断写入临时区
- `convertible` 增加**结构等价**：匿名结构体字段类型与顺序一致即兼容（元组每次
  解析生成新结构体，必须按结构而非指针比较）
- 验证：`(i32,i32)` 双值返回 + `.0`/`.1`、`(u8,u16,u32)` 三值返回、字面量直接
  初始化局部元组，逐字段核对通过（退出码 0）

**自举不动点**（`tools\selfhost.cmd`）：tcc3 == tcc4，SHA256 `586aad61…`。
`build.ps1` 示例自检已改为**优先使用 `tcc3`**（正式编译器），仅在尚未自举时回退
`tc.exe`（引导编译器），完成「新编译器顶替旧编译器」。

### M6 — 标准库 tclib ⏳ 进行中（约 60%）

**已完成并实测通过**（`13_stdlib` / `27_stdlib2` 回归；
`27_stdlib2` 由**自举后的编译器 `build\tcc1.exe`** 编译验证）：
- `tclib/core/vec.tc`：泛型动态数组 `Vec<T>`（push / get / pop / 倍增扩容，单态化零装箱）
- `tclib/core/str.tc`：动态字符串 `Str`（append / 格式化 / 比较，始终维护结尾 NUL）
- `tclib/core/map.tc`：开放寻址哈希表 `MapU64<V>`（插入 / 查找 / 覆盖 / 扩容重散列）
- `tclib/math/math.tc`：整数 abs/min/max/clamp，浮点 abs/min/max/sqrt（SSE）
- `tclib/core/intern.tc`：字符串驻留器（哈希去重 + 扩容 + 回查，符号表基础）
- `tclib/core/fmt.tc`：控制台直出（字符串 / 整数 / 十六进制 / 键值行）+ Str 构造与输出
- `tclib/fs/file.tc`：整文件读写 / 追加 / 存在性 / 路径拼接 / 目录创建（`fsMakeDir`）
- `tclib/time/time.tc`：系统时间（KUSER_SHARED_DATA 用户态读取 + 毫秒换算）

**本阶段修复的编译器缺陷**（泛型单态化相关，Stage-0）：
- `cloneExpr` 漏拷贝 `builtin` 字段 → 泛型里 `sizeof(T)` 返回垃圾值
- `cloneExpr` 漏拷贝字面量 `type` → 泛型里 `null` 退化为 `i32`、无法赋给 `*T`

**待办**：
- `coll`：`Map`（字符串键）、`Set`
- `simd`（向量类型 + 内存/算法）
- `gfx`（自研软件光栅器 + 可选 GPU 后端）
- `media`（容器解析 + 视频/音频解码）

### M7 — 视频播放器验证 ⏳ 进行中

设计文档：**`docs/PLAYER.md`**（阶段拆分 / 模块划分 / 技术决策 / 语言约束）。

**总目标**：用 TC 语言从零实现可用的视频播放器（MP4 + H.264 软解 + AAC + WASAPI），
全程零外部依赖、产物导入表恒为空。

**阶段拆分**（每阶段都必须可运行、可现场验证）：

| 阶段 | 内容 | 状态 |
|---|---|---|
| **M7.0** | 播放器 UI：窗口 + 文件拖放 + 自绘控制栏 + 播放状态机 | **已完成** |
| M7.1 | 呈现通路：DIB 后台缓冲 + `StretchDIBits` 缩放呈现 + QPC 帧调度 + 键盘/尺寸消息 | **已完成** |
| M7.2 | 软件光栅器 `gfx`：画布、图元、渐变、双线性缩放、YUV→RGB 采样 | **已完成** |
| M7.3 | MP4 解复用：ISO BMFF box 树、`stbl` 四表、`avcC`/`esds`、样例抽取 | **进行中**（信息层 + `avcC` + 样例表已完成；`esds`/MKV 后续） |
| M7.4 | H.264 软解：NAL、SPS/PPS、CAVLC、帧内/帧间预测、IDCT、去块滤波 | **进行中**（第一轮：位流 / SPS/PPS / 切片头 / I_PCM 宏块已打通并接入播放器） |
| M7.5 | 音频与同步：AAC-LC 解码 + WASAPI 共享模式输出 + A/V 同步 | 未开始 |

**M7.1 落地内容**：
- `tclib/win/win32.tc` 扩展：`CreateDIBSection`（top-down 32bpp BGRA）、
  `CreateCompatibleDC` / `SelectObject` / `BitBlt` / `StretchDIBits` /
  `SetStretchBltMode`、`GetClientRect`、`PeekMessageW` / `PostMessageW`、
  `QueryPerformanceCounter` / `QueryPerformanceFrequency`、
  `GetAsyncKeyState`；消息常量补 `WM_SIZE` / `WM_KEYDOWN` / `WM_KEYUP` /
  `WM_MOUSEWHEEL` / `WM_ERASEBKGND` 与 `WS_OVERLAPPEDWINDOW` 系列样式
- `tclib/gfx/color.tc`：BGRA 打包、RGB/ARGB 构造、整数近似混色
- `tclib/gfx/canvas.tc`：BGRA 画布（宽/高/stride/像素指针）与基础图元
  （清屏 / 像素 / 矩形 / 水平垂直线 / 画线 / 缩放 blit）
- `player/player.tc`：窗口 + 非阻塞消息泵 + 逐帧动画（自研光栅器绘制）+
  实时 FPS 叠加 + `WM_SIZE` 自适应缩放 + 暂停 / 退出

**M7.1 验证结果**（构建 `tools\build_player.cmd`）：
- 窗口冒烟：`build\player.exe` 启动后持续运行（消息泵 / 帧调度 / 缩放呈现链路通），
  实测 3 秒存活、内存 13.8 MB，无崩溃
- 光栅器离线自检：`examples\28_gfx\gfx_dump.tc` 在内存画布绘制全部图元并写出
  `build\gfx_dump.bmp`（524,342 字节 = 54 + 512×256×4，尺寸精确吻合）；
  采样点逐一核对通过：
  背景 `0x141824`（= rgb(20,24,36)）、实心矩形 `0xC83C3C`（= rgb(200,60,60)）、
  斜线 `0x50F0A0`（= rgb(80,240,160)）、渐变两端 `0x050505` / `0xF2F2F2`、
  缩放 blit 命中源图像素 `0x40FF80`（= rgb(64,255,128)）
- 说明：自检覆盖清屏 / 矩形 / 描边 / 半透明混合 / 圆 / Bresenham 线 /
  垂直渐变 / 最近邻缩放 blit —— 全部为基础图元路径

**M7.4 第一轮落地内容**（H.264 解码管线，`tclib/media/h264.tc`）：
- 位流：NAL 拆分（AVCC 长度前缀）、EBSP→RBSP 防竞争字节剥离、
  位读取器（按字节批量 + 越界安全）、指数哥伦布 `ue`/`se`、位写入器（测试码流生成共用）
- 参数集：SPS（含 `max_num_ref_frames`/`gaps` 全字段、裁剪后显示尺寸按
  7.4.2.1.1 计算）、PPS、`avcC`（AVCDecoderConfigurationRecord）
- 切片头：I/P/B 公共字段全解析；对 CABAC、加权预测、SP/SI、场编码明确拒绝
  （返回 false 而非猜测）
- 宏块：**I_PCM** 全宏块解码（像素直存，无需预测/变换/残差），
  4:2:0 平面写入，pcm 字节对齐处理
- 帧缓冲：按宏块对齐的 YUV420P 三平面
- MP4 层配套：`mp4Avcc`（avcC 定位）、`mp4BuildVideoSamples`
  （stsc/stco/co64/stsz/stts/stss 展开为逐样本 offset+size+dts+关键帧标记）

**M7.4 第一轮验证结果**（`tools\build_player.cmd` 第 3、4 步）：
- `examples\30_h264\h264_selftest.tc`：手工构造 SPS/PPS/IDR 切片并由解码器解出，
  **Y 平面 256 像素、UV 128 像素逐点零误差**；切片头字段位置正确
  （dataBit=16）；非 I_PCM 宏块被明确拒绝（提示需 CAVLC 路径）
- `examples\29_mp4\mp4_selftest.tc`：**端到端闭环** —— 构造 30 帧 H.264 码流
  （每帧 60 个 I_PCM 宏块，像素图案随帧号滚动）→ 组装合法 MP4（avcC + 样例表 +
  mdat，带防竞争字节）→ 回读解析 → 样例表 30 项（#0 offset=702 size=23172，
  #5 dts=5000）→ 解码第 5 帧 → **Y 15360 像素 + UV 7680 像素逐点零误差**
  → YUV→BGRA → BMP
- 播放器接入：`build\test.mp4` 可直接拖入 `build\player.exe` **真实解码播放**
  （30 帧解码画面 + 时间码 + 进度），启动 4 秒冒烟无崩溃
- 真实视频（CAVLC/CABAC 残差 + 帧内/帧间预测）仍需下一轮，播放器会显示
  明确的「等待 M7.4b 支持」提示而不是崩溃或黑屏

**M7.0 落地内容**（播放器 UI，用户优先要求）：
- `tclib/win/win32.tc` 扩展：`DragAcceptFiles` / `DragQueryFileW` / `DragFinish`
  （shell32，运行时解析）、`WM_DROPFILES` / `WM_LBUTTONUP` 消息、
  `u16ToUtf8`（含代理对）、`winAcceptDrops` / `dragFilePath` / `dragDone`
- `tclib/media/mp4.tc`（新）：MP4/ISO BMFF 基础解析 —— box 树遍历（含 64 位
  `largesize` 与 `size==0` 到文件尾）、`ftyp` / `mvhd` / 逐 `trak`
  （`mdhd` 时间刻度与时长、`hdlr` 轨道类型、`stsd` 编码与分辨率/采样率/声道、
  `stts` 帧数、`stsz` 样本数），失败安全（越界即拒绝）
- `player/player.tc`：UI 重写
  - 窗口：1280×720 客户区、可缩放、深色主题
  - **拖放**：拖入任意文件 → 松手即解析 → 成功显示媒体信息卡，失败显示原因
  - **控制栏（GDI 自绘，78px）**：播放/暂停按钮（三角/双横杠图标）、
    进度条（含位置手柄）、时间码 `当前 / 总长`、实时 FPS 与状态
  - **交互**：点击播放按钮或空格 = 播放/暂停；点击进度条跳转；
    左右方向键 ±5 秒；ESC 退出；命令行参数 `player.exe <file>` 直接打开
  - 播放状态机：READY → PLAYING → PAUSED（播完停在末尾，再按从头重播）
  - 视频区按解析到的分辨率重建 DIB 帧缓冲（解码器的接入点 `renderVideoFrame`）
- `examples/29_mp4/mp4_selftest.tc`（新）：用 TC 构造结构合法的
  `build\test.mp4`（avc1 1280×720/150 帧 + mp4a 2ch 48kHz/200 帧，5 秒），
  再回读解析逐字段核对（同时作为 UI 拖放验证素材）

**M7.0 验证结果**（`tools\build_player.cmd`）：
- MP4 自检（终端可核对）：生成 966 字节 `test.mp4` → 解析结果
  `brand=isom`、`mvhd ts=1000 dur=5000`、
  `video avc1 1280x720 frames=150 samples=150 ts=30000 dur=150000 (5000 ms)`、
  `audio mp4a 2ch 48000Hz frames=200`、`duration=5000 ms`，全部与生成时设定一致；
  截断到 32 字节的负例被正确拒绝
- 播放器冒烟：`player.exe build\test.mp4` 启动后持续运行（17.4 MB 常驻），
  命令行打开链路（PEB → 路径 → 读文件 → MP4 解析 → 重建 DIB）无崩溃
- **需人工确认**：把 `build\test.mp4` 拖入 `build\player.exe` 窗口 →
  应显示 1280x720 / avc1 / 150 frames / 00:05 信息卡 → 点击 ▶ 或空格开始播放
  （时间码与进度条推进）→ 点击进度条可跳转 → ESC 退出
- 当前视频区为占位动态画面：真实 H.264 解码在 M7.4 接入
  （`renderVideoFrame` 是唯一替换点，UI 与呈现通路不变）

**M7.2 落地内容**（`tclib/gfx/`）：
- `canvas.tc` 扩展：平色三角形（扫描线 + 边 16.16 定点插值）、
  Gouraud 三角形（重心坐标插值，着色器管线基础图元）、
  双线性缩放 blit（像素中心对齐的 16.16 定点采样 + 边缘钳位）
- `yuv.tc`（新）：YUV420P → BGRA（BT.601 limited range，5 张 256 项 i32 查表、
  每 2 像素共享 UV、全程整数与内联钳位）；`yuv420ToBgra`（整帧）、
  `yuv420DrawScaled`（直接缩放绘制到画布）、`yuv420MakeTest`（自检图案）
- `examples\28_gfx\gfx_dump2.tc`（新）：全部新图元 + YUV 标准向量 + 1080p 性能基准

**M7.2 验证结果**（`tools\build_player.cmd` 第 2 步）：
- 平色三角：内部 `0xF08C28` = 指定色，外部 = 背景 ✓
- Gouraud：近红顶点 `rgb(249,1,3)`、近蓝顶点 `rgb(6,7,241)`、
  重心中心 `rgb(85,86,83)`（三色均值 85,85,85）✓
- 双线性：四象限源（红/绿/蓝/黄）放大 20 倍后中心为 `rgb(127,133,63)`
  —— 与 wx=wy=0.525 的四邻域插值精确值逐通道吻合 ✓
- YUV 标准向量（BT.601 limited）全部命中：
  黑 `rgb(0,0,0)`、白 `rgb(255,255,255)`、红 `rgb(255,0,0)`、
  绿 `rgb(0,255,1)`（±1 为整数近似误差）、蓝 `rgb(0,0,255)` ✓
- YUV 缩放绘制：灰度阶梯 → 近黑、色带 → 红 / 青，采样点与图案定义一致 ✓
- 1080p 整帧转换基准：**26.5 ms/帧（37 fps，标量单线程）**；
  优化路径（SIMD 内建 / 多线程行分块 / UV 行对复用）见 `docs/PLAYER.md` §7.1

**已知取舍**：
- 呈现走 GDI `StretchDIBits`（COLORONCOLOR），缩放由驱动加速；
  自研双线性已就绪（`canvasBlitBilinear` / `yuv420DrawScaled`），
  供需要精确控制的场景使用
- 解码范围先 Baseline/Main + CAVLC，High + CABAC 为 M7.4 第二轮
- 32bpp DIB 的 alpha 字节被 GDI（BI_RGB）忽略：画布 alpha 通道仅供自研
  合成使用，不影响屏幕呈现
- YUV 转换当前为标量实现：1080p 单线程 37 fps，720p 约 12 ms/帧；
  1080p 满帧需等待 SIMD 内建或多线程（已记入待办）

### M7.5 — MP3 音乐播放（`tclib/media/mp3.tc` + `player/player.tc`）✅ 已完成

**落地内容**：
- `tclib/media/mp3.tc`：Layer III 全链路软解（帧同步 / side info / 霍夫曼 /
  反量化 / 立体声 / 重排序 / 混叠消除 / IMDCT / 频域反相 / 多相合成），
  逐函数对齐 minimp3 参考实现
- **输出级处理（本轮新增）**：`mp3OutputReset` / `mp3OutputProcess` / `mp3OutputGain`
  —— 去直流（一阶高通 ≈14 Hz）→ 峰值限幅（阈值 30000、攻击 ≈0.45 ms、
  释放 ≈90 ms、多声道联动保持立体声像）→ 软削波（二次曲线，一阶连续）
  → 取整钳位；保证 `|输出| <= 32700`，绝不越界冲击扬声器
- `player/player.tc`：`BeginPaint/EndPaint` 替代 `getDC`（消除界面持续闪烁）、
  拦截 `WM_ERASEBKGND`、`WOM_DONE` 后先 `unprepare` 再 `prepare`、
  解码后接入输出级处理

**本轮定位并修复的解码缺陷**（11 项）：

| # | 缺陷 | 后果 |
|---|---|---|
| 1 | `BITS_DEQUANTIZER_OUT` 误写为 `+16`（应为 `-4`） | 反量化增益偏大 **32 倍** → 满量程饱和 |
| 2 | DCT-II 尾部 4 行丢了 28 行偏移 | 覆盖前 4 行输出，频谱被毁 |
| 3 | `l3Decode` 恒用第 0 颗粒的 side info | 每帧第 2 颗粒用错缩放因子/位长 → **位流逐帧漂移、持续炸音** |
| 4 | `reorder` 少 `+2*len` 的输入推进 | 短块频带重复读同一段数据，频谱错乱 |
| 5 | 霍夫曼查表直接消耗表宽（应"窥视 + 按实际码长消耗"） | 每码字错位 2~4 位 |
| 6 | `count1` 区 np 每样本 -1（应每 2 样本 -1） | band 提前耗尽，高频系数缺失 |
| 7 | `scfsi` 判定与 `*2` 顺序颠倒 | 该复用的去读位 → 位流错位 |
| 8 | `partIdx`（scf 分区表选择）公式写反 | 纯 long block 选到 short 表 |
| 9 | `iscf` 声明 `[40]i32` 却按 `u8` 写入 | 高 24 位为栈垃圾 → 缩放因子爆炸 |
| **10** | **DCT-II 第二阶段（8 点蝶形）索引整体转置**：应读/写组内连续 8 项 `t[8i+0..7]`，却写成 `t[i]`/`t[8+i]`/`t[16+i]`/`t[24+i]` 的"列" | **32 点 DCT 输出被打乱 → 全频段宽带噪声（约 20~45 倍高频垃圾）**，即"大量杂音" |
| **11** | **霍夫曼逃生码位序颠倒**：`lsb==15` 时应先取 linbits 扩展位再取符号位 | 量化幅值 ≥ 15 的大系数全部解错；只出现在强能量帧 → "安静段正常、响亮段崩坏" |

**M7.5 验证结果**（素材 `build\test.mp3` 观沧海，22.9 s / 44100 Hz / 2 声道 / MPEG-1 Layer III）：

1) 解码质量自检 `build\_clip.tc`（400 帧）：

| 指标 | 修复前 | 修复后 |
|---|---|---|
| 左声道 ratio（平均相邻跳变 ÷ 有效值） | 0.44（含宽带噪声） | **0.06**（与参考解码 0.054 一致） |
| 噪声特征帧数（ratio > 1.5） | 多 | **0** |
| 峰值 | 32768（顶满量程） | **7967**（垃圾尖刺消失） |
| 满刻度样本数 | 13 | **0** |
| RMS | 1307 | 1211 |

自检输出：`[PASS] 全帧无噪声特征，输出未溢出`

2) **与 ffmpeg 参考解码逐样本交叉验证**（`build\_diff.tc` / `build\_prof.tc`；
先 `ffmpeg -f s16le -ac 2 -ar 44100` 生成 `build\ref.raw` 作基准）：

| 素材 | 判据 | 修复前 | 修复后 | ffmpeg 基准 |
|---|---|---|---|---|
| 1 kHz 纯正弦 | 二阶差分 / RMS（x1000） | **815** | **17** | 18 |
| 1 kHz 纯正弦 | 平均跳变 | 893 | **237** | 249 |
| 观沧海 | 二阶差分 / RMS（x1000） | **718** | **30** | 34 |
| 观沧海 | 平均跳变 | 463 | **69** | 66 |

全曲 51 个窗口的局部对齐剖面（每窗口独立搜索最优偏移）：

```
@30000  rms=1381  d=2257 err=0      @590000 rms=1907 d=2257 err=0
@110000 rms=1625  d=2257 err=0      @650000 rms=2332 d=2257 err=2
@730000 rms=862   d=2257 err=9      @990000 rms=59   d=2257 err=0
```

- 偏移**恒定 2257**（= LAME 起始延迟 1105 + Xing 帧 1152），无丢样/抖动
- 各窗口平均误差 **0~9**（±1 LSB 级），**含最响段落（RMS 2332）在内整曲逐样本一致**
- 结论：`tclib/media/mp3.tc` 解码结果与 ffmpeg 参考实现逐样本吻合

- 限幅器在素材上不触发（峰值 7967 << 阈值 30000），其价值在于对任何音源
  都能保证 `|输出| <= 32700` 不溢出；去直流则消除音圈被推偏的低频冲击

### M7.4b — 真实视频（asdf.mp4）解码攻关 ⏳ 进行中

**素材**：`build\asdf.mp4`（小米手机拍摄，19,497 千字节，7.6 秒）
**权威特征**（本机 ffprobe `d:\dwrg2\ffmpeg\ffprobe.exe` 与项目解析器双确认，
两者结果完全一致）：

| 项 | 值 |
|---|---|
| 容器 / 视频编码 | mp42 / avc1（H.264） |
| 档次 | **High profile（100）**，level 字段写 10 |
| 尺寸 / 帧率 / 帧数 | 1920×1080 / 30.22 fps / 228 帧 |
| 像素格式 | **yuvj420p（全范围 YUV）** |
| **熵编码** | **CABAC**（PPS entropy_coding_mode=1） |
| B 帧 | **has_b_frames=1**（帧序列 `I B P B P B ...`，需要输出重排序） |
| 参考帧 | refs=1；8 个 IDR + 220 个 P/B 切片 |
| 音频 | mp4a（AAC-LC 48kHz 立体声 320 kb/s） |

**需要实现的解码特性**（按依赖顺序）：CABAC → 帧内预测（I_16x16/I_4x4/I_8x8）
→ 逆量化 + IDCT（4×4/8×8）→ 去块滤波 → 帧间预测（P 帧）→ B 帧 + POC 重排序。

**本轮已完成**：

1. **SPS 解析三处真实缺陷修复**（全部由真实文件暴露）：
   - 漏读 `seq_parameter_set_id`（在 profile/level 之后、chroma 之前）——
     此前自检因"构造端与解析端错位一致"而侥幸通过
   - High profile 额外字段：bit_depth_luma/chroma、qpprime、seq_scaling_matrix
   - `scaling_list` 解析（H.264 7.3.2.1.1.1 的 delta_scale 递推）
   - 交叉验证：解析出的 **1920×1080** 与容器 stsd、ffprobe 三者一致 ⇒ 解析路径正确
   - asdf.mp4 解析结果：profile=100 level=10 4:2:0，120×68 宏块，
     log2MaxFrameNum=4，POC type=0，frame_mbs_only=1；PPS：CABAC + 去块控制开
   - 样本表：228 项（#0 offset=487325 size=344070，关键帧标记正确）

2. **CABAC 固定表重建**（`tclib/media/cabac.tc`，新）：
   - **Table 9-44（rangeTabLPS，64 状态 × 4 档 = 256 项）**：从 ffmpeg
     `libavcodec/cabac.c` 的打包表逐项重建（有符号字节 → 原始值），
     并与标准表逐项交叉核对**完全一致**
   - Table 9-45（transIdxMPS / transIdxLPS）内联
   - 算术解码引擎：9 位 range、上下文 bin / 旁路 bin / 结束 bin、归一化、
     上下文初始化（9.3.1.1 的 preCtxState 公式）
   - 探测工具：`examples\32_probe\probe.tc`（容器 + SPS/PPS + 样本表 + NAL 统计）

**自检状态（诚实记录）**：
- `examples\33_cabac` 的往返自检**未通过**：问题在自写编码器的进位传播
  未完整实现（算术编码器需要 pending-bits 机制），该编码器不能用于验证解码器。
- 改为**真实码流验证路线**（已执行，见下）

**CABAC 真实码流验证 ✅ 通过**（`examples\34_cabacmb`）：

从 ffmpeg 源码获取了权威数据并实现：
- `cabac_context_init_I[1024][2]`（I 切片上下文表，已内联前 81 项：mb_type 0..10、
  帧内模式 64..69、CBP 73..80）
- 上下文初始化公式修正为 `preCtxState = clip3(1,126, ((m*QP)>>4)+n)`
  （**QP 本身**，此前误写为 QP-26）
- mb_type 二元化树（对照 `decode_cabac_intra_mb_type`）
- chroma 预测模式 / CBP luma / CBP chroma / I_4x4 预测模式解码
- 切片头放开 CABAC（`cabac_init_idc` 按 ue 读取）

**验证结果**（8 个关键帧 × 各自首个宏块，全部合法）：

```
样本#0   QP=19  mb_type=0（I_NxN，首块 4x4 预测模式=2）
样本#29  QP=26  mb_type=14（I_16x16，chroma=0 cbpL=9 cbpC=1）✓ 内部一致性通过
样本#58  QP=24  mb_type=0（模式=3）
样本#87  QP=24  mb_type=0（模式=2）
样本#116 QP=23  mb_type=0（模式=4）
样本#145 QP=23  mb_type=0（模式=4）
样本#174 QP=23  mb_type=0（模式=2）
样本#203 QP=23  mb_type=0（模式=2）
关键帧共 8 个，首个宏块语法全部合法：8 个
```

证据强度：8 个独立码流位置、QP 19..26 变化（验证上下文表在不同 QP 下正确）、
覆盖 I_NxN 与 I_16x16 两条不同解码分支（后者还通过 mb_type↔CBP 一致性检查）、
预测模式分布在 2/3/4 三个合法值 —— 若算术引擎或表有误，几乎不可能全部合法。

**第 1 步：残差解码 ✅ 完成**（`tclib/media/cabac.tc` + `examples\35_residual`）

实现内容：
- **完整 I 切片上下文表（1024 项）**：0..459 为 Table 9-12 原始数据（分段内联），
  460..1023 按 ffmpeg 的段复用规律以代码生成（CBF 460..483、sig 484..571、
  last 572..659、level 952..1011、CBF cat5/9/13 = 1012..1023）
- **残差解码**（对照 ffmpeg `decode_cabac_residual_internal`）：
  CBF（14 个上下文基址）→ significance map（`significant_coeff_flag` /
  `last_coeff_flag`，含 8x8 的上下文偏移表）→ 系数幅值
  （`coeff_abs_level` 的 node_ctx 状态机 + unary 前缀 + bypass 后缀 + 符号）
- **完整宏块语法循环**（`sliceDecodeI`）：mb_type → transform_8x8_flag →
  帧内模式（4x4/8x8）→ chroma 模式 → CBP（luma/chroma）→ 亮度 DC/AC、
  亮度 4x4/8x8、色度 DC/AC 残差；跨宏块维护邻块信息
  （mb_type 类别、CBP、chroma 模式、non-zero 计数、4x4 预测模式边界、8x8 标志）
- PPS 解析补全可选尾部（`transform_8x8_mode` / `second_chroma_qp_offset`）

**验证结果**（4 个 IDR 切片，每个 120×68 = 8160 个宏块）：

```
样本 #0   QP=19  已解 8160/8160 ✓  亮度系数 1,958,379  色度 65,280  175 ms
样本 #29  QP=26  已解 8160/8160 ✓  亮度系数 1,697,106  色度 48,954  140 ms
样本 #58  QP=24  已解 8160/8160 ✓  亮度系数 1,827,755  色度 65,280  151 ms
样本 #87  QP=24  已解 8160/8160 ✓  亮度系数 1,827,330  色度 48,955  151 ms
通过 4 / 4 个关键帧
```

**⚠ 上述「8160 宏块」验证作废（重要更正）**：当时的旧 `sliceDecodeI` **漏解了
`mb_qp_delta`**，且 terminate 检查只在循环末尾返回（循环跑完即返回 8160，
并不证明 end_of_slice_flag 命中）；更关键的是当时 `cabacDecodeBypass` 有 bug
（错误地把 range 翻倍），导致残差解码**恒定位率 254 位/宏块**——残差实际从未
被正确解码。该「验证」是循环自然结束 + 退化解码的假象。

**第 2 步：像素重建（去量化 / 逆变换 / 帧内预测）实现中**（`tclib/media/cabac.tc`
内联 + `examples\36_frame`）：

实现内容：
- 去量化（4x4/8x8 LevelScale 表、DC 用 (0,0) 位置系数）、色度 QP 映射（Table 8-15）
- 逆变换：4x4 IDCT（末尾 (x+32)>>6）、8x8 IDCT（四象限 DC 先 2x2 逆 Hadamard
  净 1/4，再四个 4x4 IDCT）、I_16x16 的 4x4 逆 Hadamard、色度 2x2 逆 Hadamard
- 帧内预测：亮度 16x16（4 模式）、亮度 4x4（9 模式 + 参考样本 1:2:1 滤波）、
  亮度 8x8（9 模式）、色度 8x8（4 模式）；邻块可用性按空间坐标推导
- `sliceDecodeIFrame`：语法 + 重建一体化（mb_type → 8x8 标志 → 模式 → chroma →
  CBP → mb_qp_delta → 亮度 DC/AC、4x4/8x8、色度 DC/AC → 像素写回）

**本轮定位并修复的 10 个 bug**（全部是「看似通过实则错误」的根源）：
1. `cabacDecodeBypass` 多了一行 `range <<= 1` —— 等概率位不应翻倍 range
2. `cabacInit` 多跳 1 个字节 —— CABAC 数据从 `ceil(bitPos/8)` 字节开始
3. `g允许8x8` 跨模块直接赋值失效 —— 8x8 路径从未真正执行，改 pub 函数 `允许8x8()`
4. CBF 段复用源偏移错误 —— 460..483 应为 `ms[85..96]`（原误用 `ms[88..99]`），
   直接导致 I_16x16 的 AC 块（cat 6）CBF 上下文错误
5. CBF cat5/9/13（1012..1023）值错误 —— 应为 `{-3,-8,-10,-30}/{70,93,90,127}`
6. `codIOffset`（value）位宽 9→10 位 —— bypass 里 `value<<1` 可达 1020，
   `&0x1FF` 截断错误，应 `&0x3FF`（ffmpeg 用 16 位 low 正是为此）
7. 8x8 块 non-zero 计数未同步到 4 个 4x4 位置 —— 后续 4x4 宏块的 CBF 上下文读到空槽
8. I_16x16 亮度 DC 解码后缺 `cbp |= 0x100`（bit 8）—— 邻宏块亮度 DC CBF 上下文错误
9. `重建亮度8x8` 的 8x8 模式索引误用 `模式[g]` —— 应为 `模式[左上表[g]]`
10. 4x4 残差块索引误用 `g3*4+i4`（线性）—— 8x8 组内 4 块是 2x2 布局，
    应为 `(g3/2)*8 + (i4/2)*4 + (g3%2)*2 + (i4%2)`

**验证（已通过）**：引擎 LPS/状态转移表逐项与标准一致（`lps0[0..8]=128,128,128,123…`、
`lps3[0..8]=240,227,216,205…`）；上下文表初值正确（`ctx[0]=110` 对应 pre=8）；
IDCT 单元测试 `DC=1 qp=26 → 3 3 3 3`；mb_type 出现 0/17/21/22 多样化。

**进展**：修复这 10 个 bug 后，解码宏块数从 420 → 543（terminate）→ 379（I_PCM）。
每修复一个，drift 点就明显后移。但仍未跑通完整 8160 宏块，**说明还有残余 bug**
（每宏块约少读 160 位，当前在约第 3 行 379 宏块处误判出 mb_type=25/I_PCM）。

**下一步（按依赖顺序）**：
1. 继续定位残余 bug，跑通完整 IDR 切片（8160 宏块 + 末尾终止位）
2. 像素级对比（自建参考或用可用的 ffmpeg 副本，若允许）
3. 去块滤波 → 帧间预测（P）→ B 帧 + 重排序 → 播放器接入

### M9 — 中文代码规范与 AI 协作配套 ✅ 已完成

**需求（用户明确定下）**：所有 AI 编写自研 TC 语言代码时，必须配套一份
【中文版 TC 自研代码使用规范文档】（纯中文、通俗易懂、AI 可识别），
以及【专用识别配置文件】（AI 读取后即知调用哪个编译器、按什么规则编译）。

**交付**：
- `docs/TC中文代码规范.md`（新）：纯中文规范，10 节 —— 一分钟上手 /
  完整示例 / 命名规范 / 注释规范 / 文件与模块 / 写法规范 / 编译与验证 /
  常见错误对照表 / 给 AI 的 12 条硬性规则 / 可照抄骨架
- `TC-AI配置.json`（新，项目根）：机器可读配置 —— 唯一允许的编译器
  `build\tcc3.exe`、编译命令模板与示例、规范文档路径、12 条强制规则、
  构建脚本、目录约定、语言要点提示、参考示例索引
- `AGENTS.md`（新，项目根）：AI 工具入口，指向上述两个文件 +
  三条最易错规则（只用 tcc3 编译 / 关键字英文标识符中文 / 必须验证并记录）

**配套的编译器能力扩展（词法器支持中文标识符）**：
- `tcc/lexer.tc`：`isAlpha` 接受 ≥0x80 字节（UTF-8 多字节序列），
  中文可直接用作模块名 / 函数名 / 变量名 / 结构体名 / 成员名
- 自举链重建后**不动点保持**：`tcc3` 与 `tcc4` 逐字节相同
  （SHA256 `a9eda645cfd788bf4adc9fc27ce4cedd404ae77466e2b0564eb81024230b6bda`）
- 验证：`examples\31_zhongwen\zhongwen.tc` 全中文标识符编写，
  经 `build\tcc3.exe` 编译运行输出正确 ——
  `宽 = 6，高 = 7` / `面积 = 42` / `盒面积 = 200` / `10 以内偶数个数 = 5`
- 限制（已写入规范与配置）：代码中不得出现全角标点（只能进字符串与注释）；
  Stage-0（`build\tc.exe`）不支持中文标识符，故规定
  **编译 TC 程序只允许使用 `build\tcc3.exe`**

### M8 — 安装包 ✅ 已完成（重写版：自读自身 + 尾部载荷）

**设计**：TC 编译器没有资源系统，因此不把 `tcc3.exe` 编成源码字节数组
（源文件会膨胀到 2 MB），而采用真实安装包的标准做法 —— **自读自身 + 尾部载荷**：

```
[安装器存根 tcsetup.exe][文件数据区][索引区][24 字节尾标记 'TCS1']

索引项 = [u32 名字长度][名字 UTF-8][u64 绝对偏移][u64 大小]
尾标记 = [u32 魔数 'TCS1'][u32 文件数][u64 索引偏移][u64 索引大小]
```

安装器运行时读自己的 EXE（零 DLL 导入），从尾部取出索引，把各文件原样写出。

**文件构成**：

| 文件 | 作用 |
|---|---|
| `installer/setupcore.tc` | 安装核心：PEB 取自身路径 / 读环境变量 / 载荷解析 / 释放 / PATH 注册 |
| `installer/tcsetup.tc` | GUI 安装器：欢迎提示框 + 许可页 + 进度页 + 完成页 |
| `installer/mkpayload.tc` | 打包器：按清单把载荷追加到存根，产出 `build\TC-Setup.exe` |
| `installer/setupcheck.tc` | 控制台验收：载入安装包 → 释放 → 与载荷逐字节比对 |
| `installer/payload.txt` | 载荷清单（`目标路径\|源文件路径`，可自由增删） |
| `installer/CONTRACT.txt` | 许可协议纯文本（随包释放为 `LICENSE-TC.txt`） |
| `installer/test_hello.tc` | 闭环验收用的 TC 程序 |
| `tools/build_setup.cmd` | 一键构建 + 验收（6 步） |

**安装流程（与需求逐条对应）**：
1. 打开即弹出提示框：**「哈喽，欢迎安装 TC 语言。」**
2. 许可协议页供阅读：禁止使用与调用 C 和 C++；不能删除和更改任何文件
   （除了开发这一个安装包）；如果你执意要修改，请告诉我，我会同意的
3. 点「我同意，开始安装」→ 释放 33 个文件到 `%LOCALAPPDATA%\TC`（自动建子目录）
4. 自动把安装目录追加进 PATH（先试系统级 `HKLM`，无权限则退用户级 `HKCU`），
   并广播 `WM_SETTINGCHANGE`，让新开的 CMD 立刻生效
5. 完成页提示：新开 CMD 输入 `tcc3 你的源码.tc -o 你的程序.exe`

**验证结果**（`tools\build_setup.cmd` 六步全绿）：

| 验收项 | 结果 |
|---|---|
| 打包 | 33 个文件；存根 114,176 字节，安装包合计 **1,043,613 字节** |
| 释放 + 逐字节比对 | **33 / 33 一致**（含中文文件名 `docs\TC中文代码规范.md`、`示例\player.tc`） |
| 闭环：用释放出的 `tcc3` 编译 | 成功；产物输出「你好，TC 语言！」与 `fib(90) = 2880067194370816120` |
| PATH 写入与去重 | 首次写入成功；二次调用识别「已包含」不再重复写 |
| PATH 值类型保持 | 原值 `REG_EXPAND_SZ` → 写回仍为 `REG_EXPAND_SZ` |
| **CMD 直接调用 tcc3** | 用注册表拼出的 PATH（等价新开 CMD）执行 `tcc3 hello.tc -o hello.exe` **编译成功** |
| 测试后还原 | 用户 PATH 长度 149 → 149，无残留 |

**技术要点**：
- 取自身路径：PEB → `RTL_USER_PROCESS_PARAMETERS + 0x60` 的 `ImagePathName`
- 取 `%LOCALAPPDATA%`：PEB → `+0x80` 环境块逐条扫描（不调用任何 API）
- 注册表：`advapi32` 的 `RegOpenKeyExW / RegQueryValueExW / RegSetValueExW`，
  经 `ntdll!LdrLoadDll` 运行时解析（安装器自身导入表依然为空）
- 安装由 `WM_TIMER` 一格一个文件驱动，进度条实时刷新，界面不卡死
- 本模块全部为**新增文件**，未改动 `tcrt/`、`tclib/`、`tcc/` 中任何既有文件

**已知限制（后续补齐）**：
- 未做 UAC 自我提权：拿不到管理员权限时自动退到用户级 PATH，
  完成页会明确显示用的是系统级还是用户级
- 未做卸载入口与「已安装检测 / 修复」（旧版有；重写版先保交付主线）
- 载荷可扩展：编辑 `installer/payload.txt` 后重跑 `tools\build_setup.cmd` 即可

**为支持 Win32 互操作修复的编译器缺陷**：
- 复合赋值与切片表达式使用了非易失寄存器 RDI（调用 Win32 API 会破坏
  调用者状态）→ 统一改用调用者保存寄存器 R11
- 调用点未保证 16 字节栈对齐（Win32 API 内部使用 SSE 指令）→
  引入压栈深度跟踪，调用区大小按当前栈偏移补偿
- 间接调用 12 参数（CreateWindowExW）经回归验证

---

## 3. 性能目标与验证方法

| 场景 | 基准 (C/C++) | TC 目标 |
|---|---|---|
| 整数密集循环 | 1.00x | ≥ 1.15x |
| 内存拷贝 / 扫描 | 1.00x | ≥ 1.30x (SIMD 自动向量化) |
| 函数调用密集 | 1.00x | ≥ 1.20x (零成本抽象 + 更优寄存器分配) |
| 启动时间 | 1.00x | ≤ 0.30x (无 CRT 初始化) |
| EXE 体积 | 1.00x | ≤ 0.25x |
| 虚拟内存占用 | 1.00x | ≤ 0.50x (region 分配) |

所有基准使用同一台机器、同一编译器优化等级、取 30 次中位数。

---

## 4. 目录说明

| 路径 | 内容 | 状态 |
|---|---|---|
| `docs/` | 路线图 / 架构 / 语言规格 | 已建立 |
| `tc/` | Stage-0 引导编译器（C++ 宿主） | M0 完成 |
| `build.ps1` | 一键构建 + 示例自检 | 已建立 |
| `examples/` | 验证样例 | M0 完成 |
| `tcrt/` | TC 运行时（TC 语言 + 内建系统调用） | M4 |
| `tclib/` | TC 标准库 | M6 |
| `tcc/` | Stage-1 自举编译器（TC 语言） | **已完成自举** |
| `tools/` | 自举验证 `selfhost.cmd` / Stage-1 回归 `s1regress.cmd` | 已建立 |
| `installer/` | GUI 安装程序（TC 编写，`tcsetup.tc`） | **首版完成** |

---

### M10 — 基础三维图形 API（`tclib/gfx/m3d.tc`）✅ 已完成

**需求**：用 TC 语言编写一套基础图形 API，并实现渲染立体方形（3D 立方体）的功能。

**落地内容**：

| 文件 | 内容 |
|---|---|
| `tclib/math/math.tc` | 补上 `f64Sin` / `f64Cos` 与常量 `圆周率`（该文件头原本即标注"三角后续以多项式近似补齐"）：范围归约 + 7 项 Horner 多项式 |
| `tclib/gfx/m3d.tc`（新） | 三维图形 API：向量运算、4x4 矩阵（平移/缩放/旋转/透视/视图）、深度缓冲、带深度测试与背面剔除的三角形光栅化、三角网格渲染、朗伯光照、内建立方体 |
| `examples/37_3d/cube_dump.tc`（新） | 离屏渲染验收：渲染立方体 → 写 BMP + 打印轮廓 / 面色直方图 / 耗时 |
| `examples/37_3d/cube_win.tc`（新） | 窗口实时旋转演示（800x600，QPC 驱动，ESC 退出 / 空格暂停 / 上下键调速） |
| `tools/build_3d.cmd`（新） | 一键构建 + 验收 |

**渲染管线**：

```
模型坐标 --模型矩阵--> 世界 --视图矩阵--> 视图 --透视--> 裁剪 --除以 w--> NDC --视口--> 屏幕
                                                                    |
                            屏幕空间重心坐标 --> 深度测试(1/w) --> 写色
```

**设计要点**：
- 深度缓冲存 **1/w**：它在屏幕空间内严格线性，用重心坐标线性插值即得正确深度排序，无需透视校正
- 背面剔除用屏幕空间有向面积符号判定（屏幕 y 轴向下，逆时针正面 → 面积为负）
- 三顶点同色走常量色快速路径：省掉逐像素浮点插值，同时消除 ±1 舍入抖动
- 朗伯光照：环境光 0.35 + 漫反射 0.65·max(0, n·l)，法线随模型矩阵旋转

**验证结果**（`tools\build_3d.cmd`；相机 z=+5、竖直视场角 60°、画布 640x480、
立方体半边长 1、模型固定 30° X 倾斜）：

| 指标 | 手算解析解 | 实测 |
|---|---|---|
| 0° 轮廓 x / y 范围 | 205.61~434.39 / 134.18~362.54 | **206~433 / 134~362** |
| 0° 轮廓宽 / 高 | 228.78 / 228.36 | **228 / 229** |
| 0° 可见面数 | 2（正面 + 顶面） | **2**（像素直方图 2 色） |
| 0° 中心像素 | 朗伯着色后 0xFF702B1D | **0xFF702B1D** |
| 45° 轮廓 x / y 范围 | 189.36~450.64 / 125.78~392.96 | **189~450 / 126~392** |
| 45° 轮廓宽 / 高 | 261.28 / 267.18 | **262 / 267** |
| 45° 可见面数 | 3（正面 + 顶面 + 左面） | **3**（像素直方图 3 色） |
| 45° 三面像素数 | 两个对称面必等大 | 12288 / 18045 / 18045 |
| 三角函数 | sin(π/6)=0.5、cos(0)=1 | 49999 / 100000（x100000，误差 1e-5）|

**性能**（640x480，300 帧连渲，含清屏 / 清 2.46 MB 深度缓冲 / 着色 / 光栅化）：

- 单帧 **3763 微秒**，等效 **265 帧/秒**
- 累计提交三角形 3744（12/帧）、剔除 2070、着色像素 14,462,957

**已知限制**：
- 无近平面裁剪：任一顶点落到相机后方（w ≤ 0）的三角形整体丢弃，相机贴近物体会突然缺面
- 颜色为逐三角形常量（含朗伯明暗），尚无纹理采样与透视校正插值
- 光栅化为逐像素重心坐标（约 10 次浮点乘 + 2 次除/像素），
  后续可用扫描线 DDA + 定点增量替换（预计提速 3~5 倍）

---

### M11 — TCBase 自研 .NET 框架（编译器 DLL 能力 + 8 大模块 + 启动壳 + 独立安装包）✅ 已完成

#### M11.1 编译器新增「产出动态库」能力（`-dll`）

| 位置 | 改动 |
|---|---|
| `tcc/ast.tc` | `FuncDecl` 增加 `isPub` 字段 |
| `tcc/parser.tc` | `pub fn` 首次被真正记录（此前 `pub` 被 `accept` 掉、无任何效果）|
| `tcc/pe.tc` | 新增 `ExportItem` / `cmpCStr` / `buildExportBlock`；Characteristics 置 `IMAGE_FILE_DLL`(0x2000)；ImageBase 用 `0x180000000`；数据目录第 0 项填导出表 |
| `tcc/tc_main.tc` | 解析 `-dll`；DLL 入口取 `DllMain`（可缺省）；收集「非模板、非单态化实例」的 `pub` 函数为导出表 |

**关键设计**：
- **导出的就是 `pub` 函数** —— 让语言里原本装饰性的 `pub` 成为真实语义（DLL 的公开 API）
- **不需要 `.reloc` 节**：代码生成全程 RIP 相对（`assembleText` 回填
  `disp32 = 目标RVA − (当前RVA + at + 4)`，`Call`/`LeaStr`/`Global`/`FuncAddr` 四类都是），
  天然位置无关，任意基址可加载
- **DLL 与 EXE 的 ImageBase 必须错开**（`0x180000000` vs `0x140000000`），
  否则两个模块抢同一个基址，后加载的一方装载失败
- 导出名字表按**字节升序**排列（加载器对名字表做二分查找），函数表与名字表同序，
  故 `AddressOfNameOrdinals[i] = i`、`Base = 1`

**自举**：改动 `tcc/` 后重跑 `tools\selfhost.cmd`，不动点保持 —— `tcc3 == tcc4`，
SHA256 `e4d29f1e2b3d3ab372f123ff84c77af6c0bab84081e9498d04e4c9a8d15e3d29`。

**本轮踩到的三个坑**（都记进文档，避免下次重犯）：

| 现象 | 根因与修法 |
|---|---|
| Stage-0 报 `类型 '*FuncDecl' 没有成员` | 我在 `tcc/` 里用了中文标识符。C++ 引导编译器的词法器用 `std::isalpha` 判定标识符起始（`tc_lexer.cpp:402`），**字节 ≥ 0x80 直接落空** → `tcc/` 下必须全 ASCII 标识符，中文只能出现在注释与字符串里 |
| `p->funcs.data[i].isPub` 报「没有成员」 | 该表达式的类型是 `*FuncDecl`（指针），`.` 取指针成员不被接受 → 先用临时变量接住，再写 `fnDecl->isPub` |
| 导出表 154675 字节（本应约 5 KB） | 算块长时把 **RVA 与块内偏移混用**，`blobLen` 变成 RVA 量级（≈0x248A6），白白填充约 150 KB 零；改为全部用块内偏移计算、写字段时才加 `blockRva` |

#### M11.2 框架本体（`tcbase/`，207 个导出）

| 模块 | 文件 | 代表接口 |
|---|---|---|
| 文件读写管理 | `tcbase/fs.tc` | 文件_读取全部 / 写入全部 / 追加行 / 复制 / 移动 / 删除 / 目录创建 / 路径拼接 |
| 字符串处理 | `tcbase/str.tc` | 字符串_长度 / 相等 / 查找 / 替换 / 截取 / 裁剪 / 转大写 / 转整数 / 从整数 |
| 控制台操作 | `tcbase/con.tc` | 控制台_打印行 / 打印整数 / 错误 / 换行 / 清屏 / **读行**（`NtReadFile` 直读标准输入）|
| 日期时间 | `tcbase/dt.tc` | 时间_当前本地 / 当前UTC / Unix毫秒 / 格式化 / 格式化日期 / 紧凑 / 星期名 |
| TCP 网络 | `tcbase/net.tc` | 网络_初始化 / 连接 / 发送 / 接收 / 关闭 / 解析（IP 字面量 + DNS）/ 本机主机名 |
| 内存工具 | `tcbase/mem.tc` | 内存_申请 / 申请清零 / 复制 / 填充 / 清零 / 比较 / 读U8..U64 / 写U8..U64（小端）|
| 系统信息 | `tcbase/sysinfo.tc` | 系统_计算机名 / 用户名 / 临时目录 / 处理器数 / 物理内存MB / 内存占用% / 版本 / 磁盘剩余MB |
| 弹窗提示 | `tcbase/ui.tc` | 弹窗_提示 / 错误 / 警告 / 是否 / 自定义 |

- 另有四个框架入口：`TCBase_版本` / `TCBase_名称` / `TCBase_模块数` / **`TCBase_自检`**
  （逐模块真实调用一次，把报告写进调用者缓冲，返回通过项数，满分 8）
- **导入表恒为空**：kernel32 / user32 / advapi32 / ws2_32 / ntdll 全部经
  `ntdll!LdrLoadDll` + 导出表解析在运行时取用；版本号用 `ntdll!RtlGetVersion`
  而不是 `GetVersionEx`（后者受兼容性清单影响，未声明支持 Win10 的程序会拿到 6.2 假版本号）
- 字符串接口一律 `(目的, 容量, 源…)` 返回写入字节数，**绝不原地改写** ——
  字符串字面量在只读段，原地改会直接崩

#### M11.3 启动壳与安装包

- `tcbase/launcher.tc` → **`TCBaseLauncher.exe`**：先查注册表
  `HKCU\Software\TCBase` 的 `InstallDir`，再核对那个目录下 `TCBase.dll` 真的存在（两步都过才算已安装）；
  **未安装 → 弹窗「未检测到 TCBase 自研 .NET 框架。请先安装 TC 自研.NET 框架，再运行本程序。安装程序：TCBase-Setup.exe」并返回 3**；
  已安装 → 从该目录 `LdrLoadDll` 载入 DLL，调用 `TCBase_自检`，把报告与安装目录一起弹出来
- 支持 `-quiet`：不弹窗、只打印，退出码给出结论（3 = 未安装，0 = 通过），**自动化验证靠它**
- `tcbase/setup.tc` → `TCBase-Setup-stub.exe`：窗口式安装程序（开始安装 / 卸载清理 / 退出），
  释放到 `%LOCALAPPDATA%\TCBase`，逐个与载荷**逐字节校验**，再写注册表标记；
  `-auto` / `-remove` 为静默装 / 卸（同样供自动化验证）
- `tcbase/pack.tc` → `tcbase-pack.exe`：把存根 + `tcbase\payload.txt` 里的 13 个文件 + 索引 + 24 字节尾标记
  拼成 **`build\TCBase-Setup.exe`**；容器格式与 `installer\mkpayload.tc` 完全一致（尾标记魔数 `'TCS1'`），
  因此释放端直接复用 `installer\setupcore.tc`，不另写解包逻辑
- `.cmd` 与产物名一律 ASCII：**`.cmd` 由 cmd.exe 按 OEM 代码页（中文 Windows 是 GBK）读取**，
  UTF-8 中文在脚本里会变乱码；同理 `.ps1` 也要纯 ASCII（Windows PowerShell 无 BOM 时按 ANSI 读）

#### M11.4 实测数据

`tools\build_tcbase.cmd`（7 步）全绿：

| 步骤 | 结果 |
|---|---|
| 1 框架动态库 | `TCBase.dll` **149504 字节**，导出 **207** 个 |
| 2 启动壳 | `TCBaseLauncher.exe` 110592 字节 |
| 3 安装器存根 | `TCBase-Setup-stub.exe` 119808 字节 |
| 4 打包 | `TCBase-Setup.exe` **446260 字节**，载荷 13 个文件，索引 435 字节 |
| 5 框架自检 | **8 / 8 通过**；宿主侧直调：`字符串_长度("TC自研框架")=14`、`系统_计算机名=DESKTOP-9UFGMHS`、`系统_处理器数=16`、`系统_物理内存MB=15895`、`网络_解析("127.0.0.1")=127.0.0.1`、文件写入 32 字节读回一致 |
| 6 启动壳（未安装） | 退出码 **3**，提示语正确 |
| 7 装 / 卸往返 | 静默安装 **13/13 成功**（释放到 `C:\Users\Win\AppData\Local\TCBase`）→ 启动壳检测到框架并自检 **8/8**（退出码 0）→ 静默卸载 **13/13**，注册表标记清除 → 启动壳**回到「未安装」**（退出码 3）|

`TCBase.dll` 独立 PE 核对（`tools\pe_report.ps1`，不依赖 TC 自己的加载器）：

| 项 | 值 |
|---|---|
| Characteristics | `0x2022` → **IMAGE_FILE_DLL = YES** |
| ImageBase | `0x180000000` |
| EntryPoint | RVA `0x23670`（DllMain，存在）|
| 导出表 | RVA `0x248A6`，**5005 字节**，Base=1，functions=207，names=207 |
| 导入表 | RVA 0，大小 0 → **空（零外部依赖）** |
| 导出名（前 8，按字节序） | `DllMain` `TCBase_名称` `TCBase_模块数` `TCBase_版本` `TCBase_自检` `buildNtPath` `cstrEq` `dragDone` |

> 交叉验证：框架返回的本地时间是 `2026-10-01` 且星期为**星期四** —— 与当日真实日期星期吻合，
> 独立佐证 `GetLocalTime` + SYSTEMTIME 字段偏移用法正确。

**新增文件**：`tcbase/`（base / mem / str / con / fs / dt / sysinfo / net / ui / launcher / setup / pack / payload.txt / 说明.txt）、
`tools/build_tcbase.cmd`、`tools/pe_report.ps1`；
**新增产物**：`build/TCBase.dll`、`build/TCBaseLauncher.exe`、`build/TCBase-Setup.exe`

> 注：本轮只**新增**文件，未改动 `installer/` 下既有安装包（TC 语言安装包）的任何文件 ——
> 框架安装包是独立的一套，容器格式相同但清单/存根/输出路径各自独立，两者互不影响。

---

### M12 — 新简易语法（Python 风格）：第 1–2 批已落地并验证 ✅ 进行中

**目标**：把 TC 语法改造成 Python 那样直观（缩进块、`def`、类型可省、`#` 注释、
内建 `print/len/int/str`），同时**不破坏** `tcc3 == tcc4` 不动点。设计合同见 `docs/SYNTAX2.md`。

**核心策略：新语法做超集，旧语法永久保留。**
编译器自己的源码是旧语法写的，一旦它读不懂自身源码，自举链立刻断裂且无工具可修。
所以每一批只做"旧语法完全不受影响"的加法。

| 批次 | 内容 | 落地位置 | 状态 |
|---|---|---|---|
| 1 | `#` 行注释（行首/行尾）、`def` 等价 `fn` | `lexer.tc` 的 `skipTrivia` 加 `#` 分支；`token.tc` 关键字表加 `def_` 并映射到已有的 `KwFn` | ✅ |
| 2 | 换行即语句结束（分号可省） | `lexer.tc`：`Lexer` 加 `lastKind`/`parenDepth`/`pendingSemi`，换行处按 `canEndStmt` 补虚拟分号 | ✅ |
| 3 | 缩进块（`:` + INDENT/DEDENT，含嵌套与多层回退） | `lexer.tc`：`Lexer` 加缩进栈（结构体不能放数组字段 → `rtAllocZero` 配 64 级 `*i32`）+ `atLineStart`/`curIndent`/`blockPending`/`pendingIndent`/`pendingDedent`；`token.tc` 加 `Indent`/`Dedent`；`parser.tc` 的 `parseBlock` 同时支持 `{...}` 与 `: INDENT ... DEDENT`（if/while/for/else/函数体自动全覆盖）；**`tc_main.tc` 的 `loadSourceRec` 修「行首缩进被吃掉」缺陷** | ✅ |
| 4 | 类型可省：**裸赋值即声明** + 局部类型推断 | `sema.tc` 的 `checkAssign`：左侧是简单标识符且 `lookup`/globals/funcs 都查不到时，`declareLocal(..., mutable=true, ...)` 就地声明，类型取右值类型（只处理普通 `=`，`+=` 不声明）。局部变量的类型推断**原本就有**：`finalTy` 为空时取初值类型、再兜底 i64 | ✅ |
| 5 | 英文内建：**`print` 已完成**（多参数、参数间自动空格、自动换行）；**`len` 已完成**（字符串字节长度，全角按字节计）；`int` / `str` 待做 | `print`：`parser.tc` 的 `parseStmt` 脱糖成「每个参数一次 `__tcprint(x)` + 参数间 `rtPrint(" ")` + 末尾 `rtPrint("\n")`」，再由 `sema.tc` 的 `checkCall` 按参数类型把 `__tcprint` **改名**为 `rtPrint` / `rtPrintI64`。`len`：**不保留名字**（`len` 在既有代码里被广泛用作标识符），只在「`len` 不是任何已声明函数」时改名为运行时的 `utf8Len` | 🔶 部分 |
| 6 | 产出 `tcc3.1.exe`，编辑器改用它 | 编辑器已预留：`编译器路径()` 依次找 `build\tcc3.1.exe` → `tcc3.exe` → `tcc3.exe` | ⬜ |

**第 2 批的关键设计（零语法器改动）**：不在语法器里到处改"分号可省"，
而是让**词法器按规则补一个虚拟分号**，语法器完全不用动：

- 行尾是「能结束语句的记号」（标识符 / 字面量 / `)` / `]` / 类型名 / 布尔 / null）→ 补分号
- 行尾是运算符（`a +`）、`{`、`}`、`;`、`,` → 不补
- 小括号 / 方括号深度 > 0 时不补（跨行调用、跨行数组天然安全）

于是：旧代码（每句都写 `;`）行为完全不变；新代码可以省掉分号；跨行表达式不受影响。

> ⚠️ **上一条说法后来被证伪，已修正** —— 见下方「批次 2 规则的真实回归」。

**⚠️ 批次 2 规则的真实回归（2026-10-02 发现并修复）**

- **症状**：编译 `editor/editor.tc` 时报 `sema error [读U32]: 表达式不可调用` × 3。
- **根因**：`tclib/win/winproc.tc` 里有这样的跨行表达式——
  ```c
  fn 读U32(b: *u8, 偏移: u64) -> u32 {
      return (b[偏移] as u32)          // ← 行尾是 `)`，于是被判为「能收句」
           | ((b[偏移 + 1] as u32) << 8)   // ← 结果被虚拟分号切成另一条语句
  ```
  第一行以 `)` 结尾，`canEndStmt` 判定它可以结束语句 → 插入虚拟分号 → 表达式被腰斩。
  所以"跨行表达式不受影响"这句**只对行尾是运算符的写法成立**，对行尾是 `)`/`]`/字面量
  而下一行以运算符续行的写法**不成立**。
- **为什么自举链没发现**：编译器自己的源码里没有这种写法，所以 `tcc3 == tcc4` 一直绿。
  **教训：自举不动点通过 ≠ 语言没被改坏。** 真正的回归捕获来自"拿一个真实的大程序
  （编辑器，2000+ 行、依赖 `tclib/win/*`）编一遍"。
- **修法**：词法器在行首遇到**续行运算符**时撤销刚排好的虚拟分号——
  `| & + - * / % ^ < > = . ? :`（`isContinuation`）。
- **代价（明说）**：新语法里若一条语句以 `*` / `-` / `&` 开头且写在新的一行
  （例如解引用赋值 `*p = 5`），需显式写出分号。旧语法不受影响（它每句都有分号）。
- **状态**：修复已重建进 `build\tcc3.exe`，不动点 `56ca8a06e2ea57208ba55886ac3d520418801de815f8a5680975a2aa118e89f4`；
  **"编辑器能编过"这一步的实测尚待执行**（编译器源码不含该写法，自举无法覆盖它）。

**实测数据**（`tools\selfhost.cmd` + 两份验收程序）：

| 项 | 结果 |
|---|---|
| 自举不动点 | `tcc3 == tcc4`，SHA256 `4180d0f6a0882ab4a89abbacf91a38bf5c4d66553d0039fc0b349a7a272729cf` |
| 15 个编译器源文件 | 全部照旧编译通过（证明旧语法零影响） |
| `build\_newsyntax.tc`（第 1 批） | `[PASS] # 行注释与 def 关键字均可正常工作` |
| `build\_newsyntax2.tc`（第 2 批，**一个分号都没写**） | `[PASS] 分号可省、跨行表达式、跨行调用、注释行 全部正常`，输出 `5 的阶乘 = 120`、`1+2+3 = 6` |

**这两批踩到的坑（都值得记住）**：

| 现象 | 根因 |
|---|---|
| Stage-0 报 231 个「无法识别的字符」 | 我在 `tcc/lexer.tc` 里用了中文标识符（`上次种类`/`括号深度`/`补分号`/`能结束语句`）。**C++ 引导器的词法器用 `std::isalpha` 判标识符起始（`tc/src/tc_lexer.cpp:402`），字节 ≥0x80 直接落空** → `tcc/` 下必须全 ASCII 标识符，中文只能出现在注释与字符串里。这正是 M11 记过的坑，我又踩了一次 |
| 分号晚一个记号出现，把 `rtPrint( 跨行 )` 读成 `rtPrint ; ( ...`，报「未声明的标识符 / 赋值左侧必须是左值 / 重复定义」 | 虚拟分号在 `skipTrivia` 里**置位**，却在 `lexNext` 开头**兑现** —— 而 `skipTrivia` 是 `lexRaw` 内部调用的，等回到 `lexNext` 时下一个真记号已经被取走了。修正：兑现点移到 `lexRaw` 中 `skipTrivia` 之后、取记号之前 |

**批次 3（缩进块）实现要点**：

- `token.tc` 追加 `Indent` / `Dedent`（放在枚举末尾，不动既有取值）
- `lexer.tc`：`Lexer` 加缩进栈。**结构体不支持数组字段**（全仓 0 命中），
  所以用 `rtAllocZero(64*4, 4) as *i32` 分配 64 级缩进栈
- 规则：某行**以冒号结尾**（`lastKind == Colon`，注释里的冒号不算）→ 下一行缩进更深即开块（`INDENT`）；
  缩进变浅则连续弹栈（可一次发多个 `DEDENT`）；空行与纯注释行不参与缩进判定
- `parser.tc`：**只改 `parseBlock` 一个函数**就同时支持两种块写法，于是
  if / while / for / else / 函数体全部自动支持缩进形式
- 顺序：虚拟分号 → DEDENT → INDENT（先收句、再收块、最后开块）

**批次 3 实测**：

| 项 | 结果 |
|---|---|
| 自举不动点 | `tcc3 == tcc4`，SHA256 `5b78d369df766675cd9b2ca0f8ea43cc83d868fbe17ef0f1728e6e384e58b347` |
| `build\_newsyntax3.tc`（**无花括号、无分号、无 fn**） | `[PASS] 缩进块（嵌套 if/else、while、三层连续回退）全部正常`，输出 `阶乘(5) = 120` / `分类(负/零/正) = 123` / `1~10 偶数累加 = 30` |
| `build\_ni1.tc`（最小回退探针） | 退出码 0（`if false` 的假分支内的语句确实未执行） |
| 批次 1 / 2 回归 | 两份验收程序均 `[PASS]` |

**⚠️ 本轮挖出的真实缺陷：源码加载器吃掉行首缩进**

`tcc\tc_main.tc` 的 `loadSourceRec`（拼接入口文件与递归 import）在每个行首
**把"所有空白"都跳过**，然后从跳过后的位置复制整行 —— 空格与制表符因此被静默删除：

```c
while pos < fsize {
    // 跳过空白（0x20 空格 / 0x09 制表 / 0x0D 回车 / 0x0A 换行）
    while pos < fsize && (src[pos] == 0x20 || src[pos] == 0x09 || src[pos] == 0x0D || src[pos] == 0x0A) {
```

**花括号语法对缩进不敏感，所以这个 bug 从 M1 起潜伏至今无人察觉**，
而缩进语法一撞上就必死。修正：**行首只跳换行，缩进原样保留**；
探测 `import` 用独立游标（允许 import 缩进书写），但复制整行时从行首开始，一个空格都不丢。

**定位方法论（值得复用）**：用三个转储程序把"以为是"变成"看见"——

1. `build\_lexdump.tc`：内存字符串喂 `lexInit`/`lexNext`，打印记号流
   （`[INDENT] [DEDENT] [;] [:]` 都显式可见）→ 证明**词法器正确**
2. `build\_parsedump.tc`：调用 `parseModule` 后打印顶层函数表与每个函数体的语句数
   → 证明**语法器正确**（3 个函数、`main` 体 4 条语句）
3. 在 `tc_main.tc` 里临时插诊断，**打印源缓冲前 70 字节的十进制值**
   → 直接看到 `... 58 10 114 101 116 ...`（`:` `\n` `r`，缩进没了）→ 锁定 `loadSourceRec`

**批次 4（类型可省）实现要点**：

- **发现：局部变量的类型推断原本就已存在** —— `sema.tc` 的 VarDecl 处理里
  `finalTy` 为空时取初值类型、再兜底 `i64`，所以 `var x = 5` 一直可用，
  只是从来没人这么写。批次 4 真正缺的是**裸赋值即声明**
- 实现落在 `sema.tc` 的 `checkAssign`：左侧是简单标识符、且 `lookup` / `globals` /
  `funcs` 三处都查不到时，就地 `declareLocal(sem, id, 右值类型, mutable=true, false)`
  并回填 `e->lhs->sym` / `e->lhs->type_`
- **只处理普通 `=`**：`+=` 这类复合赋值要先读旧值，必须先有变量，不能就地声明
- **旧代码完全不受影响**：旧语法总先 `var`/`let` 声明，赋值时一定查得到符号；
  而"给未声明变量赋值"在旧语法里本来就是编译错误，不存在依赖这个行为的代码

**批次 4 实测**：

| 项 | 结果 |
|---|---|
| 自举不动点 | `tcc3 == tcc4`，SHA256 `9803a31b2295fb24fcdbd955faeeb1995fbe34f1c728164ff68a1ac6b615cef7` |
| `build\_newsyntax4.tc`（裸赋值 + 局部推断 + 缩进块 + 无分号） | `[PASS] 裸赋值即声明 + 局部类型推断 全部正常`，输出 `乙 = 42 丙 = 42 丁 = 101` |
| 批次 1/2/3 回归 | 三份验收程序均 `[PASS]`；最小回退探针退出码 0 |

**批次 5（英文内建 print）实现要点**：

- **语法器脱糖**（`parser.tc` 的 `parseStmt`）：`print(a, b, c)` 编译期展开为一个块语句——
  每个参数一次内部标记调用 `__tcprint(x)`，参数之间插一次 `rtPrint(" ")`，
  末尾再补一次 `rtPrint("\n")`。于是换行与空格分隔都在**编译期**完成，运行期零开销
- **语义分析定型**（`sema.tc` 的 `checkCall`）：看到 `__tcprint` 就检查该参数的类型，
  指针/字符串 → 把 callee 名**改成** `rtPrint`；整数 → 改成 `rtPrintI64`，
  然后**继续走下面的常规调用解析**。这样参数类型转换、错误检查、
  未声明函数报错等全部复用既有逻辑，不需要改代码生成
- **`print` 因此是保留名**：已用 `\bprint\b`（区分大小写）全仓搜索确认
  **没有任何代码把它当标识符**，所以保留它不会破坏任何既有源码
- 空格分隔对齐 Python 的 `print(sep=' ')` 语义（中文场景下可能显得多一个空格，
  需要时写 `print("你好，" + 名)` 或后续加 `sep` 参数）

**批次 5 实测**：

| 项 | 结果 |
|---|---|
| 自举不动点 | `tcc3 == tcc4`，SHA256 `c047d70bcf12a14ab54ba1adaaf07dc0068030be28944703668ef54193efef14` |
| `build\_newsyntax5.tc` | `[PASS] print 内建可用（字符串 / 整数 / 多参数 / 自动换行）`，实际输出 `你好， TC 自研语言` / `面积 = 42` / `单参数也行： 42` |
| 批次 3 / 4 回归 | 两份验收程序均 `[PASS]` |

**为什么用 tcc3 而不是 stage-0 来构建**：两者都能构建（编译器源码保持旧语法、
全 ASCII），但 `tcc3` 只需一步且可用语言特性更多；stage-0 的价值在于它是**冻结的**、
不会被改动波及，可作为最后的兜底构建器。

---

### M13 — 编辑器缺陷修复：多行粘贴挤成一行 ✅ 已验证

**现象**：从网页 / 聊天窗口复制多行代码，粘进编辑器后全部挤在同一行，不换行。

**根因**：多行 EDIT 控件**只把 `\r\n` 当换行**，单独的 `\n` 被它当普通字符吃掉。
而原来的 `编辑_粘贴` 直接转发 `WM_PASTE`，等于把粘贴完全交给控件自己处理。

**附带发现（更关键）**：编辑器里**没有任何加速键或键盘处理** —— 按 Ctrl+V 时按键由
EDIT 控件内建处理，**根本不经过菜单命令**。所以只改粘贴函数没有用，必须子类化控件。

**修法**（`tclib/win/winedit.tc` + `editor/editor.tc`）：

| 层 | 改动 |
|---|---|
| 粘贴实现 | 不再转发 `WM_PASTE`：`OpenClipboard` → `GetClipboardData(CF_UNICODETEXT=13)` → `GlobalLock` → UTF-16 转 UTF-8 → **规整换行** → `EM_REPLACESEL` 插入 → 解锁 / 关剪贴板 |
| 换行规整 | `编辑_规整换行`：丢掉裸 `\r`，把每个 `\n` 统一成 `\r\n`（因此 `\r\n` 与 `\n` 都正确） |
| 拦截 | `编辑_挂接粘贴(控件)` 用 `SetWindowLongPtrW(GWLP_WNDPROC)` 子类化：截 `WM_KEYDOWN`(Ctrl+V) 与 `WM_PASTE`，处理后返回 0 吃掉消息，避免控件再粘一次 |
| 防递归 | 子类只调用**不带退回逻辑**的内部实现；解析不到剪贴板接口时交回原过程，避免「子类 → 发 WM_PASTE → 又回到子类」的栈溢出 |
| 编辑器 | 控件创建后调一次 `编辑_挂接粘贴(g编辑区)` |

**自动化验证**（`build\_pastetest.tc`：造隐藏窗口 + EDIT 控件、程序自己写剪贴板，
全程无需人工，两个用例都断言「行数」与「每个 `\n` 前是否补了 `\r`」）：

```
用例1（直调粘贴）        字节 = 31  行数 = 3（应 3）  缺 \r 的 \n = 0（应 0）
用例2（WM_PASTE 子类路径）行数 = 3（应 3）  缺 \r 的 \n = 0（应 0）
[PASS] 粘贴修复生效：\r\n 与 \n 都正确识别，多行文本自动分成多行
```

用例 2 覆盖的是子类拦截路径（菜单粘贴 / 右键粘贴走这条）。
`WM_KEYDOWN`(Ctrl+V) 那条分支逻辑相同，但需要真实按键状态，**只能人工按一次 Ctrl+V 确认**。

**过程中踩到的两个坑**（都记下来）：

| 现象 | 根因 |
|---|---|
| `编辑_粘贴子类过程` 报「函数调用参数个数不匹配」 | `CallWindowProcW` 是 5 个参数，我误写成 `wcall4`（`wcallN` 的 N 含首参之外的实参个数） |
| `编辑_挂接粘贴` 报「实参类型不匹配」 | 把函数名传给 `u64` 形参需要显式写 `编辑_粘贴子类过程 as u64` |
| 探针程序一跑就 `0xC0000005` | 忘了先调 `winInit()` 初始化 win32 函数表，回调里调到空指针 |

**待办**：`editor/editor.tc` 本身（含新增的 `编辑_挂接粘贴` 调用）还需编译一次。

---

### M14 — 标识符只能 ASCII（中文命名禁令）🔶 规则已生效，迁移进行中

**规则**：变量名 / 函数名 / 类型名 / 全局量 —— 代码正文里**只能**用 ASCII 字母、数字、下划线。
**中文只允许出现在注释与字符串里**。编译器遇到中文标识符直接报 `error[TC1001]` 并中止编译。

**这推翻了项目原有约定**：`AGENTS.md` 第 2 条原本写的是"关键字用英文，**标识符与注释用中文**"，
另有 `docs/TC中文代码规范.md` 专门规定中文命名。两者都需要同步改写（**尚未改**）。

**实现**（`tcc/lexer.tc` + `tcc/tc_main.tc`，都是纯 ASCII 标识符）：

| 位置 | 内容 |
|---|---|
| `lexer.tc` `lexRaw` 开头 | `skipTrivia` 之后若 `cur >= 128`（非 ASCII 字节）→ 记一次词法错误、打印行列、**整段跳过**（避免逐字节刷屏与原地打转）。注释与字符串在上游已被整段消化，不受影响 |
| `lexer.tc` | `gLexErrors` 计数 + `gAllowCjkIdent` 开关 + `lexSetAllowCJK()` / `lexErrorCount()` |
| `tc_main.tc` | 新增 `-legacy-ident` 参数（迁移期逃生开关）；`parseModule` 之后检查 `lexErrorCount() != 0` → 中止（退出码 4） |

**实测**：

```
A 加 -legacy-ident ：编译通过并运行正常（你好，TC 自研语言！/ 这句问候有 27 字节 / Hello, World!）
B 默认（不带开关）：9 × error[TC1001] + "源码中存在不允许的中文标识符，编译中止"，退出码 4
自举不动点         ：tcc3 == tcc4，SHA256 5dc0f5266df6b7c1dd952c18728f37edb80829370d1fa93c34a7eddc25bc4ada
```

自举能保持，是因为**编译器自身源码（`tcc/`）与它导入的 `tcrt/`、`tclib/core/` 本来就全 ASCII**
（当年被 C++ 引导器约束出来的），规则对它们天然免疫。

**待修（已知瑕疵）**：报错里的行号 / 列号用了 `rtPrintI64`，而它写 **stdout**，
错误正文写 **stderr** —— 两者交错，行号会跑到消息后面单独出现。修法：给 stderr 补一个整数打印，
或自己 itoa 成字符串再 `rtPrintErr`。

**迁移计划（先后顺序按依赖）**：

1. `tcrt/`（运行时，最底层）→ `tclib/core/`（是编译器导入的）→ `tclib/` 其余（win / gfx / fs / time / media / math）
2. `tcbase/`（8 大模块）→ `installer/` → `player/` → `editor/` → `examples/`
3. 同步改文档：`AGENTS.md` 第 2 条、`docs/TC中文代码规范.md`、编辑器"帮助"菜单里的入口
4. 每批完成后：`-legacy-ident` 下编译验证 + 裸编译（应无 TC1001）验证
5. 全部迁完后，`-legacy-ident` 保留（作为历史代码的应急口）或按需删除

**⚠️ 迁移期间的注意事项**：
- **所有既有库与程序默认编不过**，必须加 `-legacy-ident` 才能构建，直到该文件迁完
- **`tcc/*.tc` 里绝不能出现中文标识符** —— 否则 C++ 引导器（stage-0）直接编译失败。
  本轮我已在这个坑上栽了第三次（在 `lexer.tc` 里写了 `g词法错误数` 等），
  教训：**改 `tcc/` 下任何文件后，先自查标识符是否全 ASCII，再跑自举**

---

## 5. 当前进度

| 里程碑 | 状态 |
|---|---|
| M0 引导编译器垂直切片 | **已完成** |
| M1 语言完备 | **已完成** |
| M1.5 切片 / 泛型 / 聚合按值 | **已完成** |
| M2 代码质量（IR + 寄存器分配） | **进行中**（IR 管道 `tir.tc` + `genFuncIR2` 已建成，5 类 lowering 缺陷已修；尚未在完整自举下跑通，`genModule` 默认仍走旧 `genFunc`） |
| M3 高性能机制 | 未开始 |
| M4 运行时 | **进行中**（系统调用/内存/控制台/文件 I/O 完成，待线程与装载器） |
| M5 自举 | **已完成**（`tcc3 == tcc4` 不动点收敛，`tools\selfhost.cmd` 一键验证） |
| M6 标准库 | **进行中**（Vec / Str / Map / Math / Interner / fmt / fs / time 完成，约 60%） |
| M7 视频播放器 | **进行中**（M7.0 UI 拖放/控制栏、M7.1 呈现通路、M7.2 光栅器、M7.3 解复用+样例表、M7.4 第一轮 H.264 解码已接入；**M7.5 MP3 音乐播放已完成**：全链路软解 + 输出级限幅，400 帧自检 `[PASS]`；待真实视频 CAVLC/CABAC） |
| M8 安装包 | **已完成**（重写版：自读自身 + 尾部载荷，把 `tcc3.exe` 与 32 个 TC 库/文档文件内嵌进 1,043,613 字节的安装包；欢迎提示 → 许可协议 → 释放到 `%LOCALAPPDATA%\TC` → 自动追加 PATH；验收 33/33 文件逐字节一致，并用释放出的 `tcc3` 闭环编译成功，新开 CMD 可直接调用） |
| M9 中文规范与 AI 协作配套 | **已完成**（`docs\TC中文代码规范.md` + `TC-AI配置.json` + `AGENTS.md`；词法器支持中文标识符，不动点保持） |
| M10 基础三维图形 API | **已完成**（`tclib/gfx/m3d.tc`：向量/4x4 矩阵/深度缓冲(1/w)/背面剔除/三角网格/朗伯光照 + 内建立方体；`examples/37_3d/` 离屏验收与窗口实时旋转演示；投影轮廓与面色像素与手算解析解逐项吻合，640x480 单帧 3763 微秒 ≈ 265 帧/秒） |
| M11 TCBase 自研 .NET 框架 | **已完成**（编译器新增 `-dll`：产出带导出表的动态库且**无需 .reloc**（代码全程 RIP 相对），`pub` 成为真实导出语义，不动点保持；`tcbase/` 8 大模块 207 导出编译为 `TCBase.dll`(149504 字节，导入表为空)；启动壳检测注册表+文件两步判定，未安装弹窗提示（退出码 3）；独立安装包 `TCBase-Setup.exe`(446260 字节 / 13 文件)；`tools\build_tcbase.cmd` 7 步全绿：自检 8/8、未安装→3、装/卸往返各 13/13） |
| M12 新简易语法（Python 风格） | **进行中**（设计合同 `docs/SYNTAX2.md`；第 1 批 `#` 行注释 + `def` 等价 `fn`、第 2 批换行即语句结束（分号可省）已落地；采用"新语法做超集"策略，旧源码零影响；每批均过 `tools\selfhost.cmd` 不动点验证，最新 SHA256 `4180d0f6…29cf`；两份验收程序 `[PASS]`；待做缩进块、类型推断、英文内建，然后产出 `tcc3.1.exe`） |
| M13 编辑器多行粘贴挤成一行 | **已修复并自动化验证**（根因：多行 EDIT 控件只认 `\r\n`，`\n` 被吃掉；且编辑器无加速键，Ctrl+V 由控件内建处理。修法：`winedit.tc` 自行读剪贴板 + `编辑_规整换行` 统一成 CRLF + 子类化拦 `WM_KEYDOWN`(Ctrl+V)/`WM_PASTE`、返回 0 吃掉消息 + 防自我递归；编辑器创建控件后 `编辑_挂接粘贴`。验证 `build\_pastetest.tc` 两用例全过：行数 3/3、缺 `\r` 的 `\n` 0/0，`[PASS]`。仅 `WM_KEYDOWN` 分支需人工按一次 Ctrl+V 确认） |

### M2 IR 自举迁移进展（本轮）

目标：`genModule` 从旧 `genFunc`（直接遍历 AST 发射）切到 IR 管道
（`tirInit` → `genFuncIR` → `allocRegs` → `genFuncIR2`），最终 `c2 == c3` 在 IR 下收敛。

**本轮定位并修复的 lowering 缺陷（均在 `tcc/tir.tc` / `tcc/x64.tc` 落地）**

| # | 缺陷 | 症状 | 修法 |
|---|---|---|---|
| 1 | 全局数组按值读 | `gIncBuf[base+i] = ...` 把数组首 8 字节当整数，写到垃圾地址 | `emitExprIR` 的 Ident/gvar 分支：`TypeKind::Array` 返回地址而非 `IR_LOAD` |
| 2 | sret 返回用了 `IR_LOAD` | 函数返回聚合时把 sret 缓冲指针指向「缓冲前 8 字节的内容」 | `emitStmtIR` Return 改成 `IR_MOV p, sretSlot` |
| 3 | 聚合成员赋值只动 8 字节 | `p.tok = lexNext(lx)` 之类整块赋值只拷首 8 字节 | `emitStoreTo` 对聚合走 `IR_MEMCPY` / `IR_MEMCPY_PTR` |
| 4 | 指针 ± 整数不缩放 | `&gNormBuf[0] + off`（u16 元素）落错位置 | `emitBinaryIR` 按 `e->elemSize` 缩放；指针相减按元素大小归一 |
| 5 | **Div/Mod/比较/右移不分有无符号** | `hashU64(k) % m->cap` 用有符号 `idiv`，哈希高位为 1 时得 **-15**，`idx` 变天文数字 → `mapPut` 第 3 次插入越界崩溃（`0xC0000005`） | 新增 `unsFlag()`，把「指针/无符号整数」标志作为 `IR_BIN` 的 `aux2` 传给 `emitBinOpIR`；无符号走 `xor edx,edx; div`、`CC_B/BE/A/AE`、`shr`，有符号走 `cqo; idiv`、`CC_L/LE/G/GE`、`sar` |

**验证数据（可复现）**

- 缺陷 5 的隔离证据：`build\_mapN.tc` 打点显示 `put 2` 的 `idx=-15`（修复后为 1），
  修复后 `build\_map3.tc`（13 次插入）输出 `MAP_OK`，600 条目版本 `_mapbig.tc` 输出 `BIG_OK`。
- 修复过程中曾观察到 `_c2_ir`（IR 自动机）里 `Sema.errCount`（偏移 1016，`sizeof(Sema)=1096`）
  在 `semaInit` 末尾被 `mapPut(&sem->typeNames, ...)` 踩成 181 —— 正是缺陷 5 的连带后果，
  修复后 `errCount` 全程保持 0。
- 工具链健康：`tools\selfhost2.cmd` → `[OK] new-syntax bootstrap converges (c2 == c3)`，
  `c2/c3 SHA256 = c295db11bdd251635124ed981d47f8d798122c2aeb67dc3fc03061fae9daaf9c`。

**第 1 轮时仍未通过的地方（历史记录；D2/D3 已在第 2 轮修复，见文末）**

把 `genModule` 切回 IR 后用 IR 自动机（`_c2_ir`）实测：

| 输入 | 结果 |
|---|---|
| `build\_lex.tc`（91 函数，**IR 产出**） | **错误**：`lx.pos=14 / lx.len=7`（实参本应是 0 / 3），`lexNext` 返回 `kind=86`（正确值 13 = KwFn） |
| 同一份 `build\_lex.tc`，用旧 `genFunc` 产出的 `build\c2.exe` | **正确**：`lx.pos=0 / lx.len=3`，`kind=13`（KwFn） |
| `build\_lex.tc` 交给 `_c2_ir`（IR 自动机）编译 | 崩溃 `0xC0000005` |
| `build\_f1.tc`（1 个函数）交给 `_c2_ir` 编译 | `parseInit` 后 `p.tok.kind = 0`（Eof）→ 解析出 0 个函数 |

即：IR 管道能跑通简单算术/结构体/泛型，但**局部结构体的成员寻址读到了错误偏移**
（`Lexer` 的 `len`@8 / `pos`@16 读出来是 7 / 14），导致词法器产出错乱记号，
进而解析器拿不到函数、自我编译失败。这解释了「小规模看似正常、规模化全崩」的全部现象。

已排除的假设：Map 规模（600 条目 `BIG_OK`）、`sizeof(V)` 泛型折叠（实测 8 正确）、
`Sema` 字段偏移（`typeNames=424 / errCount=1016 / mods=1024`，旧新两版一致）。

建议下一步：聚焦 `emitAddrIR` 的 `ExprKind::Ident`（局部结构体取址：`markStackOnly` + `IR_LEASLOT`）
与 `IR_LEASLOT` 发射（`leaMem(cg, RAX, RBP, 0 - loc.disp)`）——
打印 `lx` 的 `sym->offset`、`slotDisp`、`locs.disp` 三者在 IR 版里是否一致，
以及 `frameSize` 是否按 `sizeof(Lexer)≈550` 正确留出栈空间（重叠会让后续局部变量互相踩踏）。

### 第 2 轮：溢出区重叠（D2/D3 已关闭）

**根因（实测确认）**：`allocRegs` 给溢出临时槽分配栈偏移时用的是

```tc
t->locs.data[i].disp = t->frameSize + nextDisp;   // 错
```

而槽地址是 `[rbp - disp .. rbp - disp + size - 1]`，**从低地址往高地址生长**；
局部变量区占 `[rbp - frameSize, rbp - 1]`。`nextDisp` 从 0 起，于是第一个溢出槽的
首字节正好落在 `rbp - frameSize`，即「偏移恰好等于 `frameSize`」的那个局部变量上。

**观测证据链**

1. `build\_kw.tc`（`Outer{pre0,pre1,inner:Inner(64B),post}`）：`before pre0=100` →
   `after pre0=14 pre1=21 post=300 f7=14`；`Inner` 的 f0..f7 = 7,14,21,…
   即 `&o+0` 被写进了 `mk()` 返回值里的 **f1**，`&o+8` 被写进 **f2** —— 整体错位一个字段。
2. 同一程序打点 `&o.inner - &o = 16`、`&o.post - &o = 80`、`&o.inner.f7 - &o = 72`
   —— **地址计算完全正确**，排除 `IR_LEAOFF`/成员偏移。
3. 在 `genFuncIR2` 打印每函数 `frameSize` 与所有 `disp >= frameSize` 的槽：
   `main` 为 `FR=112 SPILL=16  d=112 s=16` —— 16 字节的 sret 临时槽 `disp=112`
   恰好等于 `frameSize`，把偏移 112 的局部变量 `o` 整块盖住。

**修法（`tcc/tir.tc`，`allocRegs`）**

```tc
var sz2: i32 = 8
if i < t->slotSize.len && t->slotSize.data[i] > 8 { sz2 = t->slotSize.data[i]; }
sz2 = ((sz2 + 7) / 8) * 8;
t->locs.data[i].disp = t->frameSize + nextDisp + sz2;  // 改：+sz2，整体下移
nextDisp = nextDisp + sz2;
```

改后 `main` 变为 `FR=112 SPILL=16  d=128 s=16`（`128 = 112 + 16`），区域
`[rbp-128, rbp-113]` 完整落在局部变量区之下。

**修复后实测**

| 用例 | 修复前 | 修复后 |
|---|---|---|
| `build\_mc.tc`（16B 结构体整体赋值） | `pre0=14 pre1=200` | `pre0=100 pre1=200 post=300 f0=7 f1=14` ✓ |
| `build\_kw.tc`（64B 结构体整体赋值） | `pre0=14 pre1=21 f7=14` | `pre0=100 pre1=200 post=300 f7=56` ✓ |
| `build\_lex.tc` 的 `lexInit/lexNext` | `lx.pos=14 lx.len=7 kind=86` | `lx.pos=0 lx.len=3 kind=13`（KwFn）✓ |

### 第 4 轮：函数下标二分 + 聚合值约定（D5 部分关闭）

**新方法：按函数下标二分。** 在 `genModule` 加开关

```tc
pub const IR_SPLIT: i32 = 0   // 下标 < IR_SPLIT 用 IR，其余走旧 genFunc
```

配合 `build\_bs.bat <N>`（先用 tcc3 生成 s1，再用 s1 生成 s2，最后用 s2 编译 `build\_f1.tc`），
因为**只有「s1 再生成 s2」这一步的机器码才是 IR 产物**，直接用 tcc3 生成的不算。

二分轨迹（EXIT 0 = 通过）：

| SPLIT | 结果 | SPLIT | 结果 |
|---|---|---|---|
| 0 | OK | 124 | FAIL(3) |
| 62 | OK | 108 | OK |
| 93 | OK | 116 | FAIL(3) |
| 110 | OK | 112 | FAIL(3) |
| **111** | **OK** | **112** | **FAIL(3)** |

→ **元凶 = `p->funcs[111] = lexOperator`**（`tcc/lexer.tc:443`）。

**根因：聚合值的表示约定不一致。**
`lexOperator(lx, t: *Token) -> Token` 结尾是 `return *t` —— 对 48 字节 struct 解引用返回值。
`tir.tc` 的 `emitExprIR` 在 `Deref` / `Member` / `Index` 分支一律发 `IR_LOAD`，
而 `IR_LOAD` 只搬一个槽大小的字节，于是「48 字节的 Token」被压成一个地址值，
`Token.kind` 读出来就是内存地址（`1358024`、`1358656`，步长正好 `sizeof(Token)=48`）。

IR 的既有约定是「**聚合值放在一个栈槽里，下游用 `IR_LEASLOT` 取该槽地址**」
（`emitStoreTo` 的聚合分支、`Return` 的 sret 分支都这么用）。所以正确做法是：
聚合值位于任意地址时，先整块复制进一个新栈槽，再返回该槽号。

**修法（`tcc/tir.tc`，新增 `aggToSlot`）**

```tc
fn aggToSlot(t: *TirCtx, addr: i32, ty: *Type) -> i32:
    let sz: i32 = ty->size
    let tmp: i32 = newSlot(t)
    if (tmp as u64) < t->slotSize.len: t->slotSize.data[tmp as u64] = sz
    markStackOnly(t, tmp)
    let tmpAddr: i32 = newSlot(t)
    emit(t, IR_LEASLOT, tmpAddr, tmp, 0, 0, opImm(0), opImm(0))
    emit(t, IR_MEMCPY_PTR, -1, 0, sz, 0, opSlot(tmpAddr), opSlot(addr))
    return tmp
```

`emitExprIR` 的 `Deref` 与 `Member`/`Index` 在 `tyIsAggregate(e->type_)` 时改用它。

**修复后状态变化**

| 指标 | 修复前 | 修复后 |
|---|---|---|
| `_s2.exe build\_f1.tc` | EXIT=3（语法错误，位置 `1073825781`） | EXIT=5（`存在未解析的外部符号`） |

错误从「词法产出垃圾」前移到「外部符号解析」，说明词法/语法这一段已经打通。
`build\tcc3.exe` 生产路径不受影响：`tools\selfhost2.cmd` → `[OK] c2 == c3`，`56e1a71d…`。

### 第 5 轮：二分法连续追打（再修 2 处）

同一套 `IR_SPLIT` 二分法，修一处就重新二分一次，得到「失败点持续后移」的轨迹：

| 轮次 | 失败点 | 表现 |
|---|---|---|
| 修 aggToSlot 前 | `p->funcs[111] = lexOperator` | EXIT=3（词法垃圾） |
| 修 aggToSlot 后 | `p->funcs[229] = emit` | EXIT=5（未解析外部符号） |
| 修 byRef 后 | `p->funcs[324] = loadOpndIR` | EXIT=0xC0000005（崩溃） |

**第 2 处根因（`emit` 坏了 → 全盘坏）**：`emit()` 里 `ins.a = a` 的 `a: Opnd` 是**聚合参数**，
按引用传递，槽里存的是指针；而 `emitExprIR(Ident)` 没有 byRef 分支，直接 `return slotFor(...)`，
下游按「槽里装的是字节」去 `IR_LEASLOT` 取地址，就把「指针槽」的内容当成了 Opnd 本体。
因为 `emit()` 是所有 lowering 的公共函数，一处坏就污染全部 IR。

修法（`tcc/tir.tc`，`emitExprIR` 的 `Ident` 分支）：

```tc
if e->sym->byRef:
    let s0: i32 = slotFor(t, e->sym)
    if e->type_ != null && tyIsAggregate(e->type_):
        return aggToSlot(t, s0, e->type_)      // 复制进新栈槽（s0 的值就是地址）
    let d0: i32 = newSlot(t)
    emit(t, IR_LOAD, d0, 8, 0, 0, opSlot(s0), opImm(0))
    return d0
return slotFor(t, e->sym)
```

**当前状态**：`tools\selfhost2.cmd` → `[OK] c2 == c3`，`ce4e1287…`（生产路径不受影响）。
IR 全量路径的最新失败点是 `loadOpndIR`，下一步从它继续。

**操作注意**：`build\_bs.bat <N>` 会把 `IR_SPLIT` 改成 `<N>` 留在源码里，
跑完必须把它改回 `0`，否则默认走 IR 会破坏不动点（本轮踩过一次）。

### 第 6 轮：byRef 取址（仍未越过 loadOpndIR）

**修法（`tcc/tir.tc`，`emitAddrIR` 的 `Deref` 分支）**：`&(*x)` 原先一律 `emitExprIR(x)`；
对 byRef 聚合参数，`emitExprIR` 为了满足「聚合值放槽里」会先复制一份，拿到的是**副本地址**，
于是 `p->field = v` 会写到副本上、原对象不变。改为：`x` 是 byRef 的 `Ident` 时走
`emitAddrIR(x)`（取槽里的指针值）。

**记录更正**：上一轮我把 `SPLIT=324 EXIT=0` 误读成「修复生效、失败点后移」。
实际上 `SPLIT=324` 只表示下标 0..323 走 IR 时正常，`SPLIT=325` 才含下标 324。
所以失败点**没有移动**，仍是 `p->funcs[324] = loadOpndIR`。

**当前状态**：`tools\selfhost2.cmd` → `[OK] c2 == c3`，`584defa0…`。
IR 全量路径的失败点仍是 `loadOpndIR`（`EXIT=0xC0000005`）。
它的可疑点集中在两处：`o: Opnd` 的 byRef 成员读（`o.kind` / `o.v`），
以及 `var r: Reloc` 之后 `vecPush::<Reloc>(&cg->relocs, r)` 的聚合传参。

### 第 7 轮：IR 改为可选后端 + 差分测试（决策 C）

**决策**：IR 不做「必须立刻跑通」的阻塞项，改为**长期重构 + 差分测试逐步收敛**；
生产路径固定为旧 `genFunc`（`c2 == c3` 不动点靠它保证）。

**改动 1：`IR_SPLIT` 常量 → 命令行开关，消掉踩过的坑**

原来靠改源码常量 `IR_SPLIT` 二分，忘了改回 0 就直接破坏不动点（实测踩过两次）。
现在改为：

```tc
pub var gUseIR: bool = false   // tcc/x64.tc：IR 后端总开关
```

`tcc/tc_main.tc` 参数解析新增 `-ir`：

```
build\c2.exe -ir 源文件.tc -o 输出.exe     :: 走 IR 管道
build\c2.exe     源文件.tc -o 输出.exe     :: 生产路径（默认）
```

**改动 2：`tools/ircheck.cmd` 差分测试（可量化收敛进度）**

对每个测试程序做两次编译（参考 / `-ir`），运行两个产物，比对
**标准输出逐字节 + 进程退出码**。另加一段**嵌套自举差分**：
用两个编译器各自编译同一份 `_spec1.tc`，再运行产物比对输出。

**改动 3：修掉一个我自己引入的错（`IR_BIN.aux2` 传错位置）**

`emit()` 的签名是 `(t, op, dst, aux, aux2, lbl, a, b)`，
我加 `unsFlag(...)` 时把它写在了第 6 个位置，于是 `uns` 落进 **`lbl`**、`aux2` 拿到 0 ——
有符号/无符号分派从未生效过。发射端打点证实：lowering 打 `uns=1`，发射端收到 `uns=0`。

```tc
// 错：emit(t, IR_BIN, d, op as i32, 0, unsFlag(...), ...)
// 对：
emit(t, IR_BIN, d, op as i32, unsFlag(e->lhs->type_), 0, opSlot(a), opSlot(b))
```

**差分测试基线（本轮实测）**

```
[PASS] build\_spec1.tc
[PASS] build\_mc.tc
[PASS] build\_kw.tc
[PASS] build\_lex.tc
[PASS] build\_map3.tc
[PASS] build\_mapbig.tc
-- self-host differential --
[FAIL] self - generated compiler produces different program
PASS=6  FAIL=1  SKIP=0
```

即：**生产编译器用 `-ir` 生成代码，6/6 正确**（含 u64 除模右移、64B 结构体整体赋值、
词法器、哈希表 600 条插入）；**用 IR 编出的编译器再去编程序，结果不一致**。
后者是当前唯一未闭环项，也是后续收敛的唯一验收指标（`tools\ircheck.cmd` 退出码 0）。

生产路径不受影响：`tools\selfhost2.cmd` → `[OK] c2 == c3`，`6ea73400…`。

### 第 8 轮：D7 闭环（`ArrayLit` 未实现）

**定位**：给 `-ir` 加可选数字后缀 `-ir<N>`（只让前 N 个函数走 IR），二分得到
`N=318 → OK，N=319 → OUTPUT_DIFF`，元凶 = `p->funcs[318] = assignRegs`。

**根因**：`assignRegs` 第 2250 行是数组字面量初值

```tc
let pool: [7]i32 = [3, 6, 7, 12, 13, 14, 15];
```

而 `tir.tc` 的 `emitExprIR` **没有 `ArrayLit` 分支**，落进「未覆盖的表达式：暂时返回 0 槽」，
于是 `pool` 变成槽 0（垃圾），`pool[n2]` 读出垃圾 → `locals[i]->reg = 垃圾` →
492 个函数的寄存器分配全部错位。这正好解释「IR 编出的编译器产物行为不一致」，
而不影响「生产编译器用 `-ir` 编小程序」（小程序里没有数组字面量）。

**修法（`tcc/tir.tc`，`emitExprIR` 新增 `ArrayLit` 分支）**

```tc
if k == ExprKind::ArrayLit:
    var aty: *Type = e->type_
    var esz: i32 = 4
    var asz: i32 = 0
    if aty != null:
        asz = aty->size
        if aty->elem != null && aty->elem->size > 0: esz = aty->elem->size
    let tmpA: i32 = newSlot(t)
    if asz > 0 && (tmpA as u64) < t->slotSize.len: t->slotSize.data[tmpA as u64] = asz
    markStackOnly(t, tmpA)
    let baseA: i32 = newSlot(t)
    emit(t, IR_LEASLOT, baseA, tmpA, 0, 0, opImm(0), opImm(0))
    var ai2: u64 = 0
    while ai2 < e->args.len:
        let ve: i32 = emitExprIR(t, e->args.data[ai2])
        let ea: i32 = newSlot(t)
        emit(t, IR_LEAOFF, ea, 0, (ai2 as i32) * esz, 0, opSlot(baseA), opImm(0))
        emit(t, IR_STORE, -1, esz, 0, 0, opSlot(ea), opSlot(ve))
        ai2 = ai2 + 1
    return tmpA
```

**验收（`tools\ircheck.cmd`，退出码 0）**

```
[PASS] build\_spec1.tc
[PASS] build\_mc.tc
[PASS] build\_kw.tc
[PASS] build\_lex.tc
[PASS] build\_map3.tc
[PASS] build\_mapbig.tc
-- self-host differential --
[PASS] self
PASS=7  FAIL=0  SKIP=0
```

**唯一剩余项**：`build\c2.exe -ir tcc\tc_main.tc -o _ir1.exe` 成功且 `_ir1.exe` 作为编译器正确，
但 `_ir1.exe -ir tcc\tc_main.tc` 崩溃 —— 即「IR 编出的编译器再走 IR」这一步未通。
它只在**把 IR 设为默认后端**时才需要；`-ir` 作为可选后端已经验证正确。

生产路径：`tools\selfhost2.cmd` → `[OK] c2 == c3`，`72190ecd…`。

**仍未关闭（D5）**：`genModule` 切到 IR 后，IR 自动机 `_c2_ir` 编译 `build\_f1.tc`
（`def main() -> i32 { return 0 }`）报

```
error[TC1001]: 语法错误：此处期望 '('
  --> 编译单元第 0 行:1073825781
```

即：**词法已正常（不再是 Eof），但解析器拿到的 Token 位置字段是垃圾值**
（`gTokLine`/`gTokCol` 来自 `p->tok.line` / `p->tok.col`）。
下一步应检查 `Token` 结构体的 `line`/`col` 字段在 IR 下的写入与读取
（与刚修好的溢出区问题同类：字段落点是否被别的槽压住），
以及 `pAdvance` 里 `p->tok = lexNext(...)`（16 字节级 sret+成员整体赋值）在 IR 下的落点。

### 第 3 轮：语言规范文档落地

新建 **`docs/TC语言规范.md`**（11 章，面向新手、无历史上下文）：

| 章 | 内容 |
|---|---|
| 0 | 术语、验证方法、版本基准 SHA256 |
| 1 | 快速上手、I/O 函数全表、最小程序 |
| 2 | 词法：注释/标识符（含中文）/字面量/47 个关键字/运算符优先级/缩进换行规则/`Token` 结构/`TokKind` 编号表 |
| 3 | 语法：9 类顶层项、类型语法、函数与泛型、语句（含 switch 缩进体与花括号体）、表达式 |
| 4 | 类型大小/对齐表、struct 布局算法（伪代码）、enum |
| 5 | 语义：有/无符号分派表、截断、sret、指针算术、短路求值 |
| 6 | 完整流水线 9 阶段表（源码→IR→优化→后端→机器码→obj→exe）+ 各阶段输入/输出 + 退出码约定 |
| 7 | IR 完整规范：`TirCtx`/`Opnd`/`Inst`/17 条指令表/槽与虚拟寄存器/栈帧公式/寄存器池 `[3,6,7,12,13,14,15]`/线性扫描算法/规范文本格式 |
| 8 | 设计约束与常见坑（8 条，含「溢出区压局部变量」「无符号不能一律有符号」「自适应布局别用编译期常量」） |
| 9 | 4 组固定测试案例与逐字节预期输出 |
| 10 | 五步判归属流程 + 禁止行为 + 差异对照表（D1..D6） |
| 11 | 收敛判据、当前结论（未完成）、迭代记录 |

**第 9 章四组案例本轮全部实测通过**：

| 案例 | 实测输出 |
|---|---|
| 9.1 `build\_spec1.tc` | `a/b=-2 a%b=-1 a>>1=-4` / `u/3=6148914691236517205 u%3=0 u>>1=9223372036854775807` / `point=3,4` / `add=7` / `color=Green` / `sum=25` / `switch=two`（退出码 0） |
| 9.2 `build\_mc.tc` | `pre0=100 pre1=200 post=300 f0=7 f1=14`（退出码 0） |
| 9.3 `build\_lex.tc` | `lxpos=0 lxlen=3` / `kind=13` |
| 9.4 `tools\selfhost2.cmd` | `[OK] new-syntax bootstrap converges (c2 == c3)`，`2602db89…` |

**文档已声明为未完成**：10.4 表中 D4（IR 路径不动点）、D5（IR 下词法非法 token）、D6（文档逐条复核）仍为「未关闭」，
因此 11.2 明确结论为「未完成」。后续按 11.4 的计划推进。

### 编译器选型定案：`tcc3` 为生产编译器（D8 新发现）

**现象**：把最新自举产物 `c3.exe` 装进 `tc64\bin\tc.exe` 后，用它编译的窗口程序
**双击只出现控制台、不出现窗口**；换回 `build\tcc3.exe` 编译同一份源码，窗口正常弹出。

**复现（最小样本）**：`examples\mail\tcmin.tc`，112 个函数，只做
`rtInit → winInit → winRegisterClassEx → winCreateWindowFramed → winShow → winRun`，
**无网络、无文件 IO、无子控件**，用于把「窗口能否显示」与业务逻辑彻底解耦。

| 编译器 | 编译器 SHA256 | 产物 SHA256 | 双击结果 |
|---|---|---|---|
| `build\tcc3.exe` | `a0bfb19f0c4d2bd5d11c41e4e594e72ed2cca1accd46071d842a69d5e83e526a` | `e465c5320420eea64bcf578c15c9bf5b1b21ba20848cc836903efc846e98ff42` | ✅ 正常弹窗 |
| `build\c3.exe` | `72190ecdcd68fb745e6b24fcf2e5d7cd92ec40cc7887105129efeee8d27bc7f5` | `2f2982a93a6b2fd10489b0d5b8675025be8a3e8b03de2923d691d847bc0d6b5f` | ❌ 只有控制台，无窗口 |

**已排除前端**：两者对 `tcmail.tc`（158 函数）与 `tcmin.tc`（112 函数）都能编译成功、
函数数量一致、零报错 —— 说明词法/语法/语义都正常，差异在**代码生成阶段**。

**规则（重要）**

1. **生产编译器固定为 `build\tcc3.exe`**（SHA256 `a0bfb19f…`）。
   `tc64\bin\tc.exe` 已替换为该二进制，指纹一致；用它编 `tcmin.tc` / `tcmail.tc`
   的产物分别为 `e465c532…` / `b0c72b5377323fb283acee20dce26b562d8d21492dd5cf38952ace3c71e43a7e`，
   与「已验证能开窗」的产物逐字节相同。
2. **`c2`/`c3` 仅作自举能力证明，不得用于编译实际程序。**
   `c2 == c3` 只证明「它能正确编译自己」，**不证明它能正确编译别的程序** —— 这正是本轮踩坑的根因。
3. 窗口程序必须用 `tcc3` 编译；用 `c3` 编出的产物表现为「控制台出现、窗口不出现」。

**未关闭（D8）**：`c3` 的代码生成回归点尚未定位。

已准备的定位手段（下一步）：
对 `min_ok.exe`（tcc3 产物）/ `min_bad.exe`（c3 产物）用 `tc-objdump.exe` 反汇编，
逐函数比对，第一个字节级差异即嫌疑点；若无法映射到源码行，
则按 `tir.tc` / `x64.tc` 的改动逐批回退重建并二分。

**兜底方案（若长期定位不到）**：`tcc3` 作为「可信种子二进制」永久固定，
`c2`/`c3` 只保留在自举验证链中。代价是每次改编译器源码后，
除 `tools\selfhost2.cmd`（不动点）外，还必须用 `tcc3` 重编并实际验证窗口程序能运行。

### GPU 图形 API（D3D11）进展与结论

**目标**：用 TC 调用显卡 DrawCall，最终画出旋转立方体。

**新增文件**

| 文件 | 内容 |
|---|---|
| `tclib/gfx/dx11.tc` | D3D11 绑定：设备/交换链/RTV/视口/清屏/Present + 着色器编译 + 顶点·常量缓冲 + Draw |
| `tclib/gfx/gl.tc` | OpenGL 1.1 绑定（备用路线，**尚未实机验证**） |
| `examples/39_dx/clear.tc` | **可用**：D3D11 设备 + 清屏 + Present（窗口颜色循环） |
| `examples/39_dx/tri.tc` | 最小用例：`SV_VertexID` 三角形（无顶点缓冲/无输入布局/无常量缓冲） |
| `examples/39_dx/tri2.tc` | 顶点缓冲三角形（3 顶点 × 12 字节） |
| `examples/39_dx/cube.tc` | 立方体（**未跑通**，保留作为后续排查入口） |
| `examples/38_gpu/cube.tc` | OpenGL 立方体（**尚未实机验证**） |

**已验证可用（有实机证据）**

| 环节 | 证据 |
|---|---|
| `D3D11CreateDeviceAndSwapChain` | 打印 `B3 device+swapchain ok`，`hr >= 0` |
| `GetBuffer` / `CreateRenderTargetView` | `B3b/B3d/B4 ok` |
| 清屏 + Present | 用户实机看到颜色循环窗口（`clear.exe`） |
| `D3DCompile`（HLSL 运行时编译） | `C3 shaders compiled`，vs_4_0 / ps_4_0 |
| `Draw` 确实执行 | `tri.exe` 画面出现橙黄色光点（退化三角形，说明光栅化发生） |
| 顶点缓冲对象创建 | 标题栏 `vb=216738144 cb=216731488 lay=8242312 ps=5368867664`，`ps` 即 `0x140026B50` = ImageBase + 偏移，指向程序自己的顶点数组 |

**未解决（D9）**

`tri2.exe`（3 顶点 × 12 字节 + 单一 POSITION 输入布局）**未能画出三角形**，因此「顶点缓冲 → 输入布局 → Draw」这条路径存在未定位的缺陷。

**已排除**

- 不是设备/交换链/RTV/Present 问题（`clear.exe` 实机可见）
- 不是 HLSL 编译问题（`D3DCompile` 返回成功）
- 不是着色器对象创建问题（句柄非 0）
- 不是顶点数据本身错（CPU 侧打印 `v0=-1,-1,1` `c0=1,0.2,0.2` 正确）
- 不是上传源指针错（`pSys` 指向程序自己的数组）
- 早期「画面扭曲成团块」的两条真 bug 已修：
  1. `IDXGISwapChain` vtable 漏算 `IDXGIObject`/`IDXGIDeviceSubObject`，
     `Present`/`GetBuffer` 应为 **8/9**（原写 3/4，实际调到了 `SetPrivateDataInterface`，直接访问冲突）
  2. 画家算法排序失效导致背面覆盖正面 → 改为**背面剔除**（凸多面体无需排序、无需深度缓冲）

**下一步（未执行，待续）**

- **A（优先，零成本）**：实机验证 `examples/38_gpu/cube.exe`（OpenGL 1.1 路线，一次都没跑过）
- **B（专业手段）**：`D3D11CreateDeviceAndSwapChain` 加 `D3D11_CREATE_DEVICE_DEBUG`(0x2)，
  再用 `ID3D11InfoQueue` 读出 D3D 自己的诊断文本写入 `dx.log`。
  D3D 会直接报出「输入布局与着色器签名不匹配」「顶点缓冲为空」这类具体原因，
  不必再靠画面颜色和标题栏反推。代价：新增 2 个 vtable 下标。

**遗留的临时诊断代码（下次收尾时清理）**

`tclib/gfx/dx11.tc` 目前含：`标记()`（往控制台打 `[step]`）、`dxLog` 失败弹窗、
`dxDiagTitle`（标题栏写句柄）、颜色编码诊断。功能正确但有噪音，等 D9 定位后统一移除。
