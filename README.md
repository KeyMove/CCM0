# [stdcc — JS 版标准 C 语言编译器（ARM Thumb）](https://keymove.github.io/CCM0/)

一个用纯 JavaScript 实现的标准 C11 编译器，目标为 ARM Cortex-M0 (Thumb-1)，
完全遵循经典编译器标准流程，可将 C 源码编译为可烧录的 `.bin` 固件。

## 标准编译流程

```
[预处理] → 展开宏 / include / 条件编译
  ↓
[词法/语法/语义分析] → 检查错误，生成 AST
  ↓
[中间代码生成] → 转为三地址码 IR
  ↓
[优化] → 常量折叠、死代码剔除、公共子表达式消除
  ↓
[代码生成] → 转为 ARM Thumb 汇编 (.s)
  ↓
[汇编] → 转为机器码 (.o)
  ↓
[链接] → 合并运行时库和地址 → 最终可执行文件 (.bin)
```

## 特性

- **C11 语法**（自研递归下降解析器，附 ANTLR4 文法 `grammar/C.g4`）
- **预处理**：`#include` / `#define`（含带参宏、变参、`#` 字符串化、`##` 连接）/ `#if` 条件编译 / `#error` / `#warning`
- **基础语法**：变量、表达式、控制流（`if`/`for`/`while`/`do-while`/`switch`）、`goto`/标签（向前/向后）、函数、数组、指针、宏、条件编译、`include`
- **C11 特性**：`for` 内声明、`//` 注释、复合字面量基础用法
- **结构体与位域**：`struct` / 位域（多字段/有符号/复合赋值/位域数组）/ 匿名位域、嵌套结构体成员访问
- **数据类型**：8/16/32 位整型、float/double（软浮点预留）、32 位乘除（软除法库）
- **标准库**：`printf`/`sprintf`/`snprintf` 编译器内联展开（`%d/%u/%x/%c/%s/%f/%ld`）、常用数学库（`sqrt/sin/cos` 软实现）
- **单片机扩展**：`weak` 关键字、`interrupt N` 关键字
- **中文字符串**：按 GB2312 / UTF-8 可选编码嵌入 flash（与 Keil 默认行为一致）

## 目录结构

```
stdcc.js              # 主入口 / CLI
include/              # 内置头文件（stdint.h / stdio.h / stddef.h / math.h）
grammar/C.g4          # ANTLR4 文法（参考设计）
src/
  tokens.js           # Token 定义与关键字表
  lexer.js            # 词法分析器
  preprocessor.js     # 预处理器（宏/include/条件编译/字符串编码）
  constexpr.js        # 预处理器常量表达式求值
  parser.js           # 递归下降语法分析器 → AST
  ast.js              # AST 节点定义
  types.js            # 类型系统
  semantic.js         # 语义分析（符号表/类型检查/结构体偏移）
  ir.js               # 三地址码 IR
  irgen.js            # IR 生成器（AST → IR）
  optimizer.js        # 优化 Pass（常量折叠/死代码/公共子表达式/跳转优化）
  codegen.js          # ARM Thumb 代码生成器（IR → .s 汇编）
  assembler.js        # Thumb-1 汇编器 + 链接器（.s → .o → .bin）
  runtime.js          # 运行时库（软除法/取模例程）
  c4_reg.js           # 供 legacy thumb_backend.js 加载的依赖（位于仓库根目录）
examples/demo.c       # 示例程序
tests/run.js          # 基础回归测试套件（95 用例，通过 ThumbCPU 执行校验）
tests/stress.js       # 压力测试第一套：架构/优化器压力（37 用例）
tests/stress2.js      # 压力测试第二套：位域/goto/字节数组/真实算法（41 用例）
thumb_backend.js      # 仓库原有 Thumb 汇编器/反汇编器/CPU 虚拟机
```

## 使用方法

