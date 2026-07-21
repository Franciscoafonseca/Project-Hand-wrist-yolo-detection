# Hand and Wrist Detection with YOLO11

> **Recommended repository name:** `hand-wrist-yolo-detection`  
> **Current repository:** `Project-Bone_Identify_Age`  
> **Status:** YOLO dataset, training and cropping project

This repository contains the object-detection stage of the bone-age assessment project.

Its purpose is to create and verify YOLO labels, train a YOLO11 detector, locate the hand and wrist in radiographs, and export cropped images or a reusable `best.pt` model.

It does **not** estimate bone age. Bone-age regression is performed in the separate `bone-age-estimation-yolo-efficientnet` repository.

> This software is intended for research and educational purposes only. It is not a certified medical device.

---

## Pipeline

```text
Original radiographs
        ↓
Initial or manually reviewed YOLO labels
        ↓
YOLO dataset creation
        ↓
Dataset validation
        ↓
YOLO11 training
        ↓
Detection evaluation
        ↓
best.pt
        ↓
Hand/wrist crops or downstream age-estimation project
```

---

## Repository structure

```text
hand-wrist-yolo-detection/
├── data/
│   └── yolo_dataset/
│       ├── images/
│       │   ├── train/
│       │   ├── val/
│       │   └── test/
│       ├── labels/
│       │   ├── train/
│       │   ├── val/
│       │   └── test/
│       └── data.yaml
├── detection/
│   ├── check_dataset.py
│   ├── create_yolo_dataset.py
│   └── train_yolo11.py
├── models/
│   └── yolo_hand/
│       ├── README.md
│       └── best.pt
├── src/
│   └── crop_with_yolo.py
├── tools/
│   └── auto_label_xray_region.py
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Important corrections required before publishing

### 1. Rename the repository

The current name suggests age identification, but the code is a detection and cropping project.

Recommended name:

```text
hand-wrist-yolo-detection
```

### 2. Add a reproducible `data.yaml`

The repository documentation expects:

```text
data/yolo_dataset/data.yaml
```

Commit either the real configuration without private paths or an example:

```text
data/yolo_dataset/data.example.yaml
```

Example for a single hand/wrist class:

```yaml
path: data/yolo_dataset

train: images/train
val: images/val
test: images/test

names:
  0: hand_wrist
```

The `names` section must match the class identifiers used in every label file.

### 3. Remove absolute Windows paths from scripts

All scripts must use paths relative to the repository root or command-line arguments.

Avoid:

```python
"D:/Projetos/..."
```

Prefer:

```python
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
DATASET_DIR = ROOT / "data" / "yolo_dataset"
```

### 4. Add command-line arguments

The main scripts should accept at least:

```text
--images
--labels
--data-yaml
--model
--epochs
--imgsz
--batch
--device
--output
```

This prevents collaborators from editing source code only to change paths.

### 5. Keep generated labels reviewable

Automatically generated labels are initial proposals, not guaranteed ground truth.

Before final YOLO training:

- draw the boxes over every training image;
- verify fingertips are included;
- verify the thumb is included;
- verify the carpal region is included;
- verify the distal radius and ulna are included;
- correct failed or oversized boxes;
- ensure each image has the expected number of labels.

### 6. Store `best.pt` outside normal Git history

The trained detector should be available through:

- GitHub Releases;
- Git LFS;
- private institutional storage.

The repository should contain:

```text
models/yolo_hand/README.md
```

explaining how to obtain and place the weight file.

### 7. Record the YOLO experiment configuration

For every selected model, preserve:

- YOLO version;
- base checkpoint;
- number of classes;
- class names;
- image size;
- batch size;
- epochs;
- confidence threshold;
- IoU threshold;
- data split;
- random seed;
- mAP@50;
- mAP@50–95;
- precision;
- recall.

---

## Installation

Recommended Python version:

```text
Python 3.11
```

Create the environment:

```bash
conda create -n hand_wrist_yolo python=3.11 -y
conda activate hand_wrist_yolo
python -m pip install -r requirements.txt
```

Verify Ultralytics:

```bash
python -c "from ultralytics import YOLO; print('YOLO OK')"
```

---

## Dataset placement

Do not commit private radiographs.

Expected generated dataset:

```text
data/yolo_dataset/
├── images/
│   ├── train/
│   ├── val/
│   └── test/
├── labels/
│   ├── train/
│   ├── val/
│   └── test/
└── data.yaml
```

Each image must have a label file with the same base name:

```text
images/train/15.jpg
labels/train/15.txt
```

YOLO label format:

```text
class_id x_center y_center width height
```

All coordinates must be normalised between 0 and 1.

---

## Generate initial labels

Run:

```bash
python tools/auto_label_xray_region.py
```

Recommended future command-line version:

```bash
python tools/auto_label_xray_region.py \
  --images data/source_images \
  --labels data/generated_labels \
  --preview data/label_previews
