# BEV Map Annotator

一个独立的 **本地桌面 BEV 地图标注工具**。支持从连续点云生成地面反射强度底图，再标注道路、分隔线、斑马线和停止线。

当前版本：**v4.4 Mint**。薄荷绿＋奶白色界面，地图默认黑色背景。无需浏览器、HTTP 服务、GPU 或 PyTorch。

## 1. 下载并解压

下载仓库中的 [v4.4 ZIP](./bev_map_annotator_desktop_v4_4_mint.zip)，点击文件页的下载按钮后解压。

也可以通过 Git 下载：

```bash
git clone https://github.com/TaozheLi/BEV-Map-Annotator.git
cd BEV-Map-Annotator
unzip bev_map_annotator_desktop_v4_4_mint.zip
cd bev_map_annotator_desktop
```

仓库当前存放的是打包文件；实际 Python 代码、依赖和示例地图都在 ZIP 内。

## 2. 安装环境

已验证环境：**Python 3.10.21、Tk 8.6**。依赖包括 Pillow、Shapely、NumPy 和 KISS-ICP，具体版本见解压后的 `requirements.txt`。

### Linux / macOS

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python app.py
```

### Windows

在解压后的目录打开终端：

```powershell
py -3.10 -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe app.py
```

本次安装与功能测试在 Linux 上完成。其他系统安装 KISS-ICP 时，需要对应平台支持的 wheel 或本地编译环境。

### 检查桌面组件

```bash
python -m tkinter
```

应弹出测试窗口。如果提示缺少 tkinter，请为所使用的 Python 安装 Tk 支持：Ubuntu/Debian 使用对应版本的 `python3-tk` / `python3.10-tk`，Conda 环境安装 `tk`。Tkinter 是 Python/系统组件，不是通过 pip 安装的普通依赖。

应用需要图形桌面。纯 SSH 终端不能直接显示窗口。

## 3. 先用示例地图体验

```bash
python app.py
```

程序加载包内示例 BEV 图片和 85 个已有标注。可以直接选择要素、编辑顶点并保存。

默认人工标注输出：

```text
bev_map_annotator_desktop/saved/reviewed_annotations.geojson
```

示例原文件不会被覆盖。输出已经存在时，程序优先载入上次保存的结果。

## 4. 用自己的连续点云建图

### 方法 A：桌面操作

1. 点击顶部 **Point clouds**。
2. 选择点云根目录。
3. File pattern 默认是 `**/*.pcd`，递归读取子目录；KITTI BIN 使用 `**/*.bin`。
4. 可填写 First frame / Last frame；留空表示全部。
5. 默认 stride 为 **10**，底图分辨率为 **0.15 米/像素**。
6. 默认不选位姿文件，自动使用 **KISS-ICP** 配准。
7. Output 留空即可自动生成新目录。
8. 点击 **Build map and annotate**，等待进度完成，程序自动进入新底图的标注界面。

支持以下目录布局：

```text
root/
├── 01/
│   ├── 000001.pcd
│   └── 000002.pcd
└── 02/
    ├── 000001.pcd
    └── subfolder/
        └── 000002.pcd
```

也支持原始采集布局：

```text
capture/
├── 000001/lidar.pcd
├── 000002/lidar.pcd
└── ...
```

读取顺序按目录和文件名自然排序，例如 `01 → 02`、`2 → 10`。不同子目录同名文件用相对路径区分。

起止帧可填写：

```text
01/000001.pcd
02/000100.pcd
```

如果只填一个重复的文件名，程序会提示补全相对路径。

### 方法 B：命令行

只指定点云目录，使用默认 KISS-ICP 和默认输出：

```bash
python app.py --cloud-dir /data/root
```

指定范围和采样参数：

```bash
python app.py \
  --cloud-dir /data/root \
  --pattern '**/*.pcd' \
  --start 01/000001.pcd \
  --end 02/000100.pcd \
  --stride 10 \
  --resolution 0.15
```

自动配准会处理选中范围内的 **每一帧**；stride 只控制底图采样，不跳过定位帧。

如果已有位姿文件，可以主动覆盖自动配准：

```bash
python app.py \
  --cloud-dir /data/root \
  --poses /data/poses.json \
  --map-output /data/new_bev_map
