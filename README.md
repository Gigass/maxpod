# maxpod

当前仓库保留的是 **iPod 4th gen click wheel 最小转接板** 的最终版产物。

目标：

- `click wheel FPC -> 转接板 -> ESP32-C3 开发板`
- 不带电池
- 不集成主控
- 不保留板载 `CFG1` 2.54mm 跳线头
- 保留右侧 `U1/U3` 通孔位，但默认 **不焊排针**
- 不保留右侧独立测试点列

原因：

- 板载 `U2` 三针跳线头会和 FPC 扇出区、机械插拔空间冲突
- 对这块最小 breakout，外部短接 `CFG1` 更合理

最终产物目录：

- [/Users/gigass/DEVELOP/GitHub/maxpod/artifacts/final_board2_1](/Users/gigass/DEVELOP/GitHub/maxpod/artifacts/final_board2_1)

包含：

- `Board2_1_final_Gerber.zip`
- `Board2_1_final_BOM.xlsx`
- `Board2_1_final_PnP.csv`
- `Board2_1_final_PCB.pdf`
- `Board2_1_final_Info.xlsx`

说明：

- 当前板框尺寸约 **26.16 mm × 22.35 mm**。
- `Board2_1_final_Info.xlsx` 实际是文本格式导出，不是标准 Excel 文件。这是 EasyEDA 导出行为，不影响制板文件。
- EasyEDA 仍可能显示 `Import Changes`。当前这份最终版以 **PCB 实际规则检查无真实几何/连线错误** 为准。
- 右侧 `U1/U3` 是保留的通孔引出位。当前推荐装配方式是 **裸板不贴排针**；需要调试时再自行焊接排针或直接飞线。
- 专用测试点列已经删除；需要测量时直接用 `U1/U3` 通孔位。
