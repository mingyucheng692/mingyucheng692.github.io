---
title: "Linux 发布 CI/CD 多轮收敛复盘：“能跑”是表象，不是证据"
date: 2026-09-18T12:00:00+08:00
draft: false
tags: ["C++", "Qt", "Linux", "Windows", "Industrial", "CI/CD", "Multithreading", "Postmortem"]
categories: ["industrial-software"]
summary: "开源工业调试工具 Modbus-Tools（Qt 6 / C++）在建立 Linux 自动化发布管线时，通过多轮迭代收敛出七类跨平台深坑。其中六类共享同一模式：“在别处能跑”往往只是特定环境下的局部表象，而非逻辑正确。本文从 GNU ld 符号剔除、epoll/Winsock 事件时序、Qt 双 Socket 争抢 fd 的未定义行为，到 ELF RUNPATH 契约切分，复盘一个高确定性发布管线的建立过程。"
url: "/zh-cn/blog/tech/linux-release-ci-postmortem/"
---

> **TL;DR**：[Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)（Qt 6 / C++）的首条 Linux 自动化发布管线多轮 CI 才收敛：七类故障中六类同根——**“在别处能跑”只是特定环境下的局部表象，不是正确性证据；跨平台交付必须依赖显式契约，而不是隐式环境。** 最终双平台全量测试用例稳定通过，自包含打包流程端到端闭环，跨平台发布资产自动化构建就绪。

---

## 一条从未端到端闭环的发布流水线

Modbus-Tools 的发布管线由 `v*` tag 触发 GitHub Actions，Windows 与 Linux 双平台并行。Linux 侧跑在 `ubuntu-latest` 宿主拉起的 `ubuntu:22.04` 容器里（锁 glibc 2.35 作 ABI 基线），用 aqtinstall 装 Qt 6.10.3（gcc_64 / Ninja），先以 `RelWithDebInfo` + ASan + `offscreen` 运行全量单元测试套件，再将纯净 `Release` 构建产物交给部署脚本 `deploy-linux.sh` 执行自包含打包与静态校验，生成独立的 `.tar.gz` 发布包。

同一时期，Windows 侧 QA、打包、发布一路顺利通过，4 项资产顺利上传；Linux 侧却从未实现端到端完整闭环。发布预演前后跑了多轮 CI 才收敛，且阻塞点呈现严格的时序级联：流水线作为串行长链路，前序阶段一旦中断，后续逻辑根本没有执行机会——每修复一个阻断点，下一层的缺陷才第一次真正运行。排查过程如同剥洋葱，而每剥一层都需要消耗一次完整的 CI 周期。

先排除两个基础设施初始化阶段的一次性阻塞项：GitHub 官方 action 内嵌的 CPython 基于 glibc 2.38 动态链接，在 glibc 2.35 的容器内因缺失动态符号直接崩溃（`GLIBC_2.38 not found`）；Qt 官方二进制依赖的 `libdbus-1-dev` 在容器镜像中未预装。解决上述两个环境阻断项后，剩下的七类故障才是本文的主体。事后审视，其中六类都在回答同一个问题：它们此前在 Windows CI 或本地开发机上，为何能一路通过？

---

## 为什么此前在 Windows CI 与本地能通过？

对前六处故障此前能够“顺利放行”的成因逐一溯源，每一处都依赖了特定环境的隐式容错：

