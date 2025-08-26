# GFM_Galileo

Geospatial Foundation Model (GFM): Galileo

# Galileo Pretrained Remote Sensing Model – Rwanda Crop Type Classification

A fine-tuning pipeline for the **[Galileo Pretrained Remote Sensing Model](https://github.com/nasaharvest/galileo)** to classify crop types using pixel-wise segmentation with multi-temporal Sentinel-1 and Sentinel-2 data in Rwanda.

---

## Quickstart

### 1. Clone the repository

```bash
git clone <repo-url>
cd FM
```

### 2. Create Environment

With **conda**:

```bash
conda env create -f environment.yml
conda activate gfm-env
```

## Dataset Preparation

### Step 0: Configure GEE

Follow these steps to set up a **Google Cloud project**, create a **service account with a JSON key**, and enable **Earth Engine (GEE)** for programmatic access used by the scripts in this repo.

#### 0.1 Create or select a Google Cloud project

- Open: https://console.cloud.google.com/projectcreate
- Create a project (note the **Project ID**) or select an existing one from the top project picker.

#### 0.2 Enable the Google Earth Engine API

- Open: https://console.cloud.google.com/apis/library
- Make sure the **correct project** is selected (top bar).
- Search **Google Earth Engine API** → **Enable**.

#### 0.3 Register the project for Earth Engine

- Open: https://code.earthengine.google.com/register
- Choose the same **Cloud project** and complete registration (Non-commercial/Commercial as appropriate).

#### 0.4 Create a service account

- Open: https://console.cloud.google.com/iam-admin/serviceaccounts
- **Create Service Account** (e.g., `gee-service-account`).
- Grant minimum roles you need:
  - **Writer** (read/write assets/exports): `Earth Engine Resource Writer`

#### 0.5 Create and download a private key (JSON)

- In the service account → **Keys** tab → **Add key** → **Create new key** → **JSON** → **Create**.
- Store the downloaded `*.json` securely (never commit it to Git).

#### 0.6 Access required Earth Engine assets

To avoid manual sharing, it’s recommended to **keep your assets under the same registered Cloud project** you initialized with:

1. Open the Earth Engine Code Editor: [https://code.earthengine.google.com/](https://code.earthengine.google.com/)
2. Go to the **Assets tab** (left panel).
3. Under **Cloud Assets**, locate your project folder:
   ```
   <your-project-id>
   ```
   This folder represents your registered Cloud project.
4. When exporting or creating assets, always place them under this project path. Example:
   ```python
   task = ee.batch.Export.image.toAsset(
       image=my_image,
       description='my_export',
       assetId='projects/<your-project-id>/assets/my_dataset/my_asset'
   )
   ```
   This ensures assets are automatically accessible to your service account without extra sharing.

> **Note**: If you don’t see a `<your-project-id>` folder under **Cloud Assets** in the Assets tab, make sure your Cloud project has been registered with Earth Engine (see Steps above).

#### 0.7 Point the repo to your project and key

- Place the JSON key at the root directory (e.g., `Root Directory/private_key.json`).
- Update project/key references in this repo:
  - **`src/data/config.py`**: set `EE_PROJECT = "<your-project-id>"`.
  - **`eo.py`**: configure `ee.batch.Export.image.toAsset()` target **assetId** path(s) that match your project/folder structure.

### Step 1: Export Sentinel-1/2 Data from Earth Engine

Use the script below to export Sentinel-1/ Sentinel-2 images over your area of interest for specific session (Season A or Season B):

```bash
python 0_get_data.py
```

### Step 2: Download Exported Tiles from GEE

After GEE completes your export tasks, download the resulting GeoTIFF tiles::

```bash
python 1_download_tiles_from_gee.py
```

### Step 3: Extract Pixel-wise Patches from Tiles

Use preprocessed Sentinel-1/Sentinel-2 tiles and polygon labels to generate 8×8 pixel-wise training patches:

```bash
python 2_extract_patches_from_tiles.py
```

This script will:

- Read raster tiles and label masks
- Extract valid 8×8 image and mask patches
- Save `.npz` patch datasets for `train`, `val`, and `test` splits

### Folder Structure Example

```
data/
  patches/
    train/
      inputs/
      masks/
    val/
    test/
```

---

## Fine-Tuning

Train the pixel-wise classifier using the Galileo encoder:

```bash
python 3_finetune_gfm.py \
  --data_dir data/patches/ \
  --encoder_ckpt data/models/nano/ \
  --save_dir checkpoints/ \
  --batch_size 8 \
  --epochs 50 \
  --lr 0.0001
```

To run in server background.

```
 nohup python 3_finetune_gfm.py \
  --data_dir /cluster01/Projects/USA_IDA_AICCRA/1.Data/FINAL/Galileo/data/patches/ \
  --encoder_ckpt models/nano/ \
  --save_dir /cluster01/Projects/USA_IDA_AICCRA/1.Data/FINAL/Galileo/data/checkpoints/ \
  --batch_size 8 \
  --epochs 50 \
  --lr 0.0001 \
  > finetune.log 2>&1 &
```

## Inference

Run inference on prepared raster tiles:

```bash
python 4_inferencing.py
```

This script will:

- Load each tile (60-band: 5 timesteps × 12 bands)
- Apply a sliding window to generate 8×8 predictions
- Merge predictions and output a GeoTIFF (`merged_prediction.tif`)

### Important:

- Your input tiles must have **60 bands** (5 months × 12 bands).
- Invalid or low-confidence pixels are masked using a configurable `confidence_threshold`.

---
