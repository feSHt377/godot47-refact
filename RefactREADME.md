# Godot 4.7 编辑器多实例 Dock 改造

## 项目目标

本项目直接修改并编译 **Godot 4.7 引擎源码**，为编辑器的内置 Dock 引入类似 Unreal Engine 的多实例工作流。

目标不是单纯将面板拖离主窗口（Godot 已支持浮动/分离布局），而是让同一类型的内置面板可以创建多个相互独立的实例。例如：

- FileSystem #1：固定浏览 `res://resources`
- FileSystem #2：固定浏览 `res://scripts`
- Inspector #1：编辑 `Level_A` 节点
- Inspector #2：编辑 `Level_B` 节点（锁定后不跟随主实例）

每个实例都应拥有独立的路径、选中项、筛选条件和布局状态，并可以停靠、浮动及在重启后恢复。

## 当前技术路线

**修改 Godot 4.7 源码并编译自定义编辑器。**

GDScript `EditorPlugin`、GDExtension 或对内置节点的反射/注入都不是本项目的主方案：它们可以创建自己的自定义 Dock，但不能可靠地实例化 Godot 原生的 `FileSystemDock`、`SceneTreeDock`、`InspectorDock` 等内置面板。

| 方案 | 自定义多实例面板 | 原生 Dock 多实例 | 本项目采用 |
|---|:---:|:---:|:-:|
| GDScript EditorPlugin | ✅ | ❌ | 否 |
| GDExtension | ✅ | ❌ | 否 |
| UI 反射/节点注入 | ⚠️ | ❌ | 否 |
| Godot 引擎源码改造 | ✅ | ✅ | **是** |

---

## 已完成阶段

### ✅ 第一阶段：FileSystemDock 多实例化

**提交**: `eae9b1ca09`

验收标准完成情况：

1. ✅ 编辑器提供"New File Browser"入口，可新建多个 FileSystem Dock 实例
2. ✅ 每个实例独立保存当前路径、选择项、筛选和显示模式
3. ✅ 同类实例可同时停靠在主窗口、作为标签页或浮动为窗口
4. ✅ 编辑器布局保存/恢复时保留所有实例及其各自状态
5. ✅ 既有默认 FileSystem Dock 工作流不回归

核心改动：
- `editor/editor_node.cpp`: 新增 `create_file_system_dock()` 工厂方法
- `editor/editor_node.h`: 实例计数器 `filesystem_dock_instance_count` 与 key 列表
- 动态生成 `layout_key` (`FileSystem_1`, `FileSystem_2`)
- 布局序列化新增 `filesystem_dock_instances` 字段

### ✅ 第二阶段：InspectorDock 多实例化

**提交**: `cb7570cec4`

核心功能：
- **主实例/次实例区分**: 第一个创建的 Inspector 为主实例，持有 `singleton` 指针
- **实例追踪**: 静态 `LocalVector<InspectorDock *> instances` 管理所有实例
- **跟随模式**: 次实例默认跟随主实例的对象选择
- **锁定模式**: 次实例可点击 "Lock" 按钮锁定当前对象，不再跟随
- **快捷键去重**: 仅主实例注册全局快捷键，避免命令冲突
- **运行时创建**: 工具栏新增 "New Inspector" 按钮
- **布局持久化**: `inspector_dock_instances` 保存/恢复所有实例

核心改动：
- `inspector_dock.h`: 新增 `is_primary_instance`, `locked`, `lock_button`, `instances`
- `inspector_dock.cpp`: 构造函数区分主/次实例，`follow_primary()` 同步逻辑
- `editor_node.cpp`: `create_inspector_dock()` 工厂方法，布局保存/恢复
- `editor_node.h`: 实例计数器与 key 列表

---

## 待完成阶段

### 第三阶段：SceneTreeDock 多实例化

目标：支持多个 SceneTree 面板同时编辑不同场景。

### 第四阶段：抽象统一的 Dock 工厂

目标：将 FileSystemDock 和 InspectorDock 的改造模式抽象为通用机制。

- 统一的 `DockFactory` 创建接口
- 自动实例 ID 生成
- 自动布局 key 管理
- 快捷键去重策略

### 第五阶段：布局格式兼容与迁移

目标：确保旧版布局文件可正确加载到新引擎。

---

## 改造架构

### 工厂模式

```
EditorNode
├── create_file_system_dock(layout_key)
│   ├── 动态生成 layout_key (FileSystem_1, FileSystem_2...)
│   ├── 区分主实例/次实例
│   ├── 次实例不注册全局快捷键
│   └── 注册到 EditorDockManager
├── create_inspector_dock(layout_key)
│   ├── 动态生成 layout_key (Inspector_1, Inspector_2...)
│   ├── 主实例持有 singleton
│   ├── 次实例支持锁定/跟随
│   └── 注册到 EditorDockManager
├── 布局保存
│   ├── filesystem_dock_instances → [FileSystem_1, FileSystem_2]
│   └── inspector_dock_instances → [Inspector_1, Inspector_2]
└── 布局恢复
    └── 遍历实例列表，调用工厂方法重建
```

### 实例状态管理

| Dock 类型 | 主实例 | 次实例 | 实例追踪 | 快捷键去重 |
|------|:---:|:---:|:-:|:-:|
| FileSystemDock | ✅ | ✅ | ✅ | ✅ |
| InspectorDock | ✅ | ✅ | ✅ | ✅ |
| SceneTreeDock | ❌ | ❌ | ❌ | ❌ |

---

## 已知限制

1. **单例保留**: `get_singleton()` 仍返回主实例，全局调用需改造
2. **快捷键冲突**: 仅通过主/次实例区分避免，未实现动态快捷键
3. **SceneTreeDock**: 尚未改造，依赖单例调用较多

---

## 源码与构建

引擎源码位于当前工作区。构建命令：

```powershell
scons platform=windows target=editor dev_build=yes compiledb=yes -j8
```

Windows 构建环境需要 Visual Studio 的 C++ 工具链、Windows SDK、Python 3.9+ 和 SCons 4.4+。
详见 [Godot 官方 Windows 编译文档](https://docs.godotengine.org/en/stable/engine_details/development/compiling/compiling_for_windows.html)。

---

## 分支信息

- `dock-refact`: 多实例 Dock 改造分支
- `main`: 原始 Godot 4.7 基线