```bash
# 编译为 .bin / .o
node stdcc.js examples/demo.c

# 仅生成汇编 (.s)
node stdcc.js examples/demo.c -s

# 显示 IR
node stdcc.js examples/demo.c -S

# C↔汇编对照（汇编中插入 C 源码行注释，便于调试）
node stdcc.js examples/demo.c -s --src-asm

# 仅预处理
node stdcc.js examples/demo.c -E

# 中文字符串按 GB2312 编码（默认 UTF-8）
node stdcc.js examples/demo.c --gb2312

# 指定优化级别
node stdcc.js examples/demo.c -O2

# 运行测试套件（基础回归，95 用例）
node tests/run.js

# 稳定性压力测试第一套（架构/优化器压力，37 用例）
node tests/stress.js

# 稳定性压力测试第二套（位域/goto/字节数组/真实算法，41 用例）
node tests/stress2.js

# 全量功能回归（345 用例）
node test.js

# 可指定优化级别（O0-O3）
node tests/stress.js 0
node tests/stress2.js 3
```

## 输出说明

- **`.bin`**：可烧录固件。flash 段（text + rodata）从 `0x08000000` 起，RAM 段从 `0x20000000` 起。
- **`.o`**：汇编后的机器码对象。
- **`.s`**：ARM Thumb 汇编文本。

编译产物包含一个启动桩（`start: BL main` → 死循环），入口地址即 `main`。

## 内建启动代码（对齐 thumb 后端 genROM）

默认编译出的 `.bin` 为轻量启动桩；如需生成**可上电自启动的标准 ROM 镜像**，加 `--startup` 选项：

```bash
node stdcc.js examples/demo.c --startup          # 默认向量表 2 项（SP + Reset）
node stdcc.js examples/demo.c --startup --vec 5  # 向量表扩展到 5 项
```

`--startup` 会在 `.bin` 内建完整启动逻辑，与 `thumb_backend.js` 的 `genROM` 方法一致：

| 元素 | 说明 |
|---|---|
| **向量表** | `[0]=SP`（RAM 顶）、`[1]=Reset`，后续中断槽按 `--vec` 定义大小，**默认 2 项**，空槽用 `B .`（`0xE7FE`）无限循环填充 |
| **init 搬运** | 启动代码把带初始化的全局变量从 ROM 数据池拷入 RAM |
| **bss 清空** | 启动代码把未初始化变量区（bss）清零 |
| **跳转 main** | 启动代码 `BL main`，main 返回后进入 `SWI 0xFF` + 死循环 |

`--startup` 生成的 `.bin` 布局为 `[向量表][init 数据池][代码][rodata]`，入口指向 Reset 向量，适合直接烧录/上电运行（无需宿主预初始化 RAM）。

### interrupt N 关键字（中断向量）

`interrupt N` 可把函数声明为第 N 号中断处理函数，向量表对应槽会自动用 `BL handler` 回填（4 字节正好占满一个槽），并**自动把向量表扩展为 `max(N)+1` 项**；未声明的槽用 `B .`（`0xE7FE`）自跳兜底。槽 0/1 保留给 SP 与 Reset。

```c
int g = 0;
interrupt 3 void isr_timer(void){ g = 7; }   // 触发中断 3 跳转 isr_timer
int main(){ return g; }
```

```bash
node stdcc.js isr.c --startup
# 启动: 内建向量表(4 项) SP=0x20008000 Reset=0x8000015
# 中断: 3:isr_timer@0x8000032
```

### ROM 布局（init / const / string 顺序，bss 不占 ROM）

链接器严格按 `init → const → 字符串` 的顺序把需初始化的数据放在 ROM，且**未初始化变量（bss）只占 RAM、不写入 ROM**：

| 数据 | 段 | 存放 |
|---|---|---|
| 有初始化的全局变量（`int g1=7`） | `.data` init 池 | 初始值进 ROM，启动时搬运到 RAM |
| `const` 全局（`const int ci=5`、`const char msg[]="Hi"`） | `.rodata` | 只读 ROM，**不占 RAM** |
| 字符串字面量 | `.rodata` | ROM |
| 未初始化全局（`char gram[8192]`） | bss | 仅 RAM，启动时清零，**ROM 中不出现** |

例如 `char gram[8192]` 全 0 数组，flash 里 init 池仅含实际初始化的几个字节，绝不会出现 8192 个 0 去初始化它。

## 优化 Pass

