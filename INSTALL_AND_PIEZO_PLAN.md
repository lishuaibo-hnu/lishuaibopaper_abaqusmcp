# Abaqus 插件安装与压电耦合仿真实施方案

> 目标：安装 `abaqus-mcp-server` 与 `text-to-cae` 两个插件，并在 Abaqus 中完成一个尽量贴近实际工况的压电耦合仿真。

## 1. 当前环境状态（2026-05-07）

我已在当前环境尝试直接克隆两个仓库：

- `https://github.com/jianzhichun/abaqus-mcp-server`
- `https://github.com/Cai-aa/text-to-cae`

但当前容器访问 GitHub 返回 `CONNECT tunnel failed, response 403`，因此**无法在这里直接联网安装**。你可以在本机（能访问 GitHub 的环境）执行下述命令完成安装。

---

## 2. 推荐安装方式（本机执行）

### 2.1 前置条件

- Abaqus（建议 2021+）
- Python 版本与 Abaqus 的 Python 兼容（常见是 3.8/3.9；部分 Abaqus 内置解释器版本较老）
- 能访问 GitHub

### 2.2 拉取插件

```bash
git clone https://github.com/jianzhichun/abaqus-mcp-server.git
git clone https://github.com/Cai-aa/text-to-cae.git
```

### 2.3 建立独立 Python 环境（建议）

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\\Scripts\\activate
python -m pip install --upgrade pip
```

### 2.4 安装依赖

分别查看两个仓库中的依赖文件（`requirements.txt` / `pyproject.toml`），按仓库说明安装：

```bash
cd abaqus-mcp-server
pip install -r requirements.txt  # 若存在
# 或 pip install -e .

cd ../text-to-cae
pip install -r requirements.txt  # 若存在
# 或 pip install -e .
```

> 若依赖冲突，优先固定 `numpy`、`pydantic`、`fastapi`、`langchain/openai` 等核心版本，并将两个项目放到同一个 venv 里统一解析。

### 2.5 Abaqus 侧接入

通常有两种路径：

1. **Abaqus 脚本调用模式**：将插件脚本路径加入 `PYTHONPATH`，从 Abaqus 的 `File -> Run Script` 调用。
2. **外部服务模式（MCP/HTTP）**：先启动 `abaqus-mcp-server`，再由 `text-to-cae` 或你的上层 agent 通过接口驱动建模与求解。

建议先做最小联通测试：

- 启动 server；
- 用一个最简单的“创建草图->拉伸->网格”请求验证；
- 最后再切到压电模型。

---

## 3. 压电耦合仿真：贴近实际工况的建模方案

下面给出可直接落地的“压电片 + 金属基底”结构（典型传感/致动器场景）。

## 3.1 场景定义（工程化）

- 结构：PZT 压电陶瓷片粘贴在铝合金梁表面（单贴片）
- 工况：
  - **致动模式**：施加电压，看梁挠度/应变输出
  - **传感模式**：施加机械载荷，看电极输出电荷/电压
- 目标输出：
  - 机电耦合位移响应
  - 电势分布
  - 电荷-力/电压-位移灵敏度

## 3.2 分析步建议

1. **Static, General（线性小变形）**：先跑通耦合与边界
2. **Frequency（可选）**：提取模态与电机耦合特性
3. **Steady-state dynamics（可选）**：做频响

> 第一次务必先静力，避免几何/材料/电边界错误叠加。

## 3.3 单元与材料

- 压电层：3D 实体压电单元（Abaqus 中选择支持 piezoelectric coupling 的连续体单元）
- 基底层：标准实体单元（弹性）
- 材料：
  - PZT：弹性矩阵 + 介电常数 + 压电常数（`d` 或 `e` 形式，注意单位一致）
  - 铝：E、ν、ρ

### 单位系统（强烈建议）

统一 SI：`m, kg, s, V, C, N, Pa`。

常见错误是把 mm-MPa 与 SI 混用，导致介电/压电常数数量级失真。

## 3.4 几何与网格

- 梁尺寸示例：`L=100 mm, b=15 mm, t=1.5 mm`
- 压电片：`20 mm x 10 mm x 0.3 mm`（贴在梁根部附近）
- 网格建议：
  - 压电片厚度方向至少 2~3 层单元
  - 贴片边缘与高梯度区局部加密

## 3.5 关键相互作用与约束

- 压电片与基底：
  - 理想粘接先用 **Tie**；
  - 若考虑胶层，后续可建 Cohesive 层。
- 力学边界：悬臂根部全约束（典型）
- 电边界：
  - 上电极：`Uelectrical = V0`（如 100V）
  - 下电极：接地 `0V`

## 3.6 载荷工况

### 致动工况

- 仅施加电压（100V 可作为初值）
- 观察自由端位移、贴片附近应力

### 传感工况

- 在梁端施加集中力（如 1N）或位移激励
- 输出电极总电荷/电势差

## 3.7 输出请求（避免漏项）

- Field: `U, S, E, PE, PHILSM`（具体电学变量名按 Abaqus 版本）
- History:
  - 梁端位移
  - 电极参考点电势/电荷
  - 反力/电流积分量

---

## 4. 结果校核（非常重要）

至少做 4 类 sanity check：

1. **量纲检查**：位移量级是否合理（通常微米~毫米，不应离谱）
2. **对称/边界检查**：变形方向与极化方向是否一致
3. **网格收敛**：加密后位移与电势变化 <5%
4. **解析对比**：与简化梁-压电理论趋势一致（电压↑，挠度↑）

---

## 5. 我建议你接下来这样做

1. 你在本机先完成两个仓库拉取与依赖安装；
2. 把两个项目中的以下文件贴给我：
   - `README`
   - 依赖文件
   - 启动命令说明
3. 我可以基于它们给你：
   - 一套**可直接执行**的启动脚本（Windows/Linux 各一份）
   - 一份 `text-to-cae` 提示词模板，自动生成压电模型
   - 一份 Abaqus 参数化脚本（尺寸、材料、载荷可一键改）

如果你愿意，我下一步可以直接给你“悬臂梁+PZT贴片”的 Abaqus 脚本骨架（含参数表与后处理点位定义）。