```

位姿 JSON 示例：

```json
[
  {
    "frame": "000001",
    "T_world_from_lidar": [
      [1, 0, 0, 0],
      [0, 1, 0, 0],
      [0, 0, 1, 0],
      [0, 0, 0, 1]
    ]
  }
]
```

矩阵表示原始 LiDAR 坐标到世界坐标的变换，必须覆盖选中的帧。同名点云的 frame ID 应使用相对路径，例如 `01/000001`。

### 默认输出目录

未填写 output 时，自动保存到工具目录下：

```text
generated_maps/<输入目录名>_<时间戳>_<唯一后缀>/
├── ground_reflectivity.png
├── ground_map.npz
├── metadata.json
├── poses.json
├── annotations.geojson
└── reviewed_annotations.geojson   # 人工保存后生成
```

每次新建独立目录，不覆盖旧地图。状态栏会显示实际保存位置。手动指定输出时，应使用新的空目录。

### 点云输入条件

- 支持 ASCII PCD、未压缩 binary PCD，以及 KITTI float32 XYZI 四列 BIN。
- 暂不支持 `binary_compressed` PCD。
- 需要 `reflectivity` 或 `intensity` 字段生成反射强度底图。
- 输入为米制、原始 LiDAR 坐标、Z 轴向上。
- 当前地面提取参数适配传感器下方约 1.75 米为地面的采集：候选高度 -3.5 至 -0.6 米、范围 40 米、近水平 RANSAC。
- 自动配准假定排序后的点云是连续采集。如果 01 和 02 属于不连续的采集，应分开建图，或提供统一世界坐标的位姿。
- 如有匹配的 `lidar_timing.npz` 和 PCD `column` 字段，自动配准使用列时间戳去畸变；否则没有逐点时间矫正。
- 底图生成后，先检查漂移或重影，再正式标注。地面提取不等于自动识别可行驶区域。

## 5. 标注类别

| 类别 | 几何类型 | 用途 |
|---|---|---|
| `drivable_area` | Polygon | 可行驶区域 |
| `divider` | LineString | 道路分隔标线 |
| `ped_crossing` | Polygon | 斑马线整体覆盖区域 |
| `boundary` | LineString | 道路边界 |
| `stopline` | LineString | 停止线 |

停止线至少画两个点。斑马线通常画整体区域的四个角，不需要逐条画白色条纹。

## 6. 绘制和编辑

1. 选择类别，点击 **New feature**。
2. 沿边界依次点击顶点。
3. 点击 **Finish drawing**；多边形自动闭合。
4. 点击 **Save draft** 保存。

多边形顶点应按顺时针或逆时针顺序排列，不能交叉。保存时会检查无效几何，并提示要素编号。

| 操作 | 方法 |
|---|---|
| 选择要素 | 左侧要素下拉框 |
| 移动顶点 | 拖动选中要素的顶点 |
| 增加顶点 | 双击边线 |
| 删除顶点 | 右键顶点 |
| 删除要素 | Delete selected feature |
| 撤销 / 重做 | Undo / Redo，或 Ctrl+Z / Ctrl+Y |
| 取消绘制 | Cancel / Esc |
| 删除当前绘制最后一点 | Backspace |
| 缩放 | 鼠标滚轮、＋ / － |
| 显示全图 | Fit map |
| 平移 | 空格＋拖动、鼠标中键拖动，或勾选 Pan |
| 保存 | Save draft / Ctrl+S |
| 另存为 | Save as |
| 修改字体 | 顶部 Font…，可预览并保存字体与字号 |

当前版本不支持地图旋转。默认黑色地图背景，勾选 **Light map** 可切换显示；显示风格不改变原始图片或标注坐标。

## 7. 打开现成 BEV 图片

点击 **Open map**，依次选择图片、metadata JSON、可选初始 GeoJSON 和输出文件。

也可运行：

```bash
python app.py \
  --image /data/map.png \
  --metadata /data/metadata.json \
  --annotations /data/initial.geojson \
  --output /data/saved/reviewed_annotations.geojson
```

已有图片模式的 `--output` 是 **GeoJSON 文件路径**；点云建图模式的 `--map-output` 是 **地图目录**。

最小 metadata 示例：

```json
{
  "coordinate_frame": "local_world_meters_not_WGS84",
  "resolution_m": 0.15,
  "minimum_world_xy": [-300.0, -120.0],
  "image_size": [2527, 2117]
}
```

图片向右对应世界 +X、向上对应世界 +Y。`minimum_world_xy` 是左下角外边界，单位米。输出使用局部米制坐标，不是经纬度。目前支持轴对齐、等比例栅格。

## 8. 保存与备份

- 未点击 Save 的修改不会写入文件。
- 每次覆盖保存前，旧版本备份到输出文件旁的 `backups/`。
- 初始标注与输出文件必须不同。
- 其他编辑器修改了输出文件时，会拒绝覆盖；可使用 Save as 保留自己的版本。
- stopline 已支持桌面标注和 GeoJSON 输出，接入其他训练框架时仍需适配其类别定义。

## 9. 可选：校验下载

将 ZIP 和校验文件放在同一目录：

```bash
sha256sum -c bev_map_annotator_desktop_v4_4_mint.zip.sha256
```

显示 `OK` 表示与上传版本一致。

## 10. 测试

解压后，在工具目录运行：

```bash
python -m unittest discover -v
```

桌面测试需要可用显示环境；没有显示环境时会跳过桌面测试。