| 故障类别 | 故障表象 | 此前环境（Windows CI / 本地）放行的真实成因 |
| --- | --- | --- |
| i18n 资源丢失 | 5 例 `QTranslator::load` 返回 false | **Windows CI (MSVC)** 默认保留带 CRT 初始化段的静态库成员，GNU ld 的丢弃行为从未暴露 |
| Socket 错误被吞 | 告警日志在，信号断言 false | **Windows CI** 下 `errorOccurred` 先于 `stateChanged` 到达，guard 放行；Linux 顺序恰好反转 |
| 回环通信死锁 | 数据流不通，最小复现段错误 | 双引擎争抢同一 fd 是未定义行为，**Windows CI** 仅仅容忍了重复 close 的报错 |
| POSIX 移植陷阱 | Windows CI 编译失败 | 方向相反的一例：**Linux 本地**永远有 `unistd.h`，缺陷只在 MSVC 编译期显形 |
| Qt 依赖收集落空 | Step 2 报 `no Qt libraries collected` | **Linux 开发机**的动态加载器（ldconfig 缓存、环境变量）预设了 Qt 安装前缀 |
| Wayland 假阳性 | Step 5 拦截 `libwayland-cursor` | **真实桌面环境**必有 Wayland 会话，验证器把“CI 容器极简”误判为 bundle 缺陷 |

测试通过，只说明在那个特定的语义边界下，缺陷恰好未被触发。在跨平台工程中，“能够运行”只是待验证的假设，其成立的完备条件才是工程证据。

---

## 静态库链接语义：无引用符号导致嵌入资源被整块剔除

5 例国际化测试断言失败，根源在于 `QTranslator::load` 返回 false——Qt 资源系统 `:/i18n/` 和文件系统两条路径都没命中。同一套测试，Windows CI 全部通过，本地 Linux 稳定复现。

根因是静态库在两种链接器下的语义差异：

```text
qrc 编译产物 .o ：只含 Q_CONSTRUCTOR_FUNCTION 初始化器（运行时调用 qRegisterResourceData），
                  不定义任何可被外部引用的未解析符号。
GNU ld (Linux)  ：按传统单遍扫描算法，仅当 .o 能解析当前未决符号（Undefined Symbol）时才提取该成员 → 这些 .o 被整块丢弃。
MSVC (Windows)  ：默认保留包含 CRT 初始化段（.CRT$XC*）的目标文件 → 从未暴露。
```

根因验证仅需一条命令，失败时它没有任何输出（注：`strings` 查不到资源路径，因为 rcc 生成的 qrc 内部路径默认以 UTF-16 格式存储）：

```bash
nm -C build_qa/tests/test_ui_widgets | grep -i qRegisterResourceData
```

在实施修复前，三个事实决定了技术方向。第一，波及面不仅限于测试：生产可执行程序同样静态链接 `libui.a`，若不彻底修复，Linux 发布包的运行时多语言切换将静默失效——单测失败正是生产缺陷的前哨报警。第二，本地 7 败与 CI 的 7 败在数量和类别上严格对应，证明这是确定性的工具链与代码行为差异，而非 CI 容器的环境偶发抖动（Flaky Environment）。第三，拒绝平台条件分支：备选方案还有 OBJECT 库、显式 `Q_INIT_RESOURCE`、qrc 上移可执行目标，最终选中 CMake 3.24+ 原生的 `$<LINK_LIBRARY:WHOLE_ARCHIVE,ui>`——零侵入、单一语义同时覆盖生产 app 与 3 个测试目标，MSVC 端自动映射为 `/WHOLEARCHIVE:ui.lib`，行为等价，零 `#ifdef`：

```cmake
target_link_libraries(Modbus-Tools PRIVATE $<LINK_LIBRARY:WHOLE_ARCHIVE,ui>)
```

---

## 事件顺序不保证，fd 所有权必须唯一

### 瞬时状态竞争导致错误事件被吞没

日志中早已留存确凿证据：`[warning] socket error ... Connection refused` 已明确输出，表明 `onSocketError` 回调已被触发，但测试断言 `errorEmitted` 却为 false——底层网络错误虽已发生，对外业务信号却未派发。用例执行耗时 2018ms，直至超时窗口耗尽仍未捕获到预期信号。

根因在于 Winsock 与 epoll 将底层事件投递至 Qt 事件循环的时序存在根本差异：

* Windows (Winsock)：网络错误发生时，`errorOccurred` 通常先于 `stateChanged(UnconnectedState)` 到达。
* Linux (epoll)：连接被拒时，epoll 迅速上报错误事件，Qt 内部状态先跃迁，`stateChanged(UnconnectedState)` 先将状态机置为 `Closed`，`errorOccurred` 随后才投递。

