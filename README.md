# DP-GEN_fep

面向 DP-GEN + DeepMD + LAMMPS 的自由能计算示例仓库，覆盖从双态数据驱动势训练到 TI（Thermodynamic Integration）积分采样的完整实践流程。

![FEP workflow](./fep.png)

## 项目优点

- 面向实际任务：将 `DP-GEN` 迭代训练与 `FEP/TI` 采样拆分为两个清晰目录，便于按阶段执行。
- 可追溯配置：核心参数集中在 `param.json`、`machine.json`、`resource.json` 与 CP2K 输入模板中。
- 含示例数据与结果：仓库内提供 `data/` 与 `ti-out/`，可直接对照输入输出结构。
- 适配远程调度：脚本内已集成 `dpdispatcher`/Bohrium(Lebesgue) 提交逻辑。

## 主要功能

- 使用 `DP-GEN_fep/run_fep.py` 启动 DP-GEN 迭代流程（训练、模型偏差、第一性原理标注、数据汇总）。
- 支持双态输入（`neu`/`red`）的外部 CP2K 单点计算模板。
- 在 LAMMPS 中通过双 DeepMD 势叠加与 `fix adapt` 完成 FEP 混合采样。
- 自动根据阈值追加 TI 插点（`FEP_TI/fep_ti.py` + `FEP_TI/fc.py`）。
- 保留多组 TI 样例输出目录（`ti-out/*`）用于结果检查。

## 技术栈与架构概览

- 语言与脚本：Python
- 核心库：`dpgen`、`dpdata`、`dpdispatcher`、`numpy`、`scipy`、`pandas`、`pymatgen`
- 计算引擎：DeepMD-kit、LAMMPS、CP2K
- 调度平台：Bohrium / Lebesgue（通过 `machine.json`、`resource.json`）

仓库结构（关键部分）：

```text
.
├── DP-GEN_fep/
│   ├── run_fep.py          # DP-GEN 工作流入口
│   ├── lammps.py           # FEP LAMMPS 输入生成
│   ├── param.json          # DP-GEN 参数
│   ├── machine.json        # 训练/采样/标注机器配置
│   ├── input_neu.inp       # CP2K 输入模板（neutral）
│   ├── input_red.inp       # CP2K 输入模板（redox）
│   └── data/               # 初始结构与训练数据
├── FEP_TI/
│   ├── fep_ti.py           # TI 任务自动插点与提交入口
│   ├── fc.py               # GAP 分析与 LAMMPS 输入生成
│   ├── machine.json        # dpdispatcher 机器配置
│   └── resource.json       # dpdispatcher 资源配置
└── ti-out/                 # TI 示例输出
```

## 快速开始

> 说明：仓库当前没有 `requirements.txt`/`pyproject.toml`。下面安装命令依据现有 Python 脚本 `import` 生成。

### 1) 获取代码

```bash
git clone https://github.com/LJS1124/DP-GEN_fep.git
cd DP-GEN_fep
```

### 2) 安装依赖（Python）

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install numpy scipy pandas dpdata dpgen dpdispatcher pymatgen packaging
```

### 3) 运行 DP-GEN_fep 工作流

```bash
python DP-GEN_fep/run_fep.py DP-GEN_fep/param.json DP-GEN_fep/machine.json
```

### 4) 运行 FEP_TI 插点采样

先按你的工作目录修改 `FEP_TI/fep_ti.py` 中的 `work_path`，并确保该目录下已有：

- `machine.json`
- `resource.json`
- `1.data`
- `neu.pb`
- `red.pb`

然后执行：

```bash
python FEP_TI/fep_ti.py
```

## 配置说明

### DP-GEN_fep

- `DP-GEN_fep/param.json`
- 关键字段：
- `type_map`、`mass_map`：元素类型与质量映射
- `init_data_prefix`、`init_neu_data_sys`、`init_red_data_sys`：初始数据路径
- `neu_external_input_path`、`red_external_input_path`：CP2K 输入模板路径
- `model_devi_jobs`：采样温度/压力/步数/系综

- `DP-GEN_fep/machine.json`
- 包含 `train` / `model_devi` / `fp` 三段任务命令与资源配置
- 需填写可用的 Bohrium/Lebesgue 账户信息与项目 ID

### FEP_TI

- `FEP_TI/machine.json`：`dpdispatcher.Machine` 配置
- `FEP_TI/resource.json`：`dpdispatcher.Resources` 配置
- `FEP_TI/fep_ti.py`：
- `work_path`：TI 任务根目录
- `threshold`：插点阈值
- `init_data`：初始结构文件名（默认 `1.data`）

### 环境变量

- 当前代码未定义必须的 `.env` 环境变量。
- 若你在本地或 CI 管理账号密码，建议通过环境变量注入并在脚本/JSON 中读取，避免明文提交。

## 常见问题与排错

- 运行 `run_fep.py` 报 `ModuleNotFoundError`：确认虚拟环境已激活，且依赖已安装。
- 远程任务提交失败：检查 `machine.json` / `resource.json` 中的账号、项目、资源规格是否有效。
- `fep_ti.py` 找不到 `*.pb` 或 `1.data`：确认这些文件位于 `work_path` 指向目录。
- `lmp: command not found`：确保 LAMMPS 可执行文件 `lmp` 在 PATH 中，且支持 DeepMD。
- 插点不收敛：可尝试降低 `threshold` 或提高每个窗口 `nsteps`（在 `fep_ti.py` 生成输入参数处调整）。

## Roadmap（建议）

- 增加 `requirements.txt` 或 `pyproject.toml`，固定依赖版本。
- 将 `FEP_TI/fep_ti.py` 的硬编码参数改为命令行参数。
- 补充自动化结果汇总脚本（自由能积分、误差条、图表）。
- 增加最小可运行测试数据与 CI 检查。

---

## 📄 License

Distributed under the MIT License. See `LICENSE` for more information.

---

## ⭐ Support

If you find this project useful, please consider giving it a star ⭐ on GitHub.

## English Summary

`DP-GEN_fep` is a practical workflow repo for free-energy calculations using DP-GEN, DeepMD, LAMMPS, and CP2K. It includes two stages: (1) DP-GEN-based dual-state data/training workflow in `DP-GEN_fep/`, and (2) TI window generation and adaptive refinement in `FEP_TI/`.
