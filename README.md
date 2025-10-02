# Future-Guided Learning 复现笔记

本仓库整理自 [Future-Guided Learning](https://github.com/SkyeGunasekaran/FutureGuidedLearning) 原始实现，聚焦癫痫检测/预测与马基-格拉斯时间序列实验。为了便于本地分析与复现，这里保留了原有脚本，并补充了运行指引与架构说明。

## 目录速览
- `src/AES/`：Kaggle 2014 癫痫预测挑战的教师/学生训练脚本与数据预处理工具。【F:src/AES/exp/create_teacher.py†L1-L112】【F:src/AES/exp/FGL_AES.py†L1-L200】
- `src/CHBMIT/`：CHB-MIT 数据集的蒸馏流程与模型定义。【F:src/CHBMIT/exp/FGL_CHBMIT.py†L1-L196】
- `src/mackey_glass/`：马基-格拉斯长时预测实验及辅助函数。【F:src/mackey_glass/exp/base_exp.py†L1-L205】
- `docs/`：补充文档，包括架构总览与复现流程。

更多背景介绍与模块关系图示，请参阅《[Future-Guided Learning 架构总览](docs/architecture-overview.md)》。

## 快速开始
1. **创建环境**：根据 `src/requirements.txt` 安装依赖，可使用 `conda` 或 `pip`。若需 GPU 训练，请确保 CUDA 驱动与 PyTorch 版本匹配。
2. **准备数据**：按照《[复现流程指南](docs/reproduction-guide.md)》下载原始数据、设置 `student_settings.json`/`teacher_settings.json` 中的路径，并预生成缓存。
3. **运行实验**：
   - 先执行 AES 教师模型脚本 `python src/AES/exp/create_teacher.py --subject Dog --epochs 50` 以训练通用检测器。【F:src/AES/exp/create_teacher.py†L76-L112】
   - 再调用学生蒸馏脚本 `python src/AES/exp/FGL_AES.py --subject Dog_1 --alpha 0.5`，或参照文档运行 CHB-MIT、Mackey-Glass 实验。【F:src/AES/exp/FGL_AES.py†L136-L200】【F:src/CHBMIT/exp/FGL_CHBMIT.py†L123-L196】【F:src/mackey_glass/exp/base_exp.py†L136-L205】

## 文档索引
- [Future-Guided Learning 架构总览](docs/architecture-overview.md)：模块拆解与流程图解。
- [复现流程指南](docs/reproduction-guide.md)：数据下载、预处理及各实验运行命令。

若遇到缺失模型或缓存文件，请先确认对应脚本是否成功运行并在配置文件中指向正确目录。