原代码的 guard 判定的是瞬时状态：

```cpp
// 缺陷代码：依赖瞬时状态判定，在 Linux 乱序下会吞错
if (state_ != State::Opening && state_ != State::Open) {
    return; // 判定为非活动连接的历史噪音，直接吞噬
}
```

在 Linux (epoll) 架构下，错误事件延迟一步投递，此时状态机已前置迁入 `Closed` 态，导致后续到达的连接拒绝事件被守卫逻辑作为“过期噪音”静默丢弃。换言之，若 Linux 用户尝试连接一个不可达端口，UI 将陷入无响应的静默断开状态——这是真实的产品缺陷，绝非测试断言的设计偏差。

设计改进的关键在于重构状态判定模型。守卫逻辑不应依赖瞬时连接状态，而应仲裁“该错误是否属于当前连接会话”。引入会话标志 `socketDropIsError_`：连接发起或接管时置位，主动 close、超时或错误消费后复位。只要属于本轮会话的未决异常，即便状态机已先一步迁入 `Closed`，错误照常派发：

```cpp
// 修正逻辑：基于生命周期会话判定
if (state_ != State::Opening && state_ != State::Open) {
    if (state_ == State::Closed && socketDropIsError_) {
        // 属于本轮会话未决掉线，错误照常派发
    } else {
        return;
    }
}
```

### 描述符多头竞争：从 ::dup() 移植困局到对象所有权接管

`NetworkDebuggerLoopback` 测试中交织着两层不同维度的缺陷。表层缺陷属于线程亲和性违背：`QTcpServer server_` 为未声明 parent 的值成员，工作线程执行 `moveToThread` 时其滞留主线程，导致 `nextPendingConnection` 跨线程创建子对象，触发大量跨线程警告。深层缺陷则是严重的未定义行为（UB）：即便消除警告，数据流依然死锁。基于独立用例的最小复现证明：原实现保留 `sourceSocket`，同时将其底层 native fd 交给 `TcpChannel` 采纳——Qt 的两个内部 Socket Engine 同时监听同一个系统 fd 的读就绪事件，引发严重的读饥饿（Read Starvation），脱离框架的裸复现直接触发段错误（SIGSEGV）。Windows CI 此前能够通过，仅仅是由于底层系统容忍了对已关闭句柄的重复释放报错。

第一版修复在 Linux 本地通过 POSIX `::dup()` 复制私有 fd 并注销原 Socket，本地测试全量通过。然而推送到 Windows CI 后，MSVC 编译期直接拦截：

```text
TcpServerHandle.cpp(15,10): error C1083: Cannot open include file: 'unistd.h'
```

`_dup()` 路线同样走不通：Windows 下 `socketDescriptor()` 返回的是 Winsock `SOCKET` 句柄而非 CRT 文件描述符；若换用 `WSADuplicateSocket`，不仅需要引入 `<windows.h>`，更会堆砌大量平台 `#ifdef` 分支，直接违反项目工程红线。

在跨平台 C++ 工程中，当一个平台专有的系统调用在另一端找不到干净的等价物时，多半是抽象层次选错了。

最终放弃在描述符层面的复制与析构修补，转向对象所有权采纳（Object Adoption）：

```cpp
// TcpServerHandle 交付对象所有权：
channel->adoptSocket(socket);

// TcpChannel 内部安全接管：脱离 QTcpServer 父子树，独占原生 fd 并重绑信号
socket->setParent(nullptr);
delete socket_;
socket_ = socket;
wireSocketSignals();
```

通过纯 Qt 对象树完成生命周期接管，双引擎争抢、POSIX 兼容陷阱与平台条件分支一并消除。

---

## 本地能过不算数：加载器隐式依赖、验证契约与离屏盲区

### 动态加载器的隐式依赖

