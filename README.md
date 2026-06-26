# ESP32-P4 Tiny CNN 海洋视觉轻量化项目

本项目是在原 ESP32-P4 智能无线视觉浮标工程基础上整理出的轻量化升级版。原项目以 YOLO/COCO 检测为主要路线，输出 `bbox + class + score`，但检测框后处理、JPEG 解码、候选框筛选和 NMS 在 ESP32-P4 端计算压力较大，不利于达到连续实时识别所需的 `5 FPS+` 目标。

本版本将默认识别路线切换为 Tiny CNN 海洋目标分类器。模型不再输出检测框，而是直接输出 `label / score / top_k / inference_ms / analysis_ms`，识别任务从“检测目标位置”改为“判断当前画面或中心裁剪区域属于哪一类海面目标”。这样可以保留原工程的摄像头、Web 控制台、NVS 配置、状态 API、历史记录和网络框架，同时显著降低端侧推理计算量。

当前默认模型为 `TinyCNN-M`，类别为：

```text
unknown
plastic_bottle
foam
buoy
net
ship_part
```

## 当前状态

- 默认识别方法：`tinycls`
- 默认模型：`models/tiny_cls_m_128_6cls_s8_p4.espdl`
- 默认输入尺寸：`128x128 RGB`
- 默认推理间隔：`0 ms`
- 已完成 `idf.py build`
- 已生成 S/M/L/XL/XL-Deep 五个固件变体
- `/api/status`、`/api/config`、`/api/recognition?method=tinycls` 保留
- `/api/validate/run` 已支持 `method=tinycls`
- Tiny CNN 实时路径使用摄像头 raw frame 同步处理，不再走 YOLO 的 JPEG queue、bbox、NMS 路径

## 项目结构

```text
main/                         ESP32-P4 主程序和 Tiny CNN 桥接层
components/                   本地视频组件
models/                       Tiny CNN PT / ONNX / ESP-DL INT8 模型
tools/                        数据集处理、训练、量化、延迟采样脚本
docs/                         中文说明文档
data/                         本地训练数据目录，通常不上传完整大数据集
reports/                      训练报告和采样报告
release/firmware_variants/    S/M/L/XL/XL-Deep 固件变体
build/                        当前默认 M 模型构建产物
README.md                     项目总说明
```

核心接入文件：

```text
main/camera_web_main.c
main/tiny_cls_espdl_bridge.cpp
main/tiny_cls_espdl_bridge.h
main/tiny_cls_task.h
main/CMakeLists.txt
main/Kconfig.projbuild
sdkconfig.defaults
```

## 模型版本

| 版本 | 输入 | 参数量 | valid_acc | test_acc | `.espdl` 大小 | 用途 |
| --- | --- | ---: | ---: | ---: | ---: | --- |
| TinyCNN-S | 96x96 | 8,270 | 92.88% | 91.25% | 25,648 B | 速度 baseline |
| TinyCNN-M | 128x128 | 16,382 | 96.91% | 95.67% | 36,400 B | 默认推荐 |
| TinyCNN-L | 160x160 | 28,054 | 97.82% | 96.55% | 50,896 B | 精度优先 |
| TinyCNN-XL | 192x192 | 51,622 | 98.38% | 97.41% | 78,480 B | 大模型对比 |
| TinyCNN-XL-Deep | 192x192 | 270,342 | 96.42% | 94.12% | 338,592 B | 加深加宽实验版 |

当前默认配置：

```ini
CONFIG_APP_DEFAULT_RECOGNITION_METHOD=5
CONFIG_APP_TINY_CLS_MODEL_FILE="../models/tiny_cls_m_128_6cls_s8_p4.espdl"
CONFIG_APP_TINY_CLS_INPUT_SIZE=128
CONFIG_APP_INFERENCE_INTERVAL_MS=0
```

如果要切换到 XL：

```ini
CONFIG_APP_TINY_CLS_MODEL_FILE="../models/tiny_cls_xl_192_6cls_s8_p4.espdl"
CONFIG_APP_TINY_CLS_INPUT_SIZE=192
```

