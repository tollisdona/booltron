# ModLib 模块文档

本文档解释 `booltron/source/lib/modlib.py` 中布尔运算的实现原理。

---

## 核心概念：为什么用几何节点？

Booltron 使用 **几何节点 (Geometry Nodes)** 而非传统的布尔修改器来实现布尔运算。

```
传统方式:  Object → Boolean Modifier → Result
Booltron:  Object → Geometry Nodes Modifier → Boolean Node → Result
```

**优势**：
- 可以在节点树中组合多个对象
- 支持随机位置偏移
- 更灵活的参数控制

---

## 1. `disable_mods(value: bool)`

**功能**：批量启用/禁用场景中所有布尔相关修改器的视口显示。

### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `value` | `bool` | `True` 启用显示，`False` 禁用显示 |

### 执行步骤

1. 遍历当前场景所有对象
2. 筛选 `MESH` 类型对象
3. 遍历对象的修改器
4. 若修改器类型为 `BOOLEAN` 或 Booltron 几何节点修改器
5. 设置 `show_viewport` 属性

### 输出

无返回值，直接修改修改器属性。

---

## 2. `_ng_import(filepath: Path, ng_name: str)`

**功能**：从外部 .blend 文件导入节点组。

### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `filepath` | `Path` | .blend 文件路径 |
| `ng_name` | `str` | 要导入的节点组名称 |

### 执行步骤

1. 使用 `bpy.data.libraries.load()` 打开 .blend 文件
2. 指定要导入的节点组名称
3. 返回导入的节点组

### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `node_group` | `BlendData` | 导入的节点组 |

---

## 3. `_walk_tree(node: GeometryNode)`

**功能**：递归遍历节点树，获取指定节点的所有上游节点。

### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `node` | `GeometryNode` | 起始节点 |

### 执行步骤

1. 遍历节点的所有输入接口
2. 遍历每个输入的连接
3. 若连接来源不是 `GROUP_INPUT`，则 yield 该节点
4. 递归遍历来源节点的上游

### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `Iterator` | `Iterator[GeometryNode]` | 上游节点迭代器 |

---

## 4. `secondary_visibility_set(ob: Object, display_type="TEXTURED")`

**功能**：设置次级对象的渲染可见性。

### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `ob` | `Object` | 目标对象 |
| `display_type` | `str` | 显示类型，默认 `"TEXTURED"` |

### 执行步骤

1. 根据 `display_type` 计算 `visible` 布尔值
2. 设置 `display_type` 和 `hide_render`
3. 设置通用可见性（相机、阴影）
4. 设置 Cycles 渲染器可见性（漫射、光泽、透射、体积散射）
5. 设置 EEVEE 渲染器探针可见性

### 输出

无返回值，直接修改对象属性。

---

## 5. `ModGN` 类

**核心类** - 封装几何节点布尔运算的创建、应用和管理。

### 类属性 (`__slots__`)

| 属性 | 描述 |
|------|------|
| `mode` | 布尔模式：`DIFFERENCE`, `UNION`, `INTERSECT` |
| `solver` | 主对象求解器：`EXACT`, `FAST`, `MANIFOLD` |
| `use_self` | 主对象自相交检测 |
| `use_hole_tolerant` | 主对象孔洞容差 |
| `solver_secondary` | 次级对象求解器 |
| `use_self_secondary` | 次级对象自相交检测 |
| `use_hole_tolerant_secondary` | 次级对象孔洞容差 |
| `merge_distance` | 顶点合并距离 |
| `use_loc_rnd` | 启用随机位置偏移 |
| `loc_offset` | 位置偏移量 |
| `seed` | 随机种子 |

---

### 5.1 `ModGN.__init__(self, mode, settings)`

**功能**：初始化 ModGN 实例。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `mode` | `str` | 布尔运算模式 |
| `settings` | `dict` | 设置字典 |

#### 执行步骤

1. 设置 `self.mode`
2. 从 `settings` 字典读取并设置其余属性

---

### 5.2 `ModGN.add(self, ob1, obs, md=None, show_viewport=True)` ⭐

**功能**：创建几何节点布尔修改器（核心方法）。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `ob1` | `Object` | 主对象（被操作对象） |
| `obs` | `list[Object]` | 次级对象列表（操作工具） |
| `md` | `Modifier` | 可选，复用已有修改器 |
| `show_viewport` | `bool` | 是否显示在视口 |

#### 执行步骤

**第一阶段：创建节点组基础结构**

```
┌─────────────────────────────────────────────────────────────────┐
│  Node Group: "{ob1.name} {mode}"                                │
│                                                                 │
│  [Group Input] ─→ (Geometry) ─→ ... ─→ [Group Output]          │
│                   (Offset)                                      │
│                   (Seed)                                        │
└─────────────────────────────────────────────────────────────────┘
```

