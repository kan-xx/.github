# kan-xx

围绕 **Sony Alpha 7R III (`ILCE-7RM3`, FW `3.10`)** 构建的模块化机内运行时与原生功能扩展项目。在不拆机、不改动原厂固件与内核映像的前提下，打通从 USB 服务连接、离线指令仿真、机内自愈常驻底座，到自定义快门音效、机身原生中文菜单与全界面触控的完整链路。

---

## 免责声明

> 大家请以学习为目的使用本组织下的所有仓库⚠️⚠️⚠️⚠️，[安全研究法律边界与合规须知](https://github.com/kan-xx/.github/blob/main/DISCLAIMER.md#法规与合规参考)
>
> 本组织下所有仓库的内容仅供学习和参考之用，禁止用于商业用途。任何人或组织不得将相关内容用于非法用途或侵犯他人合法权益。所涉及的固件分析、系统逆向与硬件接口研究技术仅用于个人合法自有设备上的学习与研究，不得用于未经授权侵入或破坏他人设备、绕过技术保护措施、提取或传播第三方专有固件或其他非法行为。对于因使用相关内容而引起的任何法律责任、硬件损坏（变砖）、数据丢失或保修失效，各仓库及作者不承担任何责任。使用相关内容即表示您同意本免责声明的所有条款和条件。
>
> 点击查看更为详细的免责声明。[点击跳转](https://github.com/kan-xx/.github/blob/main/DISCLAIMER.md)

---

## 架构与设计原则

整套系统不对机身闪存中的 Linux 内核映像（`uImage`）、RTOS 固件或原厂核心可执行代码段（`.text`）做任何静态修改，亦不涉及固件重打包与分区烧录。持久化入口仅通过向模块注册表 `libIMDB.so` 追加一段纯数据段描述符实现，由原厂动态加载器在启动期按标准生命周期拉起独立编译的外部共享库。用户态行为扩展在运行态内存（RAM）中通过 GOT 重定位替换与 UI 回调桥接完成；内核态硬件观测与总线边界控制则以零外部符号依赖的独立 `.ko` 模块在运行期按需挂载，断电后即恢复原厂初始内存状态。

全栈面向无外部硬件调试口（不拆机焊接主板串口、不依赖底层刷机短接器）的真实单机环境设计。任何持久化部署均受早期启动守护进程 `a7r-boot-guard` 的双槽 SHA-256 事务日志保护：若主进程未能在 60 秒窗口内完成生命周期握手或发生连续异常，守护进程自动将注册表原子切回原厂备份；运行期文件更新通过保留活跃映射底层 inode 的硬链接引用计数（`nlink >= 1`），确保 `/usr` 分区始终可安全重挂载为只读。在涉及跨核 DMA 与编码总线的硬件耦合路径上，凡未闭合完整停机与资源回收证明的状态均遵循失效关闭（Fail-Closed）原则，优先锁定租约并隔离通道，杜绝盲目释放资源引发二次破坏。

在接触物理设备之前，核心加载链路、重定位兼容层与跨核参数路径均先在隔离的 QEMU 原厂动态加载器环境及跨 ARM、Thumb、MIPS32 的无桩指令仿真器（Stub-Free Unicorn）中直接运行原厂机器码完成验证，在离线阶段提前收敛 ABI 边界、算术位宽截断与缓存一致性约束。

依托支持电源休眠与唤醒屏障（`INACT / ACT`）的常驻运行时底座，上层模块能够在原厂并未预留接口的领域独立构建完整的原生功能子系统。所有公开仓库仅包含独立编写的 C、ARMv7 汇编与 Python 源码及确定性构建脚本，不携带、不分发任何专有固件二进制文件，并提供无需物理机身即可独立运行的宿主机测试套件。

---

## 仓库索引

| 仓库 | 层级 | 职责概述 |
| :--- | :--- | :--- |
| [**a7r3-usb-senser**](https://github.com/kan-xx/a7r3-usb-senser) | 连接与会话层 | 主机到相机 USB 模式状态机、USB 3.1 SuperSpeed `516B -> 1024B` 包对齐认证传输、`16B / 2ms` 节流终端与 `/tmp_bt` 内存暂存会话 SDK |
| [**sony-fw-inspect**](https://github.com/kan-xx/sony-fw-inspect) | 离线分析与仿真 | 512 KiB 窗口化逻辑闪存采集与差异调和、16 分区与 `WBI1` 热启动映像解包、`ld-2.11.2.so` `DT_RELCOUNT = 0` 构建审计、跨三核 Unicorn 无桩执行套件 |
| [**a7r3-boot-runtime**](https://github.com/kan-xx/a7r3-boot-runtime) | 机内常驻底座 | `libIMDB.so` 纯数据段扩展、双槽事务日志与单次冷启动重试守护进程 `a7r-boot-guard`、生命周期屏障协调器与 `nlink >= 1` 映射保留原子发布器 |
| [**a7r3-kernel-video-quarantine**](https://github.com/kan-xx/a7r3-kernel-video-quarantine) | 内核探针与总线隔离 | Linux `3.0.27` 独立无符号 `.ko` 被动观测探针、RTOS VCD 编码总线 `LDREX/STREX + DMB` 失效关闭隔离门（`br_native_fence_only`）、DMM/GIC/SGC 回收门控与原码验证集 |
| [**a7r3-custom-shutter**](https://github.com/kan-xx/a7r3-custom-shutter) | 快门音效后端 | RTOS `AudioIP` 32B 有符号残差编解码器、`/dev/mmcb` `'A'..'Z'` 动态盘符加载、`START_EXPOSURE_NOTIFY` `16 ms` 曝光起点 GOT 观察器与双队列连拍抢占宿主 |
| [**a7r3-native-ui**](https://github.com/kan-xx/a7r3-native-ui) | 原生 UI 与菜单路由 | `View 0xFFC` 27 槽 C++ 虚表继承、`WidgetServer` UTF-8 中文图层渲染、`ViewMenu` `SettingNode` `@VFFC` 原子路由、`RTLD_NOLOAD` 后端事务栅栏与实时拓扑检视器 |
| [**a7r3-full-touch**](https://github.com/kan-xx/a7r3-full-touch) | 全界面触控与手势 | `ViewSystem + 0x14` 单点输入回调桥、`20 ms` 异步队列排空栅栏、`MENU`/`Fn`/`IroiroCon`/`0xFFC` 多界面路由，以及支持 JPEG 与 RAW 的 `ViewZoomPb` 双指捏合缩放引擎 |