切换后重新构建：

```powershell
idf.py build
```

## 固件变体

已经为四个模型分别跑过 `idf.py build`，应用固件保存在：

```text
release/firmware_variants/esp32p4_buoy_vision_lab_tinycnn_s.bin
release/firmware_variants/esp32p4_buoy_vision_lab_tinycnn_m.bin
release/firmware_variants/esp32p4_buoy_vision_lab_tinycnn_l.bin
release/firmware_variants/esp32p4_buoy_vision_lab_tinycnn_xl.bin
release/firmware_variants/esp32p4_buoy_vision_lab_tinycnn_xl_deep.bin
release/firmware_variants/bootloader.bin
release/firmware_variants/partition-table.bin
release/firmware_variants/flash_args_tinycnn_m
```

当前普通构建目录里的应用固件仍是推荐的 M 版本：

```text
build/esp32p4_buoy_vision_lab.bin
```

## 编译

进入工程目录：

```powershell
cd C:\Users\cyj\Downloads\esp32p4_buoy_vision_lab-main-privacy\esp32p4_buoy_vision_lab-main-privacy
```

使用 ESP-IDF 环境编译：

```powershell
idf.py build
```

如果当前 PowerShell 没有加载 ESP-IDF，也可以直接调用本机 IDF 工具：

```powershell
python C:\esp\v6.0.1\esp-idf\tools\idf.py build
```

编译成功后主要产物：

```text
build/bootloader/bootloader.bin
build/partition_table/partition-table.bin
build/esp32p4_buoy_vision_lab.bin
build/flash_args
```

## 烧录

连接 ESP32-P4 后执行：

```powershell
idf.py -p COM3 flash monitor
```

如果只使用 release 里的 bin 文件，可以按地址烧录：

```powershell
python -m esptool --chip esp32p4 -p COM3 -b 460800 `
  --before default-reset --after hard-reset write-flash `
  --flash-mode dio --flash-size 16MB --flash-freq 80m `
  0x2000 release\firmware_variants\bootloader.bin `
  0x8000 release\firmware_variants\partition-table.bin `
  0x10000 release\firmware_variants\esp32p4_buoy_vision_lab_tinycnn_m.bin
