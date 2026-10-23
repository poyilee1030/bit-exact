# Phase 3 — NPU 功能模型（int8，逐位元）

> Parent roadmap: `ROADMAP.md`
> 規格：一份公開的 int8 整數量化規格（Step 11 決定，例如 TensorFlow Lite 的 int8 量化規格）；標準答案：電腦端用 NumPy 依同一份規格**獨立**寫的參考實作

## 目的

NPU 的 C-Model 第一層是**功能模型**：不管硬體怎麼排程，只保證「每一層的輸出和規格定義的整數運算逐位元相同」。這一層是後面架構模型（Phase 4）與韌體（Phase 5）的標準答案。

```text
                  量化規格（文件）
                 ╱                ╲
     C++ 功能模型（本 Phase）    NumPy 參考實作（scripts/golden/，獨立寫）
                 ╲                ╱
            同一組輸入、權重 → 輸出逐位元比對
                        │
           分歧時：第幾層、哪個座標、兩邊的值、累加器的中間值
```

## 必須理解

- **仿射量化**：實數 `r = scale × (q − zero_point)`；int8 的範圍與飽和
- **卷積的整數運算**：int8 × int8 累加成 int32，加 int32 的 bias
- **再量化（requantization）**：int32 累加器乘一個浮點的比例，轉回 int8。硬體不用浮點，而是「定點乘數 + 右移 + 捨入 + 加 zero point + 飽和」。**捨入方式（四捨五入、遇到 .5 怎麼辦）就是逐位元正確與否的關鍵**
- 每一個通道各自的比例（per-channel）與整個張量一個比例（per-tensor）
- 資料排列（NHWC、NCHW）、padding、stride

## 完成標準

- 一個小 CNN 在 C++ 功能模型上推論，每一層都和 NumPy 參考實作逐位元相同
- 量化的捨入與飽和邊界全部有測試

---

<!-- STEP_PLAN_START -->
## Step 規劃

### Step 11 — 張量與量化

**目標**

定義 int8 張量與量化參數，實作再量化；選定並寫下這個 repo 採用的量化規格。

**規模目標**

```text
預估新增 code：約 200 lines
```

**實作範圍**

- `docs/quant-spec.md`：採用哪一份公開規格、哪一版、哪幾條公式，與本 repo 的取捨（例如只做對稱的權重量化）
- `Tensor<T>`（形狀、排列、資料）、`QuantParams`
- `requantize(int32 acc, multiplier, shift) → int8`：定點乘法、捨入、飽和
- 浮點比例 → 定點乘數與右移量的換算
- `scripts/golden/quant.py`：NumPy 依同一份規格**獨立**寫

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- 定點乘法：`acc × multiplier` 需要 64 位元的中間結果，取高位、再右移，**每一步的捨入照規格**
- 「兩次捨入」與「一次捨入」結果可能差 1：規格寫哪一種就是哪一種
- 飽和：結果超出 [-128, 127] 夾住

先寫測試（從規格長，不從實作長）：

```text
test_requantize:
  - 規格文件裡的例子（若有）逐一照抄成測試
  - 捨入的邊界：剛好在 .5、-.5 的值
  - 飽和：最大正、最大負的累加器
  - multiplier、shift 的極端值
  - 隨機 10 萬組：和 NumPy 參考實作產生的向量檔比對
tests/test_golden_quant.py:
  - NumPy 參考實作本身對規格例子的測試（標準答案也要被測）
```

### 預期檔案

```text
docs/quant-spec.md
include/bitexact/npu/{tensor,quant}.h  src/npu/quant.cpp
test/test_requantize.cpp
scripts/golden/quant.py  tests/test_golden_quant.py
scripts/gen_quant_vectors.py  test/vectors/requant.bin   # 資料檔，另外標示大小
```

### 驗證

- C++ 與 NumPy 在 10 萬組隨機輸入上逐位元相同；邊界測試都紅過
- 突變實驗：把捨入改成截斷，向量比對指出第一個不同的輸入（記進 baseline）

### 這一步不做

- 不做卷積（Step 12）
- 不做浮點推論、不做 int16／int4
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
量化規格寫下來
  ↓
C++ 與獨立的 NumPy 實作逐位元相同
  ↓
