# Destructive Boolean 操作模块文档

本文档描述 `booltron/source/operators/destructive/__init__.py` 文件中各函数的作用。

---

## 1. `_cursor_state(func)`

**装饰器函数** - 用于在执行耗时操作时更改鼠标光标状态。

### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `func` | `Callable` | 被装饰的函数 |

### 运算过程

1. 定义内部 `wrapper` 函数，接收 `self` 和 `context` 参数
2. 将光标设置为 `"WAIT"`（等待状态）
3. 执行被装饰的函数并保存返回值
4. 将光标恢复为 `"DEFAULT"`（默认状态）
5. 返回被装饰函数的执行结果

### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `wrapper` | `Callable` | 包装后的函数 |

---

## 2. `Destructive` 类

**基类** - 所有破坏性布尔运算操作符的基类。

### 类属性

| 属性 | 类型 | 描述 |
|------|------|------|
| `mode` | `str` | 布尔运算模式（由子类定义） |
| `is_overlap` | `bool` | 对象是否重叠，默认 `True` |
| `keep_objects` | `BoolProperty` | 是否保留原始对象（快捷键：Alt） |

---

### 2.1 `Destructive.draw(self, context)`

**UI 绘制方法** - 绘制操作符的属性面板界面。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `self` | `Destructive` | 操作符实例 |
| `context` | `bpy.types.Context` | Blender 上下文 |

#### 运算过程

1. 获取 `booltron.destructive` 属性组
2. 配置布局属性（`use_property_split`, `use_property_decorate`）
3. **Primary Object 区块**：
   - 显示求解器选项 (`solver`)
   - 若求解器为 `EXACT`，显示 `use_self` 和 `use_hole_tolerant`
4. **Secondary Object 区块**：
   - 若非 `SLICE` 模式，显示次级求解器选项
   - 显示 `keep_objects` 属性
   - 显示随机位置偏移选项（`use_loc_rnd`, `loc_offset`, `seed`）
5. **Pre-processing 区块**：
   - 显示 `merge_distance`（合并距离）
   - 显示 `dissolve_distance`（退化溶解距离）

#### 输出

无返回值，直接修改 UI 布局。

---

### 2.2 `Destructive.execute(self, context)`

**执行方法** - 执行布尔运算的核心逻辑（被 `@_cursor_state` 装饰）。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `self` | `Destructive` | 操作符实例 |
| `context` | `bpy.types.Context` | Blender 上下文 |

#### 运算过程

1. **准备阶段**：
   - 获取 `destructive` 属性配置
   - 检查是否使用 `MANIFOLD` 求解器
   - 调用 `objectlib.prepare_objects()` 分离主对象和次级对象
   - 调用 `meshlib.prepare()` 对所有对象进行网格预处理（合并顶点、溶解退化面）

2. **非流形检测**：
   - 如果检测到非流形几何体，报错并退出
   - 若 `keep_objects=True`，删除复制的网格数据

3. **布尔运算**：
   - 创建 `ModGN` 几何节点修改器实例
   - **重叠模式** (`is_overlap=True` 或单个次级对象)：
     - 直接应用修改器到主对象
   - **非重叠模式**：
     - 先将所有次级对象合并为一个
     - 再应用修改器

4. **结果验证**：
   - 检查结果是否为非流形，若是则报错

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `{"FINISHED"}` | `set` | 操作完成状态 |

---

### 2.3 `Destructive.invoke(self, context, event)`

**调用方法** - 操作符被调用时的入口点，处理用户交互。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `self` | `Destructive` | 操作符实例 |
| `context` | `bpy.types.Context` | Blender 上下文 |
| `event` | `bpy.types.Event` | 用户输入事件 |

#### 运算过程

1. **模式切换**：
   - 若当前不在对象模式，切换到对象模式并推送撤销记录

2. **对象过滤**：
   - 遍历选中对象，取消选择非支持类型（仅保留 `MESH`, `CURVE`, `SURFACE`, `META`, `FONT`）

3. **选择验证**：
   - 检查选中对象数量，少于 2 个则报错取消

4. **重叠检测**：
   - 若选中超过 2 个对象且非 `SLICE` 模式
   - 调用 `meshlib.detect_overlap()` 检测次级对象间是否重叠

5. **属性初始化**：
   - 首次运行时从偏好设置加载默认值
   - 重置 `use_loc_rnd` 为 `False`
   - 根据 `Alt` 键状态设置 `keep_objects`