```

Inspect all previews before creating the final dataset.

---

## Create the dataset

Run:

```bash
python detection/create_yolo_dataset.py
```

Recommended future command-line version:

```bash
python detection/create_yolo_dataset.py \
  --images data/source_images \
  --labels data/verified_labels \
  --output data/yolo_dataset \
  --train 0.70 \
  --val 0.15 \
  --test 0.15 \
  --seed 42
```

The split must be deterministic and saved for reproducibility.

---

## Validate the dataset

Run:

```bash
python detection/check_dataset.py
```

The validation must check:

- missing label files;
- missing image files;
- duplicated filenames;
- invalid class identifiers;
- coordinates outside `[0, 1]`;
- zero-width or zero-height boxes;
- empty labels;
- split overlap.

Do not train while any critical validation error remains.

---

## Train YOLO11

Run the project script:

```bash
python detection/train_yolo11.py
```

Or use Ultralytics directly:

```bash
yolo detect train \
  model=yolo11n.pt \
  data=data/yolo_dataset/data.yaml \
  epochs=100 \
  imgsz=640 \
  batch=8 \
  seed=42
```

Adjust `batch` to the available GPU memory.

Typical output:

```text
runs/detect/train/weights/best.pt
```

Copy or link the selected model to:

```text
models/yolo_hand/best.pt
```

---

## Evaluate the detector

Record at least:

- precision;
- recall;
- mAP@50;
- mAP@50–95;
- per-class results;
- examples of correct detections;
- examples of failed detections.

Do not evaluate the final detector only on training images.

---

## Crop images with the trained model

Run:

```bash
python src/crop_with_yolo.py
```

Recommended future command-line version:

```bash
python src/crop_with_yolo.py \
  --model models/yolo_hand/best.pt \
  --input data/test_images \
  --output outputs/crops \
  --conf 0.30 \
  --iou 0.45 \
  --imgsz 640
```

Before using the crops downstream, confirm that:

- the complete hand is visible;
- no fingertips are cut;
- the thumb is visible;
- the wrist is sufficiently included;
- crop orientation is consistent;
- background is not excessive.

---

## Connection with the age-estimation repository

The resulting file:

```text
models/yolo_hand/best.pt
```

is used by:

```text
bone-age-estimation-yolo-efficientnet
```

The age-estimation repository should load the detector for inference only. YOLO training data and YOLO training outputs do not need to be duplicated there.

---

## Privacy

Never commit:

- radiographs;
- DICOM files;
- patient identifiers;
- spreadsheets containing dates or clinical data;
- previews that retain identifying annotations;
- private credentials.

---

## Recommended repository topics

```text
yolo11
object-detection
hand-xray
medical-imaging
pytorch
ultralytics
computer-vision
```

---

## License

Add a project licence.

Ultralytics, pretrained weights and medical datasets may have separate licences and usage restrictions.