- **常量折叠**：`2 + 3` → `5`（编译期求值）
- **代数简化**：`x+0`→`x`，`x*1`→`x`，`x&0`→`0`，`x^x`→`0` 等
- **强度削减**：乘/除/取模 2 的幂 → 移位/位与（`x/8` → 算术右移序列，避免慢速软件除法；`x*8` → 左移）
- **死代码剔除**：删除无副作用的未使用临时变量计算
- **公共子表达式消除**：`a*b` 重复出现时复用结果
- **跳转线程化**：合并连续跳转

> 强度削减（c4_reg 优化思路）：除以/取模 2 的幂用 `ASR` 移位 + 修正序列替代 `__sdiv32/__smod32` 软除法，大幅提升整数除法性能，对有符号数按“向零取整”语义正确处理正负值。

## 关于 `thumb_backend.js`

仓库原有的 `thumb_backend.js` 是一份 ARM Thumb 汇编器 + 反汇编器 + CPU 虚拟机
（依赖缺失的 `c4_reg.js`）。本编译器：

1. 补充了根目录的 `c4_reg.js`，使 `thumb_backend.js` 可正常加载；
2. 测试套件 `tests/run.js` 使用其中的 `ThumbCPU` 虚拟机执行编译产物并校验 main 返回值；
3. 代码生成采用自研集成的 `src/codegen.js` + `src/assembler.js`（更贴合本 IR）。

## Web 编译器 IDE（webide.html）

`webide.html` 是一个**纯浏览器**的现代化编译器 IDE，可在网页上编译大型 C 项目，
无需安装任何本地工具链：

```bash
node build_webide.js      # 生成 webide_stdcc.js（把 src/ 模块打包成浏览器 bundle）
open webide.html          # 浏览器打开即可使用
```

### 界面布局

- **左侧 · 文件树**：支持文件夹展开/折叠、新建文件/文件夹、重命名、删除，
  以及 **📂 打开文件夹**（`webkitdirectory` 上传整个本地项目到浏览器）。
- **中间 · 编码窗口**：多标签代码编辑器，内置 C 语法高亮、行号、面包屑导航。
- **右侧 · tab 分栏**：
  - **编译输出**：编译日志、错误/警告定位、段大小统计。
  - **HEX 预览**：Intel HEX 固件记录 + 地址/字节/ASCII 对照表。
  - **C·汇编对照**：汇编中内嵌 C 源码行注释（`--src-asm` 模式），逐行对照。
- **顶部工具栏**：⚙️ 内存设置（模态弹窗：Flash/RAM 地址、RAM 大小、优化级别、
  中文字符串编码、启动方式）、▶ 编译、⬇ 下载 .bin / HEX。

### 编译能力

- 编译器 bundle 把 stdcc 全部 `src/` 模块（预处理→语法→语义→IR→优化→代码生成→
  汇编→链接）打包进浏览器，**完整保留 stdcc 的全部编译能力**（C11、宏、include、
  结构体位域、软浮点、`interrupt N` 等）。
- `#include` 通过浏览器内虚拟文件系统（VFS）解析：内置 `include/` 头文件 + 当前项目文件树。
- 多文件项目：文件树中的 `.c` / `.h` 均可被 `#include` 引用；入口文件默认取 `main.c`。
- 编译产物与 CLI 完全一致（字节级对齐），可在浏览器内直接下载固件。

## Web wegui 模拟器

`simulator.html` 是一个**纯浏览器**的 ARM M0 虚拟机 + wegui 模拟器，
内置了 `wegui_demo.c` 编译产物（bin + 符号表），打开即可运行：

```
node build_demo.js wegui_demo.c simulator.html   # 一键编译 + 内嵌 bin/符号表
open simulator.html                               # 浏览器打开即可运行
```

核心链路：

