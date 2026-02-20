# YOLOX-ROS H264 Camera Pipeline

H264カメラ映像をデコードし、YOLOXで物体検出を行うパイプライン。

## アーキテクチャ

```
/cam0/h264 (CompressedImage, H264, ~24fps)
    │
    ▼
[h264_decoder_node]  GStreamer + nvv4l2decoder (HWデコード)
    │
    ▼
/cam0/image_raw (sensor_msgs/Image, bgr8, 1920x1280)
    │
    ▼
[yolox_ros_cpp]  TensorRT FP16 推論
    │
    ├─► /yolox/bounding_boxes (検出結果)
    └─► /yolox/image_raw (描画済み画像)
```

## 前提条件

- Jetson AGX Orin / JetPack 6 (TensorRT 10.3, CUDA 12.5)
- ROS 2 Humble
- GStreamer 1.0 + gstreamer1.0-plugins-bad (nvv4l2decoder)

## ビルド

```bash
cd /mnt/m2ssd/agv_humble_ws/autoware
source /opt/ros/humble/setup.bash

# H264デコーダ
colcon build --packages-select h264_decoder_node --symlink-install

# YOLOX (TensorRT有効)
colcon build --packages-up-to yolox_ros_cpp --symlink-install \
  --cmake-args -DCMAKE_BUILD_TYPE=Release -DYOLOX_USE_TENSORRT=ON
```

## TRTエンジン作成

```bash
cd /mnt/m2ssd/agv_humble_ws/autoware/src/YOLOX-ROS/weights/tensorrt

# 利用可能モデル: yolox_nano, yolox_tiny, yolox_s, yolox_m, yolox_l
bash convert.bash yolox_s
```

ONNXモデルが未ダウンロードの場合は自動でダウンロードされる。

## 起動方法

全てautowareディレクトリから実行する。

```bash
cd /mnt/m2ssd/agv_humble_ws/autoware
source install/setup.bash
```

### ターミナル1: H264デコーダ

```bash
ros2 launch h264_decoder_node h264_decoder.launch.py
```

パラメータ:
| パラメータ | デフォルト | 説明 |
|---|---|---|
| `input_topic` | `/cam0/h264` | H264入力トピック |
| `output_topic` | `/cam0/image_raw` | デコード画像出力トピック |
| `frame_id` | `cam0` | フレームID |

### ターミナル2: YOLOX (モデルサイズ別)

**yolox_tiny** (416x416, ~80 FPS):
```bash
ros2 launch yolox_ros_cpp yolox_tensorrt_cam0.launch.py \
  model_path:=/mnt/m2ssd/agv_humble_ws/autoware/src/YOLOX-ROS/weights/tensorrt/yolox_tiny.trt
```

**yolox_s** (640x640, ~42 FPS):
```bash
ros2 launch yolox_ros_cpp yolox_tensorrt_cam0.launch.py \
  model_path:=/mnt/m2ssd/agv_humble_ws/autoware/src/YOLOX-ROS/weights/tensorrt/yolox_s.trt
```

**yolox_m** (640x640, ~24 FPS):
```bash
ros2 launch yolox_ros_cpp yolox_tensorrt_cam0.launch.py \
  model_path:=/mnt/m2ssd/agv_humble_ws/autoware/src/YOLOX-ROS/weights/tensorrt/yolox_m.trt
```

### ターミナル3: rosbag再生 (テスト用)

```bash
ros2 bag play -s mcap -l <bag_path> --read-ahead-queue-size 5000
```

## ベンチマーク (AGX Orin, FP16, batch=1)

| モデル | 入力サイズ | パラメータ | GFLOPs | 実測FPS | mAP (COCO) |
|---|---|---|---|---|---|
| yolox_tiny | 416 | 5.06M | 6.45 | ~80 | 32.8 |
| yolox_s | 640 | 9.0M | 26.8 | ~42 | 40.5 |
| yolox_m | 640 | 25.3M | 73.8 | ~24 | 46.9 |

## カスタマイズ

```bash
# 信頼度しきい値を変更
ros2 launch yolox_ros_cpp yolox_tensorrt_cam0.launch.py \
  model_path:=<path>.trt \
  conf:=0.5 \
  nms:=0.45

# 別のカメラトピックを使用
ros2 launch h264_decoder_node h264_decoder.launch.py \
  input_topic:=/cam1/h264 \
  output_topic:=/cam1/image_raw

ros2 launch yolox_ros_cpp yolox_tensorrt_cam0.launch.py \
  src_image_topic_name:=/cam1/image_raw
```
