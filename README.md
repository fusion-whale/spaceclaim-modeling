# spaceclaim-modeling

**把 Ansys SpaceClaim 变成命令行里可复现的建模工具**：用几行 Python 建几何、给边界命名（`inlet` / `outlet` / `wall`），后台跑出带命名分区的 `.scdocx`，并自动做**独立校验**。

> Drive Ansys SpaceClaim headlessly from the command line: build geometry and name boundary zones from a short Python script, then verify the saved `.scdocx` by re-reading it in a fresh session. Docs are in Chinese.

```python
body = box(4.0, 4.0, 10.0, origin=(0, 0, 0), name="Channel")
name_boundaries(body, bottom="inlet", top="outlet", sides="wall", axis="z")
finish(r"channel.scdocx", body)
```

```powershell
& .\scripts\Invoke-Scdm.ps1 -Script .\channel.py -Out .\channel.scdocx -Verify
```

```
[scdm] artifact_size=25979 bytes
--- verify (fresh session, re-read from disk) ---
[verify] body[0] size = 4.000 x 4.000 x 10.000 mm
[verify]   inlet      center=(2.000, 2.000, 0.000) mm  area=16.00 mm2
[verify]   outlet     center=(2.000, 2.000, 10.000) mm  area=16.00 mm2
[verify]   wall       center=(2.000, 0.000, 5.000) mm  area=40.00 mm2
[scdm] status=ok
```

## 为什么有这个项目

SpaceClaim 的脚本 API 本身能用，但**"从命令行把它跑起来"这条路几乎没有一处文档写全**。以下每一条都是实测踩出来的，任何一条错了脚本都会**静默不执行**或**直接中止**：

- `/RunScript` **只认 `.py`**；传 `.scscript` 只会打开界面、根本不执行
- 参数必须带引号**经 `cmd.exe` 传递**；PowerShell 直接调用会丢引号，开关被忽略
- `/Headless=True` 下没有窗口，脚本报错**只能**从 `/ScriptOutput` 日志看
- **退出码 0 不等于成功**（脚本抛异常也是 0），必须用哨兵自己判定
- 脚本在打印第一行之前就失败时，`/ScriptOutput` 文件是**空的**——真正的错误在 `%APPDATA%\SpaceClaim\Log Files\*.log` 里的 `Script failed:` 行
- `NamedSelection.Create` **不接受名字参数**，而且默认名是本地化的（中文界面下不是 ASCII）
- 面的几何信息必须走 `face.Shape`，直接用 `face.Edges` 会报 `'DesignEdge' object has no attribute 'StartPoint'`
- **新建的体与已有体重叠会被自动并集**（实测：板上横放一个方块，最后只剩 1 个体）
- `CylinderBody.Create` 的三个点顺序与直觉相反，写反会得到一个轴向和半径都错的圆柱

这个仓库把上述路径封装成一个可以直接调用的工具，并把结论沉淀成文档，省掉后来人的试错。

## 环境要求

| 项 | 说明 |
|---|---|
| 操作系统 | Windows |
| SpaceClaim | 已安装即可，脚本会自动探测。开发与验证基于 **2022 R1（`v221`）** |
| PowerShell | 5.1 及以上（运行器是 5.1 兼容的，且刻意只用 ASCII 编写） |
| 权限 | 调用进程需能写入 `%APPDATA%\SpaceClaim\`（SpaceClaim 启动要写日志、日志文件与用户配置，并读取许可）。在受限沙箱里请放宽文件权限，否则 SpaceClaim 会**静默退出、什么都不做** |

## 安装

```bash
# 方式一：作为 DSH（DeepSeek Harness）技能安装，之后 agent 会自动发现并调用
git clone https://github.com/fusion-whale/spaceclaim-modeling.git \
          ~/.dsh/skills/spaceclaim-modeling

