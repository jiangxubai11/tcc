# TC64 —— 中文发行版

TC 是一门面向 Windows x64 的自举系统级语言。编译器**直接生成 PE 可执行文件**：
不需要链接器、不依赖 C 运行库、产物的导入表为空。

本目录是**中文版**发行包（英文版见同级的 `TC64eng`）。

---

## 目录结构

```
tc64/
  toolchain/       tc.exe（编译器）+ 8 个配套工具
  include/
    tcrt/          运行时：内存、文件 IO、系统调用、PEB/导出表解析
    tclib/         标准库
      core/        std str vec map fmt intern mathx
      fs/          文件读写
      gfx/         canvas color m3d render yuv gl dx11
      math/  time/  media/  win/
  lib/             库入口（libtccore.tc 等）
  samples/
    mail/          可用的 SMTP over TLS 邮件客户端（GUI）
    opengl/        OpenGL 1.1 旋转立方体（尚未实机验证）
    d3d11/         Direct3D 11 示例（见下方「状态」）
  manual/          ROADMAP.md 开发记录 + 本文件
  config/          配置文件（tc.conf）
  license/         许可说明
  resource/        随包文档与示例
```

### 两个目录名不能改

`include` 和 `lib` 这两个名字**写死在 `toolchain/tc.exe` 里**，编译器按自身位置推导：

```
<tc.exe 所在目录>\..\include      <- 短路径 import 在这里解析
<tc.exe 所在目录>\..\lib          <- 库入口
<tc.exe 所在目录>\..\include\x86_64-w64-tcwin32\include
<tc.exe 所在目录>\..\include\x86_64-w64-tcwin32\lib
```

改名后所有程序里的 `import "tcrt/rt.tc"` 立刻失效。
其余目录（`toolchain`、`samples`、`manual`、`config`、`license`、`resource`）
纯属约定，编译器完全不看，随便改。

---

## 快速上手

```bat
cd tc64

toolchain\tc.exe samples\mail\tcmail.tc -o samples\mail\tcmail.exe
samples\mail\tcmail.exe
```

最小程序：

```tc
module hello

import "tcrt/rt.tc"

def main() -> i32:
    if !rtInit():
        return 1
    rtPrint("你好\n")
    return 0
```

```bat
toolchain\tc.exe hello.tc -o hello.exe
hello.exe
```

因为运行时用的是短路径 `tcrt/rt.tc`，程序源文件可以放在任何位置，
只要用**这个** `tc.exe` 编译即可 —— 搜索根是按编译器自身路径算的，
与当前工作目录无关。

---

## 编译器选项

| 选项 | 作用 |
|---|---|
| `-o <路径>` | 输出文件（默认 `a.exe`） |
| `-I <目录>` | 追加 import 搜索目录（可重复） |
| `-l <库名>` | 链接库（可重复） |
| `-console` | 控制台子系统（默认） |
| `-gui` | GUI 子系统，不弹控制台 |
| `-c` | 只生成 `.obj`（再用 `tc-ld.exe` 链接） |
| `-dll` | 生成 DLL |
| `-cache` | 增量编译缓存 |

**不要用 `-ir`**：IR 后端尚未完成。

---

## 语言速览

* 类型：`i8 i16 i32 i64`、`u8 u16 u32 u64`、`f32 f64`、`bool`、`ptr`、`*T`、
  `[]T`（切片）、`[N]T`（数组）、`struct`、`union`、`enum`、`fn(A) -> R`
* 泛型：`Vec<T>`、`mapPut::<V>(...)`
* 语句：`var`、`if/elif/else`、`while`、`for x in lo..hi`、
  `switch/case/fallthrough`、`break`、`continue`、`return`
* 内建：`syscall`、`read_gs`、`slice`、`sqrt`、`sin`、`cos`、`atan2`、`print`
* 模块：`module 名字` + `import "路径"`，`pub` 控制可见性

---

## 状态

| 组件 | 状态 |
|---|---|
| 编译器（自举不动点已验证） | 可用 |
| Windows 运行时（无 CRT、导入表为空） | 可用 |
| `tclib` 的 core / fs / win / gfx / math / time / media | 可用 |
| SMTP over TLS 邮件客户端（示例） | **已实机验证**（从 smtp.qq.com 真实投递并收到） |
| Direct3D 11：设备、交换链、渲染目标、清屏、Present | **已实机验证** |
| Direct3D 11：从顶点缓冲绘制几何 | **未打通（D9，未定位）** |
| OpenGL 1.1：旋转立方体 | **从未实机验证** |

未解决问题（详见 `manual/ROADMAP.md` 的 D9）：通过顶点缓冲提交的几何到不了屏幕。
`Draw` 确实执行了，但没有像素被光栅化；而设备/交换链/清屏/Present 这条链已被证明是好的。

同一份文档里还记录了两个已修的坑：`IDXGISwapChain` 的 vtable 下标必须算上
继承来的 `IDXGIObject` / `IDXGIDeviceSubObject`（`Present` = 8、`GetBuffer` = 9），
以及示例为什么用背面剔除而不是深度缓冲。

---

## 关于重建编译器

`toolchain/tc.exe` 是**固定版本**。它是一个自举不动点：
拿编译器源码喂给它自己，会产出同一个二进制。请保留这个二进制，
不要用别的版本替换 —— 除非你重新验证过全部示例。
