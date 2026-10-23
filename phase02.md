# Phase 2 — 差分驗證、RV32M 與效能

> Parent roadmap: `ROADMAP.md`
> 標準答案：Spike 的逐指令紀錄、`riscv-tests` 的 rv32um、x86 上跑同一份 C 的輸出

## 目的

一致性測試只告訴你「有沒有過」。這個 Phase 建立 C-Model 工作最常用的工具：**和參考模型逐步比對、自動找出第一個分歧點**；再加上乘除法擴充、跑真正的 C 程式，最後量效能、在有基線的前提下優化。

```text
同一個 ELF
   ├──▶ bitexact --trace ──▶ 每條指令：PC、指令字、寫了哪個暫存器、值
   └──▶ spike --log-commits ──▶ 同樣的資訊（格式不同）
                     │
              scripts/diff_trace.py：正規化、逐行比對
                     │
              第一個分歧點：第 N 條指令、PC、哪個暫存器、兩邊的值
```

## 完成標準

- 任一個程式都能和 Spike 逐指令比對，分歧時一行指出位置
- rv32um 全過；C 程式在模擬器上的輸出與 x86 上相同
- 有效能基線（每秒指令數），優化前後可比

---

<!-- STEP_PLAN_START -->
## Step 規劃

### Step 7 — 與 Spike 逐指令比對

**目標**

讓本模擬器輸出逐指令的紀錄，和 Spike 的紀錄自動比對，指出第一個分歧點。

**規模目標**

```text
預估新增 code：約 120 lines（模擬器的 trace 輸出；比對工具是 Python，不計入）
```

**實作範圍**

- `--trace` 選項：每條執行完的指令印一行（PC、指令字、寫入的暫存器與值、記憶體寫入）
- `scripts/diff_trace.py`：讀兩邊的紀錄、正規化、比對
- `scripts/diff_spike.sh <elf>`：兩邊各跑一次、比對、結束碼 0／1／2

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- Spike 的 `--log-commits` 格式（Step 1 查過），以及它在 RV32 下要的參數（`--isa=rv32im`、記憶體位址）
- Spike 開機時會先跑一小段自己的啟動程式（boot ROM）；比對要從兩邊都進到 ELF 的入口點之後才開始
- 正規化：兩邊印的格式不同（十六進位寬度、`x0` 的寫入印不印），寫成純函式、有測試
- 分歧的定義：PC 不同、寫入的暫存器或值不同、記憶體寫入不同；`mcycle` 這類和實作有關的 CSR 讀值要排除，並寫明為什麼

先寫測試：

```text
tests/test_diff_trace.py（不需要 Spike，用手寫的兩份紀錄）:
  - 完全相同 → 結束碼 0
  - 第 57 條的 x5 不同 → 結束碼 1，訊息含「第 57 條」、PC、x5、兩邊的值
  - 一邊提早結束 → 指出在哪裡結束
  - 讀 mcycle 的差異被排除
```

**先紅**：在模擬器裡故意改壞一條指令（例如 `SRA` 寫成邏輯右移），用 `rv32ui-p-sra` 跑差分比對，確認它指出第一條執行到 `SRA` 且結果不同的位置。這個「突變」不入庫，輸出記進 baseline。

### 預期檔案

```text
src/trace.cpp  include/bitexact/trace.h
scripts/diff_trace.py  scripts/diff_spike.sh
tests/test_diff_trace.py
```

### 驗證

- 全部 rv32ui-p、rv32mi-p 和 Spike 逐指令比對都相同（`regress.sh` 加入這一項）
- 突變實驗：指出的分歧點正確

### 這一步不做

- 不和 QEMU 逐指令比對（QEMU 沒有同等的逐指令紀錄；需要時只比最終輸出）
- 不做 RV32M（Step 8）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
比對工具的測試通過
  ↓
所有一致性測試與 Spike 逐指令相同；突變被正確定位
```

### Step 8 — RV32M（乘除法）

**目標**

實作乘除法擴充，包含規格定義的特殊情形。

**規模目標**

```text
預估新增 code：約 100 lines
```

**實作範圍**

- `MUL`、`MULH`、`MULHSU`、`MULHU`、`DIV`、`DIVU`、`REM`、`REMU`
- `misa` 加上 M

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂（規格的表，逐項寫成測試）：

- **除以 0 不會例外**：`DIV` 得 -1、`DIVU` 得 0xFFFFFFFF、`REM`／`REMU` 得被除數
- **有號溢位**：`-2^31 / -1` 得 `-2^31`、餘數 0。在 C++ 裡直接寫 `INT32_MIN / -1` 是未定義行為，sanitizer 會抓到
- `MULH` 系列要 64 位元的中間結果；`MULHSU` 是一個有號乘一個無號

先寫測試：

```text
test_muldiv:
  - 規格表裡的除以 0、溢位，每一格一個測試
  - MULH／MULHSU／MULHU 對同一組輸入的三種結果
  - 隨機輸入：和用 __int128 或 int64_t 獨立算出的結果比對