- `stdcc` 把 C 源编译为 ARM Thumb-1 `.bin`，同时导出符号表（C 变量名 → 内存地址）；
- 浏览器内用 `thumb_backend.js`（UMD，`window.C4Thumb`）执行 `.bin`；
- M0 VM 开启**性能模式**（`enablePerf`），并以**分片模式**（`runChunked`）连续运行不阻塞浏览器；
- 浏览器每毫秒通过符号表地址 **`setmem` 写全局 `ms`** → 驱动 GUI 动画；
- 触摸 **`touch_x / touch_y / touch_state`** 同样通过 `setmem` 注入；
- 程序执行到 `printf("%p", GRAM)` 时，宿主捕获 GRAM 地址并把 RGB565 显存绘制到 `<canvas>`；
- 从而实现**交互式 web wegui 模拟器**，全程无需真机。

演示程序 `wegui_demo.c` 展示：全局 `ms` 驱动的按钮往返动画、进度条推进，
以及触摸点按按钮变色的交互效果。

### printf 地址传递 + while(1) 常驻 demo（testgui01）

`testgui01.html` 演示一种更贴近真实 MCU 的接入方式：**标准启动 + while(1) 常驻主循环**，
所有输入输出统一通过 `printf` 完成：

```
node build_testgui.js testgui01.html   # 一键编译 testgui01.c + 内嵌 bin
open testgui01.html                    # 浏览器打开即可运行
```

核心思路（见 `examples/testgui01.c`）：

- **标准 MCU 启动**：以 `--startup --ram-size 0x20000` 编译，内建向量表（SP + Reset），
  程序从向量表上电启动，`BL main` 进入 `while(1)` 主循环**永不返回**；
  web 端以分片（`runChunked`）常驻运行，无需担心卡死。
- **printf 地址传递**：`main()` 第一行
  `printf("@ms,@touchx,@touchy,@touchstats", &ms, &touchx, &touchy, &touchstats)`，
  宿主检测到第一条以 `@` 开头的字符串即视为“地址传递”，按逗号解析名字并绑定地址，
  之后浏览器直接对 `ms / touch*` 的 RAM 读写注入输入（毫秒时钟、触摸坐标/状态），无需符号表查址。
- **延时靠全局 `ms`**：浏览器每毫秒 `setmem` 写 `ms`，主循环据此推进动画帧。
- **GRAM 改为 RGB565**：每帧 `printf("%p", GRAM)` 把 RGB565 显存地址交回宿主转 `<canvas>`。

> 为规避编译器对复杂控制流的既有缺陷，`testgui01.c` 把绘图逻辑内联在 `main` 的
> `while(1)` 循环内，只使用位移+掩码驱动动画（不依赖除法/取模/软除法例程）。

### 多文件链接

`build_wegui.js` 支持把多个 C 源文件（wegui 库 + demo）分别编译后合并为单个 `.bin`，
自动为每文件内部标签加命名空间，并导出 C 符号名 → 地址映射（供 setmem 使用）。

```
node build_wegui.js -o app.bin main.c wegui/we_gui_driver.c wegui/dirty_driver.c ...
```

### 本次改进（编译器 / 链接器）

- **结构体返回值整体赋值**：`*dst = foo_struct_ret()...` 的 RHS 为返回结构体的函数调用时，
  直接把返回值写入目标（修复“不能作为左值”的 IR 报错）；
- **rodata 基址稳定迭代**：链接器在多次重编码后仍可能因字面量池就地冲刷改变 text 长度，
  导致 `.str/.LC` 字符串符号地址与落盘位置错位；现迭代到 `rodataBase == ceil(textLen/4)*4` 收敛；
- **非启动模式 textAddr**：非 `--startup` 构建下 flash 不嵌入向量表/init 池，
  修正 `textAddr` 叠加 initPoolSize 造成的字符串/rodata 地址 16 字节错位；
- **多文件链接 start 桩位置**：`build_wegui.js` 不再在 text 中间插入第二个 `.section .text`
  （会重置段内地址计数器使 `start` 标签错乱到 0x08000000），改为把启动桩放到 text 最前。

> 说明：当前编译器对“多 if + 嵌套循环 / 函数内局部变量进循环”等复杂控制流仍存在
> 代码生成缺陷，故 web demo 采用每帧渲染一帧后返回、由宿主重复运行的稳健模式，
> 避免在 `main` 内使用 `for(;;)` 无限循环。完整 wegui driver 已能编译并链接通过。

## License

MIT
