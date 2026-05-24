# 3D 語意空間重建（Semantic 3DGS）系統架構規格書

---

## 目錄

0. [系統目標與使用者旅程](#0-系統目標與使用者旅程)
1. [系統架構概覽](#1-系統架構概覽)
2. [前端網頁規格](#2-前端網頁規格)
3. [控制層與儲存層](#3-控制層與儲存層)
4. [後端 GPU 算力層規格](#4-後端-gpu-算力層規格)
5. [開發流程對接與工項分配](#5-開發流程對接與工項分配)

---

## 0. 系統目標與使用者旅程

### 系統目標

讓裝潢完工的房間可以在手機上被數位化成**可互動的 3D 語意模型**。任何人打開分享連結，就能在瀏覽器中自由瀏覽空間，**點擊任何物件即可查詢完整的 BIM 規格資訊**（製造商、型號、年份、材質、尺寸等）。

### 使用者旅程

```
【前置：BIM 物件建檔】            （設計師 / 施工方負責）
  │
  ├─ 開啟 BIM 錄入介面
  ├─ 對每件物品多角度拍照（20-50 張）
  ├─ 填寫 BIM 屬性（IFC 類別、品牌、型號、年份、規格…）
  └─ 上傳 → 系統建立視覺識別特徵 → 物件存入 BIM 物件庫
             （每件約 1-2 分鐘）

【主流程：空間數位化】            （業主 / 驗收方負責）
  │
  ├─ 開啟空間掃描介面
  ├─ 以手機環繞錄影整個房間（60-180 秒）
  ├─ 自動上傳 → 雲端 GPU 解算（約 15-30 分鐘）
  └─ 收到通知後開啟房間連結

【瀏覽 & 分享】                   （業主、監造、維修人員等）
  │
  ├─ 手機瀏覽器開啟房間連結，進入 3D 場景
  ├─ 手指拖曳旋轉視角、雙指縮放
  ├─ 點擊（Tap）任意物件 → 彈出 BIM 資訊卡
  │     顯示：品牌、型號、年份、材質、尺寸、產品頁連結…
  └─ 可複製分享連結傳給其他人，無需登入即可瀏覽
```

### 兩條流程的依賴關係

```
BIM 物件建檔（流程 A）→ 必須先完成 → 主空間掃描（流程 B）
                          ↑
              掃描時系統查詢此庫，才能識別出 BIM 物件
```

---

## 1. 系統架構概覽

### 1.1 流程 A：BIM 物件預建庫

```
[ 前端：BIM 錄入介面 (/bim-register) ]
       │
       ├── (A1) 單一物件多角度連拍（20~50 張 JPEG）
       ├── (A2) 填寫 BIM 屬性表單（IFC 類別、品牌、規格…）
       └── (A3) 分片上傳至 S3 bim-staging/[bim_id]/
       │
       ▼
[ 控制層：POST /bim/register → 觸發 BIM Job ]
       │
       ▼
[ GPU Worker：bim_register_worker ]
       ├── DINOv2 萃取每張照片的視覺特徵向量（512-dim）
       ├── 多角度向量取平均 → 單一物件 Embedding
       └── 寫入 Qdrant Vector DB + S3 bim-library/
```

### 1.2 流程 B：主空間重建

```
[ 前端：空間掃描介面 (/scan) ]
       │
       ├── (B1) MediaRecorder 引導錄影（1080p / 30fps）
       ├── (B2) 分片上傳（Presigned URL）→ S3 raw-videos/
       └── (B6) 房間瀏覽器：載入 .glb，手機 Touch 互動
                （Tap 物件 → 讀 Semantic_ID_Map → 查 semantic_meta → 彈出 BIM 卡）
       ▲
       │ (B5) Webhook 通知「處理完成」+ 房間 URL
       ▼
[ 控制層：API Gateway / Lambda ]
       │                    └─────────────────────► S3 / CloudFront CDN
       └── (B3) 喚醒 GPU Serverless Container              ▲
                     │                                     │ (B4) 讀寫
                     ▼                                     ▼
             [ GPU Worker：pipeline_worker ]     [ Qdrant Vector DB ]
                     │                                     ▲
                     └── 查詢 BIM 庫，識別場景物件 ────────┘
```

### 1.3 核心技術鏈

```
影片
 │
 ├─[COLMAP]──────────────────── 相機姿態（poses）
 │
 ├─[YOLO 偵測 + DINOv2 比對]── 每幀 BIM 物件標籤（bim_id per bbox）
 │
 ├─[語意遮罩生成]──────────────  每幀 Semantic Mask（像素 → label_index）
 │
 ├─[Semantic 3DGS 訓練]────────  每顆 Gaussian 帶有語意 logits
 │
 ├─[Splat → Mesh]─────────────  幾何網格（OBJ）
 │
 ├─[語意貼圖烘焙]──────────────  Semantic_ID_Map（UV 貼圖，每像素 RGB = label）
 │
 └─[Draco 壓縮]────────────────  model.glb（含 Albedo_Map + Semantic_ID_Map）
```

---

## 2. 前端網頁規格

前端基於 **TypeScript + Next.js（或 Vite）+ Three.js** 建構，設計目標以**手機瀏覽器**為主要執行環境。

| 路由 | 功能 |
|------|------|
| `/bim-register` | BIM 物件錄入（流程 A） |
| `/scan` | 空間錄影與上傳（流程 B） |
| `/room/[task_id]` | 3D 語意瀏覽器，可分享連結 |

---

### 2.1 BIM 物件錄入介面（`/bim-register`）

**拍照引導**

- 使用 `MediaStream API` 調用後置鏡頭（靜態連拍，非錄影），每次拍攝存為 JPEG
- Overlay 顯示「環繞進度環」（360° 分 8 象限），標示哪些角度尚未覆蓋
- 建議拍攝量：**20～50 張**（正面、側面、背面、俯視各至少 1 張）
- 若 `DeviceMotionEvent` 偵測到加速度 $> 1.5\ m/s^2$，彈出「請靜止後再拍攝」

**BIM 屬性表單**

```
┌──────────────────────────────────────┐
│  新增 BIM 物件                        │
├──────────────────────────────────────┤
│  IFC 類別 *     [ 下拉選單 ▼ ]        │
│                 IfcFurnishingElement   │
│                 IfcDoor / IfcWindow    │
│                 IfcWall / IfcSlab      │
│                 IfcEquipmentElement    │
│                                       │
│  品牌 *         [__________________]  │
│  型號 *         [__________________]  │
│  出廠年份       [____]                │
│  材質           [__________________]  │
│  顏色           [__________________]  │
│  尺寸 (cm)      W[___] D[___] H[___]  │
│                                       │
│  自訂屬性       ＋ 新增 Key / Value    │
│  產品頁面 URL   [__________________]  │
└──────────────────────────────────────┘
   [ 上傳並建立 BIM 物件 ]
```

**資料送出流程**

1. 前端向控制層請求 S3 Presigned URL（路徑：`bim-staging/[bim_id]/`）
2. 分片上傳所有照片（每片 5 MB）
3. POST BIM 屬性 JSON 至 `POST /bim/register`
4. 輪詢或 Webhook 等待處理完成，顯示「物件已建立：`bim_id`」確認卡片

---

### 2.2 空間錄影引導（`/scan`）

**錄影規格**

```javascript
const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: 1920, height: 1080, frameRate: 30, facingMode: "environment" }
});
const recorder = new MediaRecorder(stream, { mimeType: "video/mp4; codecs=avc1" });
```

**UI 引導機制**

- 畫面疊加半透明「慢速提示儀」，偵測 `DeviceMotionEvent`
  - 加速度絕對值 $> 1.5\ m/s^2$ → 彈出「請放慢移動速度」
- 底部進度條顯示錄製時長，建議 **60～180 秒**

---

### 2.3 大檔案直傳（Presigned URL Multipart Upload）

1. 向控制層請求 S3 Presigned URL
2. AWS SDK 分片上傳（每片 5 MB），支援斷點續傳
3. 上傳完成後 POST 通知控制層，控制層喚醒 GPU Worker

---

### 2.4 3D 語意瀏覽器（`/room/[task_id]`，手機優先）

| 項目 | 技術 |
|------|------|
| 底層引擎 | Three.js + WebGL2 |
| 模型載入 | `GLTFLoader` + Draco 解壓 |
| 互動方式 | `touchstart` / `touchend`（手機）+ `click`（桌面）|
| 進場動畫 | 淡入 + 相機從上往下推進 |

**Mesh 貼圖結構**

每個物件的 `MeshStandardMaterial` 包含兩張貼圖：

| 貼圖 | 用途 |
|------|------|
| `Albedo_Map` | 固有色（正常外觀顯示） |
| `Semantic_ID_Map` | 語意標籤（每像素 RGB = label_index 編碼） |

`Semantic_ID_Map` 僅在前端查詢時讀取像素值，**不參與渲染**（不影響視覺外觀）。

**手機 Tap → BIM 資訊查詢**

```javascript
const raycaster = new THREE.Raycaster();

function onTap(clientX, clientY) {
    const ndcX =  (clientX / window.innerWidth)  * 2 - 1;
    const ndcY = -(clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera({ x: ndcX, y: ndcY }, camera);

    const hits = raycaster.intersectObjects(scene.children, true);
    if (!hits.length) return;

    const { uv, object } = hits[0];
    const semanticRgb = sampleTexture(object.material.semanticMap, uv);
    const bimEntry    = lookupBimByRgb(semanticRgb, semanticMeta);

    if (bimEntry) {
        highlightObject(object, semanticRgb);  // Fragment shader 高亮
        showBimCard(bimEntry);                 // 彈出 BIM 資訊卡
    }
}

// 統一處理手機 touch 與桌面 click
window.addEventListener('touchend', (e) => {
    const t = e.changedTouches[0];
    onTap(t.clientX, t.clientY);
});
window.addEventListener('click', (e) => onTap(e.clientX, e.clientY));
```

**BIM 資訊卡規格（底部 Sheet）**

```
┌────────────────────────────────────┐
│  ╳                       [ 分享 ]  │  ← 頂部工具列
├────────────────────────────────────┤
│  ● 品牌     IKEA                   │
│  ● 型號     KIVIK Sofa             │
│  ● IFC 類別 IfcFurnishingElement   │
│  ● 材質     Fabric / Beige         │
│  ● 出廠年份 2024                   │
│  ● 尺寸     W95 × D85 × H120 cm   │
│  ── 自訂屬性 ──────────────────── │
│  ● 採購編號 PO-2024-0312           │
│  ● 保固到期 2027-03-12             │
├────────────────────────────────────┤
│  [ 開啟產品頁面 ↗ ]                │
└────────────────────────────────────┘
```

**房間分享機制**

- 每個房間有固定 URL：`https://app.example.com/room/[task_id]`
- 連結**無需登入**即可瀏覽（read-only 公開存取）
- 房間擁有者可在設定中將連結切換為「私密」（需輸入存取碼）
- 分享按鈕呼叫 `navigator.share()` API（手機原生分享選單）

---

## 3. 控制層與儲存層

部署於 **AWS Lambda / Node.js**，僅做輕量調度，不做運算。

### 3.1 儲存目錄結構

```
s3://spatial-reconstruction-bucket/
│
├── bim-staging/                       # 流程 A：BIM 錄入暫存照片
│   └── [bim_id]/
│       ├── img_001.jpg  …
│
├── bim-library/                       # 流程 A：BIM 物件庫（處理完成）
│   └── [bim_id]/
│       ├── bim_meta.json              # BIM 屬性（IFC 欄位）
│       └── embedding.npy              # 視覺特徵向量（512-dim float32）
│
├── raw-videos/                        # 流程 B：原始錄影
│   └── [task_id].mp4
│
└── processed-models/                  # 流程 B：重建成果（透過 CloudFront CDN 下發）
    └── [task_id]/
        ├── model.glb                  # Draco 壓縮網格（含 Albedo + Semantic_ID_Map）
        ├── bim_label_palette.json     # label_index ↔ RGB ↔ bim_id 對照表
        ├── semantic_meta.json         # 語意對照表（含完整 BIM 屬性）
        └── camera_path.json           # 拍攝者軌跡
```

> `processed-models/` 透過 **CloudFront CDN** 下發，前端直接從 CDN URL 載入 `.glb`，不走 Lambda。

---

### 3.2 BIM 物件庫條目（`bim_meta.json`）

```json
{
  "bim_id": "bim_ikea_kivik_sofa_001",
  "registered_at": "2026-05-25T10:00:00Z",
  "ifc_class": "IfcFurnishingElement",
  "properties": {
    "brand": "IKEA",
    "model": "KIVIK Sofa",
    "year": 2024,
    "material": "Fabric",
    "color": "Beige",
    "dimensions_cm": { "width": 95, "depth": 85, "height": 120 },
    "product_url": "https://www.ikea.com/...",
    "custom": {
      "採購編號": "PO-2024-0312",
      "保固到期": "2027-03-12"
    }
  },
  "reference_image_count": 32,
  "embedding_path": "bim-library/bim_ikea_kivik_sofa_001/embedding.npy"
}
```

---

### 3.3 語意標籤色盤（`bim_label_palette.json`）

> **關鍵橋樑**：定義 `label_index`（整數）→ `semantic_id_rgb`（RGB）→ `bim_id` 的三向對照。此對照表由 pipeline_worker 在步驟 2 完成後產生，供 3DGS 訓練、貼圖烘焙、前端查詢三端共同使用。

```json
{
  "task_id": "usr_998237_task_001",
  "background": { "label_index": 0, "rgb": [0, 0, 0] },
  "objects": [
    {
      "label_index": 1,
      "rgb": [47, 131, 211],
      "bim_id": "bim_ikea_kivik_sofa_001"
    },
    {
      "label_index": 2,
      "rgb": [94, 6, 166],
      "bim_id": "bim_hm_aeron_chair_001"
    },
    {
      "label_index": 3,
      "rgb": [141, 137, 121],
      "bim_id": null,
      "note": "偵測到物件但未比對到 BIM 庫"
    }
  ]
}
```

**RGB 編碼規則**

```python
# label_index → 固定 RGB（避免顏色衝突，使用三個互質的步長）
def label_to_rgb(label_index: int) -> tuple[int, int, int]:
    return (
        (label_index * 47)  % 256,
        (label_index * 131) % 256,
        (label_index * 211) % 256,
    )
```

---

### 3.4 語意對照表（`semantic_meta.json`）

```json
{
  "task_id": "usr_998237_task_001",
  "generated_at": "2026-05-25T12:00:00Z",
  "detected_objects": [
    {
      "label_index": 1,
      "semantic_id_rgb": [47, 131, 211],
      "bim_id": "bim_ikea_kivik_sofa_001",
      "match_confidence": 0.945,
      "bounding_box_3d": {
        "center": [0.12, -0.45, 1.82],
        "size": [0.95, 0.85, 1.20]
      },
      "bim_properties": {
        "ifc_class": "IfcFurnishingElement",
        "brand": "IKEA",
        "model": "KIVIK Sofa",
        "year": 2024,
        "material": "Fabric",
        "dimensions_cm": { "width": 95, "depth": 85, "height": 120 }
      }
    },
    {
      "label_index": 2,
      "semantic_id_rgb": [94, 6, 166],
      "bim_id": "bim_hm_aeron_chair_001",
      "match_confidence": 0.891,
      "bounding_box_3d": {
        "center": [-0.85, 0.10, 0.52],
        "size": [0.60, 0.75, 0.60]
      },
      "bim_properties": {
        "ifc_class": "IfcFurnishingElement",
        "brand": "Herman Miller",
        "model": "Aeron Chair",
        "year": 2023,
        "material": "Mesh"
      }
    },
    {
      "label_index": 3,
      "semantic_id_rgb": [141, 137, 121],
      "bim_id": null,
      "match_confidence": 0.0,
      "bim_properties": null
    }
  ]
}
```

---

## 4. 後端 GPU 算力層規格

封裝成 Docker 鏡像，部署於 **Modal.com** 或 **RunPod Serverless**（1× RTX 4090，24GB VRAM）。

### 4.1 Dockerfile

```dockerfile
FROM nvidia/cuda:11.8.0-devel-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3-pip python3-dev colmap ffmpeg libgl1-mesa-glx xvfb

RUN pip3 install torch torchvision --index-url https://download.pytorch.org/whl/cu118

# 3DGS 重建
RUN pip3 install nerfstudio
RUN pip3 install git+https://github.com/gaussian-splatting/gaussian-splatting.git
RUN pip3 install ultralytics open3d gltf-pipeline

# BIM 特徵提取 & 向量資料庫
RUN pip3 install transformers qdrant-client

# 工具
RUN pip3 install Pillow numpy boto3 xatlas
```

---

### 4.2 BIM 特徵提取 Worker（`bim_register_worker.py`）

```python
import os
import json
import numpy as np
import boto3
from PIL import Image
from transformers import AutoProcessor, AutoModel
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct

QDRANT_URL    = os.environ["QDRANT_URL"]
QDRANT_APIKEY = os.environ["QDRANT_APIKEY"]
COLLECTION    = "bim_objects"
BUCKET        = "spatial-reconstruction-bucket"

def extract_embedding(image_paths: list[str]) -> np.ndarray:
    processor = AutoProcessor.from_pretrained("facebook/dinov2-base")
    model     = AutoModel.from_pretrained("facebook/dinov2-base").cuda()

    embeddings = []
    for path in image_paths:
        img    = Image.open(path).convert("RGB")
        inputs = processor(images=img, return_tensors="pt").to("cuda")
        feat   = model(**inputs).last_hidden_state[:, 0, :].cpu().detach().numpy()
        embeddings.append(feat[0])

    return np.mean(embeddings, axis=0)  # 多角度平均，得到穩定物件向量

def register_bim_object(bim_id: str):
    s3          = boto3.client("s3")
    local_dir   = f"/tmp/bim_staging/{bim_id}"
    os.makedirs(local_dir, exist_ok=True)

    resp = s3.list_objects_v2(Bucket=BUCKET, Prefix=f"bim-staging/{bim_id}/")
    image_paths = []
    for obj in resp.get("Contents", []):
        local = os.path.join(local_dir, os.path.basename(obj["Key"]))
        s3.download_file(BUCKET, obj["Key"], local)
        image_paths.append(local)

    embedding = extract_embedding(image_paths)

    # 存 S3
    npy_local = f"/tmp/{bim_id}.npy"
    np.save(npy_local, embedding)
    s3.upload_file(npy_local, BUCKET, f"bim-library/{bim_id}/embedding.npy")

    # 寫 Qdrant
    client = QdrantClient(url=QDRANT_URL, api_key=QDRANT_APIKEY)
    client.upsert(
        collection_name=COLLECTION,
        points=[PointStruct(
            id=abs(hash(bim_id)) % (2**63),
            vector=embedding.tolist(),
            payload={"bim_id": bim_id}
        )]
    )
    print(f"[BIM] Registered {bim_id} with {len(image_paths)} images.")

if __name__ == "__main__":
    register_bim_object(os.environ["BIM_REGISTER_ID"])
```

---

### 4.3 主重建解算腳本（`pipeline_worker.py`）

```python
import os
import json
import subprocess
import numpy as np
import boto3
import open3d as o3d
from ultralytics import YOLO
from transformers import AutoProcessor, AutoModel
from qdrant_client import QdrantClient
from PIL import Image, ImageDraw

QDRANT_URL    = os.environ["QDRANT_URL"]
QDRANT_APIKEY = os.environ["QDRANT_APIKEY"]
BIM_THRESHOLD = 0.82
BUCKET        = "spatial-reconstruction-bucket"


def label_to_rgb(label_index: int) -> tuple[int, int, int]:
    return (
        (label_index * 47)  % 256,
        (label_index * 131) % 256,
        (label_index * 211) % 256,
    )


def build_bim_palette(detections_by_frame: dict) -> dict:
    """從所有幀的偵測結果建立 bim_id → label_index 對照表。"""
    bim_ids = set()
    for dets in detections_by_frame.values():
        for d in dets:
            if d["bim_match"]:
                bim_ids.add(d["bim_match"]["bim_id"])

    palette = {}
    for idx, bim_id in enumerate(sorted(bim_ids), start=1):
        palette[bim_id] = {"label_index": idx, "rgb": list(label_to_rgb(idx))}

    return palette  # {bim_id: {"label_index": int, "rgb": [R,G,B]}}


def query_bim_library(crop: Image.Image, qdrant, dino_processor, dino_model) -> dict | None:
    inputs    = dino_processor(images=crop, return_tensors="pt").to("cuda")
    embedding = dino_model(**inputs).last_hidden_state[:, 0, :].cpu().detach().numpy()[0]
    results   = qdrant.search(
        collection_name="bim_objects",
        query_vector=embedding.tolist(),
        limit=1, with_payload=True
    )
    if results and results[0].score >= BIM_THRESHOLD:
        return {"bim_id": results[0].payload["bim_id"], "confidence": float(results[0].score)}
    return None


def run_reconstruction_pipeline(task_id: str):
    video_path = f"/data/raw-videos/{task_id}.mp4"
    output_dir = f"/data/processed/{task_id}"
    os.makedirs(output_dir, exist_ok=True)

    s3 = boto3.client("s3")

    # ── 步驟 1：影片抽樣與 COLMAP 相機軌跡解算 ──────────────────────
    print("[1/6] Running COLMAP...")
    subprocess.run([
        "ns-process-data", "video",
        "--data", video_path,
        "--output-dir", f"{output_dir}/nerfstudio_format",
        "--num-frames-target", "300"
    ], check=True)

    images_dir = f"{output_dir}/nerfstudio_format/images"

    # ── 步驟 2：2D 偵測 + BIM 物件庫比對 ───────────────────────────
    print("[2/6] Detecting objects & matching BIM library...")
    yolo_model     = YOLO("/models/furniture_generic_v11.pt")
    dino_processor = AutoProcessor.from_pretrained("facebook/dinov2-base")
    dino_model     = AutoModel.from_pretrained("facebook/dinov2-base").cuda()
    qdrant         = QdrantClient(url=QDRANT_URL, api_key=QDRANT_APIKEY)

    detections_by_frame = {}
    img_names = sorted(f for f in os.listdir(images_dir) if f.endswith(('.jpg', '.png')))

    for img_name in img_names:
        img_path  = os.path.join(images_dir, img_name)
        full_img  = Image.open(img_path).convert("RGB")
        results   = yolo_model(img_path, conf=0.5)
        frame_dets = []
        for box in results[0].boxes:
            x1, y1, x2, y2 = map(int, box.xyxy[0].tolist())
            crop      = full_img.crop((x1, y1, x2, y2))
            bim_match = query_bim_library(crop, qdrant, dino_processor, dino_model)
            frame_dets.append({"bbox_2d": [x1, y1, x2, y2], "bim_match": bim_match})
        detections_by_frame[img_name] = frame_dets

    with open(f"{output_dir}/2d_semantics.json", "w") as f:
        json.dump(detections_by_frame, f)

    # ── 步驟 3：建立 BIM Label Palette ──────────────────────────────
    print("[3/6] Building BIM label palette...")
    bim_palette = build_bim_palette(detections_by_frame)

    bim_id_to_label = {bim_id: v["label_index"] for bim_id, v in bim_palette.items()}

    palette_json = {
        "task_id": task_id,
        "background": {"label_index": 0, "rgb": [0, 0, 0]},
        "objects": [
            {"label_index": v["label_index"], "rgb": v["rgb"], "bim_id": bim_id}
            for bim_id, v in bim_palette.items()
        ]
    }
    with open(f"{output_dir}/bim_label_palette.json", "w") as f:
        json.dump(palette_json, f)

    # ── 步驟 4：為每幀生成語意遮罩（監督 3DGS 訓練用）──────────────
    #
    #   每幀輸出一張 Semantic Mask（灰階 PNG），像素值 = label_index。
    #   遮罩與原始影像同名存放於 semantic_masks/ 目錄，
    #   供 splatfacto-semantic 讀取作為訓練監督訊號。
    #
    print("[4/6] Generating per-frame semantic masks...")
    masks_dir = f"{output_dir}/nerfstudio_format/semantic_masks"
    os.makedirs(masks_dir, exist_ok=True)

    for img_name, frame_dets in detections_by_frame.items():
        src   = Image.open(os.path.join(images_dir, img_name))
        w, h  = src.size
        mask  = Image.new("L", (w, h), 0)  # 背景 = 0
        draw  = ImageDraw.Draw(mask)

        for det in frame_dets:
            if det["bim_match"]:
                label_idx = bim_id_to_label.get(det["bim_match"]["bim_id"], 0)
            else:
                label_idx = len(bim_palette) + 1  # 未知物件給最後一個標籤
            x1, y1, x2, y2 = det["bbox_2d"]
            draw.rectangle([x1, y1, x2, y2], fill=label_idx)

        mask.save(os.path.join(masks_dir, img_name.replace(".jpg", ".png")))

    # ── 步驟 5：訓練 Semantic 3DGS ──────────────────────────────────
    #
    #   splatfacto-semantic 讀取 semantic_masks/ 作為監督，
    #   訓練後每顆 Gaussian 帶有語意 logits（維度 = 標籤數）。
    #
    print("[5/6] Training Semantic 3DGS...")
    subprocess.run([
        "ns-train", "splatfacto-semantic",
        "--data", f"{output_dir}/nerfstudio_format",
        "--output-dir", f"{output_dir}/training_outputs",
        "--max-num-iterations", "20000",
        "--vis", "none"
    ], check=True)

    config_yaml = (
        f"{output_dir}/training_outputs/nerfstudio_format"
        "/splatfacto-semantic/config.yml"
    )

    # ── 步驟 6：Splat → Mesh + 語意貼圖烘焙 + Draco 壓縮 ───────────
    #
    #   6a. Poisson 表面重建 → raw mesh（100 萬面）
    #   6b. Open3D 減面 → 15 萬面
    #   6c. 從訓練相機視角渲染語意場 → 投影到 mesh UV → Semantic_ID_Map
    #       （每像素 RGB = label_to_rgb(argmax(semantic_logits))）
    #   6d. Draco 壓縮 → model.glb（Albedo_Map + Semantic_ID_Map）
    #
    print("[6/6] Splat → Mesh, bake Semantic_ID_Map, compress...")

    # 6a: 幾何導出
    subprocess.run([
        "ns-export", "poisson",
        "--load-config", config_yaml,
        "--output-dir", f"{output_dir}/raw_mesh",
        "--target-num-faces", "1000000"
    ], check=True)

    # 6b: 減面
    mesh       = o3d.io.read_triangle_mesh(f"{output_dir}/raw_mesh/mesh.obj")
    simplified = mesh.simplify_quadric_error_metrics(target_number_of_triangles=150000)
    o3d.io.write_triangle_mesh(f"{output_dir}/simplified_mesh.obj", simplified)

    # 6c: 從已訓練模型渲染語意通道（各訓練相機視角）
    subprocess.run([
        "ns-render", "dataset",
        "--load-config", config_yaml,
        "--output-path", f"{output_dir}/semantic_renders",
        "--rendered-output-names", "semantics",
    ], check=True)

    # 6c（續）: 使用 xatlas UV 展開 + 將語意渲染投影烘焙成 Semantic_ID_Map 貼圖
    #           每像素 RGB = label_to_rgb(argmax(semantic_logits)) per 3DGS
    bake_semantic_texture(
        mesh_path       = f"{output_dir}/simplified_mesh.obj",
        semantic_renders= f"{output_dir}/semantic_renders",
        palette         = palette_json,
        output_path     = f"{output_dir}/semantic_id_map.png"
    )

    # 6d: 打包成 GLB 並 Draco 壓縮
    pack_glb_with_textures(
        mesh_path    = f"{output_dir}/simplified_mesh.obj",
        albedo_path  = f"{output_dir}/raw_mesh/albedo.png",
        semantic_path= f"{output_dir}/semantic_id_map.png",
        output_path  = f"{output_dir}/model.glb"
    )
    subprocess.run([
        "gltf-pipeline",
        "-i", f"{output_dir}/model.glb",
        "-o", f"{output_dir}/model_compressed.glb",
        "-d"
    ], check=True)

    # 上傳成果
    s3.upload_file(f"{output_dir}/model_compressed.glb",
                   BUCKET, f"processed-models/{task_id}/model.glb")
    s3.upload_file(f"{output_dir}/bim_label_palette.json",
                   BUCKET, f"processed-models/{task_id}/bim_label_palette.json")
    generate_and_upload_semantic_meta(task_id, detections_by_frame, bim_palette, output_dir)
    trigger_frontend_webhook(task_id)
    print(f"Task {task_id} done.")

if __name__ == "__main__":
    run_reconstruction_pipeline(os.environ["RECONSTRUCTION_TASK_ID"])
```

**Pipeline 流程摘要**

| 步驟 | 任務 | 關鍵工具 |
|------|------|---------|
| 1 | 影片抽樣 & 相機軌跡 | COLMAP via `ns-process-data` |
| 2 | 2D 偵測 + BIM 庫比對 | YOLOv11 + DINOv2 + Qdrant |
| 3 | 建立 BIM Label Palette | `label_to_rgb()` + `bim_label_palette.json` |
| 4 | 每幀語意遮罩生成 | Pillow + bbox → 灰階 mask PNG |
| 5 | 訓練 Semantic 3DGS | `ns-train splatfacto-semantic`（20,000 iter）|
| 6 | Splat→Mesh + 貼圖烘焙 + 壓縮 | Open3D + xatlas + gltf-pipeline（Draco）|

---

## 5. 開發流程對接與工項分配

### 整體開發時序

```
Phase 1（並行）：工項 A + 工項 B + 工項 C
Phase 2（串行）：工項 D（依賴 Phase 1 全部完成）
```

---

### 工項 A｜前端開發

**BIM 錄入介面（`/bim-register`）**
- 多角度連拍 + 360° 象限進度 Overlay
- IFC 表單（類別下拉、自訂 Key-Value 欄、出廠年份）
- 分片上傳 + 輪詢建立狀態

**空間掃描介面（`/scan`）**
- MediaRecorder 引導錄影 + 慢速提示儀
- 分片上傳

**房間瀏覽器（`/room/[task_id]`）**
- Three.js + WebGL2，`touchend` + `click` 雙模點擊
- Custom Fragment Shader（接受 Albedo_Map + Semantic_ID_Map）
- UV 取樣 → RGB → 查 `semantic_meta.json` → BIM 資訊卡底部 Sheet
- `navigator.share()` 分享 + 公開/私密連結切換

---

### 工項 B｜BIM 物件庫建立

- 架設 **Qdrant**（Cloud 或 Self-hosted），建立 `bim_objects` Collection（COSINE 距離，512 維）
- 實作並驗證 `bim_register_worker.py`
- 收集首批物件照片（每物件 ≥ 20 張）並完成建庫驗證

---

### 工項 C｜深度學習模型

- 訓練通用家具/設備偵測 **YOLOv11 模型**（輸出 bounding box，取得 ROI 供 DINOv2 裁切）
- 驗證偵測框格式與步驟 4 語意遮罩生成的相容性

---

### 工項 D｜算力運維（依賴 A + B + C）

- 在 Modal.com 或 RunPod 上部署兩條 Webhook 觸發 Job：
  - `bim_register_worker`（流程 A）
  - `pipeline_worker`（流程 B）
- 兩條 Job 冷啟動（Cold Start）各優化至 **15 秒內**
- 設定環境變數：`QDRANT_URL`、`QDRANT_APIKEY`、`AWS_*`
- 設定 CloudFront CDN 指向 `processed-models/`，確保前端可直接串流 `.glb`
