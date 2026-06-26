# 模型目录说明

本目录保存 Tiny CNN 海洋视觉分类器的训练权重、ONNX 导出文件、ESP-DL INT8 量化产物和转换报告。当前固件默认嵌入并运行的是：

```text
tiny_cls_m_128_6cls_s8_p4.espdl
```

嵌入位置由 `sdkconfig.defaults` 控制：

```ini
CONFIG_APP_TINY_CLS_MODEL_FILE="../models/tiny_cls_m_128_6cls_s8_p4.espdl"
CONFIG_APP_TINY_CLS_INPUT_SIZE=128
```

板端桥接代码在：

```text
main/tiny_cls_espdl_bridge.cpp
main/tiny_cls_espdl_bridge.h
main/tiny_cls_task.h
```

## 类别

```text
unknown
plastic_bottle
foam
buoy
net
ship_part
```

Tiny CNN 是分类模型，不输出检测框。板端输出重点为：

```text
label
score
top_k
inference_ms
analysis_ms
```

## 已训练模型

| 版本 | 输入 | 参数量 | valid_acc | test_acc | `.espdl` 文件 | 大小 |
| --- | --- | ---: | ---: | ---: | --- | ---: |
| TinyCNN-S | 96x96 | 8,270 | 92.88% | 91.25% | `tiny_cls_96_6cls_s8_p4.espdl` | 25,648 B |
| TinyCNN-M | 128x128 | 16,382 | 96.91% | 95.67% | `tiny_cls_m_128_6cls_s8_p4.espdl` | 36,400 B |
| TinyCNN-L | 160x160 | 28,054 | 97.82% | 96.55% | `tiny_cls_l_160_6cls_s8_p4.espdl` | 50,896 B |
| TinyCNN-XL | 192x192 | 51,622 | 98.38% | 97.41% | `tiny_cls_xl_192_6cls_s8_p4.espdl` | 78,480 B |
| TinyCNN-XL-Deep | 192x192 | 270,342 | 96.42% | 94.12% | `tiny_cls_xl_deep_192_6cls_s8_p4.espdl` | 338,592 B |

推荐默认使用 TinyCNN-M。S 用于速度 baseline，L/XL 用于精度优先和大模型对照。XL-Deep 是加深加宽实验版，通道为 `48-96-144-192-288`，每个 stage 后额外增加 1 个 `stride=1` DSConv。

## 主要文件

```text
tiny_cls_96_6cls.pt
tiny_cls_96_6cls.onnx
tiny_cls_96_6cls_s8_p4.espdl

tiny_cls_m_128_6cls.pt
tiny_cls_m_128_6cls.onnx
tiny_cls_m_128_6cls_s8_p4.espdl

tiny_cls_l_160_6cls.pt
tiny_cls_l_160_6cls.onnx
tiny_cls_l_160_6cls_s8_p4.espdl

tiny_cls_xl_192_6cls.pt
tiny_cls_xl_192_6cls.onnx
tiny_cls_xl_192_6cls_s8_p4.espdl

tiny_cls_xl_deep_192_6cls.pt
tiny_cls_xl_deep_192_6cls.onnx
tiny_cls_xl_deep_192_6cls_s8_p4.espdl
```

`.pt` 用于继续训练或复现实验，`.onnx` 用于模型交换和量化，`.espdl` 是实际嵌入 ESP32-P4 固件的 INT8 模型。

## 重新训练

```powershell
python tools\train_tiny_cnn_classifier.py --dataset data\tiny_cls_merged6 --variant m --epochs 20 --batch 64 --device cpu
```

可选 `--variant`：

```text
s   96x96
m   128x128
l   160x160
xl  192x192
xl_deep  192x192, channels 48-96-144-192-288, extra stride=1 DSConv per stage
```

## ESP-DL INT8 量化

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

校准集必须使用真实海面、漂浮物、浮标、船体局部和 unknown 样本，不建议用随机图像。

## 切换固件模型

修改 `sdkconfig.defaults`：

```ini
CONFIG_APP_TINY_CLS_MODEL_FILE="../models/tiny_cls_l_160_6cls_s8_p4.espdl"
CONFIG_APP_TINY_CLS_INPUT_SIZE=160
```

然后重新编译：

```powershell
idf.py build
```

## 历史模型

目录里可能保留旧的 `tiny_cls_96.*` 二分类或早期实验文件，用于过程记录。当前 6 类主线以 `*_6cls*` 文件为准。