1. 创建新节点组，命名为 `"{ob1.name} {mode}"`
2. 标记节点组 `ng["booltron"] = self.mode`（用于识别）
3. 创建接口：
   - 输入/输出 `Geometry` 接口
   - `Randomize Location` 面板（`Offset`, `Seed`）

**第二阶段：创建输入/输出节点**

4. 创建 `NodeGroupInput` 节点（位置 x=-200）
5. 若主对象非 MESH 类型，添加 `MergeByDistance` 节点（合并顶点）
6. 创建 `NodeGroupOutput` 节点（位置 x=400）
7. 创建 `GeometryNodeBake` 节点（位置 x=200）

**第三阶段：创建布尔运算节点**

```
                    ┌─────────────┐
   Input Geo ──────→│   PRIMARY   │──→ Bake ──→ Output
                    │   Boolean   │
                    │  (mode)     │
           ┌───────→│   Mesh 2    │
           │        └─────────────┘
           │
   ┌───────┴─────┐
   │  SECONDARY  │
   │   Boolean   │
   │  (UNION)    │←── ob1 ←── ob2 ←── ob3 ...
   └─────────────┘
```

8. 创建 **PRIMARY** 布尔节点：
   - 操作模式 = `self.mode`（DIFFERENCE/UNION/INTERSECT）
   - 求解器 = `self.solver`
   - 若 EXACT 求解器，设置自相交和孔洞容差
   - 连接：输入几何体 → Mesh 1（差集）或 Mesh 2（其他）

9. 创建 **SECONDARY** 布尔节点：
   - 操作模式 = `UNION`（始终为并集）
   - 用于将所有次级对象合并为一个
   - 连接输出到 PRIMARY 的 Mesh 2

**第四阶段：添加次级对象**

10. 遍历 `obs` 列表，调用 `_ob_add()` 添加每个对象
11. 将每个对象的输出连接到 SECONDARY 的 Mesh 2

**第五阶段：配置修改器**

12. 创建或复用几何节点修改器
13. 设置随机偏移参数（若启用）
14. 配置修改器显示选项
15. 关联节点组

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `md` | `Modifier` | 创建的修改器 |

---

### 5.3 `ModGN.add_and_apply(self, ob1, obs, remove_obs=True)`

**功能**：创建修改器并立即应用（破坏性操作）。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `ob1` | `Object` | 主对象 |
| `obs` | `list[Object]` | 次级对象列表 |
| `remove_obs` | `bool` | 是否删除次级对象 |

#### 执行步骤

1. 调用 `add()` 创建修改器（`show_viewport=False`）
2. 保存节点组引用
3. 使用 `bpy.ops.object.modifier_apply()` 应用修改器
4. 删除节点组
5. 若 `remove_obs=True`，删除所有次级对象的网格数据

#### 输出

无返回值。

---

### 5.4 `ModGN.extend(self, md, obs)`

**功能**：向已有修改器添加更多次级对象。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `md` | `Modifier` | 已有修改器 |
| `obs` | `list[Object]` | 要添加的对象列表 |

#### 执行步骤

1. 禁用修改器视口显示
2. 获取节点组中已有的所有对象
3. 合并新旧对象集合
4. 删除旧节点组
5. 调用 `add()` 重新创建（包含所有对象）
6. 启用修改器视口显示

#### 输出

无返回值。

---

### 5.5 `ModGN._ob_add(self, ng, ob, in_ofst, in_seed, seed=0)`

**功能**：在节点组中添加单个次级对象的节点链。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `ng` | `NodeGroup` | 目标节点组 |
| `ob` | `Object` | 要添加的对象 |
| `in_ofst` | `NodeSocket` | 偏移输入接口 |
| `in_seed` | `NodeSocket` | 种子输入接口 |
| `seed` | `int` | 当前对象的种子偏移 |

#### 执行步骤

创建如下节点链：

```
┌──────────────┐    ┌─────────────────┐    ┌────────────────────┐
│ Object Info  │───→│ Merge by Dist   │───→│ Randomize Location │───→ Output
│ (ob)         │    │ (非MESH时)      │    │                    │
└──────────────┘    └─────────────────┘    └────────────────────┘
                                                    ↑
                                           ┌───────┴───────┐
                                           │  Integer Add  │
                                           │ (seed + n)    │
                                           └───────────────┘
```

1. 创建 `ObjectInfo` 节点，引用次级对象
2. 若对象非 MESH 类型，添加 `MergeByDistance` 节点
3. 导入或获取 `Randomize Location` 节点组
4. 创建 `IntegerMath` 节点（seed + 偏移）
5. 连接所有节点

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `output` | `NodeSocketGeometry` | 随机位置节点的几何输出 |

---

