# Godot 4.7 编辑器多实例 Dock 改造

## 项目目标

本项目直接修改并编译 **Godot 4.7 引擎源码**，为编辑器的内置 Dock 引入类似 Unreal Engine 的多实例工作流。

目标不是单纯将面板拖离主窗口（Godot 已支持浮动/分离布局），而是让同一类型的内置面板可以创建多个相互独立的实例。例如：

- FileSystem #1：固定浏览 `res://resources`
- FileSystem #2：固定浏览 `res://scripts`
- Scene #1：编辑 `Level_A`
- Scene #2：编辑 `Level_B`

每个实例都应拥有独立的路径、选中项、筛选条件和布局状态，并可以停靠、浮动及在重启后恢复。

## 当前技术路线

**修改 Godot 4.7 源码并编译自定义编辑器。**

GDScript `EditorPlugin`、GDExtension 或对内置节点的反射/注入都不是本项目的主方案：它们可以创建自己的自定义 Dock，但不能可靠地实例化 Godot 原生的 `FileSystemDock`、`SceneTreeDock`、`InspectorDock` 等内置面板。

| 方案 | 自定义多实例面板 | 原生 FileSystem Dock 多实例 | 本项目采用 |
|---|:---:|:---:|:---:|
| GDScript EditorPlugin | 可以 | 不可以 | 否 |
| GDExtension | 可以 | 不可以 | 否 |
| UI 反射/节点注入 | 有限且脆弱 | 不可以 | 否 |
| Godot 引擎源码改造 | 可以 | 可以 | 是 |

## 第一阶段：FileSystemDock 多实例化

第一阶段只改造 `FileSystemDock`，不同时修改 Scene 和 Inspector。

验收标准：

1. 编辑器提供“New File Browser”入口，可新建多个 FileSystem Dock 实例。
2. 每个实例独立保存当前路径、选择项、筛选和显示模式。
3. 同类实例可同时停靠在主窗口、作为标签页或浮动为窗口。
4. 编辑器布局保存/恢复时保留所有实例及其各自状态。
5. 既有默认 FileSystem Dock 工作流不回归。

## 预期改动区域

- `editor/editor_node.cpp`：内置 Dock 的创建、注册、生命周期和菜单入口。
- `editor/filesystem_dock.cpp` / `editor/filesystem_dock.h`：解耦 FileSystem Dock 的实例状态，支持独立创建与恢复。
- `editor/editor_dock.cpp` / `editor/editor_dock.h`：同类 Dock 的注册、唯一布局标识和持久化。
- 视 Godot 4.7 的实际代码结构，补充布局序列化、编辑器设置及测试文件。

## 源码与构建

引擎源码位于 [`Godot4.7`](./Godot4.7)。先使用未修改的源码完成一次基线构建，再开始改造：

```powershell
cd Godot4.7
git submodule update --init --recursive
scons platform=windows dev_build=yes
```

Windows 构建环境需要 Visual Studio 的 C++ 工具链、Windows SDK、Python 3.9+ 和 SCons 4.4+。详见 [Godot 官方 Windows 编译文档](https://docs.godotengine.org/en/stable/engine_details/development/compiling/compiling_for_windows.html)。

## 后续阶段

- Scene Dock 多实例化。
- Inspector Dock 多实例化及独立锁定/上下文。
- 抽象统一的 Dock 工厂和实例注册机制。
- 布局格式兼容、迁移与回归测试。