打包脚本在 CI 容器首次运行时，Step 2 抛 `no Qt libraries collected` 退出。一串连锁落空：`cmake --install` 剥掉了二进制的 build rpath；容器内既无 `LD_LIBRARY_PATH`，Qt 前缀也未注册进 ldconfig；而此刻 bundle 内部的 `lib/` 尚为空白——`ldd` 解析依赖陷入死锁。

开发机之所以能够跑通，全靠本地特定环境未受控的隐式依赖。修复原则遵循显式声明与最小作用域：不污染全局容器环境，仅在 `ldd` 静态分析阶段注入进程级隔离前缀：

```bash
LD_LIBRARY_PATH="${QT_LIBS}" ldd "${TARGET_BIN}"
```

### 验证逻辑失效：审视校验器自身的规格契约

Step 5 密封性检查拦截了 `libQt6WaylandClient.so.6` 所声明的系统依赖 `libwayland-cursor.so.0 not found`。

三条技术路线摆在面前：最轻率的做法是向 CI 容器执行 `apt install libwayland-cursor0`，但这属于“为迎合错误的验证语义而修补环境”；只打包 xcb 插件可收窄依赖面，但剥离原生 Wayland 支持属于产品决策，不应由一次发布构建修复来逾越；若将 Wayland 系统库打包进 bundle，则直接破坏了“宿主基础系统库不随包分发”的架构设计。三条路线均被否决。在真实运行场景中，运行 Wayland 会话的桌面环境必预装此库；在纯 X11 极简环境中，Qt 亦可安全降级回落至 xcb。容器并非最终桌面，因此应重构的是校验器的契约，严格落实所有权切分：

1. 防前缀泄漏：任何 ELF 产物不得解析回构建机的 Qt 安装前缀。
2. 发布完整性：未解析依赖项中，凡属 Qt 官方发行包自带者（`$QT_LIBS`），必须全部收录进 bundle；系统级动态库缺损属于宿主基线范畴，予以容忍。

为这套校验逻辑补全 4 分支测试套件时，还顺带排查出旧脚本中一个潜伏已久的 Bug：

```bash
# 真实 ldd 输出样例: "libwayland-cursor.so.0 => not found"
# 字段解析: $1="libwayland-cursor.so.0", $2="=>", $3="not", $4="found"

# 旧脚本（恒假 Bug：空格切分下，$3 永远不可能等于包含空格的 "not found"）:
awk '$3 == "not found"'

# 修复后（严格字段匹配）:
awk '$2 == "=>" && $3 == "not" && $4 == "found"'
```

未经测试的验证逻辑本身也是生产代码。旧版本此前静默放行的假象，本质上是检验逻辑长期处于“失明”状态（Unexercised Validation）。

### RUNPATH 深度：离屏冒烟测试的结构性盲区

第七类故障不在“此前能跑”的模式内——它源于部署脚本覆写 RUNPATH 时，将原本正确的路径破坏。

脚本曾粗暴地给所有插件统一注入 `$ORIGIN/../lib`。然而插件物理分布于 `plugins/<category>/lib*.so` 两级目录下，该相对路径将解析至不存在的 `plugins/lib`。Qt 官方预编译插件原生烧录的 RUNPATH 本就是符合目录层级的 `$ORIGIN/../../lib`。

自动化冒烟测试（Step 6）为何毫无感知？因为 CI 冒烟测试运行于 `QT_QPA_PLATFORM=offscreen` 模式下，离屏平台插件所依赖的底层核心库（QtCore / QtGui）早已被主程序预先加载进内存。真实用户的 X11 桌面环境则截然不同：动态加载 `libqxcb.so` 必须解析其独有的 `libQt6XcbQpa.so.6`（主程序并未直接链接此库）。一旦 RUNPATH 破损，用户在桌面双击启动应用，进程将因无法加载平台插件而直接异常退出。