```

把最后一个 app bin 换成 `tinycnn_s/m/l/xl/xl_deep.bin`，即可烧录不同模型版本。

如果只想上板对比 XL-Deep，直接把 app bin 换成
`release\firmware_variants\esp32p4_buoy_vision_lab_tinycnn_xl_deep.bin`。如果要让源码工程默认编译为
XL-Deep，再修改配置并重新构建：

```ini
CONFIG_APP_TINY_CLS_MODEL_FILE="../models/tiny_cls_xl_deep_192_6cls_s8_p4.espdl"
CONFIG_APP_TINY_CLS_INPUT_SIZE=192
```

## Web 页面和 API

板子启动后默认使用 AP+STA。手机可连接板子热点，或电脑连接同一路由器后访问串口打印的地址。

常用页面和接口：

```text
/                                  Web 控制台
/validate                          板端验证页面
/stream                            MJPEG 视频流
/api/status                        状态 JSON
/api/config                        读取或修改配置
/api/recognition?method=tinycls    切换到 Tiny CNN
/api/validate/run?method=tinycls   运行板端验证
/api/power?cmd=wake                唤醒摄像头
/api/power?cmd=standby             待机
```

Tiny CNN 返回结果重点看：

```text
vision.label
vision.score
vision.top_k
vision.inference_ms
vision.analysis_ms
vision.model
vision.model_bytes
inference_fps_x100
```

本项目不再要求 Tiny CNN 返回 `detections` 和 `bbox`。如果需要检测框，应切回 YOLO/COCO 对比路线。

## p95 和 FPS 采样

上板后可以用脚本连续采样 `/api/status`，统计 `vision.analysis_ms` 的 p95：

```powershell
python tools\sample_status_latency.py 192.168.4.1 --duration-min 1 --interval-ms 200 --method tinycls --wake --set-interval-zero
```

5 分钟采样：

```powershell
python tools\sample_status_latency.py 192.168.4.1 --duration-min 5 --interval-ms 200 --method tinycls --wake --set-interval-zero --output reports\tinycls_status_latency_5min.csv
```

验收目标：

```text
p95 analysis_ms < 200
inference_fps >= 5
```

没有实体板时不能证明真实板端 FPS，只能完成无板工程验证：模型已生成、ESP-DL INT8 量化产物已嵌入、固件能编译、API 路径已接通、采样脚本已准备好。

## 数据集和训练流程

分类数据来自三个来源：

- PoTATO 海面漂浮物数据，用于 `plastic_bottle / foam / net / unknown` 等类别补充
- LaRS 海面/船舶场景数据，用于 `buoy / ship_part` 以及 hard negative
- 原 YOLO 标注和自建裁剪样本，用于从检测框区域生成分类正样本

数据整理思路：

```text
正样本：裁剪 bbox 区域，适当放大 1.2~1.6 倍
负样本：从无 bbox 区域随机裁剪，作为 unknown
整图样本：有目标图标为目标类，无目标图标为 unknown
hard negative：海浪、反光、泡沫、船体局部、天空、岸线
```

常用脚本：

```text
tools/prepare_tiny_cls_dataset.py
tools/prepare_potato_tiny_cls_dataset.py
tools/prepare_lars_yolo_dataset.py
tools/merge_tiny_cls_datasets.py
tools/train_tiny_cnn_classifier.py
tools/quantize_tiny_cls_espdl.py
```

训练 TinyCNN-M：

```powershell
python tools\train_tiny_cnn_classifier.py --dataset data\tiny_cls_merged6 --variant m --epochs 20 --batch 64 --device cpu
```

量化 TinyCNN-M：

```powershell
python tools\quantize_tiny_cls_espdl.py `
  --onnx models\tiny_cls_m_128_6cls.onnx `
  --output models\tiny_cls_m_128_6cls_s8_p4.espdl `
  --dataset data\tiny_cls_merged6 `
  --report reports\tiny_cls_m_128_6cls_report.json `
  --input-size 128 `
  --calib-limit 672 `
  --calib-steps 200 `
  --batch-size 1 `
  --device cpu
```

## 与原 YOLO 项目的区别

原项目更适合展示“目标检测”：可以输出检测框、类别和置信度，但在 ESP32-P4 上实时性压力较大。

本项目更适合展示“轻量分类”：保留原项目的 Web、状态、NVS、网络和摄像头框架，把默认识别路径换成 Tiny CNN。它牺牲了 bbox 定位能力，换取更低的端侧计算量和更接近实时的推理速度。

推荐演示说法：

```text
本项目是在 ESP32-P4 智能视觉浮标工程基础上的轻量化升级版。原项目使用 YOLO 检测模型，能够输出目标框，但端侧推理和后处理耗时较高。本项目将默认识别路线切换为 Tiny CNN 海洋目标分类器，直接输出 label、score、top_k 和延迟指标，不再执行 YOLO 的 bbox、JPEG decode、NMS 路径。工程仍保留网页控制台、状态 API、NVS 参数、历史记录和网络框架，并提供 S/M/L/XL 四个模型变体，用于在速度和准确率之间做对比。
```

## 文档

```text
docs/tiny_cnn_classifier_cn.md
docs/tinycls_project_explanation_cn.md
docs/marine_vision_datasets_cn.md
docs/developer_guide.md
docs/customer_manual.md
models/README.md
```

## 注意事项

- `tools/*.py` 是 PC 端脚本，不是烧进 ESP32-P4 的脚本。
- 真正烧进板子的是 `build/*.bin` 或 `release/firmware_variants/*.bin`。
- 上传 GitHub 时建议保留源码、模型、README、docs、tools 和 release 小体积固件；完整原始数据集通常不要直接上传。
- 如果要证明 `5 FPS+`，必须用真实板子运行 `/api/status` 采样脚本，不能只用 PC 编译结果代替。
