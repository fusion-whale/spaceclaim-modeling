# spaceclaim-modeling

**把 Ansys SpaceClaim 变成命令行里可复现的建模工具**：用几行 Python 建几何、倒圆角/倒角、按自然语言描述给边界命名（`inlet` / `outlet` / `wall` / `interface` / `baffle`…），后台跑出带命名分区的 `.scdocx`，并自动做**独立校验**。

> Drive Ansys SpaceClaim headlessly from the command line: build geometry, fillet/chamfer it, and name boundary zones from a short Python script, then verify the saved `.scdocx` by re-reading it in a fresh session. Docs are in Chinese.

> ### 👉 第一次来，直接读 [`HANDOFF.md`](HANDOFF.md)
> 一份**自包含的能力交接书**：全部实测基线、**15 条踩坑总表**（每条带实测数字）、
> "哪些 API 在这个版本不能用"的确定性结论、8 个示例模型的数值、以及怎么自己再扩能力。
> 读完那一份就能上手，不用翻别处。
>
> - 想直接看东西：[`docs/`](docs/) 里有 4 张效果图 + 一份能力汇报页
> - 想直接打开模型：[`samples/`](samples/) 里有 8 个已校验的 `.scdocx`（不用跑脚本）

```python
body = box(4.0, 4.0, 10.0, origin=(0, 0, 0), name="Channel")
round_edges(edges_parallel(body, "z"), 1.0)          # 四条长边倒圆 r=1
name_faces_by_rules(body, [
    ("inlet",       {"normal": "z", "sign": -1}),
    ("outlet",      {"normal": "z", "sign": +1}),
    ("fillet_wall", {"kind": "cylinder"}),
    ("wall",        {"rest": True}),
])
finish(r"channel.scdocx", body)
```

```powershell
& .\scripts\Invoke-Scdm.ps1 -Script .\channel.py -Out .\channel.scdocx -Verify
```

```
[scdm] artifact_size=594563 bytes
--- verify (fresh session, re-read from disk) ---
[verify] body[0] size = 4.000 x 4.000 x 10.000 mm
[verify] body[0] edges = 24 [('circle', 8), ('line', 16)]
[verify]   inlet      center=(2.000, 82.000, 0.000) mm  area=15.14 mm2
[verify]   outlet     center=(2.000, 82.000, 10.000) mm  area=15.14 mm2
[verify]   fillet_wall center=(3.500, 83.500, 5.000) mm  area=15.71 mm2
[verify]   wall       center=(2.000, 80.000, 5.000) mm  area=20.00 mm2
[scdm] status=ok
```

> 注意上面端面是 **15.14 mm²** 而不是 16.00 —— 四个角被 r=1 的圆角吃掉了，`16 − (4 − π) = 15.142`。数值对得上手算，才算真的建成了。

## 为什么有这个项目

SpaceClaim 的脚本 API 本身能用，但**"从命令行把它跑起来"这条路几乎没有一处文档写全**。以下每一条都是实测踩出来的，任何一条错了脚本都会**静默不执行**或**直接中止**：

- `/RunScript` **只认 `.py`**；传 `.scscript` 只会打开界面、根本不执行
- 参数必须带引号**经 `cmd.exe` 传递**；PowerShell 直接调用会丢引号，开关被忽略
- `/Headless=True` 下没有窗口，脚本报错**只能**从 `/ScriptOutput` 日志看；日志为空时真正的错误在 `%APPDATA%\SpaceClaim\Log Files\*.log` 的 `Script failed:` 行
- **退出码 0 不等于成功**（脚本抛异常也是 0），必须用哨兵 `<<<SCDM_OK>>>` 自己判定
- **带非 ASCII 文字的异常会让宿主静默中止整个脚本**——连 traceback 都没有。所以库里所有 `raise` 都是 ASCII
- `NamedSelection.Create` **不接受名字参数**，而且默认名是本地化的（中文界面下不是 ASCII）
- 面的几何信息必须走 `face.Shape`；边的几何在 `DesignEdge.Shape` 上
- **新建的体与已有体重叠会被自动并集**（实测：板上横放一个方块，最后只剩 1 个体）
- `CylinderBody.Create` 的三个点顺序与直觉相反，写反会得到一个轴向和半径都错的圆柱
- `Move.Rotate` 的角度是**弧度**，传 `45` 会得到 58.3° —— 看着很像对的
- 倒圆角/倒角**只接受"边"的选择**：传面报 StandardError、传体报 ValueError；而且每次倒完，之前抓的边对象全部失效