修一层，露一层。离屏测试因符号预加载而存在结构性盲区，基于 ELF 依赖图的静态断言成为了阻断此类启动崩溃的最后一道防线。修复后的整包解包审计：26/26 插件 RUNPATH 规范统一、零 Qt 前缀泄漏、裸 `ldd` 零未解析，该套断言随后在 Step 5 的 CI 端复验中顺利通过。

---

## 哪些已有防御机制按预期发挥了作用（What Went Well）

复盘不能只列失败项。这一轮里，有几处已有机制按设计发挥了作用：

- 本地基线与 CI 失败集 7/7 严格对应：绝大多数修复在本地实现闭环验证，CI 周期仅消耗在验证真实的环境差异上，避免了针对已知故障反复空耗流水线。
- Windows CI 充当了 POSIX 污染的编译期哨兵：`unistd.h` 在 Linux 本地编译期静默放行，Windows CI (MSVC) 在初次提交时即实施硬性拦截。
- 全量测试套件（开启 ASan 动态插桩）在发布前精准拦截了真实的未定义行为（双引擎争抢 fd）与产品逻辑缺陷（错误信号吞没）——高质量的测试体系本身就是最敏锐的报警哨位。
- Step 5 在验证语义重构后的首次端到端演练中，立即捕获了深层的插件 RUNPATH 深度缺陷：分层防御机制按预期生效。

## 沉淀下来的四条防线

从首轮流水线失败到双平台全量测试稳定通过、多平台发布资产自动化构建就绪。事后审视，逐轮暴露的故障只是表征，根源在于两个长期事实——Linux 发布链路此前从未被完整执行过，以及多处“能够运行”建立在局部环境的特例表现之上。多轮收敛最终凝结为四条工程防线，每条均有已落地的技术机制作为保障，可直接复用到其他跨平台 C++/Qt 交付工程：

1. “在别处能跑”往往只是逃脱了未定义行为的惩罚。链接器归档裁剪策略、异步事件投递时序、系统级原生句柄抽象差异——每一处“在另一端看似正常”，往往只是那一侧的环境偶然容忍了未定义行为。硬性工程约束随之建立：双平台全量单测套件（带 ASan 动态插桩）出现任何断言失败即无条件熔断流水线，彻底杜绝“带伤交付、上线再补”的侥幸心理。
2. 当系统迫使你编写 `#ifdef` 时，首先审视抽象层次。丑陋的平台分支往往源于过早陷入操作系统专用句柄的细节纠缠；将控制权提升至对象生命周期所有权，平台差异便不复存在。落地成果：网络句柄接管彻底收敛至 `adoptSocket` 单一通道，项目自有代码库保持零平台条件编译。
3. 验证逻辑本身也是生产代码，必须经过严格自测。恒真放行（Vacuously True）的校验脚本比显式报错更具欺骗性——静默放行往往只是掩盖了检验逻辑的彻底失效。落地成果：`check_ldd` 核心函数体从生产脚本中抽离，直接驱动 4 分支测试 Harness（包含正向对照组），实现校验逻辑与其自测套件的零漂移。
4. 捍卫静态不变量，而非修补环境表象。面对构建容器报告的依赖缺失，深究一步即可明确契约边界：厘清“哪些依赖必须随二进制包分发”与“哪些依赖本应由宿主发行版提供”，交付链路方具备确定性。落地成果：Step 5 确立的两项静态不变量（零前缀绝对路径泄漏、Qt 随包依赖零缺失）固化为打包门禁，容器极简基线不再引发假阳性误报。

文末附上一份可直接复用的排查检查清单。文中所用手段不与本项目绑定：使用 `nm` 验证关键资源注册符号是否存留；通过最小复现隔离未定义行为；以进程级 `LD_LIBRARY_PATH` 替代对全局环境的污染；对 `ldd` 输出实施严格的字段级比对而非正则模糊匹配；打包后执行整包解包审计，全面校验 RUNPATH 与前缀泄漏。若你的流水线亦陷入“本地能过、CI 必挂”的困境，不妨对照此清单逐项排查。
