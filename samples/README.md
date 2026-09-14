# samples/ · 8 个已校验的成品模型

这些 `.scdocx` 是 `examples/` 里那些脚本**实际跑出来的产物**（SpaceClaim 2022 R1）。
每个都做过独立校验：另开一个 SpaceClaim 会话、从磁盘重新打开，回读尺寸、面/边统计、
以及每个命名选择的面心与面积，数字和脚本里打印的手算式逐位一致。

**不需要装脚本、也不需要跑任何东西** —— 直接用 SpaceClaim 打开就能看。

| 文件 | 是什么 | 打开后建议先看 |
|---|---|---|
| `tube_bank_demo.scdocx` | 管壳式换热器壳程：100×50×40 壳 + 12 根贯穿换热管 + 2 块弓形折流板 | 截面视图看 12 个管孔；结构树里点 `tubes` 高亮 24 张管壁面 |
| `pin_fin_demo.scdocx` | 针翅散热器流道：120×40×30 + 10×4 根 r2 高 20 的针翅 | 俯视看 40 根针阵；点 `fins` 高亮 40 张针壁 |
| `surface_asm_demo.scdocx` | 曲面加厚（40×30 面 → 3mm 板）+ 装配组件搬进搬出 | 结构树里三个体；看 `baffle_up` / `baffle_down` |
| `bend90_demo.scdocx` | 90° 弯管流域：真圆截面弯头 + 两段直管拼成一个体 | `bend_wall` 是一张解析环面（torus） |
| `cht_tube_bundle_demo.scdocx` | **三层共轭传热**：壳程流体 + 12 根管壁固体 + 管程流体（25 个体） | 点 `shell_tube_a` 再点 `shell_tube_b`，两次高亮位置**完全重合**（那就是 Fluent 里要配对的那对 interface） |
| `cht_baffled_demo.scdocx` | **四层共轭传热**：再加 2 块折流板固体，而且折流板带管孔（27 个体） | `baffle_tube_a/b` 是折流板上被管子穿过的孔 |
| `fuel_assembly_5x5.scdocx` | **5×5 核反应堆燃料组件流动通道**：75×75 通道挖 25 根棒（棒径 10 / 栅距 15），长 3000mm | 转 +Z 视图看 5×5 孔阵；`wall-heated` 是 25 张棒表面 |
| `pche_demo.scdocx` | **PCHE 一对冷热通道**：D 形截面、平边朝 +z、上下对齐、长 100mm | 截面看两个 D 叠在一起，中间 0.6mm 薄壁 |

## 怎么用它们

```powershell
# 直接打开
& "C:\Program Files\ANSYS Inc\v221\scdm\SpaceClaim.exe" .\samples\pche_demo.scdocx
```

或者用脚本里的运行器把对应示例重新建一遍（会覆盖同名文件）：

```powershell
& ..\scripts\smoke_examples.ps1                 # 8 个全跑一遍，打印对照表
& ..\scripts\Invoke-Scdm.ps1 -Script ..\examples\pche_demo.py -Out .\pche_demo.scdocx -Verify
```

## 看模型时最有用的一个操作

**点结构树里的命名选择**。这些模型的体名/分区名就是 CFD 里要用的 zone 名，
点一下就能高亮对应的面 —— 这是验证"边界条件命名对不对"最快的方式，
比数面数直观得多。

> 注意：文件里的**配色是 SpaceClaim 默认的**，脚本没做上色（API 里没找到稳定的上色入口）。
> 要让不同层次看起来分明，得在 SpaceClaim 里手动给体上色（右键体 → 属性 → 颜色）。