```

### 預期檔案

```text
src/hart.cpp（新增）  test/test_muldiv.cpp
```

### 驗證

- 單元測試全過、紅過；rv32um-p 全部通過；與 Spike 逐指令相同

### 這一步不做

- 不做 A（原子）、F／D（浮點）、C（壓縮）擴充
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
規格表的特殊情形全部有測試、紅過
  ↓
rv32um 全過、與 Spike 相同
```

### Step 9 — 跑真正的 C 程式

**目標**

用交叉編譯器編一般的 C 程式（CRC-32、排序、矩陣乘法），在模擬器上跑，輸出和 x86 上跑同一份 C 的結果相同。

**規模目標**

```text
預估新增 code：約 200 lines（啟動碼、最小的輸出函式、console 裝置）
```

**實作範圍**

- `fw/crt0.S`、`fw/link.ld`：設堆疊、清 `.bss`、呼叫 `main`、把回傳值寫到 `tohost`
- `Console` 裝置：寫一個位元組就輸出一個字元
- `fw/lib/`：自寫的最小 `putchar`、`print_hex`（不依賴 C 函式庫；若 Step 1 查到有 RV32 的 newlib 也不用，保持可重建）
- `fw/apps/`：三支程式，同一份原始碼也能在 x86 上編

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

誰和誰互動：

```text
fw/apps/crc32.c ──riscv64-unknown-elf-gcc──▶ crc32.elf ──bitexact──▶ console 輸出
           └──────────────g++（x86）─────────▶ crc32（host） ──────▶ 標準答案
```

要實際弄懂：

- 啟動碼做的事和 `bare-pixels` Step 6 的 `Reset_Handler` 一樣：設堆疊、搬 `.data`、清 `.bss`、呼叫 `main`
- 編譯器在 `-O2` 下會用到哪些指令（看 `objdump`），確認都在 RV32IM 之內；`-march` 不含 `c`，才不會產生壓縮指令
- 編譯器可能自動呼叫 `memcpy`／`memset`：沒有 C 函式庫時要自己提供

先寫測試：

```text
tests/test_fw_apps.sh:
  - 每支程式：模擬器的輸出 == x86 版的輸出，否則結束碼 1 並印出 diff
  - 程式回傳非 0 → 結束碼 1
```

### 預期檔案

```text
fw/crt0.S  fw/link.ld  fw/lib/*.c  fw/apps/{crc32,sort,matmul}.c
src/console.cpp  include/bitexact/console.h
scripts/run_fw_apps.sh  tests/test_fw_apps.sh
```

### 驗證

- 三支程式在模擬器與 x86 的輸出相同；在 Spike 上也逐指令相同
- 編譯參數（`-O0`、`-O2`）兩種都跑

### 這一步不做

- 不做系統呼叫模擬（`open`、`read` 等）
- 不跑作業系統
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
三支 C 程式：模擬器 == x86 == Spike
```

### Step 10 — 效能基線與優化

**目標**

量出模擬器每秒執行多少指令，找出熱點，在測試全過的前提下優化，並量化改善。

**規模目標**

```text
預估新增 code：約 150 lines
```

**實作範圍**

- `--stats`：執行的指令數、時間、每秒指令數（MIPS）
- `scripts/bench.sh`：固定的程式、固定的次數、取中位數
- 一項優化（例如預先解碼快取：同一個位址的指令只解碼一次；或用跳躍表取代大 `switch`），由量測決定

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- 先量再改：用 `perf` 找熱點，不憑感覺
- 量測的雜訊：同一個程式跑多次的差異，作為「有改善」的門檻
- 預先解碼快取要處理「程式改寫自己」：寫入被快取過的位址時要作廢（`FENCE.I` 的語意）

先寫測試：

```text
test_decode_cache:
  - 寫入已快取的指令位址之後，執行的是新的指令（自我修改程式）
  - 快取開與關，同一個程式的逐指令紀錄完全相同
```

### 預期檔案

```text
src/stats.cpp  scripts/bench.sh
（優化的檔案依量測結果決定）
```

### 驗證

- 優化前的基線（MIPS、中位數、變異）先記進 baseline，再動手
- 優化後：所有回歸照樣全過、與 Spike 逐指令相同；MIPS 的改善大於量測雜訊
- 熱點分析的結果（`perf` 的前幾名）記進 baseline

### 這一步不做

- 不做 JIT（動態編譯成 x86 指令）
- 不做多執行緒
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
基線先量
  ↓
優化後回歸全過、逐指令相同
  ↓
改善大於雜訊
```