6. **执行分支**：
   - 按住 `Ctrl`：弹出属性对话框
   - 否则：直接执行 `execute()`

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `{"CANCELLED"}` | `set` | 操作取消（对象不足） |
| `{"FINISHED"}` | `set` | 直接执行完成 |
| `{"RUNNING_MODAL"}` | `set` | 弹出对话框（由 `invoke_props_dialog` 返回） |

---

## 3. `OBJECT_OT_destructive_difference` 类

**差集操作符** - 从主对象中减去选中对象。

### 继承

`Destructive`, `Operator`

### 属性

| 属性 | 值 | 描述 |
|------|-----|------|
| `bl_label` | `"Difference"` | 操作符标签 |
| `bl_idname` | `"object.booltron_destructive_difference"` | 操作符 ID |
| `mode` | `"DIFFERENCE"` | 布尔模式 |

---

## 4. `OBJECT_OT_destructive_union` 类

**并集操作符** - 合并选中对象。

### 继承

`Destructive`, `Operator`

### 属性

| 属性 | 值 | 描述 |
|------|-----|------|
| `bl_label` | `"Union"` | 操作符标签 |
| `bl_idname` | `"object.booltron_destructive_union"` | 操作符 ID |
| `mode` | `"UNION"` | 布尔模式 |

---

## 5. `OBJECT_OT_destructive_intersect` 类

**交集操作符** - 保留主对象与选中对象的公共部分。

### 继承

`Destructive`, `Operator`

### 属性

| 属性 | 值 | 描述 |
|------|-----|------|
| `bl_label` | `"Intersect"` | 操作符标签 |
| `bl_idname` | `"object.booltron_destructive_intersect"` | 操作符 ID |
| `mode` | `"INTERSECT"` | 布尔模式 |

---

## 6. `OBJECT_OT_destructive_slice` 类

**切片操作符** - 沿选中对象的体积切割主对象，生成两个独立部分。

### 继承

`Destructive`, `Operator`

### 额外属性

| 属性 | 类型 | 描述 |
|------|------|------|
| `overlap_distance` | `FloatVectorProperty` | 切片间的重叠偏移量 |

---

### 6.1 `OBJECT_OT_destructive_slice.draw(self, context)`

**UI 绘制方法** - 扩展父类绘制，添加切片特有选项。

#### 运算过程

1. 调用父类 `draw()` 方法
2. 添加 "Slice" 区块
3. 显示 `overlap_distance` 属性

---

### 6.2 `OBJECT_OT_destructive_slice.execute(self, context)`

**执行方法** - 切片操作的核心逻辑（被 `@_cursor_state` 装饰）。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `self` | `OBJECT_OT_destructive_slice` | 操作符实例 |
| `context` | `bpy.types.Context` | Blender 上下文 |

#### 运算过程

1. **准备阶段**（与父类 `execute` 相同）：
   - 获取属性、准备对象、预处理网格
   - 非流形检测

2. **创建修改器**：
   - `ModDiff`: 差集修改器
   - `ModIntr`: 交集修改器

3. **遍历每个次级对象**：

   a. **复制主对象**：
      - 复制对象和网格数据
      - 链接到相同集合
      - 设为选中状态

   b. **主对象差集运算**：
      - 将次级对象向正方向偏移 `overlap_distance / 2`
      - 对主对象应用差集修改器（保留次级对象）
      - 检查结果是否非流形

   c. **复制对象交集运算**：
      - 将次级对象向负方向偏移 `overlap_distance`（总共偏移 `-overlap_distance / 2`）
      - 对复制对象应用交集修改器
      - 检查结果是否非流形

4. **完成处理**：
   - 取消选择主对象
   - 将最后一个复制对象设为活动对象

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `{"FINISHED"}` | `set` | 操作完成状态 |

---

## 流程图

```
invoke()
   │
   ├── 切换到对象模式
   ├── 过滤非支持类型对象
   ├── 验证选择数量 (≥2)
   ├── 检测重叠 (>2 对象时)
   ├── 初始化属性
   │
   ├─── [Ctrl 按下] ──→ 弹出属性对话框 ──→ execute()
   │
   └─── [直接执行] ──→ execute()
                           │
                           ├── 准备对象
                           ├── 预处理网格
                           ├── 非流形检测
                           ├── 应用布尔修改器
                           └── 验证结果
```
