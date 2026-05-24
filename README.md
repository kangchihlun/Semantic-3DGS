# 3D 語意空間重建（Semantic 3DGS）系統架構規格書

---

## 目錄

1. [系統拓撲與資料流](#1-系統拓撲與資料流)
2. [前端網頁規格](#2-前端網頁規格)
3. [控制層與儲存層](#3-控制層與儲存層)
4. [後端 GPU 算力層規格](#4-後端-gpu-算力層規格)
5. [開發流程對接與工項分配](#5-開發流程對接與工項分配)

---

## 1. 系統拓撲與資料流

本系統採用「輕量前端 Web」與「觸發式 GPU Serverless 後端」架構，利用 **Splat-to-Mesh 逆向著色技術**，將 3DGS 語意場解耦成傳統 3D 引擎可讀取的幾何網格（Mesh）與自訂語意貼圖（Semantic ID Map）。

```
[ 用戶網頁前端 (Web) ]
       │
       ├── (1) 呼叫 Camera API 限制規格錄影
       ├── (2) 直傳（Presigned URL）原始影片
       └── (6) 載入 .glb + Semantic Shader 實作 3D 互動
       ▲
       │ (5) Webhook / 輪詢通知（下發 .glb 網格與語意 JSON）
       ▼
[ 雲端控制層 (API Gateway / Lambda) ] ───► 物件儲存 (S3 / OSS)
       │                                          ▲
       └─► (3) 喚醒 Serverless Container           │ (4) 讀寫大量數據
                 │                                ▼
           [ GPU Serverless 算力層 (RunPod / Modal / RTX 4090) ]
```

---

## 2. 前端網頁規格

前端基於 **TypeScript**、**Next.js**（或 Vite）與 **Three.js**（透過 `@react-three/fiber` 或原生 WebGL2）建構，主要負責引導錄影、高效率上傳與語意模型的實時互動渲染。

### 2.1 錄影引導模組

**API 限制**

- 透過 `MediaRecorder API` 調用後置鏡頭
- 強制設定：
  ```
  video: { width: 1920, height: 1080, frameRate: 30, facingMode: "environment" }
  ```
- 錄製格式：強制採用 `video/mp4` 或 `video/webm`（H.264 編碼）

**UI 引導機制**

- 畫面重疊半透明「慢速提示儀」，透過手機內建 `DeviceMotionEvent` 監測加速度
  - 若 $x, y, z$ 的加速度絕對值超過 $1.5\ m/s^2$，彈出警示「請慢速移動以防畫面模糊」
- 要求用戶環繞物體或空間 **60～180 秒**

---

### 2.2 大檔案直傳機制（Presigned URL Upload）

為防伺服器記憶體溢出（OOM）：

1. 前端向控制層請求 **S3 Presigned URL**
2. 使用 AWS SDK S3 Client 實作**分片上傳**（Multipart Upload，每片 5MB）
3. 支援斷點續傳
4. 上傳完成後發送訊號給控制層後端

---

### 2.3 3D 語意模型瀏覽器（Semantic 3D Viewer）

| 項目 | 技術 |
|------|------|
| 底層引擎 | Three.js + WebGL2 Renderer |
| 檔案載入 | `GLTFLoader`（含 Draco 幾何壓縮的 `.glb`） |

**核心邏輯：物件點擊與語意著色**

載入後的 Mesh 包含兩張貼圖：
- `Albedo_Map`：固有色貼圖
- `Semantic_ID_Map`：語意 ID 貼圖（RGB 值代表特定物件類別）

```javascript
// 三維空間點擊選取（Raycaster）實作
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);

    const intersects = raycaster.intersectObjects(scene.children, true);
    if (intersects.length > 0) {
        const hit = intersects[0];
        const uv = hit.uv; // 取得點擊點的 UV 座標

        // 透過 Custom Shader 或 WebGL 讀取該 UV 座標在 Semantic_ID_Map 上的 RGB 值
        const objectIdColor = getPixelColorFromTexture(hit.object.material.semanticMap, uv);
        const matchedObject = lookupSemanticJson(objectIdColor);

        if (matchedObject) {
            triggerUIHighlight(matchedObject);             // 觸發前端 UI 彈窗：顯示品牌與型號
            updateShaderHighlight(hit.object.material, objectIdColor); // 讓該物件在 3D 中發光
        }
    }
});
```

---

## 3. 控制層與儲存層

作為輕量化中介調度者，通常部署於 **AWS Lambda / Node.js** 環境。

### 3.1 儲存目錄結構

```
s3://spatial-reconstruction-bucket/
├── raw-videos/                   # 用戶上傳的原始 MP4 影片
│   └── [task_id].mp4
└── processed-models/             # 後端解算完成的成果
    └── [task_id]/
        ├── model.glb             # 經過 Draco 壓縮的 3D 網格（含色彩與語意貼圖）
        ├── semantic_meta.json    # 語意對照表
        └── camera_path.json      # 拍攝者軌跡資料
```

---

### 3.2 語意對照表規格（`semantic_meta.json`）

```json
{
  "task_id": "usr_998237_task_001",
  "generated_at": "2026-05-24T12:00:00Z",
  "detected_objects": [
    {
      "semantic_id_rgb": [255, 0, 0],
      "bounding_box_3d": {
        "center": [0.12, -0.45, 1.82],
        "size": [0.95, 0.85, 1.20]
      },
      "classification": {
        "brand": "IKEA",
        "model": "KIVIK Sofa",
        "confidence": 0.945
      }
    },
    {
      "semantic_id_rgb": [0, 255, 0],
      "bounding_box_3d": {
        "center": [-0.85, 0.10, 0.52],
        "size": [0.60, 0.75, 0.60]
      },
      "classification": {
        "brand": "Herman Miller",
        "model": "Aeron Chair",
        "confidence": 0.891
      }
    }
  ]
}
```

---

## 4. 後端 GPU 算力層規格

封裝成 Docker 鏡像，運行於 **Modal.com** 或 **RunPod Serverless**。

**硬體配置：** 1× NVIDIA RTX 4090（24GB VRAM）

### 4.1 核心環境與依賴（Dockerfile）

```dockerfile
FROM nvidia/cuda:11.8.0-devel-ubuntu22.04

# 安裝 Python 與 3D 幾何處理庫
RUN apt-get update && apt-get install -y python3-pip python3-dev colmap ffmpeg libgl1-mesa-glx
RUN pip3 install torch torchvision --index-url https://download.pytorch.org/whl/cu118

# 安裝 Nerfstudio 與 3DGS 相關擴充
RUN pip3 install nerfstudio
RUN pip3 install git+https://github.com/gaussian-splatting/gaussian-splatting.git
RUN pip3 install ultralytics open3d gltf-pipeline
```

---

### 4.2 核心解算腳本（`pipeline_worker.py`）

Serverless 容器被喚醒後自動執行以下流程：

```python
import os
import sys
import subprocess
import json
import torch
from ultralytics import YOLO
import open3d as o3d

def run_reconstruction_pipeline(task_id):
    video_path = f"/data/raw-videos/{task_id}.mp4"
    output_dir = f"/data/processed/{task_id}"
    os.makedirs(output_dir, exist_ok=True)

    # ── 步驟 1：影片抽樣與 COLMAP 相機軌跡解算 ──────────────────────
    print("[1/4] Running COLMAP feature extraction...")
    subprocess.run([
        "ns-process-data", "video",
        "--data", video_path,
        "--output-dir", f"{output_dir}/nerfstudio_format",
        "--num-frames-target", "300"
    ], check=True)

    # ── 步驟 2：2D 語意分割與自訂品牌物件特徵提取 ───────────────────
    print("[2/4] Running Custom Brand Object Detection (YOLOv11)...")
    images_dir = f"{output_dir}/nerfstudio_format/images"
    yolo_model = YOLO("/models/furniture_custom_v11.pt")

    semantic_results = {}
    for img_name in sorted(os.listdir(images_dir)):
        if not img_name.endswith(('.jpg', '.png')):
            continue
        img_path = os.path.join(images_dir, img_name)
        results = yolo_model(img_path, conf=0.6)
        semantic_results[img_name] = serialize_yolo_output(results)

    with open(f"{output_dir}/2d_semantics.json", "w") as f:
        json.dump(semantic_results, f)

    # ── 步驟 3：訓練 3D 語意高斯潑濺場（Semantic 3DGS） ─────────────
    print("[3/4] Training Semantic 3DGS Model...")
    subprocess.run([
        "ns-train", "splatfacto-semantic",
        "--data", f"{output_dir}/nerfstudio_format",
        "--output-dir", f"{output_dir}/training_outputs",
        "--max-num-iterations", "20000",
        "--vis", "none"
    ], check=True)

    # ── 步驟 4：Splat-to-Mesh 逆向著色烘焙與精簡化 ──────────────────
    print("[4/4] Converting Splat to Semantic Mesh & Simplification...")
    config_yaml = (
        f"{output_dir}/training_outputs/nerfstudio_format"
        "/splatfacto-semantic/config.yml"
    )

    # 導出含色彩貼圖與語意貼圖的幾何網格（Poisson 表面重建）
    subprocess.run([
        "ns-export", "poisson",
        "--load-config", config_yaml,
        "--output-dir", f"{output_dir}/raw_mesh",
        "--target-num-faces", "1000000"   # 初始生成 100 萬面
    ], check=True)

    # Open3D 二次誤差崩潰減面至 15 萬面
    mesh = o3d.io.read_triangle_mesh(f"{output_dir}/raw_mesh/mesh.obj")
    simplified_mesh = mesh.simplify_quadric_error_metrics(
        target_number_of_triangles=150000
    )
    o3d.io.write_triangle_mesh(f"{output_dir}/final_model.glb", simplified_mesh)

    # Google Draco 幾何壓縮
    subprocess.run([
        "gltf-pipeline",
        "-i", f"{output_dir}/final_model.glb",
        "-o", f"{output_dir}/model_compressed.glb",
        "-d"
    ], check=True)

    # ── 步驟 5：上傳成果與觸發 Webhook ──────────────────────────────
    upload_to_s3(
        f"{output_dir}/model_compressed.glb",
        f"processed-models/{task_id}/model.glb"
    )
    generate_and_upload_metadata_json(task_id, semantic_results, output_dir)
    trigger_frontend_webhook(task_id)
    print(f"Task {task_id} successfully finished.")

if __name__ == "__main__":
    task_id = os.environ.get("RECONSTRUCTION_TASK_ID")
    run_reconstruction_pipeline(task_id)
```

**Pipeline 流程摘要**

| 步驟 | 任務 | 關鍵工具 |
|------|------|---------|
| 1 | 影片抽樣 & 相機軌跡解算 | COLMAP via `ns-process-data` |
| 2 | 2D 語意分割 & 品牌辨識 | YOLOv11（自訂模型） |
| 3 | 訓練 Semantic 3DGS | `ns-train splatfacto-semantic`（20,000 iter）|
| 4 | Splat→Mesh 減面 & 壓縮 | Open3D + gltf-pipeline（Draco） |
| 5 | 上傳 S3 & 觸發 Webhook | boto3 / S3 API |

---

## 5. 開發流程對接與工項分配

讀取本規格書後，團隊應拆分以下三個**併行工項**立刻開工：

### 工項 A｜前端開發

- 實作 WebRTC / MediaRecorder 相機引導錄影介面
- 在 Three.js 中實作自訂 `ShaderMaterial`
  - Shader 須接受兩張貼圖：`map`（Albedo）與 `semanticMap`
  - 在滑鼠 hover / 點擊對應 UV 時，動態改變遮罩強度以實現 **Highlight** 外觀

### 工項 B｜深度學習與後端

- 收集品牌家具多角度照片
- 使用 Ultralytics 框架訓練自訂 **YOLOv11 分割模型**
- 確保模型輸出格式可透過幾何射線投影到 3D 點雲空間

### 工項 C｜算力運維

- 依第 4 節 Dockerfile 在 Modal.com 或 RunPod 上建立 **Webhook 觸發機制**
- 將冷啟動（Cold Start）時間優化至 **15 秒內**