# 方式二：当普通命令行工具用，clone 到任意位置即可
git clone https://github.com/fusion-whale/spaceclaim-modeling.git
```

## 用法

1. 复制 `scripts/template_model.py`，改成你的模型；
2. 用运行器执行：

```powershell
& .\scripts\Invoke-Scdm.ps1 -Script .\my_model.py -Out .\my_model.scdocx -Verify
```

运行器会：拼装脚本 → 按上面那条实测过的命令行调用 SpaceClaim → 用**哨兵** `<<<SCDM_OK>>>` 判定成功 → 检查产物 → 可选地开新会话回读校验。

常用参数：`-Verify`（独立校验，强烈建议）、`-Gui`（弹界面看过程）、`-TimeoutSec`、`-SpaceClaimExe`。

> `-Out` 必须和脚本里 `finish(路径)` 保存的路径**完全一致**，否则会被报成 `artifact is missing`（即使建模本身成功了）。

## 已封装的能力

建模脚本里直接调用即可（运行器会把 `scdm_lib.py` 注入到脚本最前面，**不需要 import**）：

| 函数 | 说明 | 实测结果 |
|---|---|---|
| `box(w, d, h, origin, name, cut, separate)` | 长方体；`cut=True` 布尔减、`separate=True` 重叠也保持独立 | 6×5×20 mm，6 面 |
| `cylinder(r, h, origin, axis, name, cut, separate)` | 圆柱，轴向 `"x"/"y"/"z"` | r2.5×30，3 面 |
| `tube(外r, 内r, h, origin, axis, name)` | 空心圆管 | 30×12×12，4 面 |
| `stepped_cone(r1, r2, h, segments)` | 阶梯锥（真锥台的近似，见「已知限制」） | 5 段 → 1 体 11 面 |
| `extrude_circle(r, h, center2d)` | 草图圆拉伸 | 沿默认草图平面法向（Y） |
| `move(body, dx, dy, dz)` | 整体平移 | 位移精确 |
| `name_boundaries(body, bottom, top, sides, axis, split_sides)` | 按位置自动命名边界 | inlet/outlet/wall |
| `faces_where / faces_at / faces_between` | 按条件挑面 | 支持 lambda 谓词 |
| `face_center / face_area / body_extent / body_size` | 测量（mm / mm²） | 曲面用 `GetPolyline` 采点，闭合圆边的端点会退化成圆心 |

`name_boundaries` 是最常用的一个：沿指定轴，坐标小的一端 → `bottom`、大的一端 → `top`、其余面 → `sides`（`split_sides=True` 则拆成 `wall_1..wall_n`）。它只看几何位置、不看面编号，所以**改了尺寸也不会错位**。命名选择传到 Workbench / Mechanical / Fluent Meshing 里会直接成为边界分区，名称需为 ASCII。

## 独立校验

`-Verify` 会**另开一个全新会话、从磁盘重新打开产物**，回读包围盒尺寸和每个命名选择的面心坐标与面积。**建模脚本自己的输出不算证据**——这是本项目与"随便写个脚本跑一下"的区别。

## 回归测试

```powershell
$S = "<repo>"; foreach ($c in "box","cylinder","channel_4x4x10","solids") {
  & "$S\scripts\Invoke-Scdm.ps1" -Script "$S\tests\selftest_$c.py" -Out "$S\tests\selftest_$c.scdocx" -Verify
}
```

四个用例都必须以 `[scdm] status=ok` 结束。基线值（都来自真实运行，漂移即说明流水线坏了）：

| 用例 | 回读尺寸 | 命名选择 |
|---|---|---|
| `selftest_box` | `6.000 x 5.000 x 20.000 mm` | inlet 30.00 mm² @ z=0 · outlet 30.00 mm² @ z=20 · wall 4 面 |
| `selftest_cylinder` | `5.000 x 5.000 x 30.000 mm` | inlet/outlet 各 19.63 mm² · wall_1 471.24 mm² |
| `selftest_channel_4x4x10` | `4.000 x 4.000 x 10.000 mm` | inlet/outlet 各 16.00 mm² · wall 4 面 × 40.00 mm² |
| `selftest_solids` | 4 个体：`Plate 20x20x4`(7 面) · `Pipe 30x12x12`(4 面) · `Cube 5³` · `Sep 6³` | inlet/outlet 各 62.83 mm² · wall 含内壁（1130.97 + 753.98 mm²） |

## 已知限制（诚实清单）

- **真圆锥台 / 放样做不出来**。`Loft.Create` 对原始曲线、`ConvertToCurves()` 结果、圆盘面三种选择一律报 *"must include Bodies, Faces, Edges, Curves, or Points selection"*；草图曲线在切 Solid 模式后会失效；`Geometry.Profile` 没有公开构造入口（只与钣金成形绑定），`ExtrudeProfile` 也用不了。现只能用 `stepped_cone` 近似，或在 GUI 里手工建。
- **任意草图轮廓不可靠**：只有单个圆的草图拉伸稳定；矩形/多边形草图不形成可拉伸的面（矩形实心体请直接用 `box`）。草图平面也换不了（新文档里 `DatumPlanes.Count == 0`）。
- **没有封装**：圆角、倒角、抽壳、曲面、装配、阵列、旋转/镜像。
- **不做网格与求解**：本项目只产几何和命名分区。
- 曲面测量的面心在**平面内**有小幅采样偏差（实测 r=2.5 圆的盖面心偏 0.06 mm）；沿法向的坐标是精确的，按轴分类不受影响。

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
└── tests/                      4 个回归用例
```

`references/api-notes.md` 记录了大量**负面结论**（哪些调用会失败、失败报什么错），价值不比正面文档低。

## License

未指定。如需开源授权请自行添加 `LICENSE`。
