# Board2_1 Final

这目录只保留 `Board2_1 / PCB2_1 / Schematic2_1` 的最终导出物。

板子定义：

- 最小 click wheel 转接板
- `P1 + R1 + R2 + U1 + U3`
- 不保留板载 `U2` 跳线头
- `CFG1` 通过外部短接处理
- `U1/U3` 只保留通孔引出位，最终默认 **不焊排针**
- 不保留右侧独立测试点列
- 当前板框尺寸约 **26.16 mm × 22.35 mm**

文件：

- `Board2_1_final_Gerber.zip`: 制板文件
- `Board2_1_final_BOM.xlsx`: 物料清单
- `Board2_1_final_PnP.csv`: 贴片坐标
- `Board2_1_final_PCB.pdf`: PCB 文档导出
- `Board2_1_final_Info.xlsx`: 板信息导出

注意：

- `Board2_1_final_Info.xlsx` 是文本导出，后缀名是 `.xlsx`，这不是仓库错误，是 EasyEDA 导出行为。
- `Board2_1_final_BOM.xlsx` 和 `Board2_1_final_PnP.csv` 是 EasyEDA 导出的完整工程结果；如果你要的是裸板方案，`U1/U3` 视为 **DNP/不装配** 即可。
- 专用测试点列已删除；调试和测量直接使用 `U1/U3` 通孔位。
