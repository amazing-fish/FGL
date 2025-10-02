# 复现流程指南

本文汇总 Future-Guided Learning 项目在 AES、CHB-MIT 与马基-格拉斯实验中的复现步骤，覆盖环境准备、数据预处理与脚本调用方式。

## 1. 环境准备
1. **Python 版本**：推荐 Python 3.9+，需配备 CUDA 11.x 以运行 GPU 训练。
2. **依赖安装**：在虚拟环境中执行：
   ```bash
   pip install -r src/requirements.txt
   ```
3. **额外工具**：
   - AES/CHB-MIT 流程依赖 `hickle`、`stft` 等包，已包含在 requirements 中。
   - 若需绘图，请额外安装 `matplotlib`（马基-格拉斯脚本可保存预测对比图）。

## 2. 数据准备
### 2.1 AES（Kaggle 2014）
1. 在 Kaggle 2014 Seizure Prediction Challenge 页面下载 `Dog_1`–`Dog_5`、`Patient_1`–`Patient_2` 的预测与检测数据集，并解压到 `src/AES/prediction_dataset/` 与 `src/AES/detection_dataset/`。
2. 根据本地路径调整 `student_settings.json` 与 `teacher_settings.json` 中的 `datadir`、`cachedir` 字段，例如：
   ```json
   {
       "dataset": "Kaggle2014",
       "datadir": "/data/kaggle2014/prediction_dataset",
       "cachedir": "/data/kaggle2014/cache/student"
   }
   ```
   配置文件位于 `src/AES/` 目录。【F:src/AES/student_settings.json†L1-L5】【F:src/AES/teacher_settings.json†L1-L5】
3. 若已有教师模型权重（`teacher_dog.pth`、`Patient_1.pth`、`Patient_2.pth`），请置于 `src/AES/exp/`。

### 2.2 CHB-MIT
1. 下载 MIT PhysioNet 提供的 CHB-MIT EEG 数据集，并按患者目录（`chb01/` 等）置于 `src/CHBMIT/Dataset/`。
2. 调整 `src/CHBMIT/student_settings.json` 与 `src/CHBMIT/teacher_settings.json` 的路径字段，指向数据目录、缓存目录与结果输出目录。【F:src/CHBMIT/student_settings.json†L1-L5】【F:src/CHBMIT/teacher_settings.json†L1-L5】
3. 将原作者提供的教师模型（`pytorch_models/Patient_{id}_detection`）放入 `src/CHBMIT/exp/` 同级目录。

### 2.3 马基-格拉斯时间序列
1. 直接运行 `python src/mackey_glass/utils/utils.py` 内的 `MackeyGlass.generate()` 即可生成 `data.pkl` 文件；脚本会自动缓存于当前工作目录。【F:src/mackey_glass/utils/utils.py†L20-L199】
2. 如需重复试验，可删除旧的 `data.pkl` 以触发重新生成。

## 3. 运行步骤
### 3.1 AES 教师-学生流程
1. **训练教师**（仅需一次）：
   ```bash
   python src/AES/exp/create_teacher.py --subject Dog --epochs 50
   python src/AES/exp/create_teacher.py --subject Patient_1 --epochs 50
   python src/AES/exp/create_teacher.py --subject Patient_2 --epochs 50
   ```
   脚本会将模型保存为 `teacher_dog.pth`、`Patient_1.pth`、`Patient_2.pth`。【F:src/AES/exp/create_teacher.py†L33-L112】
2. **学生蒸馏**：
   ```bash
   python src/AES/exp/FGL_AES.py --subject Dog_1 --alpha 0.5 --epochs 25 --optimizer SGD --temperature 4
   ```
   关键参数：受试者（`--subject`）、蒸馏权重 `--alpha`、优化器（SGD/Adam）与温度。训练完成后会输出灵敏度、FPR 与 AUC，并写入 `FGL_AES.txt`。【F:src/AES/exp/FGL_AES.py†L34-L200】

### 3.2 CHB-MIT 蒸馏实验
1. 确认 `pytorch_models/Patient_{id}_detection` 存在于 `src/CHBMIT/exp/`。
2. 运行单个患者或批量实验：
   ```bash
   # 单个患者
   python src/CHBMIT/exp/FGL_CHBMIT.py --patient 1 --alpha 0.4 --epochs 25 --optimizer Adam --trials 3 --temperature 4

   # 遍历默认患者列表
   python src/CHBMIT/exp/FGL_CHBMIT.py --patient all --alpha 0.4 --epochs 25 --optimizer SGD --trials 3 --temperature 4
   ```
   结果会记录到 `FGL_results.txt`，每个 trial 输出测试 AUC。【F:src/CHBMIT/exp/FGL_CHBMIT.py†L40-L196】

### 3.3 马基-格拉斯长时预测
1. 生成数据后，运行：
   ```bash
   python src/mackey_glass/exp/base_exp.py --horizon 12 --alpha 0.5 --epochs 30 --num_bins 50 --lookback_window 10 --batch_size 64 --temperature 4
   ```
2. 脚本会依次训练教师（1 步预测）、基线（直接预测）与学生（带蒸馏），最终输出三者的测试 MSE，并支持早停恢复最佳权重。【F:src/mackey_glass/exp/base_exp.py†L36-L205】

## 4. 常见问题
- **找不到教师模型**：确认已执行教师训练脚本，或放置原作者发布的权重文件，AES 学生脚本会检查 `teacher_dog.pth` 等文件是否存在。【F:src/AES/exp/FGL_AES.py†L62-L84】
- **缓存目录权限不足**：确保配置文件中的 `cachedir` 指向可写路径；脚本会自动创建目录，但在无权限时需手动调整。【F:src/AES/exp/create_teacher.py†L43-L54】【F:src/CHBMIT/exp/FGL_CHBMIT.py†L40-L86】
- **GPU 内存不足**：可适当减小 `batch_size` 或缩短 `epochs`，AES/CHB-MIT 默认批大小为 32，可在源码中调整 DataLoader 参数。【F:src/AES/exp/FGL_AES.py†L96-L110】【F:src/CHBMIT/exp/FGL_CHBMIT.py†L66-L102】

以上流程覆盖了复现论文主要实验的关键步骤，建议在首次运行时保留日志文件与模型快照，便于后续调参与结果对比。
