# preferences.py 文件说明文档

## 文件概述

这是Booltron插件的配置和属性管理模块，定义了布尔运算工具的参数、用户偏好设置以及场景属性。

**文件路径**: [source/preferences.py](source/preferences.py)
**版权**: 2014-2025 Mikhail Rachinskiy
**许可证**: GPL-3.0-or-later

---

## 模块结构

### 导入的依赖
- `bpy`: Blender Python API
- `bpy.props`: 属性类型（布尔、枚举、浮点、整数、指针）
- `bpy.types`: 类型定义（插件偏好设置、属性组）
- 内部模块: `ui`, `var`

---

## 核心类和组件

### 1. 求解器类型定义 (`_solver_items`)

**位置**: [preferences.py:15-21](source/preferences.py#L15-L21)

定义了布尔运算的三种求解器算法：

```python
_solver_items = (
    ("FLOAT", "Float", "好性能，不适用于共面几何体"),
    ("EXACT", "Exact", "最慢，可处理自相交"),
)

# Blender 4.5.0及以上版本增加：
("MANIFOLD", "Manifold", "最快，仅适用于流形网格")
```

**特性**:
- **版本兼容性**: 根据Blender版本动态添加"Manifold"求解器
- **性能权衡**: 速度与精度的平衡选择

---

### 2. ToolProps 类（工具属性基类）

**位置**: [preferences.py:24-112](source/preferences.py#L24-L112)

这是一个混入类（Mixin），定义了布尔工具的所有可配置参数。

#### 主要属性（Primary）

| 属性名 | 类型 | 说明 | 位置 |
|--------|------|------|------|
| `solver` | EnumProperty | 布尔运算求解器选择 | :29-33 |
| `use_self` | BoolProperty | 允许操作数自相交 | :34-37 |
| `use_hole_tolerant` | BoolProperty | 容错模式（处理有孔洞的情况） | :38-41 |

#### 次要属性（Secondary）

| 属性名 | 类型 | 默认值 | 说明 | 位置 |
|--------|------|--------|------|------|
| `use_loc_rnd` | BoolProperty | False | 随机化位置以避免共面错误 | :46-52 |
| `loc_offset` | FloatProperty | 0.005 | 位置偏移范围 [-x, +x] | :53-61 |
| `seed` | IntProperty | 0 | 随机种子值 | :62-65 |
| `display_secondary` | EnumProperty | "WIRE" | 视口显示模式 | :66-76 |

**显示模式选项**:
- `BOUNDS`: 边界框
- `WIRE`: 线框
- `SOLID`: 实体
- `TEXTURED`: 纹理

#### 预处理属性（Pre-processing）

| 属性名 | 类型 | 默认值 | 说明 | 位置 |
|--------|------|--------|------|------|
| `merge_distance` | FloatProperty | 0.0003 | 元素合并的最小距离 | :81-90 |
| `dissolve_distance` | FloatProperty | 0.0001 | 溶解零面积面和零长度边 | :91-100 |

- merge_distance (合并距离)
    - 定义位置: preferences.py:81-90
    - 描述: "Minimum distance between elements to merge" (合并元素之间的最小距离)
    - 默认值: 0.0003 单位长度
    - 作用:将距离小于此值的顶点合并为一个顶点消除重复或非常接近的顶点解决因精度问题导致的微小间隙
    - 使用场景:当模型有重叠的顶点时（比如从其他软件导入的模型）当有非常接近但未实际连接的顶点时,这可以避免布尔运算时出现意外的间隙或错误 
- dissolve_distance (退化溶解距离)
    - 定义位置: preferences.py:91-100
    - UI 显示名称: "Degenerate Dissolve" (退化溶解)
    - 描述: "Dissolve zero area faces and zero length edges" (溶解零面积的面和零长度的边)
    - 默认值: 0.0001 单位长度
    - 作用:删除面积几乎为零的面（退化面）删除长度几乎为零的边（退化边）清理这些"退化"的几何元素
    - 使用场景:当模型中存在压扁的三角形或四边形（面积接近0）当存在两个顶点几乎重叠的边（长度接近0）这些退化元素会导致布尔运算失败或产生错误结果

#### 后处理属性（Post-processing）

| 属性名 | 类型 | 说明 | 位置 |
|--------|------|------|------|
| `use_bake` | BoolProperty | 烘焙修改器结果 | :105-108 |

#### 核心方法

**`asdict()`** - [preferences.py:110-111](source/preferences.py#L110-L111)

将所有属性转换为字典格式：
```python
def asdict(self) -> dict[str, str | float | bool]:
    return {prop: getattr(self, prop) for prop in ToolProps.__annotations__}
```

**用途**: 便于序列化和批量处理属性值。

---

### 3. 属性复制机制

**位置**: [preferences.py:114-116](source/preferences.py#L114-L116)

```python
for prop in ("solver", "use_self", "use_hole_tolerant"):
    ToolProps.__annotations__[f"{prop}_secondary"] = ToolProps.__annotations__[prop]
```

**功能**:
- 为主求解器属性创建对应的次要版本
- 生成的属性: `solver_secondary`, `use_self_secondary`, `use_hole_tolerant_secondary`
- **用途**: 支持双重求解器配置（例如主对象和次要对象使用不同的求解器）

---

### 4. ToolPropsGroup 类

**位置**: [preferences.py:119-126](source/preferences.py#L119-L126)

继承自 `ToolProps` 和 `PropertyGroup`，用于运行时属性存储。

#### 特有属性
- `first_run`: 标识首次运行状态

#### 核心方法

**`set_from_prefs()`** - [preferences.py:122-125](source/preferences.py#L122-L125)

```python
def set_from_prefs(self) -> None:
    prefs = bpy.context.preferences.addons[var.ADDON_ID].preferences
    for prop in ToolProps.__annotations__:
        setattr(self, prop, getattr(prefs, prop))
```

**功能**: 从插件偏好设置批量加载属性值到当前实例。

---

### 5. Preferences 类（插件偏好设置）

**位置**: [preferences.py:132-136](source/preferences.py#L132-L136)

```python
class Preferences(ToolProps, AddonPreferences):
    bl_idname = __package__

    def draw(self, context):
        ui.prefs_ui(self, context)
```

**特性**:
- 继承 `ToolProps` 的所有属性
- 继承 `AddonPreferences` 使其成为Blender插件偏好面板
- `draw()` 方法: 使用 `ui.prefs_ui()` 渲染用户界面

**访问方式**:
```python
prefs = bpy.context.preferences.addons[var.ADDON_ID].preferences
```

---

### 6. WmProperties 类（窗口管理器属性）

**位置**: [preferences.py:143-145](source/preferences.py#L143-L145)

```python
class WmProperties(PropertyGroup):
    destructive: PointerProperty(type=ToolPropsGroup)
    non_destructive: PointerProperty(type=ToolPropsGroup)
```

**用途**:
- 管理两种工作模式的属性集合
- `destructive`: 破坏性布尔运算（直接应用）
- `non_destructive`: 非破坏性布尔运算（使用修改器）

**注册位置**: 通常注册到 `bpy.types.WindowManager`

---

### 7. SceneProperties 类（场景属性）

**位置**: [preferences.py:160-166](source/preferences.py#L160-L166)

```python
class SceneProperties(PropertyGroup):
    mod_disable: BoolProperty(
        name="Non-destructive",
        description="Disable boolean modifiers on all objects",
        default=True,
        update=upd_mod_disable,
    )
```

#### 更新回调函数

**`upd_mod_disable()`** - [preferences.py:152-157](source/preferences.py#L152-L157)

```python
def upd_mod_disable(self, context):
    from .lib import modlib

    modlib.disable_mods(self.mod_disable)
    action = "Enable" if self.mod_disable else "Disable"
    bpy.ops.ed.undo_push(message=f"Non-destructive [{action}]")
```

**功能**:
1. 调用 `modlib.disable_mods()` 禁用/启用所有布尔修改器
2. 推送撤销步骤到历史栈
3. 用户可以通过撤销恢复修改器状态

---

## 架构设计模式

### 1. 混入类模式（Mixin Pattern）
`ToolProps` 作为属性混入类，被多个类继承，实现属性复用：
```
ToolProps
    ├── ToolPropsGroup (运行时属性)
    └── Preferences (全局偏好)
```

### 2. 属性组模式
使用 `PropertyGroup` 和 `PointerProperty` 实现层次化属性管理：
```
WmProperties
    ├── destructive → ToolPropsGroup
    └── non_destructive → ToolPropsGroup
```

### 3. 回调更新模式
属性变更时触发回调（如 `upd_mod_disable`），实现自动同步和撤销支持。

---

## 使用示例

### 访问破坏性模式属性
```python
props = context.window_manager.booltron_props.destructive
solver_type = props.solver
use_randomization = props.use_loc_rnd
```

### 访问插件偏好设置
```python
prefs = context.preferences.addons['booltron'].preferences
default_solver = prefs.solver
```

### 从偏好设置加载属性
```python
props = context.window_manager.booltron_props.destructive
if props.first_run:
    props.set_from_prefs()
    props.first_run = False
```

### 控制修改器状态
```python
context.scene.booltron.mod_disable = False  # 启用所有布尔修改器
```

---

## 版本兼容性

### Blender 4.5.0+
- 新增 **Manifold** 求解器（性能最优）
- 代码中使用版本检查：
  ```python
  if bpy.app.version >= (4, 5, 0):  # VER
      _solver_items = (("MANIFOLD", "Manifold", "..."),) + _solver_items
  ```

---

## 关键技术点

1. **动态属性生成**: 使用 `__annotations__` 动态创建次要属性
2. **类型注解**: 使用现代Python类型提示（`dict[str, str | float | bool]`）
3. **属性单位**: 使用 `unit="LENGTH"` 确保长度单位与Blender场景单位一致
4. **精度控制**: 合并和溶解距离使用高精度（precision=5）避免数值误差
5. **撤销系统集成**: 修改器状态变更会自动推送到撤销栈

---

## 依赖关系图

```
preferences.py
    ├─ 导入: ui.prefs_ui() (绘制UI)
    ├─ 导入: var.ADDON_ID (插件标识)
    └─ 导入: lib.modlib.disable_mods() (修改器管理)
```

---

## 最佳实践建议

1. **首次运行检查**: 始终检查 `first_run` 标志，确保初始化时从偏好设置加载默认值
2. **属性验证**: 使用 `soft_min` 和 `min` 限制属性范围，避免无效输入
3. **撤销支持**: 重要操作后调用 `bpy.ops.ed.undo_push()` 提供撤销功能
4. **版本检查**: 使用新特性前检查 `bpy.app.version` 确保兼容性

---

## 相关文件

- [ui.py](ui.py) - 用户界面绘制
- [var.py](var.py) - 常量定义
- [lib/modlib.py](lib/modlib.py) - 修改器管理库

---

## 维护说明

### 添加新属性时需要：
1. 在 `ToolProps` 类中定义属性和注解
2. 如需UI显示，更新 [ui.py](ui.py) 中的 `prefs_ui()` 函数
3. 如需序列化，`asdict()` 方法会自动包含新属性

### 修改求解器选项时：
- 更新 `_solver_items` 元组
- 考虑版本兼容性检查
- 更新相关文档

---

**文档生成时间**: 2025-12-14
**Blender版本支持**: 2.8+ (部分特性需要4.5.0+)
