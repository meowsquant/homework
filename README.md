# 量化交易入门 — 作业

本文件夹为「量化交易入门」课程的作业集合,使用 Python + Quarto 文档(.qmd)完成,涵盖数据模拟、行情获取、趋势识别与均线策略回测等内容。

## 作业内容

| 文件 | 主题 | 说明 |
| --- | --- | --- |
| [task1.qmd](task1.qmd) | 模拟数据涨跌盘 | 模拟三段行情(涨 → 跌 → 恢复),对比 MA20 量化策略、随机买卖与买入持有三种策略的净值曲线 |
| [task2.qmd](task2.qmd) | 股票数据获取 | 使用 akshare 获取美股日线数据,封装 `get_stock_data` 函数并演示数据加载流程 |
| [task3.qmd](task3.qmd) | 趋势模拟与识别 | 模拟上涨、下跌、横盘三种市场状态,直观展示趋势与噪声的区别 |
| [task4.qmd](task4.qmd) | 双均线策略回测 | 基于 AAPL 的 MA 双均线策略回测,与 SPY 基准对比收益表现 |
| [task1 positron配置.qmd](task1%20positron配置.qmd) | Positron 环境配置 | Task1 的 Positron 配置说明(PDF 输出) |

## 运行环境

- Python 3.x
- Quarto(用于渲染 .qmd 文档)
- 主要依赖:`numpy`、`pandas`、`matplotlib`、`akshare`

## 如何运行

1. 安装依赖:

   ```bash
   pip install numpy pandas matplotlib akshare
   ```

2. 渲染 Quarto 文档(以 task1 为例):

   ```bash
   quarto render task1.qmd
   ```

   或在 Positron / VS Code 中直接打开 `.qmd` 文件逐块运行。

## 备注

- 各 `*_files/` 目录与 `.html`、`.pdf`、`.docx` 文件为 Quarto 渲染产物,源文件以 `.qmd` 为准。
- `task*.quarto_ipynb*` 为 Quarto 运行时生成的中间 notebook 文件,可忽略。