### 5.6 `ModGN.remove(md, obs)` [静态方法]

**功能**：从修改器中移除指定的次级对象。

#### 入参

| 参数 | 类型 | 描述 |
|------|------|------|
| `md` | `Modifier` | 目标修改器 |
| `obs` | `set[Object]` | 要移除的对象集合 |

#### 执行步骤

1. 禁用修改器视口显示
2. 遍历 SECONDARY 节点的 Mesh 2 输入连接
3. 使用 `_walk_tree()` 获取每个连接的上游节点
4. 检查 `ObjectInfo` 节点引用的对象
5. 若对象在 `obs` 中：
   - 恢复对象可见性
   - 删除相关节点
6. 若还有剩余对象，重新启用修改器
7. 若无剩余对象，删除整个修改器和节点组

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `bool` | `bool` | `True` 表示修改器已完全删除 |

---

### 5.7 `ModGN.has_obs(md, obs)` [静态方法]

**功能**：检查修改器是否包含指定对象。

#### 执行步骤

1. 遍历节点组中所有节点
2. 查找 `OBJECT_INFO` 类型节点
3. 检查其引用的对象是否在 `obs` 中

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `bool` | `bool` | 是否包含指定对象 |

---

### 5.8 `ModGN.is_gn_mod(md)` [静态方法]

**功能**：判断修改器是否为 Booltron 创建的几何节点修改器。

#### 执行步骤

检查三个条件：
1. 修改器类型为 `NODES`
2. 存在节点组
3. 节点组包含 `"booltron"` 自定义属性

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `bool` | `bool` | 是否为 Booltron 修改器 |

---

### 5.9 `ModGN.check_mode(md, mode)` [静态方法]

**功能**：检查修改器的布尔模式。

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `bool` | `bool` | 模式是否匹配 |

---

### 5.10 `ModGN.bake(md)` [静态方法]

**功能**：烘焙几何节点修改器的结果。

#### 执行步骤

1. 检查文件是否已保存
2. 调用 `bpy.ops.object.geometry_node_bake_single()` 执行烘焙

---

### 5.11 `ModGN.bake_del(md)` [静态方法]

**功能**：删除修改器的烘焙数据。

---

### 5.12 `ModGN.is_baked(md)` [静态方法]

**功能**：检查修改器是否已烘焙。

#### 输出

| 返回值 | 类型 | 描述 |
|--------|------|------|
| `bool` | `bool` | 是否已烘焙 |

---

## 布尔运算实现原理图

```
                          ┌─────────────────────────────────────────────┐
                          │           Geometry Node Tree                │
                          │                                             │
  ┌─────────┐            │   ┌─────────┐                               │
  │  Main   │ ──Geometry──┼──→│  Input  │                               │
  │ Object  │            │   └────┬────┘                               │
  └─────────┘            │        │                                     │
                          │        ▼                                     │
                          │   ┌─────────────────┐     ┌──────────────┐  │
                          │   │    PRIMARY      │     │              │  │
                          │   │    Boolean      │────→│     Bake     │──┼──→ Result
                          │   │  (DIFF/UNION/   │     │              │  │
                          │   │   INTERSECT)    │     └──────────────┘  │
                          │   └────────┬────────┘                       │
                          │            ↑ Mesh 2                         │
                          │   ┌────────┴────────┐                       │
                          │   │   SECONDARY     │                       │
                          │   │    Boolean      │                       │
  ┌─────────┐            │   │    (UNION)      │                       │
  │Secondary│ ──Reference─┼──→│                 │                       │
  │Object 1 │            │   └────────┬────────┘                       │
  └─────────┘            │            ↑                                 │
  ┌─────────┐            │   ┌────────┴────────┐                       │
  │Secondary│ ──Reference─┼──→│   Object Info   │                       │
  │Object 2 │            │   │  + Randomize    │                       │
  └─────────┘            │   └─────────────────┘                       │
       ...                │                                             │
                          └─────────────────────────────────────────────┘

关键点：
1. PRIMARY 节点执行实际的布尔运算（由 mode 决定）
2. SECONDARY 节点将多个次级对象合并为一个（始终 UNION）
3. 次级对象通过 Object Info 节点引用（非破坏性）
4. Bake 节点用于缓存计算结果
```

## 不同模式的区别

| 模式 | PRIMARY 操作 | 效果 |
|------|-------------|------|
| `DIFFERENCE` | 差集 | 从主对象减去次级对象 |
| `UNION` | 并集 | 合并所有对象 |
| `INTERSECT` | 交集 | 保留重叠部分 |

连接方式的区别（第 136 行）：
- `DIFFERENCE`: 主对象 → Mesh 1，次级 → Mesh 2
- `UNION/INTERSECT`: 主对象 → Mesh 2，次级 → Mesh 2
