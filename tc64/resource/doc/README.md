# TC64 工具链使用手册

对标 mingw64 的一级目录布局，但**整程序编译**：没有目标文件、没有链接器，
产物是**零外部依赖的原生 PE**（导入表为空）。

---

## 1. 目录用途

| 目录 | 内容 |
|---|---|
| `bin` | `tc.exe` 编译器（自举产物） |
| `include` | 运行时 `tcrt/` 与核心库 `tclib/core`、`tclib/gfx` |
| `lib` | 库入口（`-l 名字` 解析到这里） |
| `libexec` | 自举源码与阶段 0 引导器（重建编译器用） |
| `opt` | 可选附加（`tcbase` 基础库） |
| `share\doc` | 本手册 |
| `share\examples` | 示例源码 |
| `etc` | 配置（当前为说明性质，编译器尚不读取） |
| `licenses` | 许可 |
| `x86_64-w64-tcwin32` | 平台绑定预留（**当前未绑定平台**） |

## 2. 最简用法

```
set PATH=D:\zhuo miam\tanchise\tc64\bin;%PATH%
tc hello.tc -o hello.exe
hello.exe
```

默认输出 `a.exe`（与 gcc 一致）。常用选项：

| 选项 | 含义 |
|---|---|
| `-o <文件>` | 输出路径 |
| `-gui` | Windows 子系统（无控制台窗口） |
| `-console` | 控制台子系统（默认） |
| `-dll` | 产物为 DLL，导出全部 `pub fn` |
| `-l <名字>` | 并入库（见第 4 节） |
| `-I <目录>` | 追加 import 搜索根 |
| `-cache` | 编译缓存（改一处不再全量重编） |

## 3. import 解析顺序

```
1. 相对当前源文件所在目录            ← 与过去完全一致，优先
2. <工具链根>\include
3. <工具链根>\x86_64-w64-tcwin32\include
4. <工具链根>\lib
5. <工具链根>\x86_64-w64-tcwin32\lib
6. -I 指定的目录（按给出顺序）
```

**工具链根由编译器自身位置推出**（`bin\tc.exe` 的上一级），所以整个 `tc64`
目录可以拷到任意路径，只要 `bin` 在 PATH 里，源码就能写短路径：

```tc
import "tcrt/rt.tc";          // 解析到 <root>\include\tcrt\rt.tc
import "tclib/gfx/canvas.tc";
```

**include-once 按内容判重**，不按路径字符串 —— 同一文件无论写成相对路径还是
绝对路径，都只会展开一次。

## 4. `-l` 的两种形态

### 形态 A：源码合并（简单，适合纯 TC 库）

```
lib\ltcui.tc          ← 库的实现源码
```
```tc
tc app.tc -l libtcui -o app.exe
```

编译器把 `lib\libtcui.tc` 的正文并进编译单元（等同 import），**没有运行时加载**。

### 形态 B：真 DLL（适合独立发布、可选功能）

```
lib\tcplug.tc         ← 入口：只做声明与转发
lib\tcplug.dll        ← 真身：实现
```
```tc
tc app.tc -l tcplug -o app.exe
```

入口文件在**首次调用时**自己去加载 DLL：

```tc
var gBase: u64 = 0;

pub fn tcplugAnswer(a: i32, b: i32) -> i32 {
    if gBase == 0 { gBase = dlOpenNear("tcplug.dll"); }
    if gBase == 0 { return -1; }
    let p: u64 = peExport(gBase, "tcplugAnswer");
    if p == 0 { return -2; }
    let f: fn(i32, i32) -> i32 = p as fn(i32, i32) -> i32;
    return f(a, b);
}
```

DLL 查找顺序：**exe 同目录 → `..\lib` → `..\bin`**。

**用户程序里不需要任何 import**，只看得到 `-l` 带来的 `pub fn`。

> 约定：`-l 名字` 先找 `lib\名字.tc`。形态 B 下这个 `.tc` 是手写的包装，
> 不要再往里面塞实现。

## 5. 语言现状（重要）

**新旧两套语法并存**，同一文件可混用。

已经可用（2026-11-06 实测）：

```tc
# 注释用 #
module demo
import "../tcrt/rt.tc"

struct Point:                        # 声明体也能用缩进块
    x: i32
    y: i32

def area(p: Point) -> i32:
    return p.x * p.y                 # 行尾无需分号

def add(a, b):                       # 参数类型可省：由调用点推断
    return a + b

def main() -> i32:
    if !rtInit():
        return 1
    var p: Point
    p.x = 6
    p.y = 7
    x = 42                           # 局部免声明，推断为 i32
    print("area =", area(p), "=", add(x, 0))    # 实参间自动空格，末尾自动换行
    return 0
```

输出：

```
area = 42 = 42
```