这个仓库把上述路径封装成可以直接调用的工具，并把结论沉淀成文档，省掉后来人的试错。

## 环境要求

| 项 | 说明 |
|---|---|
| 操作系统 | Windows |
| SpaceClaim | 已安装即可，脚本会自动探测。开发与验证基于 **2022 R1（`v221`）** |
| PowerShell | 5.1 及以上（运行器是 5.1 兼容的，且刻意只用 ASCII 编写） |
| 权限 | 调用进程需能写入 `%APPDATA%\SpaceClaim\`（SpaceClaim 启动要写日志、日志文件与用户配置，并读取许可）。在受限沙箱里请放宽文件权限，否则 SpaceClaim 会**静默退出、什么都不做** |

## 安装

```powershell
# 方式一：作为 DSH（DeepSeek Harness）技能安装，之后 agent 会自动发现并调用
git clone https://github.com/fusion-whale/spaceclaim-modeling.git "$env:USERPROFILE\.dsh\skills\spaceclaim-modeling"

# 方式二：当普通命令行工具用，clone 到任意位置即可
git clone https://github.com/fusion-whale/spaceclaim-modeling.git
```

> 上面是 Windows PowerShell 写法。用 Git Bash / WSL 的话等价写法是
> `git clone <url> ~/.dsh/skills/spaceclaim-modeling`。
> 装完不用重启：DSH 的技能提供方会扫描 `<dshHome>/skills` 并在下一次模型步进时刷新目录。

## 用法

1. 复制 `scripts/template_model.py`，改成你的模型；
2. 用运行器执行：

```powershell
& .\scripts\Invoke-Scdm.ps1 -Script .\my_model.py -Out .\my_model.scdocx -Verify
```

运行器会：拼装脚本 → 按实测过的命令行调用 SpaceClaim → 用**哨兵**判定成功 → 检查产物 → 可选地开新会话回读校验。

常用参数：`-Verify`（独立校验，强烈建议）、`-Gui`（弹界面看过程）、`-TimeoutSec`、`-SpaceClaimExe`。

> `-Out` 必须和脚本里 `finish(路径)` 保存的路径**完全一致**，否则会被报成 `artifact is missing`（即使建模本身成功了）。

## 已封装的能力

建模脚本里直接调用即可（运行器会把 `scdm_lib.py` 注入到脚本最前面，**不需要 import**）。

**几何**（`box`/`cylinder`/`tube`/`sphere` 不走草图，随时可以加；草图类**必须最先画**）

| 函数 | 说明 | 实测结果 |
|---|---|---|
| `box(w, d, h, origin, name, cut, separate)` | 长方体；`cut=True` 布尔减、`separate=True` 重叠也保持独立 | 6×5×20 mm，6 面 |
| `cylinder(r, h, origin, axis, name, cut, separate)` | 圆柱，轴向 `"x"/"y"/"z"` | r2.5×30，3 面 |
| `tube(外r, 内r, h, origin, axis, name)` | 空心圆管 | 30×12×12，4 面 |
| `sphere(r, center, name, cut)` | 球；`cut=True` 抠球腔（内部走 `ForceCut`，普通 `Cut` 对球无效） | r5 → 1 面 314.16 mm² |
| `cone_frustum(r1, r2, h, origin, axis, name)` | **真**锥台/圆锥（走回转体） | r10→r4×20，3 面 |
| `elbow(r, R, angle_deg, origin, axis, name)` | **真圆截面**弯头（回转出的圆环段） | r3/R20/90° → 3 面，torus 592.18 + 两端面 28.27 各；包围盒 23×23×6 |
| `torus(r, R, …)` | 整圈圆环（= 弯曲 360°） | 1 个 torus 面 2368.71 mm² = 4π²Rr，包围盒 46×46×6 |
| `polygon_prism(sides, r, h, …)` / `profile_prisms([…])` | 正多边形 / 任意折线 / 椭圆截面的柱体 | 正六边形 8 面、椭圆 3 面 |
| `move(body, dx, dy, dz)` | 整体平移 | 位移精确 |
| `rotate(body, 角度, axis, center)` | 绕轴旋转（**传角度，库内部换弧度**） | 45° 旋转后包围盒 35.355 |

**倒圆角 / 倒角**（只吃"边"）

| 函数 | 说明 |
|---|---|
| `round_edges(目标, r)` | 倒圆角；目标可以是单条边、边的列表、或**一个体**（= 它的所有边） |
| `chamfer_edges(目标, d)` / `chamfer_edges(目标, d1, d2)` | 等距 / 不等距倒角 |
| `round_face_edges(面, r)` | "把这个面的四周倒圆" |
| `round_by_rules(body, [(规则, r), …])` / `chamfer_by_rules(…)` | 规则表批量倒角 |
| `edges_parallel / edges_by_kind / edges_along_axis / edges_at / edges_in_box / edges_of_faces / edges_at_point / nearest_edge / match_edges` | 挑边 |
| `edge_kind / edge_length / edge_center / edge_extent / edge_direction / edge_axis / edge_summary` | 量边 |

```python
round_edges(cube, 2.0)                              # 整个体所有边
round_edges(edges_parallel(cube, "z"), 3.0)         # 只倒四条竖边
round_edges(edges_by_kind(cube, "circle"), 1.0)     # 只倒所有圆边
round_face_edges(faces_by_normal(cube, "z", 1), 3.0)  # 顶面四周
round_outer_rims(pipe, 1.0)                         # 管口两圈圆边
```

**抽壳**

| 函数 | 说明 |
|---|---|
| `shell(body, t, open_faces=None, outward=False)` | 掏成等壁厚的壳；默认外表面不动、壁往内长；`open_faces` 给开口面 |

```python
shell(cube, 2.0)                                         # 全封闭空腔
shell(cup, 2.0, open_faces=faces_by_normal(cup, "z", 1)) # 顶面开口
shell(solid, 2.0, outward=True)                          # 原表面当内壁、壁往外长
```

⚠️ SpaceClaim 命令本身的语义是"**原表面变内壁、壁往外长**"（20mm 立方体传 `+2` 得到 24³），`shell()` 默认帮你取了负号。实测 20³ 立方体 t=2 全封闭 → 12 面、总面积 3936.00 = 6×400 + 6×256（16³ 空腔）；开口顶面 → 11 面 3552.00。

**阵列 / 镜像**（管束、针翅、叶片排）

| 函数 | 说明 |
|---|---|
| `array_linear(body, count, pitch, axis, count2, pitch2, axis2, name)` | 线性 / 二维阵列，返回所有实例 |
| `array_circular(body, count, axis, center, angle_deg, name)` | 圆周阵列（整圈或指定张角） |
| `mirror(body, plane_face, merge=True, name)` | 按一张**已存在的平面面**镜像 |

```python
array_linear(seed, 4, 20.0, axis="x", name="Pin")                  # 4 个，间距 20
array_linear(seed, 3, 20.0, axis="x", count2=2, pitch2=25.0)       # 3x2
array_circular(blade, 6, axis="z", center=(0, 0, 0), name="Blade") # 整圈 6 个
mirror(half, faces_by_normal(half, "x", -1)[0], merge=True)        # 半模型补成整模型
```

⚠️ 没用官方的 `Pattern.CreateLinear` —— 实测它会把体搬进 **component**（`Bodies.Count` 变 0、`Components.Count` 变 1），下游的命名与校验就全看不见体了。这里用"`Copy.Execute` + 平移/旋转"实现，实例始终留在根零件下。另外 **cutter 与壁面相切时 `cut=True` 会静默失效**（实测 3×3 管束只挖出 4 根）。

**曲面 + 加厚 / 装配**

| 函数 | 说明 |
|---|---|
| `rect_surface(w, h, origin, normal, name)` / `circle_surface(r, center, normal, name)` | 零厚度的面体（挡板、薄板的基础） |
| `thicken(target, value, direction, symmetric=False)` | 把面体加厚成实体，或把实体的面"拉"出来 |
| `component(name)` / `move_to_component` / `move_to_root` / `component_bodies` / `all_bodies` | 组件（装配）的建、搬、查 |
| `explode_to_components(bodies)` / `drop_empty_components()` / `assembly_summary()` | 一网打尽 / 删空组件 / 看结构 |

```python
plate = rect_surface(30.0, 20.0, normal="y", name="Baffle")
plate = thicken(plate, 1.0, direction="y")            # 必须接返回值：加厚会替换掉原面体
asm = component("Assembly1"); move_to_component(body, asm)
move_to_root(component_bodies(asm)); drop_empty_components()
```

⚠️ 三条实测坑：`symmetric=True` 的**总厚是 value 的两倍**；给曲面加厚会**替换**掉那个面体（旧对象失效，必须用返回值）；**体一进组件 `GetRootPart().Bodies` 就看不到它了**，而且搬空后留下的空组件会让 `NamedSelection.GetGroups()` 抛中文空引用 —— 所以 `move_to_root` 之后一定要 `drop_empty_components()`。`Midsurface.Convert` 实测是空操作，中面提取用不了。

**命名边界**（挑面 + 命名，规则按顺序、先匹配先占）

| 函数 | 说明 |
|---|---|
| `name_faces(名字, 面组)` | 底层：任意 ASCII 名字 |
| `name_faces_by_rules(body, [(名字, 规则), …])` | 主入口，规则见下表 |
| `name_boundaries(body, bottom, top, sides, axis, split_sides)` | 沿某轴一端 inlet 一端 outlet 其余 wall 的快速写法 |
| `name_interfaces(a, b, prefix)` / `name_internal_baffle(body, axis, v, name)` / `name_face_pair` | 交界面 / 内部挡板 / 周期面成对命名 |
| `faces_where / faces_at / faces_between / faces_by_normal / faces_by_kind / faces_by_area / faces_in_box / face_at_point / nearest_face / match_faces` | 挑面 |
| `face_center / face_extent / face_area / face_normal / face_kind / body_extent / body_size` | 测量（mm / mm²） |

**自然语言的边界条件描述 → 规则**（`name_faces_by_rules`）

| 你是这么说的 | 规则 |
|---|---|
| 入口 / 出口（沿 X） | `{"normal":"x","sign":-1}` / `{"normal":"x","sign":1}` |
| 顶面 / 底面 | `{"normal":"z","sign":1}` / `{"normal":"z","sign":-1}` |
| 某一侧、某个坐标上的面 | `{"normal":"y",…}` 或 `{"at":("y",0.0)}` |
| z 在 50~150 那一段 | `{"between":("z",50,150)}` |
| 某个角、某一块区域 | `{"in_box":(0,50,0,20,None,None)}` |
| 包含点 (10,0,5) / 离它最近的面 | `{"point":(10,0,5)}` / `{"nearest":(10,0,5)}` |
| 管子内壁、圆柱面、锥面、球面、平面端面 | `{"kind":"cylinder"}` / `"cone"` / `"sphere"` / `"plane"` |
| 中间带洞的面 / 被"盖章"的补丁 | `{"loops":2}` / `{"loops":1}` |
| 对称面 | `{"at":("y",0.0)}` 或 `{"normal":"y","sign":-1}` |
| 剩下的都算壁面 | `{"rest":True}`（放最后） |

规则是**整面判定**，不会自己切面——但可以**先切再选**：`split_face_by_line` / `split_face_by_points` 切面、`split_body_by_plane` 切体、`split_face_by_body` 在王面上"盖章"。这样"底面一半恒定热流、一半绝热"这类分区壁面条件也能表达。

命名选择传到 Workbench / Mechanical / Fluent Meshing 里会直接成为边界分区，**名称需为 ASCII**；边界条件的**类型**（velocity-inlet / pressure-outlet / symmetry / periodic / fan / porous-jump…）是在下游赋给同名分区的，SpaceClaim 里只决定"哪些面叫什么名字"。

## 独立校验

`-Verify` 会**另开一个全新会话、从磁盘重新打开产物**，回读包围盒尺寸、面/边统计，以及每个命名选择的面心坐标与面积。**建模脚本自己的输出不算证据**——这是本项目与"随便写个脚本跑一下"的区别。

## 回归测试

```powershell
$S = "<repo>"
foreach ($c in "box","cylinder","channel_4x4x10","solids","boundaries","split","rotate",
               "pairs","polygon","profile","revolve","sphere","imprint","external_flow","fillet","elbow","shell","array","surface_asm") {
  & "$S\scripts\Invoke-Scdm.ps1" -Script "$S\tests\selftest_$c.py" -Out "$S\tests\selftest_$c.scdocx" -Verify
}
```

19 个用例都必须以 `[scdm] status=ok` 结束。基线值（都来自真实运行，漂移即说明流水线坏了）：

| 用例 | 回读尺寸 / 拓扑 | 命名选择 |
|---|---|---|
| `selftest_box` | `6×5×20 mm` | inlet/outlet 各 30.00 mm² · wall 4 面 |
| `selftest_cylinder` | `5×5×30 mm` | inlet/outlet 各 19.63 mm² · wall_1 471.24 mm² |
| `selftest_channel_4x4x10` | `4×4×10 mm` | inlet/outlet 各 16.00 mm² · wall 4×40.00 mm² |
| `selftest_solids` | 4 个体：`Plate 20x20x4`(7 面) · `Pipe 30x12x12`(4 面) · `Cube 5³` · `Sep 6³` | inlet/outlet 各 62.83 mm² · wall 含内壁（1130.97 + 753.98 mm²） |
| `selftest_boundaries` | 2 个体 | 7 组：inlet/outlet 各 1600 mm² · symmetry 4000 mm² · wall 3×4000 · 管口 62.83 · 管壁 1130.97+753.98 |
| `selftest_split` | 3 个体：`Channel`(7 面) + 被切开的 `SplitMe`/`SplitMe1` | heated_wall 2000 mm² @ (25,20,0) · wall 4 面 |
| `selftest_rotate` | `Bar 35.355³`、`Tilted 20×27.32×27.32` | inlet/outlet 各 100.00 mm² · wall 4×400.00 |
| `selftest_pairs` | 4 个体（`Bar` 在 x=50 被切开） | interface_a/b 各 1600 mm² · baffle_a/b 各 1600 mm² |
| `selftest_polygon` | `HexDuct 10×8.66×20`(8 面) · `OctDuct 15×12×12`(10 面) | 各 64.95 / 101.82 mm² 端面 + 6 / 8 面侧壁 |
| `selftest_profile` | `TrapDuct`(6 面) · `EllipDuct`(3 面) | 端面 150.00 / 157.08 mm² |
| `selftest_revolve` | `Frustum 16×16×20`(3 面) · `Cone 20×16×16`(2 面) | fru_inlet 201.06 / fru_outlet 50.27 / fru_wall 768.91 · cone_inlet 201.06 / cone_wall 541.38 mm² |
| `selftest_sphere` | `Ball 10³`(1 面) · `Cavity 20³`(7 面，含球腔) | ball_surface 314.16 · cavity_wall 452.39 mm² |
| `selftest_imprint` | `Plate 40×40×10`(切分后 7 面) + `Cutter` | patch 78.54 mm² loops=1 · rest 1521.46 mm² loops=2 |
| `selftest_external_flow` | `Domain 60×40×40`(7 面，圆柱障碍物被吃成空腔) | inlet/outlet 1600 · obstacle 1256.64 · 顶底壁 2321.46 · 侧壁 2×2400 mm² |
| `selftest_fillet` | `RoundCube 20³`(26 面/48 边) · `RoundCubeZ`(10) · `ChamferCube`(26) · `RoundTube`(8) · `RoundedDuct`(10) · `RuleBox`(26) · `ChamferTwo`(10) · `FaceRound`(10) 等 10 个体 | cu_inlet 256.00(=16²) · cu_edge_fillet 12×50.27 · cu_corner 8×6.28 · cz_inlet 392.27 · duct_inlet 15.14 |
| `selftest_elbow` | 3 个体：`Bend 44×44×6.009`(7 面) · `Elbow45`(3) · `UTurn`(3) | Bend: inlet/outlet 各 28.27 · bend_wall 573.89 · wall 4 面；Elbow45: 端面各 12.57 · 环面 148.04；UTurn: 端面各 12.57 · 环面 473.74 |
| `selftest_shell` | 7 个体：`HollowCube 20³`(12 面) · `OpenCup`(11) · `OpenDuct`(10) · `HollowCyl`(6) · `Outward 24³`(12) · `TooThick`(6，t=11 被拒且未改动) · `PreNamed`(11) | hollow_outer 6×400.00 · hollow_cavity 6×256.00 · cup_rim 144.00 · duct_inner 4×320.00 · pre_outlet 被重映射成 144.00 |
| `selftest_surface_asm` | 7 个体：`RectSurf 20×10×2` · `CircSurf 20×20×0`(**1 面**) · `SymSurf 40×40×8` · `PullMe 20×20×25` · `Baffle 30×1×20` · `CompA`/`CompB` | 加厚替换掉原面体、symmetric 总厚翻倍；装配搬进搬出根零件数 7→6→7 |
| `selftest_array` | 22 个体：`Pin_1..4` · `Fin_1..6` · `Blade_1..6`（各 6 面） · `Half`(合并后 20×10×10) · `Half2`+`Half2Mirror` · `BankDomain`(**15 面**) | inlet/outlet 各 1200.00 · bank_tubes **9×565.49** · 上下壁各 2145.53 = 2400 − 9π·3² |

`selftest_fillet` 里的数字都对着手算核过：20mm 立方体全倒圆 r=2 的总面积 2189.451 mm² = `6×256 + 12×(π·2/2)·16 + 8×(4π·2²/8)`。

## 已知限制（诚实清单）

- **草图类几何必须最先建**。`polygon_prism` / `profile_prisms` / `extrude_circle` / `revolve_*` 都走 SpaceClaim 的草图工具，而**文档里一旦有实体，新建草图就会崩**（`SketchPolygon.Create` 抛空引用，宿主日志里只有一句空的 `Script failed:`）。批量接口会先画完所有轮廓再切 Solid 模式，所以它们只能出现在脚本开头。
- **`Sweep`（扫掠）没打通，但弯管已经能做了**。`Sweep.Execute` 会在**什么都不生成**的情况下返回 `Success=True`（实测折线路径、平面内圆弧路径、参数开关两个取值都试过）。**弯头/弯管请改用 `elbow()` / `torus()`**（草图圆 + 回转 = 真圆截面圆环段），不需要 Sweep。
- **`Loft` / `ExtrudeProfile` 用不了**（`references/api-notes.md` §7.3）。
- **`FullRound` 没生效**：`FullRound.Execute(面选择, None)` 返回 `Success=True` 但面数不变。
- **没有封装**：中面提取（`Midsurface.Convert` 实测是空操作）、多实体装配的层级展开（目前只有根零件 + 一层组件）。
- **不做网格与求解**：本项目只产几何和命名分区。
- 曲面测量的面心在**平面内**有小幅采样偏差；沿法向的坐标是精确的，按轴分类不受影响。
- 倒圆角后**平面面会内缩成 `(边长 − 2r)²` 的方块**（相切处不生成边），按面积写规则时要按这个数来，别用"圆角矩形"公式。

## 目录结构

```
├── SKILL.md                    流程、踩坑清单、回归基线（agent 读的说明书）
├── scripts/
│   ├── Invoke-Scdm.ps1         运行器（唯一入口，纯 ASCII）
│   ├── scdm_lib.py             建模辅助库（注入到模型脚本前面）
│   ├── verify_model.py         独立校验脚本
│   └── template_model.py       模型脚本模板
├── references/
│   └── api-notes.md            反射验证过的 API 签名、命令行参数表、走不通的路
└── tests/                      19 个回归用例
```

`references/api-notes.md` 记录了大量**负面结论**（哪些调用会失败、失败报什么错、错误信息是什么语言），价值不比正面文档低。

## 相关项目

想把它当 **DSH 预设**一键装（技能 + 原生 `scdm_build` 工具）：见 [dsh-preset-spaceclaim](https://github.com/fusion-whale/dsh-preset-spaceclaim)。

## License

[MIT](LICENSE) © 2026 fusion-whale

可自由使用、修改、再发布（含商用与闭源），保留版权声明即可。