捨入、飽和邊界都有測試、都紅過
```

### Step 12 — 卷積與 depthwise 卷積

**目標**

逐位元正確的 int8 卷積（一般與 depthwise），支援 stride 與 padding。

**規模目標**

```text
預估新增 code：約 250 lines
```

**實作範圍**

- `conv2d(input, weights, bias, params)`：NHWC、per-channel 權重量化、stride、padding（same／valid）
- `depthwise_conv2d`
- NumPy 參考實作；向量產生器；比對工具指出第一個分歧的座標

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- padding 的值不是 0，而是**輸入的 zero point**（量化後的 0）：這是常見的錯
- 累加器不會溢位的條件（int32 夠不夠）：算出最壞情況，寫進文件
- 比對失敗時印出：哪個輸出座標 (n, y, x, c)、兩邊的 int8 值、**累加器的 int32 中間值**（定位是累加錯還是再量化錯）

先寫測試：

```text
test_conv2d:
  - 手算的小例子（3×3 輸入、1 個 3×3 濾波器）
  - padding 用 zero point：輸入 zero point ≠ 0 時，邊緣輸出和中間不同
  - stride 2、輸出大小的公式
  - 隨機形狀與參數 1,000 組：和 NumPy 的向量比對
```

### 預期檔案

```text
include/bitexact/npu/conv.h  src/npu/conv.cpp
test/test_conv2d.cpp  test/test_depthwise.cpp
scripts/golden/conv.py  scripts/gen_conv_vectors.py  scripts/compare_tensors.py
```

### 驗證

- 隨機 1,000 組逐位元相同；手算例子正確；padding zero point 的測試紅過
- 突變實驗：padding 用 0 而不是 zero point，比對指出邊緣座標

### 這一步不做

- 不做 dilation、group convolution（depthwise 以外）
- 不做任何效能優化（功能模型以清楚為先）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
卷積與 depthwise 逐位元相同（隨機 1,000 組）
  ↓
分歧時指出座標與累加器的值
```

### Step 13 — 其他層與一個小 CNN 端到端

**目標**

補上池化、ReLU、全連接，讀進一個已量化的小 CNN，整個網路逐層和 NumPy 相同，並確認推論結果正確。

**規模目標**

```text
預估新增 code：約 250 lines
```

**實作範圍**

- `max_pool`、`avg_pool`（捨入照規格）、`relu`（量化域的夾值）、`fully_connected`
- 模型格式：自訂的簡單二進位格式（層的種類、形狀、量化參數、權重），由電腦端腳本產生
- 一個小 CNN（例如手寫數字辨識，兩三層卷積），在電腦端訓練並量化一次，權重入庫
- `bitexact npu-run <model> <input>`：輸出每一層的結果

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- `avg_pool` 的除法捨入
- 逐層比對：哪一層開始不同，就停在那一層報告，不要把錯誤傳到最後才發現
- 準確率的判準：量化後的模型在測試集上的準確率，C++ 與 NumPy 完全相同（因為逐位元相同），且和浮點模型的差距記進 baseline

先寫測試：

```text
test_layers:
  - 每種層的手算例子與邊界
test_model_format:
  - 損壞的檔案（長度不對、未知的層）→ 拒收並說明
tests/test_e2e_cnn.sh:
  - 100 張測試影像：每一層 C++ == NumPy；分類結果相同
```

### 預期檔案

```text
include/bitexact/npu/{layers,model}.h  src/npu/{layers,model}.cpp
test/test_layers.cpp  test/test_model_format.cpp
scripts/train_tiny_cnn.py  scripts/export_model.py
models/tiny_cnn.bin  models/test_inputs.bin   # 資料檔，另外標示大小與產生方式
tests/test_e2e_cnn.sh
```

### 驗證

- 100 張影像、每一層逐位元相同
- 量化模型的準確率與浮點模型的差距記進 baseline
- 權重檔可由腳本重新產生（固定亂數種子）

### 這一步不做

- 不讀現成的模型格式（TFLite、ONNX）：格式解析不是這個專案的重點
- 不做週期模型（Phase 4）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
每種層逐位元正確
  ↓
小 CNN 逐層相同、分類結果相同
```
