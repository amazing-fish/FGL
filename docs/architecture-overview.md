# Future-Guided Learning 架构总览

## 顶层结构
- **数据集子目录**：项目按照数据集划分为 `AES/`、`CHBMIT/` 与 `mackey_glass/`，每个目录下都包含 `exp/`（实验脚本）、`models/`（模型定义）与 `utils/`（数据预处理与缓存）等子模块，用于复现论文中的不同实验场景。【F:src/AES/exp/FGL_AES.py†L12-L200】【F:src/CHBMIT/exp/FGL_CHBMIT.py†L1-L163】【F:src/mackey_glass/exp/base_exp.py†L1-L197】
- **运行入口**：面向癫痫预测/检测的主要入口脚本位于 `AES/exp/` 与 `CHBMIT/exp/` 下，而马基-格拉斯时间序列实验的入口脚本位于 `mackey_glass/exp/` 中。【F:src/AES/exp/FGL_AES.py†L171-L199】【F:src/AES/exp/create_teacher.py†L104-L118】【F:src/CHBMIT/exp/FGL_CHBMIT.py†L126-L163】【F:src/mackey_glass/exp/base_exp.py†L1-L197】
- **配置文件**：各数据集目录提供 `student_settings.json` 与 `teacher_settings.json`，用于配置数据路径、缓存目录与结果存储位置，便于在不同机器上重现训练流程。【F:src/AES/student_settings.json†L1-L5】【F:src/AES/teacher_settings.json†L1-L5】【F:src/CHBMIT/student_settings.json†L1-L5】【F:src/CHBMIT/teacher_settings.json†L1-L5】

## AES（Kaggle 2014 癫痫预测挑战）流水线
1. **教师模型构建**：`create_teacher.py` 会根据命令行参数选择犬类或患者数据，整合多个受试者的 ictal/interictal 片段，利用 `CNN_LSTM_Model` 训练短期检测器，并将模型持久化到磁盘（如 `teacher_dog.pth`）。【F:src/AES/exp/create_teacher.py†L22-L101】【F:src/AES/utils/load_signals_teacher.py†L32-L227】
2. **学生模型蒸馏**：`FGL_AES.py` 负责加载教师模型，对目标受试者的学生数据执行知识蒸馏训练。蒸馏损失由交叉熵与经温度缩放后的 KL 散度组成，可通过 `alpha` 与 `temperature` 调节权重。【F:src/AES/exp/FGL_AES.py†L34-L168】
3. **数据准备**：`PrepDataStudent.apply()` 会对 ictal/interictal 数据进行缓存、STFT 转换、窗口切分与重采样等处理，并将样本分组后交给 `train_val_test_split_continual_s` 生成按发作顺序划分的训练/测试集合。【F:src/AES/utils/load_signals_student.py†L219-L234】【F:src/AES/utils/prep_data_student.py†L1-L60】
4. **模型结构**：学生与教师默认均使用 `CNN_LSTM_Model`，通过 3D 卷积提取频谱时序特征，再接入多层 LSTM 与全连接分类头，支持 2 类输出（预警 vs. 正常）。【F:src/AES/models/models.py†L6-L65】

## CHB-MIT 流水线
1. **教师模型来源**：`FGL_CHBMIT.py` 直接从 `pytorch_models/Patient_{id}_detection` 加载预训练检测器，并对选定患者执行多次蒸馏实验，输出 AUC 指标。【F:src/CHBMIT/exp/FGL_CHBMIT.py†L40-L122】
2. **学生训练循环**：脚本按 trial 重置学生模型，结合教师输出与真实标签计算交叉熵 + KL 损失，支持 SGD/Adam 优化器与温度、`alpha` 参数调节，并将结果追加写入日志文件。【F:src/CHBMIT/exp/FGL_CHBMIT.py†L61-L121】
3. **模型与数据处理**：CHB-MIT 数据的网络结构与 AES 保持一致，同样依赖 `CNN_LSTM_Model`，配套的 `utils/` 包负责按患者切分 EEG 样本和缓存处理结果，配置文件指定原始数据与缓存目录。【F:src/CHBMIT/models/model.py†L6-L127】【F:src/CHBMIT/student_settings.json†L1-L5】【F:src/CHBMIT/teacher_settings.json†L1-L5】

## Mackey-Glass 时间序列实验
1. **数据生成**：`utils.MackeyGlass` 类利用 JIT 编译的延迟微分方程求解器生成时间序列，自动拆分训练/测试窗口，并支持按窗口滑动构建监督数据集。【F:src/mackey_glass/utils/utils.py†L20-L199】
2. **模型与训练**：`base_exp.py` 通过 `create_time_series_dataset` 构造教师（短期）与学生（长期）预测任务，先分别训练教师与基线，再以相同架构的 RNN 为学生模型执行知识蒸馏，蒸馏损失使用 KL 散度并带有早停机制。【F:src/mackey_glass/exp/base_exp.py†L15-L197】【F:src/mackey_glass/utils/utils.py†L144-L199】
3. **评估流程**：训练脚本在早停后恢复最佳权重，计算教师、基线与学生的测试 MSE，从而量化 FGL 框架在长时预测中的收益。【F:src/mackey_glass/exp/base_exp.py†L118-L197】

## 运行建议
- 运行 AES 实验前，需要先调用 `create_teacher.py` 预先训练通用检测器，再使用 `FGL_AES.py` 对特定受试者执行蒸馏；命令行参数可配置受试者、优化器、轮次与蒸馏超参数。【F:src/AES/exp/create_teacher.py†L104-L118】【F:src/AES/exp/FGL_AES.py†L171-L199】
- CHB-MIT 与 Mackey-Glass 实验入口均通过命令行参数指定目标患者或预测视窗、`alpha`、温度等关键超参数，便于批量扫参与结果记录。【F:src/CHBMIT/exp/FGL_CHBMIT.py†L137-L163】【F:src/mackey_glass/exp/base_exp.py†L1-L197】