| 能力 | 状态 |
|---|---|
| `def` 关键字（= `fn`） | ✓ |
| `:` + 缩进块（语句体） | ✓ if / while / for / else / switch 分支 |
| `:` + 缩进块（**声明体**） | ✓ `struct` / `enum` / `union` / `switch` |
| 分号可省（行尾自动补） | ✓ |
| `#` 行注释 | ✓ |
| 返回类型可省 | ✓ |
| 参数类型可省（`def f(a, b)`） | ✓ 由调用点反推；**没有任何调用点时会报错** |
| 局部变量免声明（`x = 6`） | ✓ 类型由右值推断 |
| `elif` | ✓ |
| 跨行 `&&` / `||` 条件 | ✓ 下一行以 `&&` / `||` 开头即视为续行 |
| `print(...)` | ✓ 多参数、按类型分派、实参间自动空格、末尾自动换行 |
| `len(s)` | ✓ 字符串字节数（`*u8` → `i64`）；是 `tcrt\core\io.tc` 里的普通库函数，**不接受数组** |
| `int(s)` / `str(v)` | ✓ 在 `tclib\core\std.tc`，需 `import` |

### 还没做的

| 缺口 | 现状 |
|---|---|
| `import` 的命名空间隔离 | **✓ 已完成（函数 / 全局量）**：每个 `module` 有自己的符号表，顶层名字按 `模块.名字` 修饰；裸名先查本模块，再查其它模块的 `pub` 导出，多于一个报歧义；非 `pub` 模块外不可见 |
| 模块隔离的剩余（类型名 / 常量） | ✗ `struct`/`enum`/`union`/`const` 仍是**全局命名空间**（本轮只隔离了函数与全局量） |
| 限定名访问 `a.b.foo` | ✗ 跨模块访问只能靠「对方 `pub` + 唯一名」，没有 `模块.符号` 语法 |
| ~~编译器源码迁到新语法（阶段 6）~~ | **✓ 已完成**：`tcc\*.tc` 全部是新语法（`def` + 缩进块 + 无分号），且 `c1 == c2` |
| 字符串类型 | ✗ 只有 `*u8` |
| 错误处理（`try` / `Result`） | ✗ 用返回哨兵值 |
| GC | ✗ 只有区域分配器 |
| 线程 / 同步原语 | ✗ |
| `print` 与浮点 | 需走 `float(x)` 或显式 `rtPrintF64`，`print(1.5)` 会按整数输出 |

**结论**：新语法这一层已经完整（缩进写法、类型推断、`elif`、`print` 都可用），
而且**编译器自己的源码就是用这套语法写的** —— 它已经用自己编译了自己。
剩下的缺口是**模块隔离**这类结构性能力，不是语法糖。

> 阶段 6 的副作用（须知）：源码改用新语法后，C++ 写的 Stage-0 编译器不再能解析它。
> 自举链因此从「Stage-0 起」改成「一个已知可用的 TC 编译器起」（`tools\selfhost2.cmd`，
> 要求 `c1 == c2`）。这是自举编译器的常规形态，但意味着**必须保留一个可用的
> `tc.exe` 二进制**；旧语法源码备份在源码仓库的 `legacy\tcc_oldsyntax\`。

## 6. 诊断格式

```
error[TC3010]: 第 2 个实参类型不匹配：期望 'u8'，实际 'u64'
  --> tcc\tc_main.tc:462:37
      |
  462 |     var uhash: u64 = fnAdd(0, fsize);
      |                                     ^
共 1 个错误。
```

- `TCxxxx` 错误码可检索
- `文件:行:列` 指到具体位置（跨 import 也能正确回指）
- 源码行 + `^` 插入符
- 末尾汇总；致命错误另给 `= help:`

## 7. 窗口层（`tclib\win`，平台绑定完成前不随工具链分发）

`gWin` 的成员是**函数指针字段**，必须经 `wcallN` 调用：

```tc
wcall1(gWin.postQuitMessage, 0);
return wcall4(gWin.defWindowProcW, h, msg as u64, wp, lp as u64) as i64;
```

高层封装直接用：

```tc
if !winInit() { return 1; }
winRegisterClass("MyClass", wndProc);
let hwnd: u64 = winCreateWindow("MyClass", "标题", 640, 480);
winShow(hwnd, 5);
winRun();
```

## 8. 校验它是不是自举的

```
certutil -hashfile "tc64\bin\tc.exe" SHA256
```

与 `build-info.txt` 记录的哈希比对。**功能性判据**（比哈希更硬）：把
`share\examples\hello.tc` 单独拷到一个没有 `tcrt` 子目录的地方编译 ——
能过就说明它具备"按自身位置推工具链根"的能力，而这只写在 TC 源码里。

**不动点判据**（在源码仓库里跑）：`tools\selfhost2.cmd` —— 用当前编译器编译
编译器源码两遍，要求 `c1` 与 `c2` **逐字节相同**。源码已经是新语法，所以这条
比"能编出东西"更能说明它内部那套前端是自洽的。
