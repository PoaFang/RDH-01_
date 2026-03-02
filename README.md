# RDH 可逆資訊隱藏系統（基礎功能版）

本專案完成你要求的「系統基本功能」與完整流程，且第 4/8 步已改為直方圖位移法（Histogram Shifting）。

## 目前完成內容（對應你的 10 個步驟）

1. 讀入影像檔案。
2. 自動判斷灰階/彩色，彩色會自動轉灰階。
3. 將影像切成多個 8x8 區塊。
4. 對每個區塊執行 RDH 直方圖位移運算，嵌入從 1 開始遞增資料；同時計算 BPP。
5. 將區塊重組為 stego 圖。
6. 計算 stego 圖與原圖 PSNR。
7. 將 stego 圖再切成 8x8 區塊。
8. 對各區塊做直方圖位移逆運算，取回資料並驗證正確性。
9. 將還原區塊重組為 restored 圖。
10. 計算 restored 圖與原圖 PSNR。

所有步驟都會印出結果。

## 專案結構

```text
rdh_project/
├─ requirements.txt
├─ README.md
├─ src/
│  ├─ main.py
│  ├─ gui.py
│  └─ rdh_system/
│     ├─ __init__.py
│     ├─ algorithms/
│     │  ├─ __init__.py
│     │  ├─ base.py
│     │  ├─ histogram_shifting.py
│     │  └─ registry.py
│     ├─ image_io.py
│     ├─ block_ops.py
│     ├─ metrics.py
│     └─ pipeline.py
└─ tests/
   └─ self_check.py
```

## 各程式段落說明（繁體中文）

- `image_io.py`：
  負責檔案讀寫。讀入時會判斷影像模式，若不是 `L` 就自動轉灰階，並回傳轉換資訊（原始模式、是否彩色）。

- `block_ops.py`：
  負責切塊與重組。`split_into_blocks` 會把影像依 8x8 切分並記錄座標；`merge_blocks` 依座標把區塊拼回完整影像。

- `metrics.py`：
  提供三種 BPP 指標與 PSNR：
  - `payload_bpp`：只計秘密資料位元數。
  - `side_info_bpp`：估算可逆所需輔助資訊成本。
  - `net_bpp`：`payload_bpp - side_info_bpp`。
  並附註三者的公式、原理與理論值範圍。

- `algorithms/`：
  採用可擴充演算法架構。每種 RDH 演算法都可獨立放在這裡，定義自己的參數規格（給 UI 動態產生欄位）與嵌入/逆運算邏輯。
  目前已實作 `histogram_shifting.py` 兩種演算法：
  - `histogram_shifting`（變長 payload）
  - `histogram_shifting_fixed_length`（固定長度 payload，可提高 payload BPP）
  - `histogram_shifting_multi_peak`（多峰直方圖位移，固定長度 payload，可進一步提高 payload BPP）
  - `histogram_shifting_multi_layer`（多層 + 多峰直方圖位移，高容量版本，可衝高到 2.5+ BPP）
  三者都提供位移層級 `0 / ±1 / ±2 / ±3`。

- `pipeline.py`：
  核心流程控制器，完整串起你要求的 10 個步驟並逐步列印結果，包含 BPP 與兩次 PSNR 計算、資料一致性驗證。
  現在支援 `algorithm_name + algorithm_params`，未來新增演算法時不用改流程架構。

- `gui.py`：
  圖形介面。可選圖檔、選演算法、調整演算法參數（目前可選直方圖位移層級 0/±1/±2/±3），所有步驟輸出都在訊息框顯示。

- `main.py`：
  命令列入口，接收輸入影像與輸出路徑，執行 pipeline 並列出摘要。

- `tests/self_check.py`：
  自我驗證腳本。會自動建立 512x512 彩色測試圖，執行完整流程並斷言關鍵條件（區塊數、資料正確性、輸出檔存在）。

## 安裝與執行

1. 安裝套件

```bash
cd rdh_project
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

2. 執行主流程

```bash
PYTHONPATH=src python3 src/main.py --input /你的/512x512圖檔 --output-dir outputs --algorithm histogram_shifting --hs-shift-level pm1

# 第二演算法（提高 payload BPP）
PYTHONPATH=src python3 src/main.py --input /你的/512x512圖檔 --output-dir outputs --algorithm histogram_shifting_fixed_length --hs-shift-level pm1 --hs-fixed-bits 13

# 第三演算法（多峰直方圖位移）
PYTHONPATH=src python3 src/main.py --input /你的/512x512圖檔 --output-dir outputs --algorithm histogram_shifting_multi_peak --hs-shift-level pm1 --hs-fixed-bits 16 --hs-multi-peaks 8

# 第四演算法（多層 + 多峰，目標 >2.5 BPP）
PYTHONPATH=src python3 src/main.py --input /你的/512x512圖檔 --output-dir outputs --algorithm histogram_shifting_multi_layer --hs-shift-level pm1 --hs-fixed-bits 168 --hs-multi-peaks 16 --hs-layers 8
```

3. 執行自我驗證

```bash
cd rdh_project
PYTHONPATH=src python3 tests/self_check.py
```

4. 啟動簡易 UI（可選圖、點擊開始轉換，訊息在視窗內顯示）

```bash
cd rdh_project
PYTHONPATH=src python3 src/gui.py
```

## 重要說明

目前已實作直方圖位移法，並且在自我驗證中確認：
- Step 6 的 stego 與原圖 PSNR 會下降（代表像素確實被修改）。
- Step 8 可正確取回嵌入資料。
- Step 10 restored 與原圖 PSNR 為 `inf`（代表成功還原）。
